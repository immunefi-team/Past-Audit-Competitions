# \#98223 [SC-Low] Scheduled treasury activation invalidates queued close and protection orders without an activation transaction

**Submitted on Nov 6th 2026 at 02:59:42 UTC by @Tradi3 for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #98223
- **Report Type:** Smart Contract
- **Report severity:** Low
- **Target:** https://github.com/immunefi-team/audit-comp-plether/tree/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src
- **Impacts:**

## Description

**Program:** Plether audit competition · **target:** `a82d4fefb02768e5b3bddcc97dee371eee24fea8` · **classification:** Insight (no Low/Medium impact claim)

## Summary

An owner can schedule a protocol-treasury successor to become effective at the current lock's expiry. The treasury getter changes by timestamp without an activation transaction or event. `OrderLifecycleBook.currentExecutionConfigHash()` includes that getter, so its value also changes at expiry even though neither admin's `activeConfigVersion` changes. A pending external V3 order that pinned the earlier hash terminally fails as `ConfigMismatch` when someone next tries to execute it, even if its execution deadline has not passed.

This matters most to an integrator or keeper that treats an already queued risk-reducing close as a live exit through the scheduled activation. After a cleanup call, that close is gone but the position remains open. A config-pinned parent open and its attached pending-open protection also fail together. The terminal receipt exposes the failure, so this is a monitoring and recommit obligation, not a silent persistent freeze or a claim of an unprivileged attack. The scheduled owner action is legitimate and public in advance.

## Conditions and path

1. The Engine owner locks treasury A, then calls `proposeProtocolTreasury(B, ...)` while the lock is active. Let `T` be the current lock expiry. The pending successor and `T` are readable before activation.
2. Before `T`, a trader commits an external V3 order with the required nonzero `expectedConfigHash = currentExecutionConfigHash()` and an execution deadline after `T`. The order is still pending at `T`.
3. At `T`, `protocolTreasury()` resolves to B by `block.timestamp` without a transaction. The digest now hashes B instead of A. A keeper or trader calling `executeOrder` after `T` receives a terminal `ConfigMismatch` before the historical Pyth parse or Engine trade path.
4. For an open, the reserved execution bounty is paid to an independent cleaner; with the pinned Router defaults, each such bounty is capped at **0.20 USDG**. If the trader self-clears, the bounty is released back to its source. For a full close, the zero-bounty order is removed and the position remains open until a fresh close is committed under the new digest and executed. An attached pending-open protection fails with its parent, so the trader must recommit the intended protected open.

The boundary requires a scheduled successor, an order whose commit precedes `T`, a deadline extending past `T`, and no execution before `T`. It is not a claim that all rotation schedules cause user loss. The bounty amount is a release-default bound per order, not a loss estimate for all users or deployments.

## Source and reproducibility

