# \#98645 [SC-Medium] executeOrderBatch's historical-basket reuse rejects every legitimately skewed basket: multi-order batches revert whole, rolling back already-finalized orders and re-burning their gas on every attempt

**Submitted on Nov 9th 2026 at 08:15:24 UTC by @Matgothmog for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #98645
- **Report Type:** Smart Contract
- **Report severity:** Medium
- **Target:** https://github.com/immunefi-team/audit-comp-plether/tree/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src
- **Impacts:**
  - Griefing (e.g. no profit motive for an attacker, but damage to the users or the protocol)

## Description

# Description

## Brief

`executeOrderBatch` is designed to execute many queued orders while parsing Pyth history **once** —
the second and later orders reuse the basket parsed for the first (project gas tests:
`test_Gas_SharedBasketPaysOnceAndBothOrdersExecute`, 30–300-order batch budgets). The reuse path's
storage-coverage check is over-strict: it demands every Pyth feed have reached the basket's **newest**
component publish time, while a legitimately parsed basket only proves per-feed coverage from its
**oldest** component time. Since the oracle's own policy explicitly admits component publish-time
skew (up to 15 seconds; execution batches use a 5-second divergence limit), any realistic basket is
rejected: the call reverts `PletherOracle__StoredFeedBehind`, the sidecar's graceful-stop catch does
not recognize this error shape, and the **entire batch rolls back — including already-finalized order
items**, re-burning their gas on every retry. The documented one-parse batch flow is therefore
undeliverable in production, while passing the project's test suite because the mock feeds all carry
identical publish times.

## Vulnerability Details

All references are against the in-scope code at commit `a82d4fef`.

1. **The over-strict check.** On reuse, `PletherOracle.updateBatchOrderExecutionPrice`
   (`PletherOracle.sol:262-266`) validates the caller-supplied cache with
   `_requireStoredFeedCoverage(pythFeedIds[i], cache.publishTime)` for every feed. But
   `cache.publishTime` is the basket's `maxPublishTime` over components
   (`_computeBasketPriceFromFeeds`, `PletherOracle.sol:874`), while `cache.oldestPublishTime` is the
   `minPublishTime` (`:875`) — the actual floor of the parse's proven per-feed coverage (each
   component's `previous publish time` proof covers its own feed from below its own publish time).
   Requiring every stored feed to reach the **max** is stronger than the documented reuse rule
   ("...falls within its proven coverage", SECURITY.md "Delayed-order model"), which is bounded by
   the **oldest** component time.

2. **Legitimate skew fails it.** A parsed basket whose components publish within seconds of each
   other (normal for the six FX feeds; policy allows 15 s, batches 5 s) has `minPublishTime <
   maxPublishTime`. The feed that published at `minPublishTime` cannot have a stored update time
   `>= maxPublishTime` at parse time, so the check reverts `PletherOracle__StoredFeedBehind`
   for every reuse attempt.

