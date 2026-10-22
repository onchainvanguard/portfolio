# Dropset — Independent Security Review

**Author:** OnChain Vanguard (Vance)
**Date:** 2026-10-23
**Target:** Dropset (dropset.io) — on-chain FX order book on Solana, built on the Pinocchio framework
**Repository:** `val-levy/dropset-alpha` (main branch, open source)
**Status:** Public disclosure. Private reporting channels were closed by the project (see Disclosure timeline), so D-1 was disclosed publicly via @onchainvanguard on X on 2026-10-23, notifying @__Dropset__. No funds were interacted with at any point; all work was read-only against public source.

---

## 1. Scope

| Component | Source | Nature |
|---|---|---|
| `market` program (order book, seats, fills) | `val-levy/dropset-alpha` main branch, 248 `.rs` files; core program + interface + price ≈ 3,900 lines | Open source, static review |
| `price` library (order-to-info conversion, mantissa validation) | same repository | Open source, static review |
| `interface/instructions` (order instruction parsing) | same repository | Open source, static review |
| Repository hygiene | repo root | Open source |

The target is pre-mainnet (testnet), program id `TESTnXwv2eHoftsSd5NEdpH4zEu7XRC8jviuoNPdB2Q`. Review was performed against a full snapshot of `main`.

## 2. Methodology

- Line-by-line review of the order book core: seat lifecycle, order posting/cancellation, market-order filling, account resizing, and the free-stack allocator.
- Cross-checking of every seat/order invariant across the full instruction surface, with specific attention to where mapping entries are created, removed, and left stale.
- A deliberate "what did they get right" pass over the four areas most prone to error in an order book: token-account validation, withdrawal ordering, seat hint type confusion, and batch-replace atomicity. These are recorded as positive observations, not re-filed as findings.
- All findings graded on a fixed rubric: High = permanent loss-of-funds or permanent liveness failure reachable without privileges; Medium = a defensive layer is missing or a primitive that can corrupt state under specific conditions; Low = correctness or hygiene issues without direct loss; Info = operational hygiene. Findings without a dynamic reproduction state this explicitly.

No dynamic PoC was run for either of the two substantive findings (see the grading notes on each); the review is static.

## 3. Findings

### D-1 — CloseSeat skips the open-order check, leaving ghost orders that permanently DoS a market side and lock order collateral — **High**

**Component:** `market` program — `process_close_seat`

**Root cause.** `process_close_seat` (`close_seat.rs:43-65`) copies the seat's `base_available` and `quote_available` back to the user, then calls `market.seats().remove_at(sector_index_hint)` — zeroing the seat sector and pushing it back onto the free stack. At no point does it inspect `seat.user_order_sectors` for open orders. The mapping maintenance that every other lifecycle path performs is simply absent on CloseSeat: `cancel_order.rs:56-78` removes the order's mapping entries, and `fill_market_order.rs:379-396` clears them on a full fill. CloseSeat does neither.

When an order is posted, the user's collateral is moved out of `available` and into the order (`post_order.rs:93-109`). CloseSeat refunds only `available` — the collateral inside any open order stays in the market's token account, orphaned with no owner.

**Three consequences, all reachable without special privileges:**

1. **Order collateral permanently locked.** After closing the seat, the user cannot cancel their open orders: `load_mut_seat_with_hint` requires `seat.user == signer` (`seat_operations.rs:117`), the old seat sector is now zeroed/recycled, and the mapping has no entry for the old price — `find_remove` returns `OrderNotFound`. The orders can never be cancelled, and their collateral can never be reclaimed.
2. **Permanent DoS of a market side.** The ghost orders remain on the asks/bids linked list. When a market order reaches them, `update_maker_seat_after_fill` (`fill_market_order.rs:363-397`) loads the maker payload through the order's stale `user_seat` pointer: the sector is zeroed, the mapping is empty, and `find_remove(encoded_price)` errors — rolling back the entire market-order transaction. Once a ghost sits at the top of the book, every market order sized to cross it fails permanently, and nobody can clear the ghost (see consequence 1).
3. **Cross-user seat recycling.** The free stack is LIFO, so the next depositing user inherits the sector. A fill against a ghost order would then credit `try_increment_*_available` to an unrelated user; if that user happens to hold an order at the same price, `find_remove` deletes the wrong mapping entry — cascading corruption.

**Trigger cost.** Rent plus dust collateral. A single transaction can plant up to 10 ghosts per side (`MAX_ORDERS=10`). An attacker can deliberately seed key price levels and abandon the seat.

**Static confidence.** Provable from source alone; the logic chain has no runtime dependency. No on-chain reproduction was run (the program is pre-mainnet; there is no live deployment we can interact with beyond testnet).

**Recommendation.** At the CloseSeat entry, assert that `user_order_sectors` is empty for both sides (e.g. `to_sector_indices().iter().all(|i| *i == NIL)`), returning a `SeatHasOpenOrders` error otherwise; or force inline cancellation of open orders before the seat is released.

---

### D-2 — `fund_then_resize_unchecked` returns early when `lamports_diff == 0`, skipping the resize and leaving the free-stack reinit to write out of bounds — **Medium**

**Component:** `account_resize.rs` → `market_account_view.rs` → `free_stack.rs`

**Root cause.** `fund_then_resize_unchecked` (`account_resize.rs:37-41`):

```rust
let lamports_diff = new_lamports_required.saturating_sub(current_lamports);
if lamports_diff == 0 {
    return Ok(());        // early return; resize_unchecked never runs
}
```

