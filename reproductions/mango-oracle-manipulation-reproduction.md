# Mango Markets Oracle Manipulation — Minimal On-Chain Reproduction

> Reproduction study of a historical exploit — not an original finding.

**Target incident:** Mango Markets, October 11 2022 · ~$114M drained (SEC/CFTC figure ~$110M–$116M)
**Reproduction:** `mango-oracle` — a minimal Solana program capturing the causal chain of the exploit
**Result:** 1 Finalized attack transaction with on-chain balance and log evidence

---

## 1. Executive Summary

On October 11, 2022, Avraham Eisenberg drained roughly $114M from Mango Markets, a Solana-based derivatives and lending venue, by manipulating the MNGO oracle price. This document reproduces the exploit's essential mechanism in a minimal Solana program and executes it against a local validator, producing a Finalized attack signature with on-chain evidence.

The exploit reduces to a single causal chain: **a lending program prices collateral off a low-liquidity spot market; an attacker pumps that market; the inflated price inflates the collateral value; the attacker borrows out the pool.**

The reproduction is deliberately minimal. It does not recreate Mango's full derivatives book, Serum order books, or MNGO tokenomics. It isolates the vulnerability that made the real attack possible — a manipulable price feed feeding an uncapped collateral valuation.

---

## 2. The Real Exploit (Factual Background)

Source: SEC complaint (Jan 2023), CFTC press release 8647-23, ESMA TRV Risk Article (Oct 2023).

- **Attacker:** Avraham Eisenberg, a 27-year-old U.S. citizen then residing in Puerto Rico.
- **Mechanics:** Eisenberg used two accounts on Mango Markets to take opposing MNGO/USDC perpetual futures positions — one selling, one buying. He then made a series of large purchases of the thinly traded MNGO spot token across three exchanges that served as inputs to Mango's oracle, pushing the MNGO price from roughly $0.03 to $0.91 — over a 13-fold spike within about 30 minutes.
- **Result:** The oracle-reported MNGO price, and therefore the value of Eisenberg's long MNGO perpetual position, spiked artificially. He borrowed against that inflated position and withdrew approximately $110M–$116M worth of USDC, BTC, SOL, mSOL, ETH and other assets, effectively draining the platform of most user deposits.
- **Aftermath:** Eisenberg returned roughly $67M under a governance proposal, retaining about $47M. The SEC, CFTC, and DOJ charged him in 2023. A jury convicted him in April 2024; the conviction was vacated in May 2025.

The key property worth studying: **the entire loss stemmed from a price feed that could be moved by ordinary market trades in an illiquid asset, combined with a collateral valuation that trusted that feed without any deviation guard.**

---

## 3. Root Cause

Three compounding defects, in order of consequence:

1. **Manipulable oracle.** The price of MNGO collateral was derived from low-liquidity spot markets. A single large buy — or, in Mango's case, coordinated wash-style futures plus spot pumps — could move the price materially. No TWAP, no multi-source median, no volume weighting.

2. **Collateral valued at spot.** The borrow ceiling was a direct function of `collateral_quantity × spot_price`. When the spot price spiked, the borrowing power spiked with it, despite no change in the attacker's actual capital.

3. **No deviation guard or LTV discount.** There was no cap on how far the oracle price could move between updates, no discounted loan-to-value for volatile collateral, and no circuit breaker. The attacker borrowed up to the full inflated valuation.

The reproduction maps each defect to a concrete line of program logic.

---

## 4. Minimal Reproduction

### 4.1 Program

`mango-oracle` = `FZAa2iyfP98jM8845rC729PhZmMc62bbDDXjPz598Vtm`

It contains a minimal constant-product AMM (MNGO/USDC) whose spot price acts as the oracle, and a collateral/borrow facility that prices MNGO collateral off that AMM.

Account layout:

| Account | Data layout |
|---|---|
| `pool` (16 B) | `[reserve_mngo: u64][reserve_usdc: u64]` — AMM reserves + borrow liquidity (lamports) |
| `collateral` (40 B) | `[owner: Pubkey][mngo_amount: u64]` — attacker's pledged MNGO |

Instructions:

| Tag | Instruction | Effect |
|---|---|---|
| `0` | `init` | Seed pool at 1,000,000 MNGO / 1,000,000 USDC (price $1.0); victim deposits 1,000,000 lamports of borrow liquidity |
| `1` | `buy_mngo` | Constant-product swap: attacker pays USDC in, receives MNGO, pledged to `collateral`; price rises |
| `2` | `borrow` | Borrow ceiling = `mngo_amount * reserve_usdc / reserve_mngo` (spot price), no LTV discount |

The vulnerable line is the borrow ceiling:

```rust
let collateral_value = mngo * u / m;   // spot price, trusted as-is
if amount > collateral_value { return Err(OverBorrow); }
```

`u` and `m` are the live AMM reserves. The attacker's `buy_mngo` has already moved them, so `collateral_value` reflects the pumped price.

### 4.2 Numbers

| Step | Reserve MNGO | Reserve USDC | Price | Attacker MNGO collateral |
|---|---|---|---|---|
| init | 1,000,000 | 1,000,000 | $1.00 | 0 |
| buy_mngo(900,000 USDC) | 526,315 | 1,900,000 | ~$3.61 | 473,685 |

