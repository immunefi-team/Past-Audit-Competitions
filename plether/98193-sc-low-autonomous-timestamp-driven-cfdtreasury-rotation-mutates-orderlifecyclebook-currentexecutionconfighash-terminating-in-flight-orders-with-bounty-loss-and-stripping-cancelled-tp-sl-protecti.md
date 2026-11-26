# \#98193 [SC-Low] Autonomous Timestamp-Driven CfdTreasury Rotation Mutates OrderLifecycleBook.currentExecutionConfigHash(), Terminating In-Flight Orders with Bounty Loss and Stripping Cancelled TP/SL Protection

**Submitted on Nov 5th 2026 at 21:37:27 UTC by @DrHermes for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #98193
- **Report Type:** Smart Contract
- **Report severity:** Low
- **Target:** https://github.com/immunefi-team/audit-comp-plether/tree/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src
- **Impacts:**
  - Griefing (e.g. no profit motive for an attacker, but damage to the users or the protocol)

## Description

## Summary

In `plether-core` at commit `a82d4fefb02768e5b3bddcc97dee371eee24fea8` (in-scope asset `packages/perps/src`), `OrderLifecycleBook.currentExecutionConfigHash()` includes `engine.protocolTreasury()`, which `CfdTreasury._activeTreasury()` switches from `_currentTreasury.account` to `_pendingTreasury.account` purely by wall-clock timestamp (`block.timestamp >= _currentTreasury.lockedUntil`) inside a `view` function. Because traders committing orders before `lockedUntil` are forced to pin the pre-expiry config hash, crossing `lockedUntil` invalidates all in-flight orders before their `executionDeadline`, causes traders on open orders to forfeit their locked `executionBountyAssets` when cleared via `OrderRouter.clearInvalidOrders()`, and causes atomic TP/SL close replacements (`cancelPositionProtectionAndCommitClose`) to cancel protection and then fail with `ConfigMismatch`.

## Root Cause

The problem starts in `CfdTreasury._activeTreasury()` (`packages/perps/src/CfdTreasury.sol`, lines 101-106): when a successor treasury has been queued via `proposeProtocolTreasury()` (lines 70-89), `engine.protocolTreasury()` (lines 45-47) switches from `_currentTreasury.account` to `_pendingTreasury.account` automatically as soon as `block.timestamp >= _currentTreasury.lockedUntil` - i.e. purely based on `block.timestamp` inside a `view` function, with no state-changing call and no `activeConfigVersion` bump:

```solidity
// packages/perps/src/CfdTreasury.sol, lines 45-47 & 100-106
function protocolTreasury() public view returns (address) {
    return _activeTreasury().account;
}

/// @dev Views resolve activation by timestamp, even before the next treasury mutation persists it.
function _activeTreasury() internal view returns (TreasuryConfig storage active) {
    active = _currentTreasury;
    if (_pendingTreasury.account != address(0) && block.timestamp >= _currentTreasury.lockedUntil) {
        active = _pendingTreasury;
    }
}
```

At the same time, `OrderLifecycleBook.currentExecutionConfigHash()` (`packages/perps/src/OrderLifecycleBook.sol`, lines 172-203) includes `engine.protocolTreasury()` at line 199 in the config digest:

```solidity
// packages/perps/src/OrderLifecycleBook.sol, lines 193-202
bytes memory engineConfig = abi.encode(
    engine.planner(),
    engine.settlementSidecar(),
    engine.terminalNavBook(),
    engineAdmin,
    IOrderLifecycleAdminConfigView(engineAdmin).activeConfigVersion(),
    engine.protocolTreasury(),
    IOrderLifecyclePoolConfigView(HOUSE_POOL).markStalenessLimit()
);
return keccak256(bytes.concat(routerConfig, engineConfig));
```

When a trader commits an order, `OrderLiquidationBatchLogic._validateFreshRequest()` (`packages/perps/src/router/OrderLiquidationBatchLogic.sol`, lines 294-297) requires `bounds.expectedConfigHash == currentExecutionConfigHash()` - otherwise it reverts with `OrderRouter__ExecutionConfigMismatch`. A trader committing an order shortly before `lockedUntil` cannot pass `bytes32(0)` and cannot pre-pin the upcoming treasury's hash; both revert immediately.

## Impact & Severity Rationale (Medium - Griefing)

As soon as `block.timestamp` reaches `lockedUntil`, `currentExecutionConfigHash()` flips to the new treasury's hash while orders committed before `lockedUntil` are still well within their execution window (`block.timestamp <= pending.timing.executionDeadline`):

