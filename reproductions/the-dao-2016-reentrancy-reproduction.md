# The DAO — Reentrancy Reproduction (3.6M ETH, 2016-06-17)

**Author:** OnChain Vanguard (Vance)
**Date:** 2026-10-23
**Type:** Reproduction of a historical exploit, for study. **Not an original finding.**
**Reference event:** The DAO hack, 2016-06-17 — ~3.6M ETH drained from the first
decentralized autonomous organization on Ethereum.

---

## 1. What happened

The DAO's `splitDAO()` routine allowed a token holder to split into a child DAO and
receive their share of ETH. The vulnerable path sent ETH to the caller **before**
updating the caller's internal credit balance. An attacker exploited that window with
a reentrant call: the fallback function re-entered `splitDAO()` before the balance was
written down, so the same credit was paid out repeatedly.

~3.6M ETH was drained (roughly one-third of The DAO's total holdings, ~$50M at the
time). The fallout forced the Ethereum hard fork of 2016-07-20 that split the chain
into Ethereum (ETH) and Ethereum Classic (ETC) — the first and still most consequential
hard-fork rescue in the ecosystem.

The fix class is the Checks-Effects-Interactions (CEI) pattern and the modern
`nonReentrant` guard. In Solidity ≥0.8.0 the default-checked arithmetic removed the
related integer-overflow subclass, but the reentrancy itself is **language-independent**
and remains one of the highest-frequency bug classes in EVM contracts today.

## 2. Key facts (multi-source, cross-checked)

- **Loss:** ~3.6M ETH (~$50M USD at the time of the June 2016 drain).
- **Root cause:** ETH transferred to the caller before the caller's balance was zeroed —
  a classic CEI violation leaving a reentrancy window.
- **Exploit window:** ~June 17, 2016, drained over a series of reentrant calls.
- **Outcome:** the 2016-07-20 hard fork reversed the drain by moving the funds to a
  recovery contract; the minority chain continued as Ethereum Classic.
- **Durability of the bug class:** reentrancy still ranks near the top of EVM incident
  causes a decade later (e.g. the 2023 Euler $197M exploit was also reentrancy-adjacent
  in the donation/liquidation path).

## 3. Reproduction

A minimal Foundry project reproduces the exact CEI-violation shape — *transfer first,
zero the balance second* — and a reentrant attacker that drains the pool.

`src/Vault.sol` — the vulnerable contract:

```solidity
function withdraw() external {
    uint256 bal = balances[msg.sender];
    require(bal > 0, "nothing to withdraw");
    // ① external call hands control to the caller first
    (bool ok, ) = msg.sender.call{value: bal}("");
    require(ok, "transfer failed");
    // ② balance is zeroed only after the external call returns
    balances[msg.sender] = 0;
}
```

`src/Attacker.sol` — the reentrant caller:

```solidity
receive() external payable {
    if (address(vault).balance >= 1 ether) {
        vault.withdraw();   // re-enter before the victim zeroes our balance
    }
}
```

`test/Reentrancy.t.sol` — the assertion that closes the loop. A victim deposits
10 ether, the attacker deposits only 1 ether, then attacks:

```
vault  before : 11 ether
attacker before: 0 ether
attacker.attack()
vault  after  : 0 ether     → drained
attacker after: 11 ether    → profited 10 ether
```

`forge test` passes with the two assertions `vaultAfter == 0` and
`attackerAfter > attackerBefore + 1 ether`, and `forge lint` flags the vulnerable line
with a `reentrancy-eth` warning — the same shape The DAO's `splitDAO` exhibited, in
minimal form.

## 4. Detection checklist this event anchors

The lesson generalizes to every EVM value-transfer and external call, not just DAOs:

1. **CEI order** — are all state effects (balance writes, flags) done *before* any
   external call?
2. **`nonReentrant` on value movement** — is the withdrawal/claim/split guarded?
3. **External calls to untrusted addresses** — `call`, `send`, `transfer`, and any
   contract interaction to a non-trusted address is a potential reentry vector.
4. **Balance-before/after invariants** — does a contract's `address(this).balance` vs.
   internal ledger stay consistent across a reentrant frame?

This is the same root-cause family as the cross-chain value-conservation checks in this
portfolio's bridge work: the invariant "you can only move what you hold" must be
enforced atomically, never split across a trust boundary.

## 5. Source

The DAO post-mortem and contemporaneous June 2016 coverage; the Solidity
documentation on reentrancy and the Checks-Effects-Interactions pattern; and the
Foundry PoC in this repository (`forge test` output reproduced verbatim). Historical
loss figures are canonical across sources; where the ~$50M USD figure is cited it
reflects the ETH price at the time of the drain.

---

*OnChain Vanguard — independent cross-chain security research. This document is a
labeled reproduction for study and claims no original finding.*
