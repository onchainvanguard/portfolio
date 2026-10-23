# OnChain Vanguard — Cross-Chain Security Research Portfolio

Independent adversarial review of production on-chain programs — Solana, EVM,
and cross-chain bridges — plus the off-chain application layer that settles
them. Hong Kong based.

This repository documents completed review passes over production code. Every
claim is anchored to verifiable evidence: program IDs, deployment slots,
deployed-binary contents, and source file/line references. Where no exploitable
condition was found, we say so, and we show the work. Where an issue survived
review, it carries a severity and the exact line that proves it.

Contact: Discord `vance467` · Warpcast / GitHub / Superteam Earn: `@onchainvanguard`

## Repository layout

- [`audits/`](audits/) — independent adversarial audits of production programs.
  Zero exploitable conditions found; each write-up shows the work and says so
  plainly.
- [`findings/`](findings/) — original findings with severity, file/line-anchored
  against the source under review.
- [`disclosures/`](disclosures/) — privilege-surface disclosures: verified
  on-chain centralization findings where **no exploitable vulnerability is
  claimed**. Kept separate from `findings/` so the record is unambiguous about
  what was a confirmed finding versus a disclosed privilege concentration.
- [`fuzzing/`](fuzzing/) — state-level fuzzing research against deployed mainnet
  ELFs. This is where the remaining attack surface lives; see the section below.
- [`reproductions/`](reproductions/) — reproductions of historical exploits for
  study. Every file is labelled a reproduction, not an original finding.
- [`checklists/`](checklists/) — reusable methodology: the cross-chain / Web2
  fast-scan ammunition table and the bridge audit report template. These are the
  standing checklists that drive every engagement above.

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
| 10 | Percolator (perp engine fork) | deploy branches (static review) | Ported-function wiring: stress envelope, insurance ceilings, policy replay | 1 Medium (F-SE-1), 2 Low, 1 Informational |
| 11 | Heaven (LIGHT) | `HEAVENoP2qxoeuF8Dj2oT1GHEnu49U5mJYkdeC8BAX2o` (AMM) | Privilege-surface disclosure: single-EOA upgrade, uncapped transfer-fee lever, treasury leverage, per-pool kill switches | Centralization disclosure — 5 findings, no exploit claimed |
| 12 | Dropset (FX order book) | `TESTnXwv2eHoftsSd5NEdpH4zEu7XRC8jviuoNPdB2Q` (testnet) | CloseSeat open-order check; seat/fill lifecycle; account resizing & free-stack allocator | 1 High (D-1), 1 Medium, 2 Low, 1 Info |
| 13 | Allbridge Classic (Solana bridge) | n/a — program id is a test placeholder; live address not verified | Signature collection vs. verification; validator/blockchain authorization | 1 Critical (A-1), 1 High, 1 Medium, 1 Info |
| 14 | Zoneless (marketplace payout API) | n/a — off-chain TypeScript/Express + MongoDB | Payout authorization, idempotency tenant isolation, KYC webhook signature | 1 Medium (Z-1), 1 Low, 2 Info |
| 15 | SSI Protocol (RWA basket / wrapped token) | `SoSoValueLabs/ssi-protocol` @ `0fd83a0` (static review) | Re-review vs Quantstamp; StakeToken lock overloads, token-hash collision, order-signature scheme | 3 Low, 3 Info |

Fifteen engagements across Solana, EVM, bridge, and off-chain layers. Twelve
Solana reviews closed with zero false-positive submissions except where a real
condition surfaced:
Percolator yielded one Medium (ported-function wiring gaps) with two Low and one
Informational bundled as supporting evidence; Heaven (closed-source) produced a
privilege-surface disclosure — five centralization findings, no exploitable
vulnerability claimed — kept in `disclosures/`; Dropset (pre-mainnet) surfaced
one High — CloseSeat's missing open-order check, which leaves ghost orders able
to permanently DoS a market side and lock order collateral.

The cross-chain and off-chain layer is newer and covered by the same
evidence-anchored standard. Allbridge Classic (Solana bridge) surfaced a
Critical — `AddSignature` stores validator signatures without ever verifying
them, combined with an unauthenticated `AddValidator` — breaking the bridge's
core "verified quorum" invariant at the root. Zoneless (a Stripe-Connect-style
payout API) surfaced a Medium — an idempotency key not scoped to its tenant —
with payout authorization and KYC-webhook signature verification both held up
under review. SSI Protocol (an RWA basket / wrapped-token protocol, first EVM
production target) was re-reviewed against its Quantstamp report; the two prior
Mediums are fixed in the current tree, and the pass surfaced three Low items —
inconsistent `lock` overloads enabling an over-lock DoS, a residual
`encodePacked` token-hash collision, and raw-keccak256 order signatures without
EIP-712 — documented in
[`findings/ssi-protocol-residual-low-findings.md`](findings/ssi-protocol-residual-low-findings.md).
We do not file informational noise as standalone findings.

The Percolator write-up (`findings/percolator-ported-function-wiring-gaps.md`)
documents a single root-cause family: **write-side wiring dropped during the
port** of the Toly perp engine. Fields, mechanisms, and validation checks all
exist; the instruction-level call chains that drive them do not. One Medium
(F-SE-1), two Low, one Informational, each file/line-anchored.

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

Flagship audit write-ups live in [`audits/`](audits/). Historical exploit
reproductions live in [`reproductions/`](reproductions/), labelled as such —
covering Solana (Cashio, Mango, Token-2022), cross-chain (Wormhole), and EVM
(The DAO reentrancy), so the record spans every layer we review.

## State-level fuzzing against deployed binaries

The highest-value work in this repository is in [`fuzzing/`](fuzzing/): a
state-level fuzz campaign against the **deployed mainnet ELFs** of Jupiter
Lend's oracle and vaults programs. The harness loads the real program binaries
(extracted from their programdata accounts) into a **LiteSVM** local fork of
live mainnet state — 808 archived accounts, zero mainnet writes — and
binary-searches accept/revert boundaries on the deployed code, not a public
source snapshot.

The campaign surfaced two reportable items — a deep-crash liquidation
*liveness* constraint and a deployed-vs-documented oracle staleness drift
chain — both measured slot-exact and reproducible from the archived account
set. Full write-up:

- [`fuzzing/jupiter-lend-oracle-drift-liquidation-liveness.md`](fuzzing/jupiter-lend-oracle-drift-liquidation-liveness.md)

The motivation: on top-tier Solana protocols the remaining attack surface is
cross-program timing and edge-state reachability, which static review alone no
longer reaches. Loading the deployed ELF — not the public repo snapshot, which
can lag or diverge — is the only way to fuzz the code that actually runs.

## Tooling

`svm-fuzz-lab`: the local-fork harness behind `fuzzing/`. It extracts deployed
mainnet ELFs from programdata accounts and loads them into LiteSVM for
state-level fuzzing of cross-program interaction timing and edge-state
reachability.
