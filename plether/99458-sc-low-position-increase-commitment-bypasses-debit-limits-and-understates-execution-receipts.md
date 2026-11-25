# \#99458 [SC-Low] Position-increase commitment bypasses debit limits and understates execution receipts

**Submitted on Nov 13th 2026 at 18:35:39 UTC by @alexhunter for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #99458
- **Report Type:** Smart Contract
- **Report severity:** Low
- **Target:** https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/MarginClearinghouse.sol
- **Impacts:**

## Description

## Brief/Intro

Committing an increase to an existing position can collect accrued carry without recording that collection in the order’s commitment effects. Consequently, `maxGrossAccountDebitAssets` and `maxActionChargeAssets` can be exceeded during commitment, while execution subsequently succeeds and produces an authenticated receipt that understates the total debit. In production, this would undermine the caller’s explicit financial limits and the accuracy of per-order accounting.

## Vulnerability Details

This finding applies to commit `a82d4fefb02768e5b3bddcc97dee371eee24fea8`.

The affected path is a public V3 increase order with `isClose == false`, submitted by an account that already has a position with collectible accrued carry.

`OrderCommitHandler._createPendingOrder()` obtains commitment effects from `_reserveExecutionBounty()`, records those effects, and then reserves committed margin:

```solidity
CfdEnginePlanTypes.CloseCommitment memory effects =
    _reserveExecutionBounty(account, order.sizeDelta, executionBountyAssets, order.isClose);

clearinghouse.recordBountyReservation(
    account, IMarginClearinghouse.BountyKind.Order, orderId, executionBountyAssets
);

_onOrderCommitted(order, effects);
_reserveCommittedMargin(account, orderId, order.isClose, order.marginDelta);
```

For an increase, `OrderReservationAccounting._reserveExecutionBounty()` sets `effects.bountyFromFreeAssets` and calls `clearinghouse.lockReservedSettlement(account, bounty)`. It does not populate `effects.carryCollectedAssets` or the before/after settlement balances.

However, `MarginClearinghouse.lockReservedSettlement()` checkpoints carry before locking the bounty:

```solidity
_checkpointCarryBeforeMarginChange(account);
_lockMargin(account, IMarginClearinghouse.MarginBucket.ReservedSettlement, amountAssets);
```

That checkpoint reaches `engine.realizeCarryBeforeMarginChange(account)`. For an existing position, it can collect accrued carry from available backing and transfer it to the pool. Therefore, the reservation operation can decrease the trader’s settlement balance even though the returned commitment effects report zero carry collection.

`OrderLifecycleBook.recordCommitment()` relies on those reported effects. Its commitment-time gross debit calculation is:

```solidity
uint256 gross = effects.carryCollectedAssets + pending.executionBountyAssets;
```

Because `effects.carryCollectedAssets` remains zero, the gross-debit check accounts only for the bounty. The action-charge check similarly compares zero carry against `maxActionChargeAssets`.

The omission persists through execution. `CfdOrderPolicyEvaluator.assessCommittedOrder()` includes commitment economics only for closes:

```solidity
assessment = _assessSnapshot(context, order, pending.bounds, planner, snapshot);
if (order.isClose) {
    _includeCommitment(assessment, pending.commitment, pending.bounds, pending.executionBountyAssets);
}
```

An increase’s execution snapshot already reflects the earlier carry collection. Execution therefore assesses the subsequent trade charges without recovering the omitted commitment debit. The final receipt copies this incomplete economic assessment.

The execution balance checks can still pass because they compare balances immediately before and after execution. The omitted collection occurred in the earlier commitment transaction.

The committed-margin reservation also checkpoints carry and occurs after commitment effects are recorded. A correction should capture carry collection across the complete reservation sequence, enforce the submitted limits against those effects, and include the collection exactly once in the eventual assessment and receipt.

**Reproduction**

The local Foundry reproduction uses unmodified production contracts, including the production Router, with the published risk and fee parameters and mocked USDG/Pyth dependencies.

1. Fund a trader with 20,000 USDG.
2. Open a 100,000-unit LONG at price `1e8`, allocating 10,000 USDG margin.
3. Advance 28 days and refresh the unchanged mark during a normal trading window.
4. Submit a 1,000-unit increase with 100 USDG additional margin.
5. Set both `maxGrossAccountDebitAssets` and `maxActionChargeAssets` to `10e6`, representing 10 USDG.
6. Commit and execute the increase through the public V3 entry points.

The passing test, `test_ReleaseRiskSettingsCarryStillBypassesCommitmentCeiling`, observes:

| Measurement | USDG |
|---|---:|
| Submitted gross-debit ceiling | 10.000000 |
| Submitted action-charge ceiling | 10.000000 |
| Carry collected during commitment | 31.134047 |
| Total settlement debit through execution | 32.636972 |
| Gross debit reported by the authenticated receipt | 1.502925 |

The receipt understates the total debit by exactly **31.134047 USDG**, matching the omitted commitment collection.

The debit is measured from total internal settlement balance. The 100 USDG committed-margin reservation merely reclassifies custody and is excluded from this measurement.

