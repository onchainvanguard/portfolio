# Zoneless — Idempotency Key Not Tenant-Scoped (Medium)

**Author:** OnChain Vanguard (Vance)
**Date:** 2026-10-23
**Target:** Zoneless — open-source Stripe Connect alternative (USDC/Solana marketplace payouts)
**Repository:** `zonelessdev/zoneless` (main branch)
**Commit:** `5febc1a` (2026-09-30)
**Stack:** TypeScript / Express + MongoDB
**Status:** Draft finding. **Disclosed via GitHub Private Vulnerability Reporting on 2026-10-23** — pending maintainer response (48h window).

---

## 1. Scope

| Component | Source | Nature |
|---|---|---|
| `apps/api/src/middleware/IdempotencyMiddleware.ts` | `zonelessdev/zoneless` @ `5febc1a` | Open source, static review |
| `apps/api/src/middleware/AuthMiddleware.ts`, `Authorization.ts` | same | Open source, static review |
| `apps/api/src/routes/payouts.routes.ts`, `routes/index.ts` | same | Open source, static review |
| `apps/api/src/modules/Payout.ts`, `Balance.ts`, `PlatformAccess.ts` | same | Open source, static review |
| `apps/api/src/modules/identity/IdentityLite.ts`, `DiditProvider.ts` | same | Open source, static review |

The target is a production-money service (dogfooded by its author with real payouts).
Review was static against a full source snapshot; no live requests were sent.

## 2. Methodology

- Line-by-line review of the payout lifecycle: create → build → broadcast.
- Focus on the four payout attack surfaces: payout amount tampering, seller balance
  accounting, idempotency, and KYC bypass.
- A deliberate "what did they get right" pass over the same four areas, recorded as
  positive observations rather than re-filed findings.

## 3. Findings

### Z-1 — Idempotency lock is keyed by `id` alone, not scoped to the tenant — **Medium**

**Component:** `apps/api/src/middleware/IdempotencyMiddleware.ts` (`AcquireIdempotencyLock`, `:35-52`)

**Root cause.** The lock lookup queries `{ id: key }` with no `account` condition, even
though `$setOnInsert` stores the `account` on insert. The middleware is mounted on
every authenticated route (`routes/index.ts:72`), so the idempotency key namespace is
shared across all tenants.

Two failure modes, both reachable by any authenticated caller:

1. **Response disclosure.** If tenant B is the first to use a given `key`, tenant A's
   later request with the same `key` hits B's cached entry and receives B's response
   body and headers — a cross-tenant data leak.
2. **Silent request drop.** In the reverse order, A's operation is never executed;
   A receives B's cached result and believes their own request succeeded. In a payout
   context this means a payout can be silently not-sent.

**Impact.** Data exposure and a silent lost-write, not a direct loss of funds — the
payout amount path itself is separately guarded (see positive observations). Graded
Medium rather than High because no funds are directly moved by this primitive.

**Recommendation.** Scope the lock to the tenant: query
`{ id: key, account: req.user.account }`, and include `account` in the unique index
that backs the lock.

### Z-2 — `broadcast` does not verify the signed transaction binds to the payout — **Low**

`broadcast` (`payouts.routes.ts`) accepts a `signed_transaction` and submits it via
`sendRawTransaction` without checking that it encodes the same amount/recipient the
payout record describes. This is by-design for a self-custodial flow where the seller
signs off-platform, and the seller is the only party harmed by a mismatch. Recorded as
a hardening note rather than an exploitable condition.

### Z-3 — MongoDB transaction has no retry — **Informational**

Balance mutations run inside a MongoDB transaction; there is no retry on transient
abort, and transactions require a replica set. Operational hardening, not a
vulnerability.

### Z-4 — `CheckWalletExists` only verifies the address is on-curve — **Informational**

The Solana wallet check accepts any on-curve address without confirming ownership.
Consistent with the self-custodial model; noted for completeness.

## 4. Positive observations

The payout amount path is the strongest part of the codebase and held up under review:

- **Seven guards on payout amount.** Amount > 0, sufficient balance, wallet ownership,
  on-curve check, `payouts_enabled`, `rejected` state — all enforced before a payout is
  created or built.
- **Seller balance is read-only.** The internal ledger is only debited inside a
  transaction; sellers have no write path to their own balance.
- **KYC webhook signature is rigorous.** The Didit webhook is HMAC-SHA256 verified with
  `timingSafeEqual`, a 300s tolerance, and V2 canonical JSON — no bypass found.
- **`platform_account` is server-forced.** `CreateAccount(req.body, req.user.account)`
  derives the platform binding server-side, so the authorization chain
  (`PlatformAccess` / `CanAccessAccount`) cannot be forged by the client.

## 5. Items marked unknown

- Whether the idempotency collision is exploitable against a specific tenant in
  production (depends on key-generation discipline across tenants).
- Whether the MongoDB deployment runs a replica set (affects Z-3's transactional
  guarantees).

## 6. Disclosure

Disclosed via GitHub Private Vulnerability Reporting (PVR) on 2026-10-23. Zoneless's
`SECURITY.md` specifies a 48h initial-response commitment; awaiting maintainer
response.

---

*OnChain Vanguard — independent security research across the on-chain / off-chain
boundary. All evidence in this report is independently verifiable against the public
source of `zonelessdev/zoneless` at commit `5febc1a`.*
