# Marginfi v2 — Cross-Protocol CPI Surface (Kamino / Drift / JupLend venue adapters)

**Target program:** `MFv2hWf31Z9kbCa1snEPYctwafyhdvnV7FZnsebVacA` (mainnet, 10 MB deployed binary)
**Source basis:** `0dotxyz/marginfi-v2` main; venue adapter sources pulled per-file and byte-count-verified against the GitHub tree manifest.
**Result:** No exploitable condition. Four observations below reporting threshold, retained on file.

## Scope

Marginfi v2 lets a margin account park collateral in external yield venues
(Kamino, Drift, JupLend) instead of idle token vaults. The integration surface
is the interesting part: every venue adapter is a trust boundary where Marginfi
believes something about the venue's accounting. Fourteen adapter and
oracle-synthesis source files were reviewed in full.

## Questions asked and how they resolved

**1. Can a caller substitute a fake venue program or fake venue state?**

No. Every adapter hard-pins the venue program ID with `address =` constraints,
binds venue state accounts with `has_one`, and re-checks the account owner via
`AccountLoader`. The oracle key chain additionally double-binds the venue
reserve account to `oracle_keys[1]`. Representative anchor: the Drift withdraw
CPI construction (`instructions/drift/withdraw.rs`, account constraints
section) — the drift program ID is a compile-time constant, not an argument.

**2. Can the venue underpay, or can share/token deltas be desynchronized?**

No. All three venue adapters assert bilateral deltas: the change in venue-side
shares and the change in vault-side tokens must match unit-for-unit, with at
most a ±1 lamport rounding tolerance. The Drift adapter additionally invokes
the venue CPI with `reduce_only = true`, which makes it structurally impossible
for the withdraw path to open a borrow on Drift.

**3. Do venue legs inherit the receivership/deleverage guards?**

Yes. Deposit/withdraw through any venue inherits the same deferred health
check, zero-price hard revert, daily withdrawal ledger, and rate limiter as the
native paths. Zero-weight collateral cannot be withdrawn through venues either.
There is no "venue escape hatch" that skips the account-state machine.

**4. Can wrapped-asset pricing be manipulated via the oracle?**

No practical path. Venue-bearing assets are priced as `Scope base price ×
venue exchange-rate multiplier`. The multiplier is computed from freshly-read
venue state; stale venue state reverts the transaction. Within a transaction,
the tx-structure gate (`ix_utils.rs`) restricts third-party programs to a
whitelist of refresh instructions only — atomic sandwich manipulation of the
venue rate is closed off.

**5. Permissionless emergency/reward entry points?**

The Kamino emergency-propagation instruction only mirrors the venue's real
flag and cannot set it. Reward harvesting and bad-debt claims pin the
destination to the protocol fee wallet ATA; a caller can trigger them but
cannot redirect proceeds.

## Observations (not submitted)

- ±1 lamport rounding tolerance in bilateral delta assertions — bounded, self-correcting.
- A forgotten refresh flag causes a temporary DoS that self-heals on the next crank.
- Scope oracle usage does not gate on confidence intervals — a privileged-configuration assumption.
- Cross-transaction rate probing is a residual the team documents in its own module comments.

These are design intent or privileged-role hygiene, not user-extractable value.

## Verification notes

- Deployment provenance: programdata account `4Q8u2ny8YYgytJEncwZQfVWbd5axZtfgDxYKHehvPQR7`,
  10 MB occupied, last deployed at slot 456,093,551.
- Deployed-binary scan confirms the venue adapter instruction set is present in the
  live program (see repository README for the binary-diff method).