Expected behavior is an atomic commitment revert when the collected carry exceeds the submitted limits. Actual behavior is successful commitment and execution, followed by an authenticated receipt reporting a debit below the caller’s ceiling.

## Impact Details

**Selected impact: Security Best practices — Insight.**

The demonstrated consequences are:

- The caller’s gross-debit and action-charge limits fail to cover carry collected during position-increase commitment.
- The authenticated receipt understates the complete operation’s settlement debit.
- Consumers relying on `receipt.economics.grossAccountDebitAssets` receive an incomplete per-order cost summary.

The reproduced carry amount was already owed by the trader and was transferred to the intended pool. The evidence therefore establishes a financial-bound and reporting defect, without demonstrating additional financial loss from an incorrect carry calculation.

The reproduction involves the submitting account’s own position. It establishes no third-party theft, freezing of funds, protocol insolvency, or loss of unclaimed yield. The reported amounts are a verified example, rather than an estimate of maximum possible losses.

## References

All code references are pinned to the competition commit.

- OrderReservationAccounting.sol — increase bounty reservation - https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/router/OrderReservationAccounting.sol#L108
- OrderCommitHandler.sol — commitment and reservation sequence - https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/router/OrderCommitHandler.sol#L21
- MarginClearinghouse.sol — bounty reservation checkpoints carry - https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/MarginClearinghouse.sol#L818
- MarginClearinghouse.sol — committed-margin reservation checkpoints carry - https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/MarginClearinghouse.sol#L620
- OrderLifecycleBook.sol — commitment-bound checks - https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/OrderLifecycleBook.sol#L327
- CfdOrderPolicyEvaluator.sol — commitment inclusion restricted to closes - https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/CfdOrderPolicyEvaluator.sol#L64
- OrderV3Types.sol — execution-bound definitions - https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/OrderV3Types.sol#L131
- DEPLOYMENT.md — initial release parameters - https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/DEPLOYMENT.md#L123

## Proof of Concept

## Proof of Concept

### 1. Overview

This PoC demonstrates that increasing an existing position can collect accrued carry during order commitment without including the collection in the order's commitment effects.

As a result, the commitment succeeds even when the actual debit exceeds both `maxGrossAccountDebitAssets` and `maxActionChargeAssets`. The subsequent execution also succeeds and produces an authenticated receipt that understates the total settlement debit.

The reproduction uses the production contracts at commit `a82d4fefb02768e5b3bddcc97dee371eee24fea8`, including the production Router, with published risk and fee parameters and mocked USDG/Pyth dependencies.

### 2. Preconditions

The following conditions are established in the local Foundry test environment:

- A trader has an existing LONG position with collectible accrued carry.
- The trader has sufficient settlement balance to cover the carry collection and requested margin.
- The new order increases the existing position (`isClose == false`).
- Both execution bounds, `maxGrossAccountDebitAssets` and `maxActionChargeAssets`, are set to 10 USDG.
- The public V3 order commitment and execution paths are used.

### 3. Steps to Reproduce

**Step 1: Initialize the trader's account**

Fund the trader with 20,000 USDG.

Open an existing LONG position using:

- Position size: 100,000 units
- Opening price: `1e8`
- Allocated position margin: 10,000 USDG

Confirm that the position is active.

**Step 2: Accrue carry**

Advance the blockchain timestamp by 28 days.

Refresh the oracle mark at the unchanged price during a valid trading window.

This establishes collectible accrued carry on the existing position.

**Step 3: Prepare the position-increase order**

Construct a public V3 increase order with:

- `isClose = false`
- `sizeDelta = 1,000` units
- `marginDelta = 100e6` (100 USDG)
- `maxGrossAccountDebitAssets = 10e6`
- `maxActionChargeAssets = 10e6`

Both limits represent a maximum of 10 USDG.

**Step 4: Commit the increase order**

Submit the order using the public V3 commitment entry point.

The Router calls `OrderCommitHandler._createPendingOrder()`, which first invokes `_reserveExecutionBounty()`.

For an increase, `OrderReservationAccounting._reserveExecutionBounty()` calls:

`clearinghouse.lockReservedSettlement(account, bounty)`

Internally, `MarginClearinghouse.lockReservedSettlement()` invokes:

`_checkpointCarryBeforeMarginChange(account)`

This checkpoints and realizes accrued carry before locking the bounty.

In the reproduction, **31.134047 USDG of accrued carry is collected from the trader during commitment**.

However, `_reserveExecutionBounty()` returns commitment effects with `carryCollectedAssets = 0`, without recording the resulting settlement-balance change.

**Step 5: Observe the incorrect commitment-bound validation**

`OrderLifecycleBook.recordCommitment()` calculates the commitment-time gross debit using:

`gross = effects.carryCollectedAssets + pending.executionBountyAssets`

Because the collected carry is omitted, the calculation excludes the actual 31.134047 USDG collection.

The action-charge validation similarly checks the zero-valued `effects.carryCollectedAssets` against `maxActionChargeAssets`.

