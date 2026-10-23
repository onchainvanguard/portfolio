# Jupiter Lend — Stateful Fuzzing of the Liquidity/Lending Core on Deployed Bytecode

**Scope:** Jupiter Lend `liquidity` (`jupeiUmn818Jg1ekPURTpr4mFo29p46vygyykFJ3wZC`) and `lending` (`jup3YeL8QhtSx1e253b2FDvsMNC87fDrgQZivbrndc9`) programs, USDC market, plus the SPL Token / ATA / Jupiter oracle / MPL metadata dependencies.
**Method:** LiteSVM local fork running the *deployed mainnet ELFs* against *real mainnet account state* (no mocks). Hand-built instruction encoders, PDA derivation verified against on-chain fields, clock warping for interest accrual.
**Result:** No solvency-breaking invariant violation found. All rounding observed favors the protocol. Details and evidence below.

## Harness milestones

| Stage | What ran | Outcome |
|---|---|---|
| M1 | Load deployed ELFs, smoke transactions | pass |
| M2.1 | `update_exchange_price` crank on real USDC reserve | 12,411 CU; supply/borrow exchange prices strictly increase after +3600s warp |
| M2.2 | deposit → redeem round-trip, warp+crank, re-deposit | pass, three invariants hold |
| M3 | 200-round single-user fuzz (seed `0x5eed0001`): random deposit/redeem/warp+crank | pass, zero invariant violations |
| M4 | 300-round three-user interleaved fuzz (seed `0x5eed0002`) + over-redeem probes | pass; 19/19 unauthorized redeems reverted |

## Invariants asserted every step

- **I1 — exchange-price monotonicity:** `liquidity_xp`, `token_xp`, `supply_xp`, `borrow_xp` never decrease across any operation sequence including multi-year equivalent warps.
- **I2 — no free minting:** immediate deposit→redeem round-trips always return ≤ deposited amount; dust is 2–3 base units in the protocol's favor, constant-sized (does not scale with amount).
- **I3 — share conservation:** fToken mint supply == baseline + sum of user balances, exact, every step.
- **I4 — monotonic share dilution:** equal-sized deposits after interest accrual mint strictly fewer shares (938,634,800 → 938,626,687 over a 1h warp at prevailing rates).
- **I5 — authorization:** `redeem(balance + 1)` reverts 19/19 (SPL Token insufficient funds at the burn layer — the share ledger lives in `lending`, funds in `liquidity` vault, mint/burn in SPL Token; the three-layer split holds).

## Notable observations (informational, not vulnerabilities)

1. **Dust-scale guard:** amounts ≤ 9 base units revert with `Custom(6030)` consistently — no zero-cost minting path.
2. **Rounding wear is constant:** 2–3 base units per round-trip regardless of size; no amplification vector via repeated splitting.
3. **User PnL matches accrual:** fuzz users (1M USDC notional) gained +297.8 USDC over simulated months, consistent with the +1.96% exchange-price drift — no anomalous emission.

## Vaults liquidation mechanics (M5.x, deployed `vaults` ELF, Vault#1 wSOL/USDC)

- Full position lifecycle on real state: `init_position` (MPL NFT CPI) → `operate(+0.05 wSOL)` → `operate(+2 USDC debt)` → position enters the tick machinery at tick 2466 (97,876 CU for the borrow step).
- **Tick math:** positions are bucketed by `ratioX48 = 1.0015^tick * 2^48` over *decimal-normalized share units* (inputs scaled to 9 decimals, then divided by the respective supply/borrow exchange prices, floor for collateral / ceil for debt — both directions favor the protocol).
- **Liquidation smoke:** with the Chainlink SOL/USD feed locally crashed 5%, `liquidate` against topmost tick 2972 executed (185,077 CU): liquidator paid exactly the covered debt, seized collateral per the tick's entry-locked ratio (documented Fluid design: `col += debt * 2^48 / tick_ratio`), tick 2972 marked liquidated (`total_ids` 1→3, `raw_debt` → 0), `topmost_tick` advanced 2972 → 2971, branch linkage updated.
- Chainlink staleness is enforced by **slot**, not timestamp (`verify_publish_time_with_slot`).

## Environment caveats

All results are from a local fork with mainnet state snapshots; no mainnet transactions were sent. Fuzz coverage is bounded by the harness's operation set (deposit/redeem/warp/crank for M2–M4; operate/liquidate for M5).

*Artifacts: `svm-fuzz-lab` (Rust, LiteSVM 0.18), `tick_calc.py` (exact integer replica of Fluid TickMath), account/ELF snapshots with md5 records. Reproducible via fixed PRNG seeds.*
