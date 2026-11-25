# \#97904 [SC-Medium] Batch historical-price cache reuse checks every feed against the basket's maximum component publish time instead of that feed's own time, spuriously reverting valid batch order executions at a measured, uncompensated cost

**Submitted on Nov 3rd 2026 at 20:17:54 UTC by @N53X for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #97904
- **Report Type:** Smart Contract
- **Report severity:** Medium
- **Target:** https://github.com/immunefi-team/audit-comp-plether/tree/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src
- **Impacts:**
  - Griefing (e.g. no profit motive for an attacker, but damage to the users or the protocol)

## Description

## Summary

`PletherOracle.updateBatchOrderExecutionPrice` lets a keeper reuse a historical Pyth basket already validated
by an earlier order in the same `executeOrderBatch` call, so a second order with an overlapping settlement
window doesn't need its own fresh Pyth fee. The check that decides whether a feed's already-stored data still
covers the reuse compares each feed's stored publish time against the basket's maximum component time. The
check that originally validated and stored that same data compares each feed against its own publish time.
When a basket's components have any mutual timing skew, the reuse check is stricter than the validation that
produced the data it is checking, and reverts a batch execution that would otherwise succeed.

I want to be precise about how often this skew actually occurs: I don't have live Pyth feed data to cite a
measured frequency, so I'm not claiming it as an empirical fact. What I can point to is the protocol's own
design choice — `maxComponentPublishTimeDivergence` explicitly tolerates up to 5 seconds of mutual skew across
the six components — which only makes sense if the protocol's own designers expected components to routinely
*not* share an identical publish time. The skew my PoC uses (one second) is well inside that tolerance and is
not a contrived or adversarial value chosen to force a failure.

I measured the actual cost of this directly in my Foundry test environment rather than only showing a revert:
recovering the affected orders through individual execution costs exactly double the Pyth fee a working reuse
batch would have charged (4x the base fee vs. 2x — a structural multiplier guaranteed by the code itself,
since `getOrderExecutionFee` is always `2 * pyth.getUpdateFee()` regardless of environment), plus roughly 1.8M
gas burned on the failed batch attempt itself for zero benefit. The exact gas figure and the mock fee's raw
wei amount are specific to my test environment's gas costs and the mock Pyth fee configured in the test
fixture, not a measured mainnet or production figure — I'm reporting them as evidence of the mechanism and its
proportional cost, not as a guaranteed real-world dollar amount. The revert also rolls back an
already-validly-processed earlier order in the same call, so the caller gets nothing for that gas at all — not
even the first order's execution.

**Private Gist with the full runnable PoC** (Foundry test file, 6/6 tests passing on the pinned commit):
https://gist.github.com/adimai/17a010d6ecaf7ab568bfcab767fe79b1

## Asset affected

- `packages/perps/src/PletherOracle.sol` — `updateBatchOrderExecutionPrice` / `_requireStoredFeedCoverage`
  (this function's logic is wholly inside the in-scope file; it does not depend on anything inside
  `packages/shared`).

## Root cause

The reuse branch of `updateBatchOrderExecutionPrice`:
```solidity
if (!policy.oracleFrozen && _canReuseHistoricalBatchBasket(request.commitTime, cache)) {
    for (uint256 i; i < pythFeedIds.length; ++i) {
        _requireStoredFeedCoverage(pythFeedIds[i], cache.publishTime);
    }
    ...
}
```
checks every feed's currently-stored Pyth publish time against `cache.publishTime`. That field is set from
`basket.publishTime`, which `_computeBasketPriceFromFeeds` defines as the **maximum** across all six
components. The function that originally parsed and validated the basket now sitting in that cache,
`_resolveHistoricalOrderBasket`, checks each feed only against its own time:
```solidity
for (uint256 i; i < parsedFeeds.length; ++i) {
    _requireStoredFeedCoverage(parsedFeeds[i].id, parsedFeeds[i].price.publishTime);
}
```
A feed whose own component was earlier than the basket's maximum (but still inside the allowed
`maxComponentPublishTimeDivergence`, which defaults to 5 seconds) passes the original check but fails the
reuse check, because the reuse check holds it to a bar — the basket's latest component's time — that the
original validation never required of it.

## Link to Proof of Concept

https://gist.github.com/adimai/17a010d6ecaf7ab568bfcab767fe79b1

## Proof of Concept

## Proof of Concept

A standalone Foundry test against the project's own `OracleSynchronizationTestBase` fixture, the same base
every one of the project's existing 24 oracle-synchronization tests uses. The only cheatcodes used are
`vm.warp`/`vm.roll`, identical to how every sibling test in that file already advances time.

