# [PROTOCOL NAME] — Cross-Chain Bridge Security Audit

**Prepared for:** [Client / Protocol]
**Prepared by:** OnChain Vanguard
**Lead Auditor:** [Name]
**Date:** [YYYY-MM-DD]
**Commit ID:** [full sha]
**Repository:** [url]
**Classification:** [Public / Confidential]

---

## 1. Executive Summary

[Protocol] engaged OnChain Vanguard to perform a security assessment of its cross-chain bridge, covering the source-chain and destination-chain contracts, the validator/relayer set, and the off-chain settlement logic. The engagement ran from [start] to [end] and was scoped to the commit listed above.

[One paragraph describing what the bridge does, the asset flow, and the custody model — lock-and-mint, burn-and-mint, or liquidity-pool — and who is trusted.]

In summary, the assessment identified [N] findings: [C] critical, [H] high, [M] medium, [L] low, [I] informational. [One sentence on the overall posture and the single most important remediation.]

### 1.1 Overview of the Findings

| Severity | Count | Fixed | Addressed | Acknowledged | Open |
|---|---|---|---|---|---|
| Critical | 0 | — | — | — | — |
| High | 0 | — | — | — | — |
| Medium | 0 | — | — | — | — |
| Low | 0 | — | — | — | — |
| Informational | 0 | — | — | — | — |

---

## 2. Scope

The assessment covered the following source files at the commit referenced throughout this report.

| File | SHA-256 | Notes |
|---|---|---|
| `src/…` | `…` | |

### 2.1 Out of Scope

- Any contract, module, or off-chain component not listed above.
- Dependencies and libraries are assumed to be correct.
- Attacks by authorized or privileged actors are considered out of scope where those roles are documented as trusted, unless the threat model states otherwise.

---

## 3. System Overview

Describe, in plain terms, how value moves across the bridge.

- **Source chain**: which contract locks or burns the asset.
- **Destination chain**: which contract mints or releases the asset.
- **Messaging**: how the two sides learn that a lock/burn occurred (validator signatures, light client, oracle).
- **Settlement**: who actually moves funds if settlement is off-chain (relayer / validator set).

### 3.1 Roles

| Role | Privileges | Trust Assumption |
|---|---|---|
| Owner / Governance | … | … |
| Validators | … | … |
| Relayers | … | … |
| Users | … | … |

### 3.2 Trust Assumptions

List every actor that is trusted, and what happens if that trust is violated. This section drives the severity of every finding that follows.

---

## 4. Methodology

- Manual line-by-line review of the in-scope source.
- Static analysis: [Slither / cargo audit / anchor / cargo clippy].
- Dynamic testing: local fork, scripted interaction (Foundry / anchor test), mainnet-state probes via [cast].
- Manual reconstruction of the bridge invariant and cross-side reconciliation.

---

## 5. Risk Methodology

Findings are rated on two axes — **Exploitability** and **Impact** — and assigned a severity from the table below. This follows the industry-standard CVSS-inspired approach used by Halborn and the OWASP risk-rating methodology used by ChainSecurity.

| Severity | Definition |
|---|---|
| **Critical** | Direct loss of funds with no privileged precondition, or permanent breakage of a core invariant. |
| **High** | Loss of funds requiring a narrow precondition, or violation of the bridge's value-conservation invariant under realistic conditions. |
| **Medium** | Funds at risk only under a specific misconfiguration or external condition; or data exposure. |
| **Low** | Best-practice deviations with no direct loss-of-funds path. |
| **Informational** | Observations, hardening suggestions, and documentation gaps. |

Each finding carries a **Status** label: `Open`, `Fixed` (addressed in code), `Addressed` (mitigated non-technically), or `Acknowledged` (accepted as a documented trade-off).

---

## 6. Findings Overview

| ID | Title | Severity | Status |
|---|---|---|---|
| B-1 | … | … | … |

---

## 7. Findings & Technical Details

> Each finding follows the template below. Replace the bracketed content.

### [B-N] [Short, descriptive title]

- **Severity:** [Critical / High / Medium / Low / Informational]
- **Status:** [Open / Fixed / Addressed / Acknowledged]
- **Location:** `path/to/file.rs:NN` or `contracts/X.sol:NN`

**Description**

What the code does, and what it should do instead.

**Impact**

The concrete consequence of exploitation, tied back to the trust assumptions in §3.2.

**Proof of Concept / Evidence**

A minimal reproduction, a storage/state dump, or the exact code excerpt that demonstrates the issue. Where a PoC was run, include the observed output.

**Recommendation**

The specific, actionable fix.

---

## 8. Bridge-Specific Attack Surface — Verification Checklist

This checklist is the bridge-specific core of the report. Each item is marked **PASS**, **FAIL**, or **N/A** with a pointer to the code location that substantiates it.

### 8.1 Value conservation (lock vs. mint reconciliation)

- [ ] The destination mints/releases **only** in response to a verifiable lock/burn on the source.
- [ ] Locked value and minted value are reconcilable; no path mints without a corresponding lock.
- [ ] Decimals and amount scaling are correct across both chains (no off-by-decimals).

### 8.2 Message uniqueness & replay protection

- [ ] Every cross-chain message carries a nonce / sequence that is checked for uniqueness on the destination.
- [ ] Replay of a previously-processed message reverts.
- [ ] Nonce handling is atomic with the state change it guards.

### 8.3 Validator threshold & quorum

- [ ] Signatures are actually **verified** (not merely collected) against the registered validator keys.
- [ ] The threshold is checked (`m-of-n`), not assumed.
- [ ] Validator set changes are permissioned and not open to arbitrary callers.
- [ ] Signature verification uses the correct curve and the correct sysvar/precompile.

### 8.4 Mint / release authorization

- [ ] Only the bridge program/contract can mint or release.
- [ ] The authority check is on the real account/address, not a caller-supplied value.
- [ ] On Solana, every account's identity is proven via owner + address (+ signer where required) — not position.

### 8.5 Emergency pause / circuit breaker

- [ ] A privileged, documented pause path exists and halts asset flow.
- [ ] Pause cannot be abused to freeze user funds indefinitely without recourse.

### 8.6 Off-chain trust (relayers / validators)

- [ ] No single relayer or validator can unilaterally move funds.
- [ ] Off-chain settlement is idempotent and cannot double-pay.

### 8.7 Cross-chain replay

- [ ] A message valid on chain A cannot be replayed on chain B (chain-id / domain separator is included in the signed payload).

### 8.8 Fee manipulation

- [ ] Fees are not derivable from caller-supplied, unvalidated inputs.

---

## 9. Limitations

A security audit is a time-boxed, best-effort exercise. It cannot guarantee the absence of vulnerabilities, and it does not replace operational security, monitoring, or incident response. Findings reflect the code at the audited commit only; subsequent changes are not covered.
