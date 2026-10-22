# Heaven (LIGHT) — Privilege-Surface Disclosure: Closed-Source AMM Launchpad, Verified On-Chain

**Scope**: Heaven AMM program `HEAVENoP2qxoeuF8Dj2oT1GHEnu49U5mJYkdeC8BAX2o` (mainnet), LIGHT mint `LiGHtkg3uTa9836RaNkKLLriqTNRcMdRAhqjGWNv777`
**Method**: read-only on-chain observation, binary string forensics on the deployed program, frontend bundle analysis, and cross-checks against the official Kamino klend repository. **No transactions were sent; no state was modified.**
**Date**: 2026-10-21 ｜ Snapshot: slot 459,655,xxx; program binary md5 `c9aff2dfafef459083bac8a686226fcb` (1,633,549 bytes)

---

## Executive Summary

Heaven raised $27M in an 11-hour ICO (August 2025) and operates a closed-source AMM/launchpad. Because no source code or IDL is published, this review reconstructed the attack surface from the chain itself: the deployed binary leaks its full instruction set and error strings (Anchor build artifacts), the program's own address lookup table discloses its integration graph, and all role accounts are directly readable.

**Bottom line: no exploitable vulnerability is claimed here. What is proven is a privilege concentration that users should price in**: three of four privileged roles are bare externally-owned accounts (no multisig, no timelock), a single EOA can replace the entire 1.6 MB program at any time, an admin instruction set can borrow against protocol treasury assets on Kamino, and the LIGHT token carries a transfer-tax lever with an uncapped maximum fee.

| ID | Finding | Classification |
|---|---|---|
| H-1 | Upgrade authority is a single bare EOA | Centralization — Critical surface |
| H-2 | LIGHT transfer-fee lever: uncapped max fee, live admin instruction, prior use at 150 bps | Centralization — Medium |
| H-3 | Treasury actively deployed into Kamino/Marinade via admin instruction family (`AdminBorrowSol`, etc.); debt position not publicly verifiable | Centralization — Medium (exposure unquantified, see *Unknowns*) |
| H-4 | Creator per-pool kill switches (`CreatorToggleProSwap/Deposit/Withdraw`) | Design disclosure — Info |
| H-5 | "Anti-MEV" marketing vs. implementation: same-slot trading ban + slot fees | Disclosure — Info |

---

## H-1 — Single-EOA upgrade authority over the entire protocol (Critical surface)

- Program: `HEAVENoP2qxoeuF8Dj2oT1GHEnu49U5mJYkdeC8BAX2o`, owned by `BPFLoaderUpgradeab1e...`, executable, 1,633,549-byte programdata account `DuevhUr92TiYhu3dgdMDkHdjsK72fqgNr5er4jkRFRCy` (deployed slot 410,580,266).
- Programdata header decodes to upgrade authority `3VAxiBzxbxgH4FYfCjGKsGc1oTkJPJN6SiwMaTWaRfuU` — a system-program-owned account (data length 0, balance 1.001 SOL). **Not a multisig, not a DAO, not a PDA.**
- This key can replace all program logic at any time, covering every liquidity pool, the treasury PDAs, and the fee machinery.

## H-2 — LIGHT transfer-tax lever (Medium)

LIGHT is a Token-2022 mint with the `transferFeeConfig` extension:

- `mintAuthority: null`, `freezeAuthority: null` (supply fixed, no freeze lever — good).
- `maximumFee = 18446744073709551615` (u64::MAX — **no economic cap**).
- `transferFeeConfigAuthority` = `withdrawWithheldAuthority` = `KpXrCt3pjJYFind2kgk7nQ3dS6bqjC2Ze3zzE5MQ78v`, a 1,792-byte PDA owned by the Heaven program (fee changes route through program logic, gated by the on-chain fee-admin role).
- **Live history**: 150 bps through epoch 831, raised/lowered to 0 bps at epoch 838 — the lever has been used.
- `withheldAmount` = 157,013,368,457,485 base units (≈157k LIGHT) sitting withheld across accounts, claimable via `AdminWithdrawTransferFee` (instruction present in the deployed binary).
- Binary-bounded context: the string `Total fees cannot exceed 99.99% (9,999 bps)` exists in the program, but which fee it constrains (transfer fee vs. pool fee) cannot be confirmed without source. Token-2022 transfer-fee updates take effect at epoch boundaries — a one-epoch delay, not an instant rug lever.

## H-3 — Active treasury leverage via Kamino/Marinade admin family (Medium, exposure unquantified)

The deployed binary contains a complete treasury-management instruction family, each with Anchor "Instruction:" log markers:

```
AdminMintMsol   AdminDepositMsol   AdminWithdrawMsol   AdminUnstakeMsol
AdminBorrowSol  AdminRepaySol      AdminClaimMsol      AdminClaimStakingRewards
```