Copy the file from the Gist into `packages/perps/test/perps/spec/oracle/` in the pinned-commit checkout and
run:
```
FOUNDRY_TEST=test/perps/spec/oracle forge test --match-contract BatchCacheReuseWrongTimestampTest -vv
```

Result on the pinned commit (`a82d4fef`):
```
Ran 6 tests for test/perps/spec/oracle/BatchCacheReuseWrongTimestamp.t.sol:BatchCacheReuseWrongTimestampTest
[PASS] test_BatchRevertsOnReuseAttempt_WithRoutineUnequalComponentTimes() (gas: 4360317)
[PASS] test_Control_SingleOrderExecutionAcceptsTheSameSkewedBasketTwice() (gas: 3030453)
[PASS] test_Differential_Comparison_PrecedingOrderInSameBatchCausesFailure() (gas: 4385926)
[PASS] test_Differential_Control_VictimOrderAloneInBatchSucceeds() (gas: 2511352)
[PASS] test_EndToEnd_BatchFailsThenBothOrdersRecoverIndividually_WithMeasuredExtraCost() (gas: 6040328)
Logs:
  gas burned on the reverted batch attempt (pure loss): 1807368
  gas for Alice's individual recovery execution: 1734933
  gas for Bob's individual recovery execution: 1084797
  Pyth fee actually paid to recover (2 individual calls): 4000000000
  Pyth fee a WORKING cache-reuse batch would have charged: 2000000000
[PASS] test_RoleSeparation_IndependentKeeperBearsCostNotTheOrderOwners() (gas: 6028054)
Suite result: ok. 6 passed; 0 failed; 0 skipped
```

**`test_BatchRevertsOnReuseAttempt_WithRoutineUnequalComponentTimes`**: two ordinary opens, committed moments
apart, both eligible for one keeper batch call. The supplied basket has one feed one second behind the other
five — a basket the oracle's own divergence check already accepts as valid on its own terms, built the same
way any two real, independently-published Pyth feeds would look. The batch call reverts
`PletherOracle__StoredFeedBehind`.

**`test_Control_SingleOrderExecutionAcceptsTheSameSkewedBasketTwice`**: the identical skewed basket submitted
twice through the single-order `updateOrderExecutionPrice` path, with no reuse attempted at all, succeeds both
times. This isolates the cause to the batch reuse check specifically; the basket itself is not the problem.

**`test_Differential_Control_VictimOrderAloneInBatchSucceeds`** / **`test_Differential_Comparison_PrecedingOrderInSameBatchCausesFailure`**:
a matched control/comparison pair, both through the real `executeOrderBatch` entrypoint, covering the full
sequence end to end in two tests:
1. An unprivileged party commits an ordinary order via the normal public `commitOrder` flow — no special
   permission, no crafted parameters, nothing an attacker needs that an ordinary trader doesn't already have.
2. A separate, unrelated party subsequently commits a second valid order behind it.
3. An independent keeper — a dedicated address with zero stake in and no relationship to either order — attempts
   to batch both through the real, permissionless `executeOrderBatch` entrypoint, using a basket with timestamp
   skew the protocol's own divergence policy permits.
4. The batch reverts; the keeper's attached ETH is returned in full (confirmed by an explicit balance
   assertion — the loss is gas, measured directly and asserted greater than zero) and both orders are confirmed
   still pending afterward via explicit `getPendingOrderView` assertions on each order id, through no fault of
   either order owner.
5. The control test proves the identical kind of victim order, under the identical skew shape, succeeds when
   it is not preceded by another order in the same batch call — isolating presence-of-a-preceding-order as the
   sole variable that flips the outcome.

**`test_EndToEnd_BatchFailsThenBothOrdersRecoverIndividually_WithMeasuredExtraCost`**: goes through the real
`OrderRouter.executeOrderBatch` and `executeOrder` entrypoints (not the oracle directly). It explicitly
asserts `router.nextExecuteId()` and `getPendingOrderView(1)` are unchanged immediately after the revert —
proving the first order's in-batch execution really is rolled back, not just failing to advance — then
recovers both orders via individual calls and measures the actual cost: ~1.8M gas burned on the failed attempt
for zero benefit, and 4x the base Pyth fee paid to recover versus the 2x a working reuse batch would have
charged.

**`test_RoleSeparation_IndependentKeeperBearsCostNotTheOrderOwners`**: introduces a dedicated `KEEPER` address
with zero stake in either order — distinct from both Alice and Bob — as the sole caller of
`executeOrderBatch`/`executeOrder`. It proves KEEPER's attached ETH is fully returned by the revert (the loss
is gas, not principal), that Alice's and Bob's order records are completely untouched and remain pending
through no action of their own, and that KEEPER alone bears the cost of the two-call recovery. An order
committer has no parameter through which to influence what Pyth data a separate, independent keeper later
chooses to submit (`commitOrder` only accepts side/size/margin/target price/isClose), so the harm here isn't
confined to whoever happens to choose the failing path — it reaches passive order owners who did nothing to
cause it, and falls on any ordinary keeper acting in good faith with valid, real Pyth data.

