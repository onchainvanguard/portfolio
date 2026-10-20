# Jupiter Lend — Deployed Oracle Constants Drift from Published Source; Deep-Crash Liquidation Liveness Constraint

**Program scope:** Jupiter Lend `oracle` (`jupnw4B6Eqs7ft6rxpzYLJZYSnrpRgPcr589n5Kv4oc`), `vaults`, `liquidity`/`flashloan` — deployed mainnet ELFs only
**Environment:** LiteSVM local fork of live mainnet state; 808 archived mainnet accounts; zero mainnet writes at any point
**Method:** 76-check fuzz campaign across 7 phases (M5.6 + M6), binary-searched accept/revert boundaries on the deployed binaries
**Result:** 76/76 checks green, 0 invariant violations, **no fund-loss finding**. Two reportable items: one liveness constraint (Medium-leaning, user-facing), one deployed-vs-documented drift chain (Low/Informational).

---

## Part I — Oracle staleness: deployed windows exceed documentation on every path

**Severity:** Low / Informational (documentation divergence; exploitability measured and priced — see Appendix A/B)

### 1. Summary

The oracle program's published source documents two staleness budgets: `MAX_AGE_OPERATE = 600s` for user operations (open/borrow) and `MAX_AGE_LIQUIDATE = 7200s` for liquidations. Measured against the deployed ELF, the Chainlink path is materially wider on both:

| Path | Documented | Deployed (measured) | Flip point (slot-exact) |
|---|---|---|---|
| liquidate | 7200s (2h) | **11520s (3.2h)** (+60%) | feed_slot+28802 accepted / +28803 `PriceTooOld` |
| operate (borrow) | 600s (10min) | **~1057s (17.6min)** (+76%) | feed_slot+2642 accepted / +2643 `PriceTooOld` |

Staleness on the Chainlink path is slot-based: `time_elapsed = (current_slot - publish_slot) * AVG_SLOT_TIME(400ms) / 1000`, rejected past the cap. The liquidate boundary divides exactly: 28800 slots × 400ms = 11520s, and the constant `11520` (`0x2D00`) appears at six offsets in `oracle.bin`. The operate boundary lands at ~2643.5 slots ≈ 1057.4s; the constant `1057` appears at six offsets in the binary (±1 slot attribution margin from slot-basis/rounding semantics).

The timestamp-based paths (Pyth/Redstone) use yet another pair of constants — **660s / 7200s** — a precise 1.6× ratio against the slot path (660×1.6≈1057, 7200×1.6=11520). Same binary, three documented-vs-deployed divergences, all in the looser direction on the Chainlink path. This is version drift between the deployed build and the published source (public repo 2026-02, v0.1.2), not a one-off typo.

### 2. Measurement method

Harness: `vault_flash_oracle_fuzz.rs` (M6), vault#1 (wSOL/USDC), Chainlink SOL/USD feed fixture (`CH31Xns5z3M1cTAbKW34jcxPPciazARpijcHj9rxtemt`).