Borrow ceiling after the pump:

```
collateral_value = 473,685 * 1,900,000 / 526,315 = 1,710,005
```

The attacker put 900,000 units of capital at risk, yet borrowed 1,710,005 — extracting 810,005 units of the victim's deposit. The pool's lendable balance dropped from 1,000,000 to 189,995.

---

## 5. Attack Walkthrough (On-Chain Evidence)

All signatures verified `finalized` via `getSignatureStatus` and `getTransaction` (double check, on-chain source of truth).

**Step 1 — init.** Victim seeds the pool at price $1.0 and deposits 1,000,000 lamports.

Signature: `37H68V7mbPALef6Dy8hxYyMViwwSSxFQYq7JZpRwU3P1tDx554t9ktLwXZLjgRNfSwX33LPR3LpmBUKjZUvhgb8q` (slot 137883, err=null)

```
Program log: mango-oracle: init pool reserve_mngo=1000000 reserve_usdc=1000000 price=1.0
```

**Step 2 — pump.** Attacker buys MNGO with 900,000 USDC, moving the price from $1.00 to ~$3.61 and pledging 473,685 MNGO.

Signature: `3ronJH1j99fWSjWeJ1GAZWfWY9ED8giPvgX31NtyXuPckaxBbPobT8oKdyRpLkRqogjytR6XA2PUaLg7Hwvenm4z` (slot 137947, err=null)

```
Program log: mango-oracle: buy_mngo usdc_in=900000 got_mngo=473685 new_reserves=526315/1900000 price~3.61x
```

**Step 3 — attack.** Attacker borrows 1,710,005 against the inflated collateral, draining the victim's deposit.

Signature: `4d1yVCrrMg8tYWWvmJCadTqQmz86JtLqABDNJ69uRcZfv2R4ZhveWxPVfwemG886qPgc1wehE84qGpAZHE3UDhZW` (slot 137979, err=null)

```
Program log: mango-oracle: borrow OK amount=1710005 collateral_value=1710005 (price=1900000/526315, mngo_collateral=473685)
```

**Balance proof:**

| Account | Address | Change |
|---|---|---|
| pool (lendable) | `9pPB1eSo8xgeAaX5i1Rgta4Z1NvgfFnp3Sc2e7uFAM64` | 1,000,000 → 189,995 |
| collateral | `J1wTSbWPELg6pNWzbFeuv73YiN4oi13qieFnPjpQ1NSv` | 473,685 MNGO |
| attacker | `GXmD7zVobTUsQ6AkK4rYpmeGANgSu64D2VVc7qQrSm3W` | +1,710,005 |

Net extraction: **1,710,005 − 900,000 = 810,005**, taken from the victim's 1,000,000 deposit.

---

## 6. Impact

The reproduction demonstrates the exact insolvency mechanism of the Mango exploit: the attacker's real capital (900,000) is far below what they borrow (1,710,005), and the difference is funded by other depositors. When the manipulated price reverts, the collateral is worth only ~526,315 (473,685 MNGO at the original ~$1.0 price), leaving the pool short by over 1.18M — the same "deficit when the price returned to pre-manipulation level" described in the SEC complaint.

In Mango's case the price reversion was ~30× (from $0.91 back toward $0.03), which is why the loss reached nine figures. The reproduction compresses the same dynamic into a 3.61× pump with a single swap.

---

## 7. Mitigations

1. **Time-weighted or multi-source price.** Use a TWAP over a window, or a median across multiple independent venues, instead of a single spot read. Mango's three-exchange feed was still manipulable because all three were thin MNGO books; the correct fix is to weigh by liquidity and require volume.

2. **Deviation / staleness circuit breakers.** Reject or bound any oracle update that moves more than X% within a window, and pause borrowing if the feed is stale.

3. **Collateral-specific LTV discount.** Volatile, low-liquidity assets should carry a low loan-to-value cap (and ideally be excluded from collateral entirely). A 90%+ effective LTV on a thin token is what turned a pump into a drain.

4. **Borrow caps and pool-level invariants.** Cap aggregate borrows per asset, and assert `collateral_value >= total_liabilities` as an invariant checked on every state transition.

5. **Do not trust a single spot source for settlement-critical valuation.** The core lesson: a price that an attacker can move with a normal trade must never, by itself, authorize the release of other users' funds.

---

## 8. Verification Note (No Fabrication)

- 1 real attack signature, `finalized` (slot 137979, err=null), verified via `getSignatureStatus` + `getTransaction`.
- Balance deltas read on-chain (pool 1,000,000 → 189,995).
- Three program logs capture the full chain: `init` (price 1.0) → `buy_mngo` (price 3.61) → `borrow` (collateral_value 1,710,005).
- Program source: `poc/mango_oracle_src/lib.rs`; attack script: `poc/mango_oracle_attack.mjs`.

**One-line summary:** Mango Markets was drained because a low-liquidity spot price — movable by ordinary trades — was trusted as an uncapped oracle for collateral valuation, letting the attacker borrow far beyond their real capital and empty the pool. The fix is time-weighted, deviation-guarded, liquidity-weighted pricing, plus conservative LTV caps.
