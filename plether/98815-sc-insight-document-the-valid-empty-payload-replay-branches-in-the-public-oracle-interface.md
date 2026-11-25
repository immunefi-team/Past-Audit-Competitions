# \#98815 [SC-Insight] Document the valid empty-payload replay branches in the public oracle interface

**Submitted on Nov 10th 2026 at 11:43:45 UTC by @FestiveSRT for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #98815
- **Report Type:** Smart Contract
- **Report severity:** Insight
- **Target:** https://github.com/immunefi-team/audit-comp-plether/tree/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src
- **Impacts:**

## Description

## Summary

At frozen commit `a82d4fefb02768e5b3bddcc97dee371eee24fea8`, seven `pythUpdateData` parameter descriptions in the public oracle interface and its concrete copy say the payload must be nonempty. The implementation deliberately accepts `[]` in defined replay branches.

A caller, SDK generator, or maintainer that enforces the current public `Nonempty` precondition will reject `pythUpdateData = []` before a valid accepted-basket or frozen-mark replay, or require signed payload acquisition that the supported zero-fee replay path does not need; live historical order execution remains the nonempty control.

This is a present source-maintenance decision, not a hypothetical runtime rewrite. The in-scope Router entrypoints `settleLpEpoch`, `updateMarkPrice`, `executeLiquidation`, and `executeLiquidationBatch` pass their caller-supplied array first to `getUpdateFee` and then to the matching oracle update without enforcing nonemptiness. The single-order execution path similarly quotes with `getOrderExecutionFee` and forwards the same array. An interface-derived wrapper or validation layer should preserve these branches instead of rejecting the payload before the Router and oracle apply their actual policy.

No current external SDK failure is claimed. The benefit is keeping generated validation, fee acquisition, and maintenance of these existing entrypoints aligned with the supported contract behavior.

## Exact discrepancy

The stale descriptions are:

- `interfaces/IPletherOracle.sol:270` — `updateOrderExecutionPrice`;
- `interfaces/IPletherOracle.sol:306` — `updateLiquidationBatchPrice`;
- `interfaces/IPletherOracle.sol:318` — `updateLiquidationPrice`;
- `interfaces/IPletherOracle.sol:331` — the two-argument `updatePrice` overload;
- `interfaces/IPletherOracle.sol:400` — `getUpdateFee`;
- `interfaces/IPletherOracle.sol:416` — `getOrderExecutionFee`; and
- `PletherOracle.sol:217` — the concrete `updateOrderExecutionPrice` copy.

The surrounding prose already exposes the contradiction. `getUpdateFee` says it returns zero for empty replay while its parameter says nonempty blobs. `getOrderExecutionFee` says it returns zero when frozen while its parameter says nonempty payloads. The order-execution function text says frozen policy replays the Engine mark at zero fee while its parameter remains unconditional.

These seven lines express one stale conditional-payload contract, not seven findings.

## Implemented behavior and boundaries

For current-price and liquidation paths, `getUpdateFee([])` returns zero. `_resolveCurrentBasket` then:

1. replays the stored Engine mark when oracle-frozen;
2. otherwise replays the accepted live basket when the array is empty; or
3. parses and accepts a new signed basket when the array is nonempty.

Replay still requires valid state. The accepted basket must exist and satisfy freshness, confidence, component-time, and mark-ordering policy. Frozen replay requires a nonzero Engine mark within the applicable age limit. A zero fee quote alone does not prove those conditions.

Order execution has the narrower exception. `getOrderExecutionFee([])` returns zero only when oracle-frozen, and `_resolveOrderExecutionBasket` uses the stored frozen mark. Outside frozen policy, `_resolveHistoricalOrderBasket` rejects `[]` with `PletherOracle__MissingUpdateData`. The separate internal batch-cache reuse branch remains unchanged.

The two-argument `updatePrice(refundRecipient, pythUpdateData)` overload has no call site under `packages/perps/src` at this pin. Its stale public parameter is part of the same contract, but this report does not claim it is presently consumed.

`ORACLE_SYNCHRONIZATION.md`, nearby function prose, and most concrete implementation comments already describe replay correctly. The mechanism is not presented as newly discovered behavior. The improvement is to stop the public parameter contract from contradicting that documented and implemented behavior at the seven remaining sites.

## Recommendation

For current and liquidation methods, describe `pythUpdateData` as signed Pyth payloads, or empty for accepted-basket or frozen-mark replay. For order execution, state that live historical parsing requires a nonempty signed payload while frozen stored-mark replay permits an empty array; retain the existing batch-cache exception separately. Apply the same distinction to both fee-quote parameters, and state that a zero quote does not establish replay-state validity.

This is a Documentation Improvements Insight. It makes no vulnerability, loss, funds-at-risk, observed external-consumer failure, or guaranteed-acceptance claim. The separately filed newest-versus-earliest timestamp documentation root is excluded.


## Proof of Concept

## Proof of Concept / Conditions under which the Insight is valuable

Use the public input:

```text
pythUpdateData = []
```

Current-price and liquidation replay:

```text
getUpdateFee([])                                  => 0
Router forwards [] and value 0
oracle frozen                                    => stored Engine-mark replay
otherwise accepted basket valid                  => accepted-basket replay
otherwise                                        => revert under the documented policy controls
```

Frozen order execution:

```text
getOrderExecutionFee([])                         => 0
updateOrderExecutionPrice(..., [], request)
policy.oracleFrozen                              => true
_resolveOrderExecutionBasket                     => _frozenBasket
historical parser                                => not called
```

Matched negative control:

```text
live historical order execution + []             => PletherOracle__MissingUpdateData
```

The Insight is valuable where source documentation drives wrapper validation, SDK generation, fee acquisition, or maintenance of the existing Router entrypoints. The correction prevents those consumers from imposing an unconditional nonempty check that rejects a valid replay or fetches unnecessary signed payloads. It does not imply that every empty call succeeds: accepted state, freshness, confidence, component-time, mark-ordering, frozen-mark existence, and the applicable age limit still control.

No attachment or Gist is included; the complete deterministic source comparison and negative control are inline.