- **Liquidate path**: real liquidate ix against topmost tick 2972 at 8% crash; sweep SVM clock slot over the fixed feed; bisect to the exact accept/reject flip.
- **Operate path**: full real position lifecycle (init_position → init_tick → init_tick_id_liquidation → deposit 0.05 wSOL), then a 2 USDC borrow operate at each staleness point; bisect the flip.
- **Pyth/Redstone paths** (unused by any of vault configs #1–#24 — verified by PDA-derived census): synthetic Oracle PDA + synthetic feed accounts, direct permissionless calls to `get_exchange_rate_operate` / `get_exchange_rate_liquidate` on the deployed ELF, binary-searched boundaries (Appendices D/E).

The operate check lives at `oracle/src/helper.rs:318` (deployed line info) — same `PriceTooOld` gate as liquidate, different constant.

### 3. Impact

**Operate side.** A ~17.6-minute window (vs documented 10) during which new borrows execute against stale prices. Exploitability was priced end-to-end (Appendix A/B): the on-chain break-even crossing sits at a **22.7%** crash inside the window with a fully frozen Chainlink feed — an event with no historical precedent given Chainlink SOL/USD's ~0.5% deviation trigger. Not a priceable attack; the finding's value is the drift itself.

**Liquidate side.** Liquidations may execute on prices up to 3.2h old. If price rebounds while the feed lags low, currently-solvent positions get liquidated at the stale price. The seizure bonus is capped — measured 99 bps constant across cover sizes — so the excess over fair liquidation is bounded but real. This side carries the more material user-facing risk (see Part II for the compounding liveness constraint).

**Integrator mismatch.** Any liquidator bot or risk engine parameterized from the documented constants (600s / 7200s) misjudges both when positions become operable and when they become unliquidatable.

No solvency break observed anywhere: answer-boundary mutations (0, 1 lamport, u64::MAX, negative i128) all revert cleanly; stale-price liquidation economics remain capped at the configured penalty; flashloan borrow/payback introspection held on all 9 attack shapes; Pyth confidence factors (operate 50, liquidate 25) match source exactly with slot-exact boundaries.

### 4. Recommendation

- Publish the deployed constants (or redeploy to match the docs). If 11520s / 1057s / 660s are intentional, they belong in the public spec; if not, the drift itself warrants review.
- Liquidators and risk dashboards should parameterize off measured on-chain behavior (28800 / ~2643 slots on the Chainlink path), not the documentation.
- Consider tightening the operate window: at 400ms/slot, 600s = 1500 slots; the deployed ~2643-slot allowance nearly doubles the documented exposure.

---

## Part II — Deep-crash liquidation liveness: absorb path structurally unreachable in one tx

**Severity:** Medium-leaning (liveness, not solvency)

### 1. Summary

Jupiter Lend Vaults liquidate tick-by-tick: a `liquidate` call walks the vault's connected branch chain and pulls every tick carrying debt down to a target tick derived from the liquidated position's collateral ratio. We fuzzed the deep-crash path — cascading price shocks of 8–74% with progressive multi-tx liquidation — and found that **the bad-debt absorption path (`Branch status = 3`) is structurally unreachable in a single transaction** once the vault's `topmost` tick is underwater.

The constraint is the interaction of two hard limits:

1. The ~108-tick gap between `topmost` and the liquidation target tick, imposed by the 85% liquidation threshold on a 0.15%-per-tick ratio grid (`gap = ln(1/0.85) / ln(1.0015) ≈ 108.4`).
2. Solana's 64-account lock cap per transaction: measured assembly budget is 27 static + ≤10 branches + 26 ticks + 1 thd ≤ 64 locks, and a 26-tick window covers only ~3.9% of price range — the ~108-tick gap is roughly **4× the maximum window**.

When `topmost` is underwater, the walk's target tick always sits beyond what one transaction can assemble, so the call reverts atomically (`VaultTickNotFound`). Deep-crash liquidation therefore proceeds only as a sequence of progressive transactions. During that window, underwater debt sits unliquidatable in one shot — a liveness constraint, not a solvency bug: across 223 successful liquidation txs we observed **zero erroneous absorptions, zero invariant violations, and full revert atomicity** on every out-of-window attempt.

### 2. Fuzz results

Harness: `vault_absorb_fuzz.rs` (deployed `vaults` ELF, all 678 debt-bearing ticks 2051–2938 archived). Assertions per tx: liquidator never loses (A1); conservation of `total_borrow` (A2); `topmost` monotonicity (A3); revert atomicity (A4); `absorbed_raw` ≤ chain debt (A5).

| Run | Config | Result |
|---|---|---|
| Final round (seed 20261020) | STAIR 8% + 2%/tx, ≤12 tx/round, 20 rounds | **223 ok_tx, 0 guard, 5 walk-limit reverts, 0 violations, absorb_events = 0** |
| Deep stair | 12% + 3%/tx, 40 tx | 40/40 ok, topmost 2972 → 2157 (~74% cumulative crash), 0 violations |
| Single-shot 35% crash | FINAL_CRASH=35 | 20/20 ok |
| Single-shot 42% crash | FINAL_CRASH=42 | 20/20 atomic revert (`VaultTickNotFound` on tick 2713, out of window) |

Every walked tick ends `debt_after = 0` while `absorb_events` stays 0: each tick's debt is fully repaid by the external liquidator **before** its collateral ratio crosses 100% — the walk's built-in 0.85×-price liquidation line means absorb never arms. For absorb to trigger, a single transaction would need its target tick both below 100% collateralization *and* inside the assembly window — which the two limits above make mutually exclusive.

### 3. Why liveness, not solvency

- No erroneous absorption in 223 txs; conservation held with zero absorbed debt.
- No stuck state: progressive multi-tx liquidation walks `topmost` arbitrarily far (2972 → 2157 observed); each tx is independent and permissionless.
- Atomic failure mode: out-of-window attempts revert with state untouched (verified at 42% crash depth).

The risk is temporal: in a fast crash, the gap between "price crosses the liquidation line" and "liquidators fire enough sequential txs to walk the chain down" is where underwater debt waits — and Part I's 3.2h liquidation staleness window widens exactly this gap. On mainnet, v0 transactions + Address Lookup Tables raise the effective account ceiling beyond the 64-lock legacy cap our harness is bound by; ALT-equipped reach is the natural next measurement before final severity assignment.

### 4. Evidence pointers

- Tick account layout (40 bytes): `disc8 + vault_id u16@8 + tick i32@10 + positions u32@14 + raw_debt u64@18`
- Branch chain: `connected_branch_id @ offset 39`; vault#1 chain measured: `39→37→36→35→33→31→30→1→0`
- Vault state: `topmost i32@11 / cur_branch u32@15 / total_branch u32@19 / total_supply@23 / total_borrow@31`
- TickMath replicated off-chain in pure integers; `ratio(t) = 1.0015^t` verified against on-chain tick prices

---

## Appendix A — Operate-window exploitability, priced (fuzz phase O5)

Maximum borrow against a stale-high feed, five price scenarios, fresh fork each (5 wSOL collateral, on-chain binary search with program-guided tick assembly):

| scenario | max borrow | collateral value | effective borrow LT |
|---|---|---|---|
| fresh  | $406.25 | $525.27 | 77.3% |
| -5%    | $390.62 | $499.01 | 78.3% |
| -8%    | $375.00 | $483.25 | 77.6% |
| -12%   | $359.38 | $462.24 | 77.7% |
| -15%   | $343.75 | $446.48 | 77.0% |

Attacker profit against a stale price: `profit = V · (LT_eff − 1 + c)`, positive only when the crash exceeds **1 − LT_eff = 22.7%** inside the ~17.6-minute window with no Chainlink update in between. All measured scenarios are unprofitable. At max leverage, withdrawing 1 lamport of collateral reverts (`VaultPositionNotHealthy`, 6043) in every scenario; max borrow is monotonically non-increasing as price falls. No bypass found.

## Appendix B — Break-even crossing, confirmed on-chain

O5 sweep extended across the predicted crossing (fresh fork per scenario):

| market crash | attacker collateral cost | protocol lend (stale price) | net P/L |
|---|---|---|---|
| -20% | $420.22 | $406.25 | **-$13.97** |
| -22% | $409.71 | $406.25 | **-$3.46** |
| -23% | $406.03 | $406.25 | **+$0.22** |
| -25% | $393.95 | $406.25 | **+$12.30** |

The on-chain crossing sits in (22%, 23%], matching the closed-form prediction c* = 1 − LT_eff = 22.7% exactly.

MEV ceiling: per-position MEV = V·(c − 22.7%). Protocol-level ceiling capped by the shared USDC pool free balance ($41.56M measured): a -30% crash with a fully stalled oracle yields at most ~$3.9M, and requires market-buying ~$37.6M of wSOL mid-crash. The attack is a bet on a >22.7% crash inside 1057s with Chainlink's ~0.5% deviation trigger never firing — a tail event the attacker cannot force. Severity stays Low / Informational.

## Appendix C — Flashloan-funded liquidation combo (single-tx)

Question: can a liquidator operate with **zero capital** via `flashloan_borrow(B) → liquidate(cover=B) → resale → flashloan_payback(B)` in one transaction?

| Vector | Result |
|---|---|
| combo cover $5 / $20 / $50 | all OK, ~288k CU, signer ATA closes to exactly 0 |
| borrow→liquidate without payback | reverts (6009), zero state drift |
| cover = 2× flash borrow | reverts (insufficient funds), zero drift |
| double borrow wrapping liquidate | reverts (6013, introspection contract), zero drift |
| incentive amplification check | margin constant **99 bps** across cover sizes — no flash-funded amplification |

Economics: incentive priced at current oracle (99 bps realised = configured 1.00%); flashloan admin fee_bps = 0, so the combo needs no capital. Gross profit = cover × 99 bps ($0.495 at $50) — viable iff aggregator resale slippage < 99 bps. Intended permissionless-liquidator design, not a violation.

Structural note (fixture-verified): the flashloan USDC token vault **is** the same Liquidity USDC pool account vault#1 borrows from (`BmkUoKMFYBxNSzWXyUjyMJjMAaVz4d8ZnxwwmhDCUXFB`); liquidation repayment flows back into the same pool (post-combo pool delta exactly +cover, fee = 0). Assessment: **no finding**.

## Appendix D — Pyth confidence/age rails (deployed oracle ELF)

Mainnet reachability census (PDA-derived scan of vault configs #1–#24): **no vault uses a Pyth or Redstone source** — every oracle config is Chainlink or StakePool based. Rails verified anyway via synthetic-source direct calls (13/13 checks green):

| rail | source | deployed (measured) | verdict |
|---|---|---|---|
| operate confidence factor | 50 (reject conf > 2% of price) | 50 (boundary exactly price/50, +1 reverts 6012) | no drift |
| liquidate confidence factor | 25 (reject conf > 4% of price) | 25 (boundary exactly price/25) | no drift |
| operate max age (timestamp path) | 600s | **660s** | +10% drift |
| liquidate max age (timestamp path) | 7200s | 7200s | matches |
| verification < Full / exponent > 0 / price ≤ 0 | revert | all revert | enforced |

The Pyth timestamp path's 660s/7200s vs the Chainlink slot path's ~1057s/11520s is a precise 1.6× ratio inside one binary — the deployed build comes from a newer source revision than the public repo (2026-02 v0.1.2). If an admin ever configures a Pyth source, its operate window would actually be *tighter* (660s) than the Chainlink vaults' effective 1057s.

## Appendix E — Redstone source rails (deployed oracle ELF)

Same synthetic-source method, 10/10 checks green:

- age boundaries: operate **660s** / liquidate **7200s** exactly — shares the timestamp-path constants with Pyth
- u256 value with any high byte set reverts (`RedstonePriceOverflow`)
- `write_timestamp = None` reverts (`TimestampExpected`)
- freshness gate is `min(timestamp, write_timestamp)`: a stale leg dominates in both directions
- price = 0 rejected downstream (`RateZero`); fresh feed passes both paths

Oracle source matrix complete: Chainlink (live-fork phases), Pyth (Appendix D), Redstone (here), StakePool/JupLend (static review). No fund-risk findings; the drift chain is consistent (600 → 660 on the timestamp path, 1057/11520 effective on the Chainlink slot path, confidence factors 50/25 unchanged).

---

*Fuzz campaign M5.6 + M6, 2026-10-20: 76 checks, 0 violations, local fork only. Reproducible from the archived account set (808 JSON fixtures) + deployed ELFs.*