plus Kamino account surfaces (`kamino_lending_obligation`, `kamino_reserve_sol_state`, `kamino_scope_prices`, …) and Marinade wiring (`marinade_finance_program`, mSOL legs).

Verified on-chain from the protocol's own LUT (`FdvM7AszARmB7fqjn98wjPPtfmVi7ZwzzD3woqXEricN`, extracted from the public frontend environment):

- Kamino **Reserve** `FBSyPnxtHKLBZ4UeeUyAnbtFuAmTHLtso9YtsqRDRWpM` — owner `KLend2g3cP87fffoy8q1mQqGKjrxjC8boSyAYavgmjD` (official mainnet deployment per the klend repo README), account size 8,624 bytes = official `RESERVE_SIZE` (8,616) + 8-byte discriminator. Exact layout match.
- Marinade-owned account `8szGkuLTAux9XMgZ2vtY39jVSowEcpBfFfD8hXSEqdGC` (2,616 bytes).
- Two program-controlled SOL pots (both off-curve PDAs): `Du3Ysj1wKbxPKkuPPnvzQLQh8oMSVifs3jGZjJWXFmHN` — **1,127.888 SOL**, and `UefNb6z6yvArqe4cJHTXCqStRsKmWhGxnZzuHbikP5Q` — **5,950.844 SOL** (≈7,078 SOL total at snapshot).

**Interpretation**: protocol funds are not idle — they are deployed into staking and lending. Borrowing introduces liquidation risk borne by token holders and LPs, while the borrowing capacity is exercised by admin keys. Whether the protocol currently carries an open borrow position, and at what health factor, could not be confirmed (see *Unknowns*).

**Role accounts** (all readable on-chain; each a 40-byte PDA state account):

| Role | Address | Form |
|---|---|---|
| protocol owner | `A3XjuDcrumkykWBpMXuSD1zKjzkjtCUby8Vn3HzuSLDZ` | bare EOA (1.95 SOL) |
| fee admin **and** staking admin (same key) | `HKZCA4EJ2e16P9n2afZHXm9ZvZkPcFAp9ZpA9E48Kyma` | bare EOA (107.18 SOL) |
| upgrade authority | `3VAxiBzxbxgH4FYfCjGKsGc1oTkJPJN6SiwMaTWaRfuU` | bare EOA (1.00 SOL) |

Three EOAs control the whole privilege surface; the fee and staking roles share one key, collapsing role separation.

## H-4 — Creator per-pool kill switches (Info)

The instruction set includes `CreatorToggleProSwap`, `CreatorToggleProDeposit`, `CreatorToggleProWithdraw` — a token creator can independently halt buys/sells, deposits, and withdrawals on their own pool. Combined with creator-set fee knobs (`CreatorSetProMarketCapCreatorFee/ReflectionFee/LpFee/SlotCreatorFee`), each pool carries self-contained honeypot-grade switches. This is presumably intentional design (creator flywheels), but users should treat per-pool creator privileges as a first-order risk.

## H-5 — "Anti-MEV" marketing vs. implementation (Info)

Public marketing frames Heaven as anti-MEV. The binary implements: `Same slot trading is disabled`, `Only single swap allowed`, leader-slot windows, and slot fees (`SetProtocolSlotFees`), and the public frontend ships `SNIPER_TAX_DECAY_RATE=1000`. This is a **slot-level rate limiter plus a decaying sniper tax** — an economic friction mechanism, not cryptographic MEV protection. Users should not read more into the label than that.

---

## Unknowns (stated, not guessed)

Per a strict no-speculation policy, the following could not be determined with available read-only channels and are recorded as **unknown**:

1. **The exact constraints on `AdminBorrowSol` / `AdminWithdrawMsol` / `AdminClaim*CreatorTradingFees`** — no source, no IDL (neither classic nor new-style on-chain IDL accounts exist), and the frontend builds transactions server-side (`tx.api.heaven.xyz`), so no IDL ships to clients.
2. **The protocol's current Kamino obligation (debt amount, collateral ratio, distance to liquidation)** — public RPC nodes do not index non-program accounts, and layout-tentative decoding of one candidate account was inconclusive; we do not assert a number we cannot verify.
3. **Admin transaction history** — not indexed by public infrastructure for these EOAs.
4. **Frontend-vs-chain consistency** — the public data API returned 503 across endpoints at snapshot time; no comparison could be run.

## Verifiability

Every address, role assignment, byte size, and balance above was read from mainnet at the stated snapshot slot and can be re-derived with any RPC node (`getAccountInfo`, plus `getProgramAccounts` where noted). The instruction inventory was extracted from a byte-faithful dump of the programdata account (md5 above) via string analysis of Anchor log markers; the Kamino account identification was validated against the official klend repository's `RESERVE_SIZE` constant. No claim in this report rests on inaccessible evidence.
