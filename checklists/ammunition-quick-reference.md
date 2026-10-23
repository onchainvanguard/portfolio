# Cross-Chain & Web2 — Quick-Reference Ammunition

**Author:** OnChain Vanguard (Vance)
**Scope:** Solana / EVM / cross-chain bridge / Web2 application layer
**Purpose:** Fast-scan triggers for audit arenas, bounty platforms, and live targets.
Each round maps a trigger signal to a concrete check and the likely hit it produces.
Every entry is anchored to a prior engagement or training reproduction; nothing here
is speculative.

---

## Solana (8 rounds)

| Trigger | Check | Hit |
|---|---|---|
| `Account<'info, T>` with no `seeds=`/`address=`/`has_one=` | Is the account's identity constrained? | Unverified account — attacker substitutes a self-controlled account |
| `bump` supplied by the caller | Is the canonical bump used (`ctx.bumps.*`)? | Non-canonical bump — PDA address not unique |
| `seeds` missing a disambiguating field | Does the seed set uniquely identify the account? | Seed collision — logical account redirected |
| `transfer(...)` return value ignored | Is the error propagated with `?`? | Silent failure — state and funds diverge |
| CPI into another program | Account order + param layout + discriminator correct? | CPI layout mismatch — cross-program call corruption |
| Account that should sign lacks `Signer<'info>` | Is the critical account a signer? | Missing signer — unauthorized operation |
| External call after a state change | Is state written before the CPI (CEI)? | Reentrancy on the Solana side |
| `amount`/`fee` unbounded | Overflow/underflow/zero/decimal scaling checked? | Integer overflow, fee manipulation, decimals mis-scaling |

## EVM (8 rounds)

| Trigger | Check | Hit |
|---|---|---|
| `call{value:}` before state zeroing | State updated before external call (CEI)? | Reentrancy |
| Proxy upgrade | Variable order/slots consistent across versions? | Storage-layout collision |
| Critical function with no `onlyOwner`/modifier | Is authorization enforced? | Missing access control |
| `delegatecall` to a mutable address | Can the target be changed? | delegatecall hijack |
| `call`/`send`/`transfer` return unchecked | Is the return value checked? | Unchecked return — silent failure |
| Arithmetic without SafeMath/0.8 checks | Overflow points | Integer overflow |
| Signature verified, no nonce/used marker | Replay protection present? | Signature replay |
| `tx.origin` used for auth | Should be `msg.sender` | Phishing authorization (tx.origin attack) |

## Cross-chain bridge (8 rounds)

| Trigger | Check | Hit |
|---|---|---|
| Destination mints/releases | Only in response to a verifiable source lock/burn? | Value-conservation break — unbacked mint |
| Cross-chain message has nonce/sequence | Uniqueness + atomicity enforced? | Replay — one lock released multiple times |
| Signatures collected then acted on | Signatures actually verified (not just collected)? | Missing verification — forged multisig |
| Mint/release has an authority | Authority points to the real account (not caller-supplied)? | Forged mint authority |
| No emergency pause | Privileged pause path exists, no indefinite freeze? | No circuit breaker |
| Relies on off-chain relayer/validator | Single point can move funds alone? Idempotent? | Off-chain trust single point, double-pay |
| Signature payload | Includes chain-id / domain separator? | Cross-chain replay |
| Fee derived from caller input | Fee manipulable? | Fee manipulation |

## Web2 (6 rounds)

| Trigger | Check | Hit |
|---|---|---|
| Resource ID straight from URL/param | Query scoped by tenant/ownership? | IDOR — unauthorized read |
| Idempotency lock / cache | Key scoped by tenant? | Cross-tenant key collision — response leak / dropped request |
| Server requests an external URL | URL user-controllable? | SSRF |
| SQL/command concatenation | Parameterized? | Injection |
| Auth middleware | Bypassable (default-allow / wrong order)? | Authentication bypass |
| Deployment config | Probe `/.env` `/debug` stacktrace `/.git` actuator | Config leak — keys / stack / source |

---

## Bridge attack surface (full checklist)

The eight bridge rounds above expand into the complete audit surface; see
[`bridge-audit-report-template.md`](bridge-audit-report-template.md) §8 for the
item-by-item PASS/FAIL checklist with code-location pointers.