When the account already holds sufficient lamports, the resize is silently skipped. But the caller `MarketAccountView::resize` (`market_account_view.rs:154-161`) proceeds regardless to `convert_zeroed_bytes_to_free_sectors(curr_n_sectors, new_n_sectors)`, and `free_stack.rs:115-124` calls `set_next` on each new sector via a raw pointer — while the account data has not actually been extended. This writes past the end of the account data buffer.

No privileges are required to trigger it: anyone can transfer lamports into the market account. An attacker pre-funds the account so that `lamports_diff == 0`, then waits until free sectors are exhausted; any user's `deposit` (`deposit.rs:94-97`) or `ExpandMarket` then triggers the resize, and the out-of-bounds write lands in the memory adjacent to the SBF input buffer — potentially corrupting serialized data of other accounts in the same transaction, or trapping.

**Grading note.** The primitive exists and the write site is confirmed in source; however the concrete landing address of the out-of-bounds write depends on runtime memory layout, and no dynamic PoC was run (building the Pinocchio + SBF test environment was out of scope for this static pass). Graded conservatively as Medium; a follow-up PoC would establish whether it upgrades to controlled corruption.

**Recommendation.** Change the early return to "transfer lamports only when `diff > 0`, but always run `resize_unchecked`."

---

### D-3 — Zero-quantity orders can be posted — **Low**

**Component:** `price` library + `post_order`

`to_order_info` (`price/src/lib.rs:256-264`) maps `base_scalar = 0` to `base_atoms = 0` and `quote_atoms = 0` with no zero rejection; `ValidatedPriceMantissa` only validates the price mantissa. `post_order` (`post_order.rs:52-109`) accepts the zero-value debit (`checked_sub(0)`) and succeeds. An attacker can mint book noise at no cost, occupying sectors and pushing resize rent onto other users; on fill the zero quantity is cleared harmlessly. **Recommendation:** reject `base_scalar == 0` in `to_order_info`.

### D-4 — `batch_replace` truncates silently — **Low**

**Component:** `interface/instructions/orders.rs`

`into_valid_order_infos_iter` (`orders.rs:121-126`) uses `map_while`, which stops silently at the first invalid parameter without erroring. A user who cancels the old orders (which have already executed) believes they placed N new orders but only the first k are live. Atomicity means no loss of funds, but the silent behavior diverges from trade intent and will trip market-making clients. **Recommendation:** reject the whole batch if any entry is invalid.

### D-5 — Hardcoded test keypair in the repository — **Info**

The repo root `test-keypair.json` holds a 64-byte keypair in plaintext. Marked testnet-only (no value), but it is an operational-hygiene issue; confirm the key is not reused before mainnet. **Recommendation:** remove it from the repo and add a pre-commit secret scan.

## 4. Positive observations

The order book core is more carefully written than most pre-audit code. Specifically:

- `TokenAccountView` / `MintAccountView` validate that the mint is one of `{base, quote}` and check the token account's internal mint/owner fields (`token_account_view.rs:53-58`, `mint_account_view.rs:32-44`); token-2022 paths pin `TransferChecked`.
- `withdraw` performs the transfer before debiting the seat, with `checked_sub` and Solana atomicity as the backstop (`withdraw.rs:44-69`, `market_seat.rs:70-91`).
- Seat-hint type confusion is blocked by `load_(mut_)seat_with_hint` enforcing `seat.user == signer`, and order sectors carry no forgeable 32-byte address field (`seat_operations.rs:84-122`).
- `batch_replace` mapping consistency is atomic across cancel / credit / post / mapping-rewrite / tail-remove, with the iterator upper bound (≤10) pinning `get_mut_entry_at(i)` (`batch_replace.rs:118-262`).
- Event forgery is blocked by `FlushEvents` requiring the `event_authority` PDA as signer and self-CPI hardcoding `program::ID` (`flush_events.rs:15-19`, `event_authority.rs:17-29`).
- Rounding economics use `mul_div` u128 intermediates with floor, and the taker-deposit == fill-amount equality check (`mod.rs:109-112`) closes transfer-fee siphoning.
- Linked-list pointers are updated in pairs, and the free-stack LIFO pop zeroes entries (`linked_list.rs:62-151`).

The test suite covers price ordering, duplicate prices, and max order count — real coverage. The D-1 omission is the textbook miss: the open-order ownership check is absent on exactly the one instruction that removes a seat.

## 5. Items marked unknown

- Whether D-2's out-of-bounds write is controllable (requires a dynamic PoC under Pinocchio + SBF).
- Whether the D-5 test keypair will be reused on mainnet.
- The Pinocchio framework itself and the front-end were out of scope; only the on-chain program and its order/liquidity accounting were reviewed.

## 6. Disclosure timeline

- 2026-10-23 — Static review of `val-levy/dropset-alpha` (main branch) completed; D-1 through D-5 identified.
- 2026-10-23 — Private reporting channels (GitHub Issues and GitHub Private Vulnerability Reporting) are disabled on the repo; no public contact email exists; X DM is restricted. D-1 was therefore disclosed publicly via @onchainvanguard on X on 2026-10-23, notifying @__Dropset__. Primary code maintainer is Matt (GitHub `xbtmatt`).
- Public disclosure — this report. The public notice has been issued; awaiting maintainer response before a fix confirmation is added.

---

*OnChain Vanguard — independent on-chain security research. All evidence in this report is independently verifiable against the public source of `val-levy/dropset-alpha`.*
