# SPL Token-2022 Transfer Fee — Rounding Asymmetry & Integration Pitfalls

> Reproduction study of a published SDK defect and its downstream impact — not an original finding.

**Target:** `solana-program-library` / `spl-token-2022` (and `spl-token-2022-interface`), `transfer_fee` extension
**Reviewer:** OnChain Vanguard
**Date:** 2026-10-09
**Severity:** Medium (SDK-level rounding asymmetry) — the on-chain transfer path itself is sound

---

## 1. Summary

The Token-2022 `transfer_fee` extension is the single most consequential extension for downstream protocols: every transfer of a fee-bearing mint withholds a portion of the amount in flight, so any protocol that assumes "sent == received" is wrong. I reviewed the reference implementation end to end — the fee math (`calculate_fee`), the inverse math (`calculate_pre_fee_amount` / `calculate_inverse_fee`), and the on-chain transfer processor.

Conclusion: **the on-chain transfer path is sound** — it withholds the fee correctly using ceiling division and can never under-charge. The one real defect sits in the SDK's *inverse* fee function: `calculate_inverse_fee` is not a strict inverse of `calculate_fee`, and the two disagree by one base unit on a wide class of inputs. This is a genuine, reproducible logic asymmetry that has already bitten real integrations (GHSA-725g-w329-g7qr is the same fee-misaccounting class at High severity).

---

## 2. Scope & Method

- Source: `spl-token-2022` / `spl-token-2022-interface`, `extension/transfer_fee/mod.rs` (current `spl-token-2022` and `spl-token-2022-interface` 3.1.2), and the byte-exact mirror in `solana-clmm-raydium` 0.3.0.
- I reproduced the fee arithmetic independently in a small BigInt harness (see §6) rather than trusting any single secondary write-up.

---

## 3. Findings

### 3.1 [Medium] `calculate_fee` and `calculate_inverse_fee` are not inverses — one-unit rounding asymmetry

#### The code

`calculate_fee` computes the withheld fee with **ceiling** division (this is deliberate and correct — it guarantees a non-zero fee for any non-zero amount at any positive rate):

```rust
const ONE_IN_BASIS_POINTS: u128 = 10_000;

fn ceil_div(numerator: u128, denominator: u128) -> Option<u128> {
    numerator.checked_add(denominator)?.checked_sub(1)?.checked_div(denominator)
}

pub fn calculate_fee(&self, pre_fee_amount: u64) -> Option<u64> {
    let transfer_fee_basis_points = u16::from(self.transfer_fee_basis_points) as u128;
    if transfer_fee_basis_points == 0 || pre_fee_amount == 0 {
        Some(0)
    } else {
        let numerator = (pre_fee_amount as u128).checked_mul(transfer_fee_basis_points)?;
        let raw_fee = Self::ceil_div(numerator, ONE_IN_BASIS_POINTS)?.try_into().ok()?;
        Some(cmp::min(raw_fee, u64::from(self.maximum_fee)))
    }
}
```

The inverse (`calculate_pre_fee_amount`, exposed as `calculate_inverse_fee` in the TS SDK) also rounds **up** when reconstructing the gross amount from a net amount:

```rust
// calculate_pre_fee_amount / reverse_apply_transfer_fee (byte-exact mirror)
pub fn reverse_apply_transfer_fee(fee: &TransferFee, post_fee_amount: u64) -> Result<u64> {
    let bps = fee.transfer_fee_basis_points as u128;
    let max_fee = fee.maximum_fee;
    match (bps, post_fee_amount) {
        (0, _) => Ok(post_fee_amount),
        (_, 0) => Ok(0),
        (b, _) if b == ONE_IN_BASIS_POINTS => post_fee_amount.checked_add(max_fee),
        _ => {
            let numerator = (post_fee_amount as u128).checked_mul(ONE_IN_BASIS_POINTS)?;
            let raw_pre_fee = ceil_div(numerator, ONE_IN_BASIS_POINTS - bps)?;
            let fee_part = raw_pre_fee.checked_sub(post_fee_amount)?;
            if fee_part > max_fee { post_fee_amount.checked_add(max_fee) } else { raw_pre_fee }
        }
    }
}
```

