# Percolator — Ported-Function Wiring Gaps: Four Findings from the Deployed v18.2 Line

**Scope**: `percolator-prog` @ `6377376a` (branch `deploy/v18.2-wrapper`), `percolator-engine` @ `deploy/v18.1-engine-267`
**Method**: static review of the deployed-source line only; no mainnet interaction
**Date**: 2026-10-21

---

## Executive Summary

Percolator is a fork of the Toly perp engine with ~310 fork-only public functions added on top of the reference implementation (engine: 24,629 LOC vs. 17,036 LOC upstream). The codebase is heavily defended: Kani harnesses, token value-flow conservation proofs, and an extensive fix history (LIEN-1, C-S-10b, Finding-A, GH#412, #377).

This report documents four findings that share a single root-cause family: **write-side wiring dropped during the port**. Fields, mechanisms, and validation checks all exist — the instruction-level call chains that drive them do not. Individually the impacts are bounded; together they describe a systematic port-hygiene weakness that future features will keep hitting.

| ID | Title | Severity |
|---|---|---|
| F-SE-1 | Stress-envelope producer never wired; `threshold_stress_active` unreachable, HMax early-brake arm dead | Medium |
| F-L4-2 | `deposits_only` insurance ceiling enforced on terminal path only; per-asset operator path bypasses it | Low |
| L-1 | `UpdateMaintenanceFeePerSlot` (tag 88) missing the `policy_sequence` replay guard its siblings all carry | Low/Info |
| F-L4-1 | `insurance_withdraw_max_bps` is a dead config field: validated, initialized, never set or read | Informational |

---

## F-SE-1 — Stress-envelope producer never wired (Medium)

**Design intent** (documented in the engine header, `v16.rs:39-40`):

> When the accumulator crosses `STRESS_ENVELOPE_TRIGGER_BPS_E9`, `threshold_stress_active` is set true and the next `h_lock_lane` lookup lifts the lane HMin→HMax.

**What exists**

- Accumulator field `stress_consumption_bps_e9_since_envelope` (`percolator-engine/src/v16.rs:7807`), producer `apply_stress_envelope_progress` (`:20481`), clearer `clear_stress_envelope_v16` (`:20522`).
- Consumer arm in the lane selector `h_lock_lane` (`:16510-16517`): `threshold_stress_active || ... → HLockLaneV16::HMax`.
- Two production consumption points for HMax:
  - `ensure_favorable_action_allowed` (`:16309`) — hard-freezes favorable actions, gating `preflight_convert_released_pnl_to_capital` (`:19855`).
  - Trade admission `locked` flag (`:19201-19212`), consumed by `finish_trade_checks_not_atomic` (`:18796-18832`), where `locked` additionally enforces `ensure_no_positive_credit_initial_margin` — i.e., during lock, initial margin may not count positive unrealized credit.

**What is missing**

`apply_stress_envelope_progress` has **zero production call sites**. Across `percolator-engine/src` and `percolator-prog/src`, the only callers are tests (`tests/proofs_v17_fork.rs`). Consequences, each independently verifiable:

1. The accumulator never advances on-chain; an engine invariant even asserts it is `== 0` (`v16.rs:8319`).
2. `threshold_stress_active` is initialized false and never set; the on-chain program additionally fail-closed validates it **must be 0** (`percolator-prog/src/v16_program.rs:9358`, `:10406`), so even a manually crafted state is rejected.
3. The stress arm of `h_lock_lane` is dead code in production.

**Impact (quantified)**

The dead arm was the only trigger that lifts HMax *before* Recovery-mode declaration or bankruptcy. With it unreachable, during the stress window the design intended to cover:

- **Risk-increasing trades may count positive unrealized credit toward initial margin** (`ensure_no_positive_credit_initial_margin` is skipped when `locked` is false) — a procyclical margin relaxation exactly when the market is deteriorating.
- **PnL→capital conversion is not frozen**, letting informed actors convert and — subject to the withdrawal path's own gates (flat positions, non-negative PnL, `amount ≤ capital`, `v16.rs:20404-20477`) — exit earlier than the design intended.

What still works: user withdrawals never consult the lane at all (independent gate stack), and all catastrophic brakes (Recovery mode, `bankruptcy_hlock_active`, `liquidation_lock`, stale states, close progress) are live code. The stress lane is defense-in-depth, not the last wall.

**Why not High**: no direct, attributable fund-loss path was demonstrated; both relaxations pass downstream solvency/support checks. **Medium**: a documented safety brake that is provably non-functional in deployment.

---

## F-L4-2 — `deposits_only` insurance ceiling is terminal-path-only (Low)

The code itself states the gap (`percolator-prog/src/v16_program.rs`, insurance withdraw family):

> "The deposits-only ceiling stays terminal-only; its budget tracks market-zero deposited principal, not per-asset insurance"

- Terminal path (tag 41 `WithdrawInsurance`, `handle_withdraw_insurance` @15968): cooldown (F-1) **and** deposits-only ceiling (F-2) both enforced.
- Per-asset path (`WithdrawInsuranceAsset` @16148): cooldown enforced (#396), epoch replay protection (W3A-1) — but the deposits-only ceiling does not apply.

An admin who sets `deposits_only = 1` reasonably expects all insurance withdrawals to be capped at deposited principal market-wide. The per-asset operator path can withdraw available insurance beyond that cap. Mitigating factors: the insurance operator is itself a privileged role (within the trust model), and the cooldown still applies. Classified **Low**: policy semantics and enforcement surface disagree; noted as intentional in a code comment, but the divergence deserves an explicit security note.

---

## L-1 — `UpdateMaintenanceFeePerSlot` (tag 88) lacks the `policy_sequence` guard (Low/Info)

All sibling admin-policy instructions in the deployed wrapper bind an `expected_sequence` / policy-sequence parameter for replay and ordering protection. `UpdateMaintenanceFeePerSlot` (tag 88, `percolator-prog/src/v16_program.rs:19511`) is the exception: no sequence parameter at all.

If an admin signs two successive fee-update transactions, an older signed transaction can land later and silently overwrite the newer value. Partial mitigations exist: a Live-mode gate (#428) and an anti-retroactivity checkpoint (GH#444, prevents new rates from applying to old positions), but nothing enforces ordering between the two updates. **Low/Info**: requires admin key misuse or operational race, and economic impact is limited to fee-parameter regression.

---

## F-L4-1 — `insurance_withdraw_max_bps` is a dead config field (Informational)

- Field exists in config (`v16_program.rs:1692`), shape-validated on load (`:3233`, `insurance_withdraw_policy_shape_ok`), initialized to 0 (`:11273`).
- **No instruction writes it**: the policy setter (tag 92) accepts only `deposits_only` and `cooldown_slots`.
- **No withdrawal path reads it**: a full-repo grep finds exactly five references, all definition/validation/initialization.
- The shape rule `max_bps == 0 → true` makes validation vacuously pass.

The bps-denominated rate cap on insurance withdrawals — a documented design intent — was never wired. No fund risk (cooldown and ceiling remain), but if documentation or the front end implies a bps cap exists, it is a misleading security claim. **Informational**.

---

## Methodology & Verifiability

Every claim above carries a `file:line` anchor against the deployed-source branches listed in the scope header. Key negative claims (zero call sites) were established by full-repository grep across both crates; production vs. test call sites were disambiguated by directory. No dynamic execution, no mainnet transactions, no modification of any state — static review only.

## Suggested Remediation Priority

1. **F-SE-1**: either wire `apply_stress_envelope_progress` into the intended instruction(s) and add an integration test that drives the accumulator across the trigger on a local validator, or remove the mechanism and its documented promise. Half-present safety machinery is worse than absent — it reads as covered in review.
2. **F-L4-2**: document the terminal-only scope of the deposits-only ceiling in the operator guide, or extend the ceiling to the per-asset path.
3. **L-1**: add `policy_sequence` binding to tag 88 for symmetry with sibling policy instructions.
4. **F-L4-1**: implement or remove `insurance_withdraw_max_bps`.
