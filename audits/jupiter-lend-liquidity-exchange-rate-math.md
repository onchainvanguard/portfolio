# Jupiter Lend (Fluid stack) — Liquidity Program Exchange-Rate & Share Math

**Target program:** `jupeiUmn818Jg1ekPURTpr4mFo29p46vygyykFJ3wZC` (Liquidity, ~791 KB deployed)
**Result:** No exploitable condition. Five observations retained as leads for state-level fuzzing.

## Approach

This review inverted a set of downstream trust assumptions extracted during a
prior audit of Marginfi's JupLend venue adapter. An integrator's code is a
formal statement of what it believes about the venue: exchange-rate
monotonicity, crank timing, share-math precision, account layout. Each belief
becomes an attack question against the venue itself.

## The four assumptions, tested

**A1. "The exchange rate is monotonically increasing."**

Holds. `calculate_exchange_prices` (the rate-update core) contains only
checked additions across the entire function — there is no subtraction path
that could lower the rate. Combined with the Liquidity program's totals-based
rate derivation (computed from internal accounting, not from vault token
balances), direct donation-style price suppression does not exist.

**A2. "Rate updates depend on a privileged crank that can be withheld."**

Stronger than assumed. `UpdateExchangePrice` is a **permissionless,
no-signer crank** — anyone can refresh the rate at any time. The main
`operate` path additionally recomputes rates inline, so withholding a crank
cannot strand a stale price.

**A3. "Share conversion loses or gains value at rounding boundaries."**

Holds tightly. All four value-transforming directions round against the user:
deposit floors, withdraw ceils, borrow ceils, repay floors. Zero-share results
revert explicitly rather than minting dust for free. Overflow is gated at a
2^60 cap with checked math throughout — no bare arithmetic in the conversion
paths.

**A4. "Account layout matches what integrators deserialize."**

Matches. The on-chain Liquidity account structures mirror the layout Marginfi's
adapter binds against, confirmed field-by-field.

## Operate-path hardening worth noting

- `pre_operate` / `operate` handshake is bound by same-transaction timestamp.
- Vault actually-received amounts are checked against expectations within ±1%.
- An EOA may self-declare as "protocol" in account constraints, but position
  activation remains an admin-privileged operation — the self-declaration is
  inert.

## Observations (leads for fuzz stage, not findings)

- `init_token_reserve` initial values — admin-only, unreviewed edge states.
- Interest-free borrow channel from Vaults distorts the utilization denominator — cross-program timing question.
- `PauseToken` instruction exists in the deployed binary but not in the public source snapshot — semantics unverifiable from public code.
- `rate_model` update permissions.
- Lending slippage-variant instructions (`DepositWithMinAmountOut` etc.) and their CPI self-consistency.

## Verification notes

- Deployment verified over RPC: program executable, programdata account size
  and deployment slot recorded (September 2026 deployment).
- Deployed-binary instruction-string scan: every instruction in the public
  source tree is present in the deployed build; `PauseToken` present in binary,
  absent in source (flagged above).
