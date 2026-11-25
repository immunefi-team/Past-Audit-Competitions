# \#97603 [SC-Medium] executeOrderBatch reverts as a whole whenever Pyth component publish times differ, because batch cache reuse checks every stored feed against the newest component time

**Submitted on Nov 2nd 2026 at 14:12:36 UTC by @Griffinf for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #97603
- **Report Type:** Smart Contract
- **Report severity:** Medium
- **Target:** https://github.com/immunefi-team/audit-comp-plether/tree/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src
- **Impacts:**
  - Contract fails to deliver promised returns, but doesn't lose value

## Description

Severity claimed: Low.

The Router's batch execution path reuses a proven historical basket for later orders in the same batch when every component time follows the later order's commit (`OrderRouterExecutionSidecar.sol:997`, `:1074-1093`). On reuse, `PletherOracle.updateBatchOrderExecutionPrice` re-proves Pyth storage coverage by requiring every stored feed to be at least `cache.publishTime`, which is the basket's newest component time (`PletherOracle.sol:264-266`). The fresh historical path, by contrast, checks each stored feed against its own parsed component time (`PletherOracle.sol:624-626`). Plether's live paths parse without writing Pyth storage, so the stored time of a quieter feed is typically the time the earlier historical update wrote for it, which is older than the newest component. With real six-feed data, where components rarely share a publish time, the reuse branch reverts `StoredFeedBehind`. The revert is not caught by the Router (`:1019-1027` absorbs only `HistoricalParserFailure`), so the entire batch reverts and all earlier progress is rolled back. The same orders execute one at a time with the same payload.

Precondition, stated plainly: two or more orders committed before the oldest component tick that follows the first commit, and a payload whose components have unequal publish times. An attacker can arrange the first condition by committing a small order right after a victim's; the second is the normal shape of Pyth FX data. The documented mitigation is that keepers fall back to single execution, which is why I claim Low rather than Medium.

The expected pushback, answered: `ORACLE_SYNCHRONIZATION.md` lists "failed cache coverage" as a whole-transaction revert. It presents that as a defence against a bad cache, not as the behaviour of the honest happy path on ordinary data. The repository's own `test_BatchReusesBasketWhenEveryComponentFollowsBothCommits` passes only because it first pushes newer Pyth storage by hand for every feed, which a production keeper does not do.

Asset: `packages/perps/src/PletherOracle.sol` and `packages/perps/src/OrderRouterExecutionSidecar.sol` at `a82d4fefb02768e5b3bddcc97dee371eee24fea8`.

## Relationship to published known issues

- Program known-issues entry "Cecuro MED-7 (partly fixed): a caller can still pick an older valid signed price per feed" concerns live-basket component selection. This report concerns the batch cache coverage check on the historical path and does not depend on which tick the caller picks.
- Not in `SECURITY.md` known limitations, the Cecuro AI audit, or the 2026-10-15 scan. Cecuro MED-7/18/20/21/23 and LOW-36/68 covered the live-basket and historical paths rewritten in #120; none concerns the batch cache coverage check.
- The known "historical price availability leaves the order pending" issue is about a missing qualifying tick, not about a coverage check failing on a basket the oracle has already proven.

## Root cause

Reuse coverage uses one timestamp for all feeds (`PletherOracle.sol:262-266`):

```solidity
if (!policy.oracleFrozen && _canReuseHistoricalBatchBasket(request.commitTime, cache)) {
    // Cache contents are caller-supplied; independently prove storage coverage, not price authenticity.
    for (uint256 i; i < pythFeedIds.length; ++i) {
        _requireStoredFeedCoverage(pythFeedIds[i], cache.publishTime);   // newest component time for every feed
    }
```

The fresh path uses each feed's own time (`PletherOracle.sol:624-626`), so the first order in the batch passes and the second fails on exactly the same data. `BatchOrderPriceCache` carries only `publishTime` and `oldestPublishTime`, not per-feed times, so the reuse branch has no way to make the per-feed check.

## Vulnerability details

Paths relative to the repository root at `a82d4fef`.

**Trigger.** Order 1 and order 2 are committed at `C` and `C+1`. A payload has feed 1 at `C+4` and feed 2 at `C+5`. The first item resolves historically: `updatePriceFeeds` writes feed 1 at `C+4`, feed 2 at `C+5`; per-feed coverage passes. The Router decides the basket is reusable for order 2 (every component follows `C+1`). The reuse branch requires feed 1's stored time to be at least `C+5`; it is `C+4`; `StoredFeedBehind(feed 1, C+4, C+5)` reverts the whole batch.

**Not absorbed.** The Router's try/catch around the oracle call treats only `HistoricalParserFailure` as a retryable item outcome; any other revert propagates (`OrderRouterExecutionSidecar.sol:1019-1027`).

**Workaround exists.** The same two orders executed with `executeOrder` one at a time, same payload, both fill.

## Impact

- Keeper batch execution fails on ordinary data whenever two batched orders qualify for cache reuse; earlier items in the batch are rolled back and the keeper pays the gas and Pyth fees for nothing.
- A griefer can make a specific victim's order always sit in a reuse-eligible pair by committing a small order immediately after it, forcing keepers to single-execute.
- No funds are lost; queue progress is delayed until keepers fall back to single execution.

## Recommended fix

1. Carry per-feed component times (or the minimum stored-coverage time actually proven) in `BatchOrderPriceCache` and check each feed against its own time in the reuse branch.
2. Alternatively, treat a failed coverage check as "not reusable" and fall through to a fresh historical parse instead of reverting.
3. Make the existing batch-reuse test use a payload with unequal component times and no manual Pyth storage push.

## Link to Proof of Concept

https://gist.github.com/aussinfosec/52e4eb169a4a878634879a8e98a304f1

## Proof of Concept

Gist (secret): https://gist.github.com/aussinfosec/52e4eb169a4a878634879a8e98a304f1

```bash
git clone https://github.com/immunefi-team/audit-comp-plether.git && cd audit-comp-plether
git checkout a82d4fefb02768e5b3bddcc97dee371eee24fea8 && git submodule update --init --recursive
mkdir -p packages/perps/test/perps/poc   # copy PoC_B_BatchReuseCoverage.t.sol from the gist here
forge test --root packages/perps --match-path test/perps/poc/PoC_B_BatchReuseCoverage.t.sol -vv
```

Stages (`test_BatchRevertsOnReuseWhileSinglesSucceed`, on the repository's `OracleSynchronizationTestBase` with the six-feed oracle and explicit payloads):

1. Two orders are committed one second apart. A payload with unequal component times after both commits is built.
2. `executeOrderBatch(2, payload)` reverts with `StoredFeedBehind`; `nextExecuteId()` is unchanged at 1.
3. `executeOrder` is called twice with the same payload; both fill; `nextExecuteId()` is 0.

Observed output:

```text
[PASS] test_BatchRevertsOnReuseWhileSinglesSucceed()
  executeOrderBatch(2) reverted StoredFeedBehind; nextExecuteId stays 1
  single executeOrder x2 succeeded; nextExecuteId 0
```