1. **Normal execution fails (`ConfigMismatch`):** `executeOrder` and `executeOrderBatch` refuse to execute the order and settle it as `TerminalReason.ConfigMismatch` (`packages/perps/src/OrderRouterExecutionSidecar.sol`, lines 400-414).
2. **Permissionless oracle-free termination & trader bounty loss on non-expired orders:** `OrderRecoverySidecar.clearInvalidOrders(uint64[] calldata orderIds)` (`packages/perps/src/OrderRecoverySidecar.sol`, lines 67-115) takes no Pyth update parameter and allows cleanup as soon as `expectedConfigHash != observedConfigHash`, even when `expired == false` (lines 90-99):
   ```solidity
   // packages/perps/src/OrderRecoverySidecar.sol, lines 90-99
   bytes32 observedConfigHash = book.currentExecutionConfigHash();
   bool expired = block.timestamp > pending.timing.executionDeadline;
   bool configInvalid = pending.bounds.expectedConfigHash != bytes32(0)
       && pending.bounds.expectedConfigHash != observedConfigHash;
   if (!expired && !configInvalid) {
       continue;
   }
   IOrderRouterExecutionHost.ItemRequest memory request =
       _cleanupRequest(host, orderId, expired, observedConfigHash);
   ```
   Because `OrderRouter._settleTerminalBounty()` (`packages/perps/src/OrderRouter.sol`, lines 314-344, specifically lines 331-342) and `OrderLifecycleBook._validateBountyDisposition()` (`packages/perps/src/OrderLifecycleBook.sol`, lines 708-722) map `ConfigMismatch` to `BountyDisposition.Paid`, the trader's locked `executionBountyAssets` (`200_000` / `0.20 USDG` default cap) is not refunded to the trader, but credited to the `clearInvalidOrders` caller.
3. **Additional impact via shared `_commitOrder` path (`cancelPositionProtectionAndCommitClose`):** Because `PositionProtectionBook.cancelPositionProtectionAndCommitClose()` (`packages/perps/src/PositionProtectionBook.sol`, lines 341-368) cancels armed TP/SL at line 361 and delegates the replacement close order through `commitProtectedClose` -> `_commitOrder` -> `_validateFreshRequest` (`packages/perps/src/router/OrderLiquidationBatchLogic.sol`, lines 168-180, 233, 294-297), any close replacement committed shortly before `lockedUntil` is forced to pin the pre-expiry hash and then fails with `ConfigMismatch` once `lockedUntil` passes.

**Why Medium (Griefing):** Although `clearInvalidOrders` transfers the trader's `0.20 USDG` (`200_000` units) execution bounty to the caller, the bounty is capped at `maxOpenOrderExecutionBountyAssets = 200_000` (`$0.20`), which is comparable to transaction gas costs rather than a large standalone profit vector. We therefore classify this conservatively as **Medium (Griefing - damage to users/protocol without net attacker profit motive after gas)** rather than Critical/High theft of funds: the primary damage to users is the forced failure of live, non-expired orders, the unrefunded `$0.20` bounty debit, and the failure of replacement close orders across the `lockedUntil` boundary.

## Recommendation

Remove `engine.protocolTreasury()` from `OrderLifecycleBook.currentExecutionConfigHash()` at line 199 of `packages/perps/src/OrderLifecycleBook.sol`. `CfdEngine` reads `protocolTreasury()` directly when crediting protocol fees or liquidated bounties, so excluding the fee-recipient address from `currentExecutionConfigHash()` has no side effects on fee routing, execution prices, or margin bounds. (Alternatively, if the protocol wants treasury changes to remain in the config digest, `CfdTreasury` rotation should require an explicit state-sync call rather than switching inside `_activeTreasury()` by timestamp.)

```diff
--- a/packages/perps/src/OrderLifecycleBook.sol
+++ b/packages/perps/src/OrderLifecycleBook.sol
@@ -196,7 +196,6 @@ contract OrderLifecycleBook is IOrderLifecycleBook {
             engine.terminalNavBook(),
             engineAdmin,
             IOrderLifecycleAdminConfigView(engineAdmin).activeConfigVersion(),
-            engine.protocolTreasury(),
             IOrderLifecyclePoolConfigView(HOUSE_POOL).markStalenessLimit()
         );
         return keccak256(bytes.concat(routerConfig, engineConfig));
```

## Link to Proof of Concept

https://gist.github.com/brunjlar/757e11afee068cb9ccb1aa76f111f8c3

## Proof of Concept

## Proof of Concept (Foundry Test)

Save the test below as `packages/perps/test/perps/spec/trader/TreasuryConfigMismatch.t.sol` at commit `a82d4fefb02768e5b3bddcc97dee371eee24fea8` and run:

```bash
forge test --match-path test/perps/spec/trader/TreasuryConfigMismatch.t.sol -vvv
```