- [`CfdTreasury.sol:43-46,100-105`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/CfdTreasury.sol#L43-L46) resolves the active treasury from time; `proposeProtocolTreasury` schedules the successor at the current lock expiry.
- [`OrderLifecycleBook.sol:172-203`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/OrderLifecycleBook.sol#L172-L203) includes `engine.protocolTreasury()` in the external order's execution-config digest. The Book stores the requested execution deadline separately at `:287-296`.
- [`OrderRouterExecutionSidecar.sol:403-414,906-917`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/OrderRouterExecutionSidecar.sol#L403-L414) terminalizes a nonzero-hash mismatch and can classify it before oracle work. [`OrderRouter.sol:287-338`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/OrderRouter.sol#L287-L338) releases the reservation, pays an independent bounty recipient, or refunds it on self-clear. The constructor's open-bounty cap is `200_000` 6-decimal atoms (`router/OrderRouterBase.sol:74`); ordinary closes have zero bounty.
- A test-only fixture at `packages/perps/test/perps/cx-insights/CxTreasuryRotation.t.sol` (SHA256 `CDF21779C23B38972BE3AD68C0FAB982A74632EE0E9419D49929B4806ED22097`) covers timestamp-only digest change, third-party open cleanup, attached-protection failure, full-close invalidation, self-clear bounty recovery, and a fresh close committed under the successor digest. The first three cases were copied from the independently reviewed DeepSeek fixture with the relative import and test clock reads adjusted; the last two are additional controls. No in-scope source was changed.

Reproduction from the pinned repository root, once the test-only file is placed at the path above:

```sh
cd packages/perps
/opt/foundry-1.5.1/forge test --offline --no-auto-detect --match-path test/perps/cx-insights/CxTreasuryRotation.t.sol --jobs 1 -vv
```

The focused command passed **5/5 tests**, zero failures, process exit `0`, on the isolated exact-pin Kali checkout; full durable output is the exact run output quoted in the proof of concept. The additional self-clear test confirms no net reserved-bounty loss when Alice clears her own mismatched open; it does not account for transaction gas. The recommit test confirms a zero-bounty full close first fails with the position still open, then a fresh close under the successor digest executes and removes the position. These are local mock results, not an observed production rotation or evidence of attacker control over governance timing.

## Suggested handling

For current deployments, surface a scheduled activation in trader clients and keeper monitoring. Watch `pendingProtocolTreasury()` and `protocolTreasuryLockedUntil()`, compare the activation with each pending order's execution deadline, and treat a `ConfigMismatch` terminal receipt as a user-action alert. Recommit a close or protected open with a new client id and the post-activation digest. A trader who self-clears a mismatched open can recover its reserved bounty, but still has to submit a new order.

For a future contract version, decide explicitly whether treasury *recipient identity* is an execution policy that should invalidate existing orders. If it only routes future protocol credits without altering user economic bounds, consider excluding the timestamp-resolved recipient from the order digest while recording the actual recipient in execution evidence. If pinning the recipient is intentional, make the transition explicit at admission: detect a scheduled activation inside the requested execution window and offer a safe bounded path for risk-reducing closes rather than letting a queued exit disappear unexpectedly. A version bump or an activation event alone would improve visibility but would not preserve the pending order. Either direction needs tests for fee routing, user bounds, FIFO cleanup, attached protections, expiry, and the exact `T-1/T/T+1` boundary.

## Prior disclosure and limits

[PR #130](https://github.com/Plether-Fi/plether-core/pull/130), merged 2026-10-27, deliberately introduced automatic transaction-free successor activation. The pinned [security guide](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/SECURITY.md#L114-L151) documents that behavior and lack of an activation event; its [configuration section](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/SECURITY.md#L754-L777) explicitly says protocol treasury is hashed and config drift terminalizes external orders. This report identifies the order-lifecycle effect at the already public activation boundary. It may be intended design or overlap prior public/private knowledge. No direct theft, sustained freeze, unprivileged ability to schedule rotation, or severity-rated impact is claimed.

## Proof of Concept

Conditions under which this Insight is valuable: the Engine owner has legitimately scheduled a treasury successor to activate at the current lock expiry T, and a trader has a config-pinned V3 order still pending across T. An integrator or keeper that assumes the queued risk-reducing close or protected open remains executable after T must detect its terminal ConfigMismatch and prompt a new commitment. This does not grant an unprivileged attacker control of T or prove a severity-rated loss.

Pinned commit: a82d4fefb02768e5b3bddcc97dee371eee24fea8. Test-only path: packages/perps/test/perps/cx-insights/CxTreasuryRotation.t.sol. SHA256: CDF21779C23B38972BE3AD68C0FAB982A74632EE0E9419D49929B4806ED22097. The source was retained on Windows and run on the isolated exact-pin Kali checkout with Forge 1.5.1 / Solc 0.8.35. No packages/perps/src file was modified.

From packages/perps:
```sh
/opt/foundry-1.5.1/forge test --offline --no-auto-detect --match-path test/perps/cx-insights/CxTreasuryRotation.t.sol --jobs 1 -vv
```
Output: 5 passed, 0 failed, 0 skipped. Passed cases: timestamp-only pending-order invalidation; attached pending-open protection failure; full-close invalidation with position still open; self-clear refund; successful post-rotation close recommit. Full source follows:

```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

import {BasePerpTest} from "../BasePerpTest.sol";
import {CfdTypes} from "@plether/perps/CfdTypes.sol";
import {OrderV3Types} from "@plether/perps/OrderV3Types.sol";
import {IOrderLifecycleBook} from "@plether/perps/interfaces/IOrderLifecycleBook.sol";
import {IOrderRouterAccounting} from "@plether/perps/interfaces/IOrderRouterAccounting.sol";
import {IOrderRouterAdminHost} from "@plether/perps/interfaces/IOrderRouterAdminHost.sol";
import {IPositionProtectionActions} from "@plether/perps/interfaces/IPositionProtectionActions.sol";
import {IPositionProtectionViews} from "@plether/perps/interfaces/IPositionProtectionViews.sol";
import {PositionProtectionTypes} from "@plether/perps/interfaces/PositionProtectionTypes.sol";
import {MockPyth} from "@plether/test-utils/MockPyth.sol";

/// @notice PoC: a scheduled automatic protocol-treasury rotation changes currentExecutionConfigHash()
///         at the treasury lock-expiry timestamp without any transaction, terminally invalidating every
///         config-pinned pending order and failing attached pending-open protections at that instant.
contract ProtectionEmergencyTreasuryRotationTest is BasePerpTest {

    address internal constant ALICE = address(0xA11CE);
    address internal constant EXECUTION_KEEPER = address(0xE0EC);
    address internal constant TREASURY_A = address(0xFEE5000A);
    address internal constant TREASURY_B = address(0xFEE5000B);

    uint256 internal constant MARK_PRICE = 1e8;
    uint256 internal constant POSITION_SIZE = 10_000e18;
    uint256 internal constant POSITION_MARGIN_ASSETS = 2000e6;

    IPositionProtectionActions internal protectionActions;
    IPositionProtectionViews internal protectionViews;

    function setUp() public override {
        super.setUp();
        protectionActions = IPositionProtectionActions(address(router.positionProtectionBook()));
        protectionViews = IPositionProtectionViews(address(router.positionProtectionBook()));

        // Enable a bounded execution window (production default is 60s) so a committed order can outlive the
        // treasury lock while still being inside its execution window.
        IOrderRouterAdminHost.RouterConfig memory config = _routerConfig();
        config.maxExecutionWindowSeconds = 60;
        config.orderSettlementWindow = 15;
        _setRouterConfig(config);

        _fundTrader(ALICE, 20_000e6);
        _refreshMark(MARK_PRICE);
    }

    /// @notice Core PoC: pending open order committed before lock expiry is terminally failed by the
    ///         automatic treasury rotation; its reserved execution bounty is paid to the cleaner.
    function test_TreasuryLockExpiryInvalidatesPinnedPendingOrderWithoutTransaction() public {
        // 1. Owner locks the current treasury and schedules an automatic successor at lock expiry.
        engine.setProtocolTreasury(TREASURY_A, 20);
        engine.proposeProtocolTreasury(TREASURY_B, 0);
        assertEq(engine.protocolTreasury(), TREASURY_A, "treasury A active before expiry");

        bytes32 pinnedHash = router.lifecycleBook().currentExecutionConfigHash();

        // 2. Alice commits an ordinary bounded open while the treasury is A. The order pins the config digest.
        OrderV3Types.OrderRequest memory request = _publicBoundedOpenRequest(keccak256("pending-open"));
        uint256 freeBefore = _freeSettlementAssets(ALICE);
        vm.prank(ALICE);
        uint64 orderId = router.commitOrder(request);
        uint256 bounty = _quoteOpenOrderExecutionBountyAssets(POSITION_SIZE);
        assertGt(bounty, 0, "fixture requires a nonzero open bounty");
        assertEq(_freeSettlementAssets(ALICE) + POSITION_MARGIN_ASSETS + bounty, freeBefore, "commit reserved margin and bounty");

        // 3. Time passes past the treasury lock expiry. No transaction runs at expiry: the effective treasury
        //    (and therefore the execution-config digest) flips purely by timestamp.
        vm.warp(vm.getBlockTimestamp() + 21);
        assertEq(engine.protocolTreasury(), TREASURY_B, "successor resolves by timestamp without any transaction");
        (address pendingTreasury,,) = engine.pendingProtocolTreasury();
        assertEq(pendingTreasury, address(0), "pending view reports matured proposal as inactive");
        bytes32 rotatedHash = router.lifecycleBook().currentExecutionConfigHash();
        assertTrue(rotatedHash != pinnedHash, "config digest changed at lock expiry without a transaction");

        // 4. Any keeper can now clear the order. The order is inside its execution window, so this is a terminal
        //    ConfigMismatch, not an expiry.
        bytes[] memory executionData = _mockPythUpdateData(MARK_PRICE);
        uint256 keeperBefore = clearinghouse.balanceAssets(EXECUTION_KEEPER);
        vm.prank(EXECUTION_KEEPER);
        OrderV3Types.ExecutionResult memory result = router.executeOrder(orderId, executionData);

        assertEq(uint8(result.status), uint8(OrderV3Types.LifecycleStatus.Failed), "order failed");
        assertEq(
            uint8(result.terminalReason),
            uint8(OrderV3Types.TerminalReason.ConfigMismatch),
            "terminal reason is ConfigMismatch"
        );
        assertEq(
            clearinghouse.balanceAssets(EXECUTION_KEEPER) - keeperBefore,
            bounty,
            "cleaner is paid Alice's reserved execution bounty for a no-work pre-oracle failure"
        );
        assertEq(
            _freeSettlementAssets(ALICE),
            freeBefore - bounty,
            "Alice loses the bounty and gets only the margin back"
        );
        assertEq(router.pendingOrderCounts(ALICE), 0, "queue entry removed");
    }

    /// @notice Extension: an attached pending-open protection is terminally failed by the same rotation, so the
    ///         position the trader opens next is left without the stop-loss/take-profit they paid to reserve.
    function test_TreasuryLockExpiryFailsAttachedPendingOpenProtection() public {
        engine.setProtocolTreasury(TREASURY_A, 20);
        engine.proposeProtocolTreasury(TREASURY_B, 0);

        OrderV3Types.OrderRequest memory request = _publicBoundedOpenRequest(keccak256("pending-protected-open"));
        vm.prank(ALICE);
        (uint64 parentOrderId, uint64 protectionId) =
            protectionActions.commitOpenOrderWithProtection(request, _params(90_000_000, 110_000_000));

        uint256 freeBefore = _freeSettlementAssets(ALICE);
        uint256 parentBounty = _quoteOpenOrderExecutionBountyAssets(POSITION_SIZE);
        uint256 keeperBefore = clearinghouse.balanceAssets(EXECUTION_KEEPER);

        vm.warp(vm.getBlockTimestamp() + 21);
        assertTrue(
            router.lifecycleBook().currentExecutionConfigHash() != request.bounds.expectedConfigHash,
            "digest rotated automatically"
        );

        bytes[] memory executionData = _mockPythUpdateData(MARK_PRICE);
        vm.prank(EXECUTION_KEEPER);
        router.executeOrder(parentOrderId, executionData);

        PositionProtectionTypes.PositionProtectionView memory protection =
            protectionViews.getPositionProtection(protectionId);
        assertEq(
            uint8(protection.status),
            uint8(PositionProtectionTypes.PositionProtectionStatus.Failed),
            "attached protection terminally failed"
        );
        assertEq(protectionViews.activePositionProtectionId(ALICE), 0, "trade lock released");
        // The parent's execution bounty was pre-locked out of free settlement and is paid to the cleaner, so Alice's
        // free settlement recovers exactly the committed margin plus the refunded protection bounty.
        assertEq(
            _freeSettlementAssets(ALICE),
            freeBefore + POSITION_MARGIN_ASSETS + 200_000,
            "margin and protection bounty returned; parent bounty is gone to the cleaner"
        );
        assertEq(
            clearinghouse.balanceAssets(EXECUTION_KEEPER) - keeperBefore,
            parentBounty,
            "cleaner is paid the protected parent open's reserved bounty"
        );
        (uint256 size,,,,,,) = engine.positions(ALICE);
        assertEq(size, 0, "parent open did not execute");
    }


    /// @notice Extension: a queued *risk-reducing* full close (zero bounty) is also silently invalidated at the
    ///         rotation instant, leaving the trader's position open when they believe an exit is queued.
    function test_TreasuryLockExpiryInvalidatesPinnedFullCloseAndLeavesPositionOpen() public {
        // Open a live LONG position for Alice and refresh the mark.
        _open(ALICE, CfdTypes.Side.LONG, POSITION_SIZE, POSITION_MARGIN_ASSETS, MARK_PRICE);
        (uint256 sizeBefore,,,,,,) = engine.positions(ALICE);
        assertEq(sizeBefore, POSITION_SIZE, "fixture position");

        // Schedule the automatic rotation inside the close order's execution window.
        engine.setProtocolTreasury(TREASURY_A, 20);
        engine.proposeProtocolTreasury(TREASURY_B, 0);

        OrderV3Types.OrderRequest memory request = _publicBoundedOpenRequest(keccak256("pending-full-close"));
        request.isClose = true;
        request.sizeDelta = POSITION_SIZE;
        request.marginDelta = 0;
        request.targetPrice = engine.CAP_PRICE();
        vm.prank(ALICE);
        uint64 closeOrderId = router.commitOrder(request);

        vm.warp(vm.getBlockTimestamp() + 21);
        assertEq(engine.protocolTreasury(), TREASURY_B, "rotation resolved by timestamp");

        bytes[] memory executionData = _mockPythUpdateData(MARK_PRICE);
        vm.prank(EXECUTION_KEEPER);
        OrderV3Types.ExecutionResult memory result = router.executeOrder(closeOrderId, executionData);

        assertEq(uint8(result.status), uint8(OrderV3Types.LifecycleStatus.Failed), "close failed");
        assertEq(
            uint8(result.terminalReason),
            uint8(OrderV3Types.TerminalReason.ConfigMismatch),
            "close failed as ConfigMismatch"
        );
        (uint256 sizeAfter,,,,,,) = engine.positions(ALICE);
        assertEq(sizeAfter, POSITION_SIZE, "position stays open: risk-reducing intent silently voided");
        assertEq(router.pendingOrderCounts(ALICE), 0, "no live exit remains queued");
    }

    /// @notice User action control: a trader who clears her own config-mismatched open
    ///         receives the reserved execution bounty back, but still must recommit.
    function test_TreasuryLockExpirySelfClearRefundsOpenBounty() public {
        engine.setProtocolTreasury(TREASURY_A, 20);
        engine.proposeProtocolTreasury(TREASURY_B, 0);

        uint256 freeBefore = _freeSettlementAssets(ALICE);
        OrderV3Types.OrderRequest memory request = _publicBoundedOpenRequest(keccak256("self-clear-open"));
        vm.prank(ALICE);
        uint64 orderId = router.commitOrder(request);
        uint256 bounty = _quoteOpenOrderExecutionBountyAssets(POSITION_SIZE);
        assertGt(bounty, 0, "fixture open bounty");

        vm.warp(vm.getBlockTimestamp() + 21);
        assertEq(engine.protocolTreasury(), TREASURY_B, "successor active");
        // ConfigMismatch is pre-oracle, so no signed Pyth update or ETH fee is needed.
        vm.prank(ALICE);
        OrderV3Types.ExecutionResult memory cleared = router.executeOrder(orderId, new bytes[](0));
        assertEq(uint8(cleared.status), uint8(OrderV3Types.LifecycleStatus.Failed), "self-cleared order failed");
        assertEq(uint8(cleared.terminalReason), uint8(OrderV3Types.TerminalReason.ConfigMismatch));
        assertEq(_freeSettlementAssets(ALICE), freeBefore, "margin and open bounty returned to source");
        assertEq(router.pendingOrderCounts(ALICE), 0, "queue freed");
    }

    /// @notice Recovery control: an automatically invalidated risk-reducing close
    ///         leaves its position open, but a new close pinned after rotation can settle.
    function test_TreasuryLockExpiryCloseCanRecommitUnderNewDigest() public {
        _open(ALICE, CfdTypes.Side.LONG, POSITION_SIZE, POSITION_MARGIN_ASSETS, MARK_PRICE);
        engine.setProtocolTreasury(TREASURY_A, 20);
        engine.proposeProtocolTreasury(TREASURY_B, 0);

        OrderV3Types.OrderRequest memory request = _publicBoundedOpenRequest(keccak256("old-close"));
        request.isClose = true;
        request.marginDelta = 0;
        request.targetPrice = engine.CAP_PRICE();
        vm.prank(ALICE);
        uint64 oldId = router.commitOrder(request);

        vm.warp(vm.getBlockTimestamp() + 21);
        vm.prank(EXECUTION_KEEPER);
        OrderV3Types.ExecutionResult memory failed = router.executeOrder(oldId, new bytes[](0));
        assertEq(uint8(failed.status), uint8(OrderV3Types.LifecycleStatus.Failed));
        assertEq(uint8(failed.terminalReason), uint8(OrderV3Types.TerminalReason.ConfigMismatch));
        (uint256 stillOpen,,,,,,) = engine.positions(ALICE);
        assertEq(stillOpen, POSITION_SIZE, "old close left position open");

        bytes32 newDigest = router.lifecycleBook().currentExecutionConfigHash();
        request = _publicBoundedOpenRequest(keccak256("new-close"));
        request.isClose = true;
        request.marginDelta = 0;
        request.targetPrice = engine.CAP_PRICE();
        vm.prank(ALICE);
        uint64 newId = router.commitOrder(request);
        assertEq(router.pendingOrderCounts(ALICE), 1, "fresh close queued");
        assertEq(
            IOrderLifecycleBook(address(router.lifecycleBook())).pendingIntent(newId).bounds.expectedConfigHash,
            newDigest,
            "new close pins successor digest"
        );

        bytes[] memory executionData = _mockPythUpdateData(MARK_PRICE);
        vm.prank(EXECUTION_KEEPER);
        OrderV3Types.ExecutionResult memory executed = router.executeOrder(newId, executionData);
        assertEq(uint8(executed.status), uint8(OrderV3Types.LifecycleStatus.Executed), "recommitted close executes");
        (uint256 sizeAfter,,,,,,) = engine.positions(ALICE);
        assertEq(sizeAfter, 0, "position closes after recommit");
    }

    // --- helpers copied from the canonical protection fixture ---

    function _refreshMark(
        uint256 price
    ) internal {
        if (vm.getBlockTimestamp() <= engine.lastMarkTime()) {
            vm.warp(uint256(engine.lastMarkTime()) + 1);
        }
        bytes32[] memory ids = _basePythFeedIds();
        MockPyth.FeedUpdate[] memory ticks = new MockPyth.FeedUpdate[](ids.length);
        uint256 previous = engine.lastMarkTime();
        for (uint256 i; i < ids.length; ++i) {
            ticks[i] =
                MockPyth.FeedUpdate(ids[i], MockPyth.MockPrice(int64(uint64(price)), 0, -8, vm.getBlockTimestamp(), previous));
        }
        bytes[] memory data = new bytes[](1);
        data[0] = abi.encode(ticks);
        router.updateMarkPrice(data);
    }

    function _params(
        uint256 takeProfit,
        uint256 stopLoss
    ) internal pure returns (PositionProtectionTypes.PositionProtectionParams memory params) {
        params.takeProfitTriggerPrice = takeProfit;
        params.stopLossTriggerPrice = stopLoss;
        params.closeLimits = PositionProtectionTypes.CloseFinancialLimits({
            maxExecutionNotionalAssets: type(uint256).max,
            maxGrossAccountDebitAssets: type(uint256).max,
            maxActionChargeAssets: type(uint256).max,
            maxExplicitFeesAssets: type(uint256).max
        });
    }

    function _publicBoundedOpenRequest(
        bytes32 clientOrderId
    ) internal view returns (OrderV3Types.OrderRequest memory request) {
        request.clientOrderId = clientOrderId;
        request.side = CfdTypes.Side.LONG;
        request.sizeDelta = POSITION_SIZE;
        request.marginDelta = POSITION_MARGIN_ASSETS;
        request.targetPrice = 1;
        request.isClose = false;
        request.bounds = OrderV3Types.ExecutionBounds({
            submitBy: uint64(vm.getBlockTimestamp() + 60),
            executionWindowSeconds: 60,
            allowedExecutionModes: 1 | 2 | 4,
            expectedConfigHash: router.lifecycleBook().currentExecutionConfigHash(),
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
