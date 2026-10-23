# SSI Protocol — Residual Low-Severity Findings (Re-review)

**Author:** OnChain Vanguard (Vance)
**Date:** 2026-10-23
**Status:** Draft — not yet disclosed to the protocol team.
**Target:** `SoSoValueLabs/ssi-protocol` (RWA basket / wrapped-token protocol), Solidity
**Commit:** `0fd83a0ba67f17f5b4694064faa883a55e39ab6a` (main, 2026-08-05)
**Prior audit cross-check:** Quantstamp, 2025-01 (commit `#4ff5f0d`), 2 Medium / 7 Low / 2 Info.

---

## Executive summary

Re-review of the current `main` against the prior Quantstamp report. No
High/Critical condition was found; the two prior Mediums are fixed in the
current tree. Three Low-severity items remain, plus informational observations.
The protocol's core trust model — order pricing is set by a maker/order-signer
signature, and the contracts validate structure and signature but **not** the
in/out value equivalence — is documented as a trust assumption rather than a
finding.

## Findings

### SSI-1 — Inconsistent `lock` overloads allow over-locking DoS (Low)

**File:** `src/StakeToken.sol:174-180` (expiry overload) vs `:185-191` (no-expiry overload)

The two `lock` overloads enforce different invariants:

```solidity
// expiry overload — no amount>0 check, no available-balance check, no whenNotPaused
function lock(address user, uint256 amount, uint256 expiry) external onlyLocker {
    _cleanExpiredLocks(user);
    uint256 len = _locks[user].length;
    if (len >= MAX_ACTIVE_LOCKS) revert TooManyActiveLocks(user, len);
    _locks[user].push(Lock(amount, expiry));   // over- or zero-lock accepted
}

// no-expiry overload — full validation
function lock(address user, uint256 amount) external onlyLocker whenNotPaused {
    require(amount > 0, "amount is zero");
    require(getAvailableBalance(user) >= amount, "insufficient available");
    ...
}
```

An authorized locker can push an over-lock (amount greater than the user's
balance) via the expiry overload. `getAvailableBalance` then returns `0`
(`:206`, `if (locked >= balance) return 0`), and the `_update` override
(`:230-238`) blocks **every** transfer / stake / unstake of that user until the
lock expires. The expiry overload also lacks `whenNotPaused`, so locks can be
placed while the token is paused, defeating the pause semantics.

**Severity rationale:** Low — requires the trusted `Locker` role, the DoS is
temporary (bounded by expiry), and there is no direct loss of funds.

**Recommendation:** enforce the same `amount > 0` and
`getAvailableBalance(user) >= amount` checks in both overloads, and add
`whenNotPaused`.

### SSI-2 — `calcTokenHash` boundary-collision persists via `encodePacked` (Low)

**File:** `src/Utils.sol:138-140`

```solidity
function calcTokenHash(Token memory token) internal pure returns (bytes32) {
    return keccak256(abi.encodePacked(token.chain, token.symbol, token.addr, token.decimals));
}
```

`abi.encodePacked` concatenates variable-length `string` fields without a length
separator, so `chain="AB", symbol="C"` and `chain="A", symbol="BC"` produce the
same bytes and therefore the same hash. The prior Quantstamp report listed a
"Token Hash Collision" as fixed, but the function body still uses
`encodePacked`; any mitigation appears to have been applied at call sites rather
than at the source.

Exploitability is limited — `addr` is a fixed 42-char string, so a colliding
description still resolves to the same contract address, and `AssetIssuer`
re-checks `chain` independently (`AssetIssuer.sol:113`). Kept at Low, but the
clean fix is `abi.encode` (or an explicit length prefix), which the
tokenset-equality checks in `AssetIssuer` already use.

### SSI-3 — Raw-keccak256 order signatures without EIP-712; unused nonce (Low)

**Files:** `src/Swap.sol:75-79`, `src/USSI.sol:222-239`

```solidity
bytes32 orderHash = keccak256(abi.encode(orderInfo.order));   // no EIP-712 domain
if (!SignatureChecker.isValidSignatureNow(orderInfo.order.maker, orderHash, orderInfo.orderSign)) return 3;
```

Order signatures are verified against a bare `keccak256` of the struct with no
EIP-712 domain separator and no EIP-191 prefix. The `Order.nonce` field
(`Interface.sol:26`) is never read, incremented, or validated; replay protection
relies solely on the `orderHashs` set plus `deadline`.

This is inconsistent with the same repository's voting contracts
(`RewardedVoting`, `ResearchHubVoting`), which use `EIP712Upgradeable` +
`NoncesUpgradeable`. The core fund-settlement contracts (Swap / USSI) are the
ones left with the weaker scheme. Impact: cross-protocol signature replay if the
same key signs another protocol, and signatures that are opaque and easier to
phish.

---

## Additional observations (informational, not filed as standalone findings)

- **Debug import in production** — `import "forge-std/console.sol";` is left
  uncommented in `AssetFactory.sol:13`, `AssetLocking.sol:11`,
  `AssetToken.sol:10`, `StakeFactory.sol:13`. Inflates bytecode and leaves a
  `forge-std` build dependency in deployed contracts.
- **Cooldown merge** — a second `unstake` overwrites `cooldownEndTimestamp`
  (`StakeToken.sol:132`), extending the lock on amounts already in cooldown.
  Availability delay only, no fund loss.
- **`burnFor` is permissionless** — `AssetIssuer.sol:398-407` lets any holder who
  has granted `AssetIssuer` an allowance destroy their `AssetToken` with no
  compensation. Footgun, not an attack vector.

## Verification against the prior audit

| Quantstamp item | Current state |
|---|---|
| Medium "Inability to Set Cooldown" | Fixed — `StakeToken.setCooldown:152`, `AssetLocking.updateLockConfig:72` |
| Medium "unlockIssue for Assets" | Fixed — `AssetToken.unlockIssue:115` |
| Low "Chain Comparison Bypassed" | Fixed — `AssetIssuer:113` uses keccak256 comparison |
| Low "Reentrancy" | Partial — `nonReentrant` present in StakeToken/RewardedVoting/ResearchHubVoting; absent in Swap/AssetIssuer/USSI, but whitelisted ERC20s have no receiver callback and the sensitive paths are `onlyOwner` |

## Trust assumption (not a finding)

Mint/redeem in/out pricing is set entirely by a maker-signed order; the
contracts validate structure and signature but do not check that `inAmount` and
`outAmount` are value-equivalent. The maker (`MAKER_ROLE`) is the protocol's
market-maker, so a mispriced order is self-harm. Documented as centralization /
trust assumption, consistent with the prior audit's conclusion.

---

*OnChain Vanguard — independent cross-chain security research. Findings are
file/line-anchored against commit `0fd83a0`; no High/Critical claimed.*
