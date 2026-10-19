# Jupiter Lend (Fluid stack) — Vaults Tick-Bitmap Liquidation Engine

**Target:** Jupiter Lend Vaults program (Fluid protocol port, ~2.19 MB deployed, September 2026 deployment)
**Source basis:** `Instadapp/fluid-solana-programs` main snapshot (February 2026); deployed-vs-source instruction diff performed first — one divergence (`PauseToken`, present in binary, absent in snapshot) was isolated and flagged before analysis.
**Result:** No exploitable condition. Four observations below reporting threshold, retained on file.

## Why this target

The Vaults program ports Fluid's tick-bitmap liquidation design from the EVM
to Solana. Ports fail at the seams: the EVM version assumes global mutable
storage and implicit enumeration, while the Solana version must pass every
account explicitly and authenticate each one. The review focused on the four
questions a port must answer differently.

## 1. remaining_accounts authentication

The liquidation path walks the tick bitmap via caller-supplied
`remaining_accounts`. Forgery here would mean liquidating against attacker-
controlled state. The implementation applies three checks per supplied
account: owner equals the Vaults program, Anchor discriminator matches the
expected account type, and `vault_id` matches the vault being liquidated —
which prevents cross-vault account mixing. Oracle and Liquidity program IDs
are hard-pinned against `vault_config`, so no component of the read set can be
substituted.

## 2. Bitmap traversal bounds

Tick-index-to-bitmap-array translation is range-checked, and the team pins the
edges with its own unit tests (MIN_TICK maps to array 0, MAX_TICK to the top
array). Bit clearing on liquidation follows the same direction as traversal,
so a partially-executed batch cannot skip or revisit ticks.

## 3. Batch-liquidation scaling math

Compounding debt factors across a batch use u256 intermediate arithmetic with
downward rounding in the protocol's favor. The denominator is mathematically
capped because liquidation threshold plus penalty is bounded below 100%
(LT + penalty ≤ 99.7%), so no division blow-up. Final per-position settlement
rounds debt ceil / collateral floor — both directions against the liquidator's
counterparty, i.e. in the protocol's favor.

## 4. Dust-position DoS against the bitmap

A classic griefing vector against tick systems: dust positions that bloat the
bitmap and stall batch liquidation. Three independent dust gates exist
(minimum tick debt, minimum position debt, minimum branch residual), and an
`absorb` path collects bad debt separately. Manufacturing dust costs the
attacker full value at each gate.

## Design notes worth stealing

- **Dual price tracks.** Opening positions checks collateral factor against
  the "operate" price; liquidation checks the liquidation threshold against a
  separate "liquidate" price. Single-point oracle manipulation has to move
  both.
- **Liquidator protection parameter.** `col_per_unit_debt` acts as a slippage
  bound so permissionless liquidators are not griefed by intratransaction
  price movement.
- **Reverting resolver.** An `ADDRESS_DEAD` resolution path deliberately
  reverts while returning the max-liquidatable amount — a clean probing
  interface for off-chain liquidators.

## Observations (not submitted)

- Extreme-batch compute-unit exhaustion is theoretical only (bounded by dust gates).
- A negative `magnifier` in the exchange-price loader is a privileged-config hygiene item.
- The dual-track oracle's two price feeds share one source set; divergence handling is by construction, not by check.
- `div_big_number` exhibits wei-level rounding drift — bounded and directionally safe.

## Verification notes

- Deployed binary provenance: programdata pulled over RPC, ELF extracted and
  instruction-string scanned; all source instructions present in the deployed
  build (plus `PauseToken`, noted above).
- Local fork testing of cross-program timing is queued on the `svm-fuzz-lab`
  harness (see repository README).
