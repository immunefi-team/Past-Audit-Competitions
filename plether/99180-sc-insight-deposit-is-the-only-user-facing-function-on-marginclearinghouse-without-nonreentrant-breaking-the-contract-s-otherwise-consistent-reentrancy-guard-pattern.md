# \#99180 [SC-Insight] deposit() is the only user-facing function on MarginClearinghouse without nonReentrant, breaking the contract's otherwise consistent reentrancy-guard pattern

**Submitted on Nov 12th 2026 at 10:17:30 UTC by @crypt0n1337 for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #99180
- **Report Type:** Smart Contract
- **Report severity:** Insight
- **Target:** https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/MarginClearinghouse.sol
- **Impacts:**

## Description

`MarginClearinghouse` inherits `ReentrancyGuardTransient` and applies `nonReentrant` to every external state-changing user entry point except one: `deposit(address,uint256)` (`:348-353`). Its functionally identical sibling `depositMargin(uint256)` (`:359`) is guarded, as are `depositFor`, `withdraw` and `withdrawMargin`.

`deposit()` and `depositMargin()` both call `_deposit(account, msg.sender, amount)`; the internal body enforces `owner == account` (`:425`), so the only difference is that `deposit()` takes the account as a parameter. The internal path performs three operations: `safeTransferFrom(owner, address(this), amount)` (`:432`), `settlementBalances[account] += amount` (`:434`), and `_realizeOrCheckpointCarryBeforeMarginChange(account)` (`:436`), a cross-contract call into the engine. The engine has its own `nonReentrant` on that hook, using a separate transient lock, so a reentrant call back into the clearinghouse would not be blocked by the engine's guard and the clearinghouse's own guard was never acquired.

Today the settlement token is a standard six-decimal ERC-20 with no transfer callbacks, and the protocol is non-upgradeable, so there is no reachable reentrancy path. The finding is therefore a defence-in-depth and consistency gap, not an exploit: the guard is free to apply, the identical path already applies it, and its absence forces every future reader to re-derive that the token has no callbacks. It also means the file's own pattern is broken in exactly one place, which is the kind of asymmetry that survives review and later becomes a bug when a token migration or a wrapper is introduced.

Recommendation: add `nonReentrant` to `deposit()`, matching `depositMargin()`.

## Proof of Concept

## Proof of Concept
This is a guard-coverage insight, so the reproduction is a code and call-path comparison.

Step 1. Enumerate the external state-changing user functions on `MarginClearinghouse` and their modifiers: `deposit` (`:348`, no modifier), `depositMargin` (`:359`, `nonReentrant`), `depositFor` (`:371`, `nonReentrant`), `withdraw` (`:403`, `nonReentrant`), `withdrawMargin` (`:414`, `nonReentrant`). The first entry is the only unguarded one.

Step 2. Compare the internal paths. `deposit(account, amount)` calls `_deposit(account, msg.sender, amount)`; `depositMargin(amount)` calls `_deposit(msg.sender, msg.sender, amount)`. `_deposit` requires `owner == account`, so the two calls are the same operation with a different parameter source. One is guarded, the other is not.

Step 3. Trace the callback surface of `_deposit`: `safeTransferFrom` on the settlement token (a standard six-decimal ERC-20 with no hooks), then the balance credit, then `realizeCarryBeforeMarginChange` on the engine. The engine call is `nonReentrant` on the engine, but that is a separate transient lock, so it does not protect the clearinghouse. A reentrant call into a guarded clearinghouse function would therefore succeed if a callback existed.

Step 4. The reachability check: the settlement token cannot be changed and has no callbacks, so no reentrancy path is reachable at this commit. The finding is the inconsistency and the missing defence in depth, not a live exploit.

Suggested fix verification: after adding `nonReentrant` to `deposit()`, a test that re-enters `deposit()` from a malicious token mock during `safeTransferFrom` should revert, while the same test on `depositMargin()` already reverts today.