#### The defect

Both functions round **up**, so the round-trip is not exact. In general:

```
calculate_fee(amount_in)  >=  calculate_inverse_fee(amount_in - calculate_fee(amount_in))
```

with equality failing by exactly one base unit. A reproducible instance (Offside Labs' published example, which I re-derived independently — see §6):

| Field | Value |
|-------|-------|
| `amount_in` | 5238206430131997821 |
| `transfer_fee_basis_points` | 1000 |
| `maximum_fee` | 5704976840628159154 |
| `calculate_fee(amount_in)` | **523820643013199783** |
| `calculate_inverse_fee(post_fee)` | **523820643013199782** |
| delta | **1** |

#### Why it matters

`calculate_fee` and `calculate_inverse_fee` are the two primitives any fee-aware integrator uses — the forward one to predict what a transfer will withhold, the inverse one to size an exact-out swap or bridge leg so the recipient lands on a precise net amount. When they disagree:

- An exact-out router that sizes input via the inverse function but reconciles via the forward function will observe a one-unit mismatch, and if that mismatch is treated as an error it becomes a DoS/accounting break; if it is absorbed, it becomes a per-transaction drain that accumulates with volume.
- The `solana-clmm-raydium` crate's own mirror documents this: *"rounding-up of the fee means `calculate_fee(reverse_apply_transfer_fee(x))` can be one unit less than `calculate_fee(y)` where `y - calculate_fee(y) == x`."*
- The same fee-misaccounting class has already produced a real High-severity advisory (GHSA-725g-w329-g7qr): `kora-lib`'s `verify_token_payment()` credited the full raw amount and never subtracted the transfer fee, causing systematic loss. That bug is downstream mis-use of this exact arithmetic, and it demonstrates the blast radius of getting the fee math wrong.

The fix is not to change the on-chain program. It is to make the SDK expose a single canonical function and document the asymmetry, or to provide a strict inverse for integrators that need exact reconciliation (Offside Labs publishes the one-line change: replace the inverse's `raw_pre_fee_amount = ceil_div(numerator, denominator)` with a formulation that satisfies the reverse inequality).

### 3.2 [Info] Two-epoch activation delay and non-real-time `withheld_amount`

Two behaviors of the extension are correct by design but frequently mis-modeled:

1. **Fee-config updates are delayed two epochs (~4 days).** When `set_transfer_fee` updates the config, the *newer* fee does not apply until the `epoch` boundary recorded in the config. A protocol that assumes the new fee is live immediately will over- or under-charge during the transition window.
2. **`TransferFeeConfig.withheld_amount` is not real-time.** It lives on the mint account and is only brought up to date by `harvest_withheld_tokens_to_mint`. Between harvests it under-reports the actual collected fees. Any accounting that reads `withheld_amount` as "fees collected so far" will be wrong.

Neither is exploitable against Token-2022 itself; both are integration traps documented here for completeness.

---

## 4. Confirmed-sound areas

I specifically checked the attack surfaces a reviewer would worry about and found them correct:

- **Fee can never under-charge.** `calculate_fee` uses ceiling division, so any positive amount at any positive rate yields `fee >= 1` (before the `maximum_fee` cap). There is no truncation-to-zero path.
- **`maximum_fee` is enforced as a hard cap** via `cmp::min(raw_fee, maximum_fee)`.
- **Zero-rate and zero-amount short-circuits** (`bps == 0 || amount == 0 → 0`) are correct.
- **Overflow is guarded** with `checked_mul` / `checked_add` / `checked_sub` / `checked_div` throughout.
- **`transfer_checked_with_fee`** lets a caller pin the exact expected fee and reverts on mismatch, which is the correct primitive for any integrator that must not be surprised by the fee.

---

## 5. Severity Assessment

- **On-chain program:** no finding. The transfer processor withholds correctly and cannot be manipulated to under-charge.
- **SDK arithmetic (Finding 3.1):** **Medium.** It is not directly exploitable against Token-2022, but it is a real logic asymmetry in the canonical fee helpers that produces a reproducible one-unit error and has a documented High-severity downstream incident in the same class. It does not reach High on its own because the asymmetry requires an integrator to mix the two functions; a correct integration using `transfer_checked_with_fee` is unaffected.
- **Finding 3.2:** Informational.

---

## 6. Proof of Concept

The rounding asymmetry is pure arithmetic and reproduces deterministically. The harness below re-derives the Offside Labs example from first principles; running it prints the one-unit delta.

```javascript
// Node >= 18 (BigInt). Reproduces the Token-2022 fee round-trip asymmetry.
const ONE_IN_BP = 10000n;
const ceilDiv = (n, d) => (n + d - 1n) / d;

function calculateFee(bps, maxFee, preFee) {
  const b = BigInt(bps);
  if (b === 0n || preFee === 0n) return 0n;
  const raw = ceilDiv(preFee * b, ONE_IN_BP);
  return raw < maxFee ? raw : maxFee;
}

function inversePreFee(bps, maxFee, postFee) {
  const b = BigInt(bps);
  if (b === 0n) return postFee;
  if (postFee === 0n) return 0n;
  if (b === ONE_IN_BP) return postFee + maxFee;
  const numerator = postFee * ONE_IN_BP;
  const denom = ONE_IN_BP - b;
  const raw = ceilDiv(numerator, denom);
  const feePart = raw - postFee;
  return feePart > maxFee ? postFee + maxFee : raw;
}

const amountIn = 5238206430131997821n;
const bps      = 1000n;
const maxFee   = 5704976840628159154n;

const fee        = calculateFee(bps, maxFee, amountIn);
const postFee    = amountIn - fee;
const invFee     = inversePreFee(bps, maxFee, postFee) - postFee;

console.log('calculate_fee            =', fee.toString());       // 523820643013199783
console.log('calculate_inverse_fee    =', invFee.toString());      // 523820643013199782
console.log('delta                    =', (fee - invFee).toString()); // 1
```

Output (independently reproduced):

```
calculate_fee            = 523820643013199783
calculate_inverse_fee    = 523820643013199782
delta                    = 1
```

---

## 7. Recommended Fix

1. **SDK:** ship a single canonical inverse and document that `calculate_fee` and the inverse differ by at most one base unit; do not leave the two functions as interchangeable. For integrators needing strict reconciliation, adopt Offside Labs' published formulation so the chosen inequality holds exactly.
2. **Integrators:** never reconcile fee-bearing token flows with `calculate_fee` on one side and `calculate_inverse_fee` on the other. Prefer `transfer_checked_with_fee` with the exact expected fee, or measure balance deltas before/after.
3. **Integrators:** treat `withheld_amount` as a lazily-synced counter and account for the two-epoch fee-config activation delay.

---

## 8. References

- `spl-token-2022` `extension/transfer_fee/mod.rs` — `calculate_fee`, `ceil_div`, `MAX_FEE_BASIS_POINTS`: https://docs.rs/spl-token-2022/latest/src/spl_token_2022/extension/transfer_fee/mod.rs.html
- `solana-clmm-raydium` 0.3.0 `transfer_fee.rs` — byte-exact mirror with the explicit non-exact-round-trip note: https://docs.rs/solana-clmm-raydium/0.3.0/src/solana_clmm_raydium/transfer_fee.rs.html
- Offside Labs — *Token-2022 Security Best Practices, Part 2: Extensions* (published numeric example + inequality + fix): https://blog.offside.io/
- GHSA-725g-w329-g7qr — `kora-lib`: Token-2022 transfer fee not deducted during payment verification (High): https://github.com/advisories/GHSA-725g-w329-g7qr
- Zooz — *Token-2022 Is Not SPL Token* (audit checklist; notes `calculate_fee` vs `calculate_inverse_fee` one-unit difference): https://engineering.zooz.com/
