# Wormhole — Fake-Message Mint Reproduction ($326M, 2022-02-02)

**Author:** OnChain Vanguard (Vance)
**Date:** 2026-10-23
**Type:** Reproduction of a historical exploit, for study. **Not an original finding.**
**Reference event:** Wormhole bridge exploit, 2022-02-02 — 120,000 wETH (~$326M) minted
without backing.

---

## 1. What happened

The Wormhole Solana-side `verify_signatures` routine confirmed that the secp256k1
precompile had validated the guardian signatures by reading the Instructions sysvar
with the **deprecated** `load_instruction_at`. That helper does **not** check that the
passed account is actually the Instructions sysvar
(`Sysvar1nstructions11111111111111111111111111111111`).

The attacker passed a self-controlled account whose data was crafted to look like a
secp256k1 instruction claiming "13 of 19 guardian signatures verified". The bridge
took that at face value and minted 120,000 wETH on Solana with no backing ETH.

The fix was a single line: replace `load_instruction_at` with
`load_instruction_at_checked`, which validates the sysvar identity before reading.

## 2. Key facts (multi-source, cross-checked)

- **Loss:** ~$326M (120,000 wETH), ~93,750 wETH bridged back to ETH.
- **Threshold:** 13-of-19 guardian signatures — the forged instruction claimed to have
  met it.
- **Backstop:** Jump Crypto replenished the 120,000 ETH within ~24h.
- **Recovery:** ~$225M recovered ~1 year later (February 2023) via a UK High Court
  order and an Oasis counter-exploit of the attacker's remaining position.
- **Bounty:** a $10M white-hat reward went unclaimed.
- **Timing detail:** the fix commit was merged 53 minutes *before* the attack but was
  not yet deployed — a publicly visible, un-deployed patch that effectively served as
  the attacker's roadmap.

## 3. Reproduction

A minimal, semantics-equivalent Rust program demonstrates the vulnerable vs. fixed
behavior against the same forged-account input. It does not run real Solana BPF; it
isolates the single decision point — *is the account identity checked before the
sysvar is read?*

```
[2] vulnerable verify_signatures (no identity check):
    [PASS] signature verification "passed" — attack succeeds
    minted 120000 wETH with no real ETH backing

[3] fixed verify_signatures (identity check):
    [REJECT] rejected: account is not the Instructions sysvar — forgery detected, mint blocked
```

Three unit assertions pass: the vulnerable path accepts the forged account, the fixed
path rejects it, and the quorum-counting logic is unchanged between the two.

## 4. Detection checklist this event anchors

The lesson generalizes to every Solana account, not just sysvars. A "position in the
argument list" is not an identity. Every account must prove what it is through three
checks:

1. **Key check** — is the address what it claims to be (sysvar / PDA / fixed program)?
2. **Owner check** — does `owner` match the expected program?
3. **Signer check** — is it a signer where signing is required?

Any account that skips one of these is attacker-controlled until proven otherwise.
This is the same root-cause family as the unverified-account and missing-signer
findings elsewhere in this portfolio.

## 5. Source

Wormhole post-mortem (2022-02-03), the `verify_signature.rs` diff
(`load_instruction_at` → `load_instruction_at_checked`), and contemporaneous coverage
of the recovery. Exact loss figures and the 53-minute timing detail are from
multi-source cross-check; where sources diverged, the conservative figure is used.

---

*OnChain Vanguard — independent cross-chain security research. This document is a
labeled reproduction for study and claims no original finding.*