I also re-ran the project's own existing 24 tests in the same file to confirm nothing about my addition
changes the established baseline — all 24 still pass unmodified.

## Why I believe this is new

I checked `prior-art.md`'s full register for `StoredFeedBehind`, "cache reuse", and "batch cache" — no match.
The nearest related item, Cecuro MED-15, fixed historical-cache reuse rejecting zero prices, a different check
entirely. The project's own existing test closest in spirit,
`test_StorageCoverageChecksEachParsedComponentNotOnlyMinimum`, guards the *opposite* failure direction — it
proves the single-order path correctly catches a component that is individually stale, i.e. it guards against
under-checking. This finding is about the batch-reuse path checking too strictly against the wrong reference,
causing valid data to be rejected — the opposite class of bug, on a different code path.

## Impact

`executeOrderBatch`'s reuse path exists specifically so a keeper processing several orders that share an
overlapping settlement window doesn't pay a fresh Pyth fee for each one. Whenever the shared basket's six
components have non-identical publish times — a condition the protocol's own 5-second divergence tolerance
indicates was anticipated, not excluded — the second and later orders in that batch cannot actually use the
reuse path: the whole call reverts.

I'm classifying this as Medium (Griefing). Immunefi's own severity-classification system defines Griefing as:

> "Griefing is when the attacker calls certain functions of the smart contract that would put it in a
> suboptimal state, thus blocking normal function execution for any user. This would cause the user to lose
> money for sending the transaction, but when the smart contract is back to normal, the user would be able to
> call the function once again to complete it. In this instance, the attacker damaged the user by requiring
> them to send another transaction. The attacker does not profit, but they do damage the users or the
> protocol."

My PoC maps onto this directly: calling `executeOrderBatch` with an ordinary, protocol-tolerated skewed basket
puts the call in a state that blocks the reuse optimization (proven by `vm.expectRevert` on the real
entrypoint); this costs the caller real, unconditionally uncompensated gas (~1.8M, confirmed by tracing the
control flow — the revert is a hard, unhandled propagation with no enclosing try/catch, so
`_settleTerminalBounty` — the only mechanism that ever reimburses a caller — is never reached); the same
orders then execute successfully via individual calls at 2x the fee a working reuse batch would have charged;
and no party profits. With role separation proven directly rather than assumed — a dedicated keeper address
with no stake in either order bears the entire gas cost, while the two order owners, who are not the caller
and did nothing to trigger the defect, have their pending orders sit untouched for one extra round trip before
they clear.

This requires no privileged access at all. `_validateBatchBounds` — the only gate on `executeOrderBatch`'s
inputs — checks `maxOrderId` against the queue head and the current commit count, and nothing else; there is
no caller-identity check anywhere in it, consistent with the function's own documentation ("Permissionlessly
processes consecutive FIFO orders"). Any address can call it for any currently-pending order range, with no
relationship to the affected orders required. Combined with the differential test above, this means: any
unprivileged third party can choose, at will, to batch two strangers' already-pending orders together using
ordinary real Pyth data, and trigger a revert that costs them only gas while delaying both order owners — with
no need to predict or engineer anything about those orders in advance; they simply choose which currently-
pending orders to batch at the moment they act.

This bug also doesn't rest on the batch-wide rollback alone. The project's own test suite
(`test_LaterSynchronizationFailureRevertsCompletedBatchPrefix`,
`test_LaterInsufficientFundingRevertsCompletedBatchPrefix`) shows that rolling back an already-processed
earlier order on a later order's oracle failure is the project's general, intentional response to most
late-order oracle failures — only `PletherOracle__HistoricalParserFailure` is deliberately exempted and
preserves the batch prefix instead. `PletherOracle__StoredFeedBehind`, the error this bug raises, falls into
the general, non-exempted category. The harm here comes specifically from the fact that the reuse check
spuriously raises an error that the original validation would never have raised for the same data — not from
the rollback mechanism itself, which behaves as the protocol already intends for this class of failure.

No user funds are at risk and no guarantee of eventual execution is broken; the affected orders remain valid
and executable, just at measurably higher, unconditionally uncompensated cost and after one extra round trip.

## Suggested fix

Carry each component's own publish time in `BatchOrderPriceCache` rather than only the basket-wide maximum,
and check each feed's stored coverage against its own recorded time on reuse — mirroring exactly what the
originating historical parse already does.