Consequently, neither bound rejects the order based on the carry already collected.

**Observed behavior:** The increase order is successfully committed despite its 10 USDG action-charge and gross-debit limits being insufficient to cover the 31.134047 USDG collection.

The subsequent committed-margin reservation also checkpoints carry, but occurs after the commitment effects have already been recorded.

**Step 6: Execute the committed increase**

Execute the pending increase through the public V3 execution path with a valid oracle update.

`CfdOrderPolicyEvaluator.assessCommittedOrder()` evaluates the execution snapshot but incorporates stored commitment economics only for close orders.

Because this order is an increase (`isClose == false`), the earlier commitment-time carry collection is not included in the execution assessment.

The execution succeeds, and the finalized authenticated receipt reports an incomplete gross debit.

**Step 7: Compare actual settlement debit against the receipt**

The supplied Foundry reproduction produces the following results:

| Measurement                                     | Amount (USDG) |
| ----------------------------------------------- | ------------- |
| Submitted gross-debit limit                     | 10.000000     |
| Submitted action-charge limit                   | 10.000000     |
| Carry collected during commitment               | 31.134047     |
| Actual total settlement debit through execution | 32.636972     |
| Gross debit reported by authenticated receipt   | 1.502925      |
| Omitted debit                                   | 31.134047     |

The discrepancy is:

**32.636972 − 1.502925 = 31.134047 USDG**

The receipt understates the total settlement debit by exactly the amount of carry collected during commitment.

The actual total debit is **22.636972 USDG above the submitted gross-debit ceiling**.

The debit measurement uses the trader's total internal settlement balance across custody buckets. The additional 100 USDG committed-margin reservation only reclassifies funds and is not counted as a settlement debit.

### 4. Expected Behavior

When accrued carry collection during commitment exceeds the submitted debit or action-charge limits, the transaction should revert atomically.

In this example, collecting 31.134047 USDG of carry with both limits set to 10 USDG should cause commitment to revert.

Any successful order should have its complete commitment and execution economics represented accurately in its final receipt.

### 5. Actual Behavior

The order commits successfully even though collectible carry exceeds the submitted limits.

The increase subsequently executes successfully, and its authenticated receipt reports a gross debit of only 1.502925 USDG despite the trader's actual total settlement balance decreasing by 32.636972 USDG across commitment and execution.

The omitted 31.134047 USDG was collected during the earlier commitment transaction and is absent from the recorded commitment effects and final execution assessment.

### 6. Root Cause

The discrepancy results from three connected accounting behaviors:

1. `OrderReservationAccounting._reserveExecutionBounty()` does not capture carry collected by `MarginClearinghouse.lockReservedSettlement()`.
2. `OrderLifecycleBook.recordCommitment()` validates submitted financial limits using incomplete commitment effects.
3. `CfdOrderPolicyEvaluator.assessCommittedOrder()` includes stored commitment economics for close orders, but not increases.

Consequently, the debit-limit checks and authenticated receipt do not account for all settlement changes caused by the increase operation.

### 7. Reproduction Evidence

The supplied local Foundry reproduction identifies the passing test:

`test_ReleaseRiskSettingsCarryStillBypassesCommitmentCeiling`

The test exercises the production Router and reports:

- Successful position-increase commitment with 10 USDG debit and action-charge limits.
- Collection of 31.134047 USDG of carry during commitment.
- Successful execution of the increase.
- Total settlement debit of 32.636972 USDG.
- Authenticated receipt gross debit of 1.502925 USDG.
- Exact accounting discrepancy of 31.134047 USDG.

These results demonstrate a commitment-bound enforcement and receipt-accounting defect.

They do not establish theft, protocol insolvency, frozen funds, or an additional incorrect carry charge. The collected carry was already owed by the trader.

### 8. Relevant Source Code

All references are pinned to the competition commit.

- [OrderReservationAccounting.sol, lines 108–127](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/router/OrderReservationAccounting.sol#L108-L127) — Bounty reservation omits carry collection.
- [MarginClearinghouse.sol, lines 818–823](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/MarginClearinghouse.sol#L818-L823) — Reservation checkpoints accrued carry.
- [OrderCommitHandler.sol, lines 21–33](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/router/OrderCommitHandler.sol#L21-L33) — Commitment-effects recording precedes margin reservation.
- [OrderLifecycleBook.sol, lines 327–359](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/OrderLifecycleBook.sol#L327-L359) — Commitment-bound calculations.
- [CfdOrderPolicyEvaluator.sol, lines 64–67](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/CfdOrderPolicyEvaluator.sol#L64-L67) — Commitment economics included only for close orders.

### 9. Conclusion

The reproduction demonstrates that a position-increase order can successfully collect 31.134047 USDG of accrued carry despite both submitted financial limits being set to 10 USDG.

Execution subsequently succeeds, while the authenticated receipt reports 1.502925 USDG instead of the actual total settlement debit of 32.636972 USDG.

The observed result confirms that position-increase commitment effects omit previously collected carry, permitting incorrect financial-bound validation and incomplete receipt accounting.
