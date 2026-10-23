# Allbridge Classic — Signature Verification Missing (Critical)

**Author:** OnChain Vanguard (Vance)
**Date:** 2026-10-23
**Target:** Allbridge Classic — Solana side master contract (`allbridge-io/allbridge-master-contract`)
**Commit:** `92c9803f` (master, 2021-06-15)
**Language:** Rust (Solana program), ~62 KB
**Status:** Draft finding. **Not disclosed** — legacy/deprecated code (repo last active 2021, "will be stopped in November 2026"); finding kept as bridge-audit methodology reference, no live disclosure.

---

## 1. Scope

| Component | Source | Nature |
|---|---|---|
| `src/processor.rs` — instruction handlers (AddBlockchain, AddValidator, AddSignature, …) | `allbridge-io/allbridge-master-contract` @ `92c9803f` | Open source, static review |
| `src/state.rs` — bridge state, validator registry | same | Open source, static review |
| `src/instruction.rs`, `src/utils.rs`, `src/error.rs`, `src/lib.rs` | same | Open source, static review |
| `tests/tests.rs` | same | Open source, static review |

The repository is a **signature-collection** program: it records validator signatures
toward a quorum and marks the bridge ready to release funds. It does not itself mint,
transfer, or burn tokens; final fund release is performed off-chain by a relayer that
reads this contract's state. This boundary is material to grading (see §5).

## 2. Methodology

- Line-by-line review of every instruction handler, with specific attention to whether
  the collected signatures are ever cryptographically verified against the registered
  validator keys.
- Search for any call into `secp256k1` / `ed25519` verification primitives.
- Cross-check of the declared error types against actual usage — an error type that is
  declared but never raised is direct evidence of unimplemented intent.
- Review of the project's own tests to see what the contract accepts as a "signature".

## 3. Findings

### A-1 — `AddSignature` stores 65-byte signatures without ever verifying them — **Critical**

**Component:** `processor.rs` — `AddSignature` (`processor.rs:354-364`)

**Root cause.** `AddSignature` appends the caller-supplied 65-byte `signature` directly
to `bridge.signatures` and then checks `signatures.len() >= quorum`. No verification
primitive is invoked anywhere in the program — a repository-wide search for
`verify`, `secp256k1`, `ed25519`, `recover`, and `check_sig` returns nothing. The
signature bytes are stored verbatim and never consumed by any cryptographic check.

**Three independent proofs that verification was intended but never implemented:**

1. `src/error.rs` declares `InvalidSignature` and `Secp256InstructionLosing` — two
   errors that only make sense if signature verification exists. Neither is ever
   raised anywhere in `processor.rs`. Declared-but-unused error types are the
   signature of a feature that was designed and then dropped.
2. The project's own test suite (`tests/tests.rs`) submits `[7u8; 65]` — 65 garbage
   bytes — as a signature and asserts the contract accepts it. The test passes.
3. The `pub_key` field on each registered validator is written at registration and
   never read again — there is nothing to verify a signature against.

**Impact.** Combined with A-2 (validator self-registration), an attacker can register
their own validator, submit arbitrary garbage "signatures" until the quorum count is
reached, and flip the bridge's `unlock_ready` flag. The off-chain relayer, trusting
this state, releases the bridged funds. The core invariant of any bridge — that
release is authorized by a verified quorum of validators — is broken at the root.

**Static confidence.** Provable from source alone. The absence of any verification
call is exhaustive (repository-wide search), not inferential.

**Recommendation.** Verify each submitted signature against the registered validator
public keys (secp256k1 recover for EVM-origin validators), count only signatures that
verify, require signatures to be distinct, and enforce `m-of-n` quorum after
verification. Restore the intended `InvalidSignature` error path.

### A-2 — `AddValidator` has no ownership check — **High**

**Component:** `processor.rs` — `AddValidator`

`bridge.owner` is written once at initialization and never read again. `AddValidator`
(and `AddBlockchain`, see A-3) do not compare the caller against `bridge.owner`. Any
caller can register themselves as a validator. This is the enabling step for A-1's
impact: without it, forging signatures would still require compromising a real
validator's key.

**Recommendation.** Gate `AddValidator` (and every privileged instruction) on
`ctx.accounts.authority.key == bridge.owner`, or require the bridge authority PDA as
signer.

### A-3 — `AddBlockchain` has no ownership check — **Medium**

Same root cause as A-2: `AddBlockchain` lets any caller register an arbitrary
blockchain mapping. Lower severity than A-2 only because it does not directly grant
signature-counting power, but it is part of the same unprivileged configuration
surface.

### A-4 — No explicit `is_signer` on authority accounts — **Informational**

The authority/owner accounts are not explicitly required to be signers. In the
current CPI context this is largely backstopped by the calling convention, but the
intent is not explicit in the account constraints. Recorded for completeness; not a
standalone finding.

## 4. Positive observations

- The program cleanly separates instruction parsing (`instruction.rs`), state layout
  (`state.rs`), and handling (`processor.rs`) — readable structure.
- State serialization and the validator/blockchain registries are straightforward and
  free of the ownership/decimals classes of bugs that dominate this era of Solana
  bridges.
- The declared error type (`InvalidSignature`) shows the designers understood the
  requirement; the gap is implementation, not design.

## 5. Items marked unknown

- **Fund-release path is out of scope.** This repository does not mint/transfer/burn;
  funds move in a downstream relayer/other-chain component not present here. The
  exploitability of A-1 therefore depends on the (unreviewed) relayer trusting
  `unlock_ready`. Graded Critical on the invariant break; a full end-to-end
  confirmation requires the relayer source.
- **Program ID is a placeholder.** `lib.rs` declares a test placeholder (marked
  `FIXME`); the real mainnet program address was not verified against a live
  deployment.
- **Version status.** Allbridge Classic predates Allbridge Core; whether this contract
  is still the live asset path was not independently confirmed.

## 6. Disclosure

Not disclosed. Allbridge Classic is legacy/deprecated code — the repository's last
activity was 2021 and the Classic line is stated to be stopped in November 2026. The
finding is retained as a bridge-audit methodology reference; no live disclosure is
being made against a codebase that is no longer the active asset path.

---

*OnChain Vanguard — independent cross-chain security research. All evidence in this
report is independently verifiable against the public source of
`allbridge-io/allbridge-master-contract` at commit `92c9803f`.*