3. **The failure is not graceful.** The batch's oracle-preparation errors are only contained for the
   exact 68-byte `HistoricalParserFailure` shape (`OrderRouterExecutionSidecar.sol:1019-1027`); every
   other error — including the 100-byte `StoredFeedBehind` — is re-thrown verbatim and bubbles to the
   outer frame. Per the documented batch contract ("a prepared-item failure can stop the batch without
   rolling back already finalized predecessors"), finalized order items are expected to persist;
   instead the whole call reverts and `nextExecuteId()` is rewound — the finalized prefix is lost and
   its gas re-burned on every retry.

4. **Deterministic in production, invisible in tests.** The project's batch gas tests pass because
   `MockPyth.setAllUniquePrices` assigns all feeds the same publish time (the equal-time control in
   the PoC reproduces exactly this). Real Pyth FX components publish at different times, so the reuse
   path fails deterministically whenever a batch contains two or more price-dependent orders.

5. **Impact.** No funds are lost and no user is frozen — keepers can execute orders one-by-one (the
   PoC's workaround test confirms the per-order path works). The damage is (a) the loss of the
   documented batch optimization for its entire real-world domain: every attempt reverts and re-burns
   the full prefix gas (the PoC measures a 2-order batch; the advertised 30–300-order batches would
   re-burn proportionally more per attempt), and (b) queue-clearing liveness depends on per-order
   execution with its much smaller per-tx throughput, degrading the documented bounded-cleanup
   machinery (MED-4's `clearInvalidOrders`, expiry sweeps) under load.

## Recommended Mitigation

On `StoredFeedBehind` (or, better, instead of reverting), fall back to the fresh-parse path the
caller already funded (`_resolveOrderExecutionBasket` with the supplied `pythUpdateData`) — exactly
what happens when the cache is absent. Alternatively, align the check with the documented rule:
require per-feed stored coverage `>= cache.oldestPublishTime` (plus the existing commit-time and
window predicates), which is what the previous-publish proof actually establishes. Additionally,
extend the sidecar's graceful-stop recognition to cover oracle-preparation failures of known typed
shapes so a single item's oracle hiccup cannot roll back a finalized batch prefix.


## Link to Proof of Concept

https://gist.github.com/Matgothmog/4b96ebd9d42659457dc567f42bf51018

## Proof of Concept

# Proof of Concept

Foundry tests against the frozen commit `a82d4fef` (Forge 1.5.1, solc 0.8.35 via-IR).
File: `poc/BatchReuseStoredFeedBehind.t.sol` (drop into `packages/perps/test/perps/regression/` and run
`forge test --root packages/perps --match-contract BatchReuseStoredFeedBehind -vv`).
Full run output: `poc/RUN_OUTPUT.txt` — **3/3 PASS**.

All three tests drive the REAL public entrypoints (`router.commitOrder`-style setup via the harness,
then `router.executeOrderBatch`) with a 2-component basket whose publish times differ by 2 seconds —
well inside the oracle's own 5-second batch divergence policy and 15-second live window.

1. **`test_BatchReuse_ComponentSkew_RevertsWholeBatchAndLosesPrefix`** (the defect):
   two orders committed at increasing commitTimes, one batch call covering both. The first order's
   historical parse succeeds and its item frame **finalizes**; when the second order takes the cache
   -reuse path, `_requireStoredFeedCoverage(feed, cache.publishTime)` reverts
   `PletherOracle__StoredFeedBehind` (the oldest-published feed's stored time cannot reach the
   basket's newest component time). The error is not the recognized 68-byte
   `HistoricalParserFailure` shape, so the sidecar re-throws it and the OUTER frame reverts the
   entire call. Asserted: the call reverts with exactly `StoredFeedBehind`, and
   `router.nextExecuteId() == 1` afterwards — the **already-finalized first item was rolled back**
   (the documented batch contract says finalized predecessors persist).

2. **`test_BatchReuse_EqualComponentTimes_CompletesBatch`** (fixture blind spot):
   identical setup but all components share one publish time — the batch completes BOTH orders.
   This is exactly why the project's own batch gas tests pass: `MockPyth.setAllUniquePrices` gives
   every feed the same publish time.

3. **`test_BatchReuse_ComponentSkew_SingleOrderWorkaroundWorks`** (impact bound):
   the same two orders executed one-per-call complete fine — proving the harm is gas/liveness of the
   batch entrypoint, not fund loss.

Measured consequences in test 1: the full gas of the finalized prefix is burned and re-burned on
every retry (each retry hits the same deterministic revert), and no partial progress is retained.

Deterministic — no fork, no timing races, no privileged actors. The failure triggers on any
component publish-time skew, which is the normal condition for the six FX feeds (EUR/JPY/GBP/CAD/
SEK/CHF publish at different moments — the policy's 15-second divergence window exists precisely
because of this).
