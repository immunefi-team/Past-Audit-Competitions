# \#98437 [SC-Medium] Cache reuse can demand stronger feed coverage than its populating parse proved

**Submitted on Nov 8th 2026 at 03:25:05 UTC by @FestiveSRT for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #98437
- **Report Type:** Smart Contract
- **Report severity:** Medium
- **Target:** https://github.com/immunefi-team/audit-comp-plether/tree/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src
- **Impacts:**

## Description

### Classification and boundary

This is one **Code Optimizations and Enhancements Insight** about the historical
basket cache at frozen competition commit
`a82d4fefb02768e5b3bddcc97dee371eee24fea8`. It is not a vulnerability-severity
claim. I do not claim theft, freezing, attacker-forced keeper loss, a current
Arbitrum One deployment, or that unequal component ticks are routine.

The relevant `PletherOracle.sol` and `OrderRouterExecutionSidecar.sol` blobs are
identical between the v2.0.0 release source and the frozen competition source.
The Insight concerns the intended batch optimization in the in-scope
`packages/perps/src` code.

### Proof mismatch in cache construction and reuse

For the first compatible order, `_resolveHistoricalOrderBasket` parses each
feed's unique post-commit tick, updates Pyth storage from the same payload, and
proves separately that:

```text
storedPublishTime[feed_i] >= parsedPublishTime[feed_i]
```

The resulting `BatchOrderPriceCache` does not retain those individual times. It
retains only their minimum and maximum as `oldestPublishTime` and
`publishTime`.

For a later compatible order, `updateBatchOrderExecutionPrice` takes the reuse
branch and instead requires every feed to cover the aggregate maximum:

```text
storedPublishTime[feed_i] >= cache.publishTime
                              = max(parsedPublishTime[])
```

The second predicate is stronger than the proof established while populating
the cache. If one valid component was parsed earlier than the newest component
and storage has not independently advanced it, reuse reverts
`PletherOracle__StoredFeedBehind` even though the same payload already produced
and synchronized the accepted basket.

The Router independently decides that the cache is reusable, forwards zero
fresh-resolution fee, and re-raises this error. It therefore cannot take the
fresh historical-resolution fallback described by the guide. The complete
batch transaction reverts, including an already executed prefix.

Relevant source:

- [`PletherOracle.updateBatchOrderExecutionPrice`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/PletherOracle.sol#L245-L299)
- [`PletherOracle._resolveHistoricalOrderBasket`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/PletherOracle.sol#L596-L643)
- [`PletherOracle._canReuseHistoricalBatchBasket`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/PletherOracle.sol#L974-L991)
- [`OrderRouterExecutionSidecar` reuse, funding, and catch boundary](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/OrderRouterExecutionSidecar.sol#L982-L1037)

### Conditions and system-level value

The Insight is valuable when all of the following hold:

1. two or more queued orders are compatible with the same in-memory historical
   cache; two orders committed in the same second are a sufficient example;
2. the signed six-feed historical basket contains at least two distinct
   component publish times within the configured five-second divergence limit;
3. at least one older component remains stored at its own parsed time rather
   than independently advancing to the newest component time; and
4. a keeper uses `executeOrderBatch` for the compatible range.

With equal component times, the retained control executes two orders in one
batch with one unique historical parse. With valid unequal times, the cache
branch can revert and the public fallback is to execute the two orders singly.
That fallback preserves order liveness, but requires two transactions, two
historical parse/storage-update cycles, and four Pyth-fee units rather than one
transaction, one parse/update cycle, and two Pyth-fee units. These are exact
call/fee-count differences, not a production gas-price or dollar estimate.

The retained authentic corpus is adverse occurrence evidence: 0 of 140
observed six-feed payload sets had unequal current component times, and 0 of
140 had unequal previous component times. This sparse corpus does not prove
impossibility, but it means I do not claim the condition is frequent or
currently occurring.

### Documentation overlap and exact novelty boundary

`ORACLE_SYNCHRONIZATION.md` already says that failed cache coverage or
`StoredFeedBehind` reverts the full transaction. I am not presenting that error
class or batch atomicity as novel. The narrower contribution is that a cache
created by a successful valid parse can fail its own reuse proof because reuse
demands a stronger per-feed time than construction established.

The same guide says that when reuse cannot satisfy cache coverage, "the Router
requests a new historical resolution." In this state it does not: both Router
and Oracle select reuse, the Router forwards zero fee, and the Oracle reverts.

### Recommendation

Retain each authenticated component publish time in the cache and check each
feed against its own retained time during reuse. Alternatively, make a coverage
miss enter a funded fresh-resolution path instead of re-raising it from the
zero-fee reuse call. Add an unequal-component-time regression beside the
equal-time reuse test, and update the synchronization guide to describe the
implemented fallback exactly.

### Non-claims

The first signed tick for each feed is not controlled by an ordinary order
creator. A later order cannot enter a keeper transaction after that
transaction's `maxOrderId` is fixed, the deterministic failure can be detected
by exact simulation, and both orders remain permissionlessly executable one at
a time. This report therefore selects no vulnerability impact or severity.


## Proof of Concept

## Proof of Concept

## Conditions under which this Insight is valuable

Runnable code is not required for an Insight. The following deterministic
source trace demonstrates the condition and the optimization consequence; it
is not presented as an attacker PoC.

Let two orders share commit time `C`. A valid six-feed payload supplies five
first qualifying ticks at `C+1` and one at `C+3`. The two-second dispersion is
within the configured five-second historical divergence limit.

The first order performs:

```text
parsed times:        [C+1, C+3, C+1, C+1, C+1, C+1]
stored after update: [C+1, C+3, C+1, C+1, C+1, C+1]
construction proof: stored_i >= parsed_i                 => true for all feeds
cache oldest time:   C+1
cache newest time:   C+3
```

The second order is cache-compatible because `C < C+1`, `C < C+3`, and
`C+3 <= C + orderSettlementWindow`. Reuse then applies this different proof:

```text
required for every feed: stored_i >= cache newest time C+3
feed 0:                  C+1 >= C+3                       => false
result:                  StoredFeedBehind(feed0, C+1, C+3)
```

`OrderRouterExecutionSidecar` re-raises that error, so the entire batch rolls
back. Repeating the exact call produces the same result. Executing each order
through `executeOrder` with the same payload succeeds because each call parses
and synchronizes the basket rather than reusing the aggregate cache.

The retained internal fixture at the frozen commit measured exactly these
controls:

```text
unequal-tick batch: exact StoredFeedBehind revert; prefix rolled back
identical retry:    same exact revert
unequal-tick singles: both orders executed; two unique parses
equal-tick batch:  both orders executed; one unique parse
```

This demonstration does not show a current production occurrence. In the
retained 140-observation authentic payload corpus, every observed current set
and every observed previous set had equal component publish times. The Insight
is valuable as a correctness and efficiency guard for a code-admitted valid
basket state, not as a claim that the state is common.