```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

import {BasePerpTest} from "../../BasePerpTest.sol";
import {CfdTypes} from "@plether/perps/CfdTypes.sol";
import {OrderV3Types} from "@plether/perps/OrderV3Types.sol";
import {IOrderLifecycleBook} from "@plether/perps/interfaces/IOrderLifecycleBook.sol";

contract TreasuryConfigMismatchTest is BasePerpTest {

    address internal constant ALICE = address(0xA11CE);
    address internal constant BOB = address(0xB0B);
    address internal constant KEEPER = address(0xE0EC);

    uint256 internal constant OPEN_SIZE = 100_000e18;
    uint256 internal constant OPEN_MARGIN_ASSETS = 10_000e6;

    IOrderLifecycleBook internal book;

    function setUp() public override {
        super.setUp();
        book = IOrderLifecycleBook(address(router.lifecycleBook()));
        _fundTrader(ALICE, 50_000e6);
    }

    function test_TreasuryLockExpiryInvalidatesInFlightOrdersAndForfeitsBounty() public {
        _startRecordingLogs();

        // 1. Owner locks current treasury (ALICE) for 1 day and schedules successor treasury (BOB).
        engine.setProtocolTreasury(ALICE, 1 days);
        engine.proposeProtocolTreasury(BOB, 1 days);
        uint256 lockedUntil = engine.protocolTreasuryLockedUntil();

        // 2. 10 seconds before lockedUntil, ALICE commits an open order with a 60s execution window.
        vm.warp(lockedUntil - 10);
        uint256 aliceBefore = _settlementBalance(ALICE);
        uint256 keeperBefore = _settlementBalance(KEEPER);

        OrderV3Types.OrderRequest memory request = _openRequest(bytes32("treasury-expiry"));
        vm.prank(ALICE);
        uint64 orderId = router.commitOrder(request);

        uint64 deadline = book.orderTiming(orderId).executionDeadline;
        assertTrue(deadline > lockedUntil, "order execution window extends past lockedUntil");

        // 3. Advance clock to lockedUntil with no intervening state-changing calls.
        //    engine.protocolTreasury() switches to BOB in _activeTreasury() and mutates currentExecutionConfigHash().
        vm.warp(lockedUntil);
        assertEq(engine.protocolTreasury(), BOB);
        assertTrue(block.timestamp <= deadline, "order still has 50s remaining before executionDeadline");

        // 4. Any caller (KEEPER) can immediately clear the non-expired order without a Pyth price update,
        //    causing ALICE to forfeit her locked executionBountyAssets (0.20 USDG) via BountyDisposition.Paid.
        uint64[] memory ids = new uint64[](1);
        ids[0] = orderId;
        vm.prank(KEEPER);
        (uint256 nextIndex, uint256 cleared) = router.clearInvalidOrders(ids);
        assertEq(nextIndex, 1);
        assertEq(cleared, 1);
        assertEq(router.pendingOrderCounts(ALICE), 0);

        OrderV3Types.CompactOutcome memory outcome = _verifiedOutcome(book, orderId);
        assertEq(uint8(outcome.reason), uint8(OrderV3Types.TerminalReason.ConfigMismatch));
        assertEq(uint8(outcome.bountyDisposition), uint8(OrderV3Types.BountyDisposition.Paid));
        assertEq(outcome.bountyRecipient, KEEPER);
        assertEq(outcome.bountyAssets, 200_000);
        assertEq(aliceBefore - _settlementBalance(ALICE), outcome.bountyAssets);
        assertEq(_settlementBalance(KEEPER) - keeperBefore, outcome.bountyAssets);
    }

    function _openRequest(
        bytes32 clientOrderId
    ) internal view returns (OrderV3Types.OrderRequest memory request) {
        uint32 window = uint32(router.maxExecutionWindowSeconds());
        request.clientOrderId = clientOrderId;
        request.side = CfdTypes.Side.LONG;
        request.sizeDelta = OPEN_SIZE;
        request.marginDelta = OPEN_MARGIN_ASSETS;
        request.targetPrice = 1;
        request.isClose = false;
        request.bounds = OrderV3Types.ExecutionBounds({
            submitBy: uint64(block.timestamp + window),
            executionWindowSeconds: window,
            allowedExecutionModes: 7,
            expectedConfigHash: book.currentExecutionConfigHash(),
            maxExecutionBountyAssets: type(uint256).max,
            maxExecutionNotionalAssets: type(uint256).max,
            maxGrossAccountDebitAssets: type(uint256).max,
            maxActionChargeAssets: type(uint256).max,
            maxExplicitFeesAssets: type(uint256).max,
            maxPostPositionSize: type(uint256).max,
            minPostSettlementBalanceAssets: 0,
            minPostPositionEquityAssets: 0,
            maxPostLeverageBps: type(uint32).max
        });
    }

}
```

