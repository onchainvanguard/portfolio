# OnChain Vanguard — Solana Security Research Portfolio

Independent adversarial review of production Solana programs. Hong Kong based.

This repository documents completed audit passes over live mainnet deployments.
Every claim is anchored to verifiable evidence: program IDs, deployment slots,
deployed-binary contents, and source file/line references. Where no exploitable
condition was found, we say so, and we show the work.

Contact: Discord `vance467` · Warpcast / GitHub / Superteam Earn: `@onchainvanguard`

## Completed engagements (self-directed, October 2026)

| # | Target | Program (mainnet) | Area under review | Result |
|---|--------|-------------------|-------------------|--------|
| 1 | Marginfi v2 | `MFv2hWf31Z9kbCa1snEPYctwafyhdvnV7FZnsebVacA` | Receivership liquidation state machine | No exploitable condition |
| 2 | Marginfi v2 | same | Premium ratchet / `tag_adjusted_premium` growth bounds | No exploitable condition |
| 3 | Marginfi v2 | same | Order & rebalance instruction set | No exploitable condition |
| 4 | Marginfi v2 | same | Deleverage sub-state machine (privileged entry, daily withdrawal ledger) | No exploitable condition |
| 5 | Marginfi v2 | same | Cross-protocol CPI surface: Kamino / Drift / JupLend venue adapters, oracle synthesis | No exploitable condition |
| 6 | Jupiter Lend (Fluid stack) | `jupeiUmn818Jg1ekPURTpr4mFo29p46vygyykFJ3wZC` (Liquidity) | Exchange-rate and share math; supply/borrow rounding directions | No exploitable condition |
| 7 | Jupiter Lend | `jup3YeL8QhtSx1e253b2FDvsMNC87fDrgQZivbrndc9` (Lending) | fToken mint/burn math, first-deposit inflation class, rewards model | No exploitable condition |
| 8 | Jupiter Lend | Vaults program | Tick-bitmap liquidation engine ported from EVM; remaining-accounts forgery class; batch-liquidation scaling math | No exploitable condition |
| 9 | Jupiter Lend | Flashloan + Oracle programs | Transaction introspection repayment enforcement; dual-track oracle pricing | No exploitable condition |

Nine consecutive passes, zero false-positive submissions. We do not file
informational noise as findings.

## What "verified against the deployed binary" means

Source repositories can lag or diverge from what runs on mainnet. Before any
analysis we reconcile the two:

1. Fetch the programdata account over RPC, verify size, deployment slot, and
   upgrade-authority state.
2. Extract the deployed ELF and scan for instruction and type markers
   (`Instruction: <Name>` compile-time strings, account discriminators).
3. Diff the deployed instruction set against the public source tree.

Example: for Marginfi v2, the deployed 10 MB binary contains
`StartLiquidation`, `StartDeleverage`, `TagLiqRecord`, `StartRebalance`,
`SettleRebalanceTip`, `ConfigureBankPremium`, `CollectBankPremiumFees` —
confirming the receivership/premium/rebalance feature set is live, while the
on-chain Anchor IDL remains a stale v0.1.8 (91 instructions, missing those
entry points). Analysis proceeded against the deployed code, not the IDL.

For Jupiter Lend, the public repository (`Instadapp/fluid-solana-programs`)
is a February 2026 snapshot while mainnet deployments are from September 2026.
Deployed-vs-source instruction-set diff isolated one divergence: a `PauseToken`
instruction present in the deployed Liquidity binary and absent from the public
snapshot. All analysis was flagged accordingly.

## Methodology in brief

- **Downstream-assumption inversion.** When a protocol integrates an external
  venue (e.g. Marginfi depositing into JupLend), the integrator's code encodes
  assumptions about the venue's invariants — exchange-rate monotonicity, rate
  crank timing, share-math precision. We extract those assumptions from the
  integrator and test them against the venue's own implementation. This turned
  one audit (Marginfi venue adapters) directly into an attack checklist for
  the next (Jupiter Lend core math).
- **Rounding-direction audit on every value-transforming arithmetic path.**
  Deposit/withdraw/borrow/repay conversions must round against the user in all
  four directions; any deviation is a value-extraction primitive.
- **Permissionless-entry mapping.** Every instruction reachable without
  privileges is enumerated from the deployed binary before deeper review.
- **EVM-port mismatch surface.** For ported designs (Jupiter Vaults tick
  liquidation), the highest-yield questions are the ones the port must answer
  differently: remaining-accounts authentication, account-model bitmap storage,
  compute-unit bounds on batch operations.

Flagship write-ups live in [`audits/`](audits/); the full per-target set is being ported over.

## Tooling

`svm-fuzz-lab` (in progress): local-fork harness that loads **real deployed
mainnet ELFs** — extracted from programdata accounts — into LiteSVM for
state-level fuzzing of cross-program interaction timing. Built because the
remaining attack surface on top-tier Solana protocols is no longer reachable
by static review alone.
