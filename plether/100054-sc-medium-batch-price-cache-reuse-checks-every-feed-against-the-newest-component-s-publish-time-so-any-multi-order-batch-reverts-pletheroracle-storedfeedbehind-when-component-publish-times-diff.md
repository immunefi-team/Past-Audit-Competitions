# \#100054 [SC-Medium] Batch price-cache reuse checks every feed against the newest component's publish time, so any multi-order batch reverts `PletherOracle__StoredFeedBehind` when component publish times differ (`PletherOracle.sol`)

**Submitted on Nov 16th 2026 at 06:53:58 UTC by @Squid11643 for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #100054
- **Report Type:** Smart Contract
- **Report severity:** Medium
- **Target:** https://github.com/immunefi-team/audit-comp-plether/tree/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src
- **Impacts:**
  - Contract fails to deliver promised returns, but doesn't lose value

## Description

## Brief/Intro

When a keeper executes orders in a batch, the first order parses the historical Pyth update and installs a cached basket; later orders reuse it. Before reuse, the oracle re-proves storage coverage by checking **every** feed against `cache.publishTime` — which is the **newest** component's publish time. A feed whose own update was published earlier is therefore always "behind", and reuse reverts `PletherOracle__StoredFeedBehind`. The fresh-parse path checks each feed against **its own** publish time, so identical update data passes for the first order and fails for the second, and the whole `executeOrderBatch` rolls back. Batched execution stops working whenever the basket's components carry different publish times — which the oracle explicitly allows up to its publish-time dispersion limit.

## Vulnerability Details

The reuse path checks each feed against the cached basket's single `publishTime`, which is the maximum component publish time; the fresh-parse path checks each feed against its own.

```solidity
// packages/perps/src/PletherOracle.sol#L262-L266  (reuse path: every feed vs cache.publishTime = max)
if (!policy.oracleFrozen && _canReuseHistoricalBatchBasket(request.commitTime, cache)) {
    // Cache contents are caller-supplied; independently prove storage coverage.
    for (uint256 i; i < pythFeedIds.length; ++i) {
        _requireStoredFeedCoverage(pythFeedIds[i], cache.publishTime);
    }
```

```solidity
// packages/perps/src/PletherOracle.sol#L874-L875
basket.publishTime  = uint64(maxPublishTime);   // cache.publishTime is the NEWEST component time
// oldestPublishTime = uint64(minPublishTime);   // the minimum is kept separately but not used here
```

```solidity
// packages/perps/src/PletherOracle.sol#L625  (fresh-parse path: each feed vs its OWN publish time)
_requireStoredFeedCoverage(parsedFeeds[i].id, parsedFeeds[i].price.publishTime);
```

This looks like a regression: the 2026-10-15 AI scan quotes the basket builder as `basket.publishTime = uint64(minPublishTime)` (oldest component). At `a82d4fef` it is `maxPublishTime`, with the minimum moved to a new `oldestPublishTime` field, but the reuse-path coverage loop still reads `cache.publishTime`. Under the old semantics the loop was a correct lower bound; under the new semantics it demands that every feed be as new as the newest one.

`_requireStoredFeedCoverage` (`:645-653`) reverts when `pyth.getPriceUnsafe(feedId).publishTime < requiredPublishTime`. Pyth stores each feed at its own publish time, so a feed published at `t_i < max(t)` is stored at `t_i`; the reuse check then demands `t_i >= max(t)` and reverts. The Router catches only `PletherOracle__HistoricalParserFailure` (`OrderRouterExecutionSidecar.sol:1009-1031`) and re-raises everything else, so `StoredFeedBehind` reverts the whole batch — including orders that already executed within it.

### Affected locations
- `PletherOracle.updateBatchOrderExecutionPrice` reuse-path coverage loop — `packages/perps/src/PletherOracle.sol#L262-L266`
- Cache `publishTime = max` vs separate `oldestPublishTime` — `#L874-L875`
- Fresh-parse per-feed check (correct behavior) — `#L625`; `_requireStoredFeedCoverage` `#L645-L653`
- Router re-raise of non-parser reverts — `packages/perps/src/OrderRouterExecutionSidecar.sol#L1009-L1031`

### Preconditions
- A keeper executes a batch of two or more orders (normal keeper operation).
- The historical update carries components with different publish times — within the oracle's dispersion limit, and the common case for a six-feed FX basket (each feed's publish time is its own first update after commit).
- No privileged role, key, or social engineering; the attacker/keeper is untrusted. No in-scope contract is modified.

### Attack path (reproduction)
1. Two traders commit LONG opens at time `commit`.
2. At `commit + 6` a keeper calls `router.executeOrderBatch` with one historical update where feed 0 is published at `commit + 1` and the other feeds at `commit + 5` (dispersion 4s, within the order-execution limit).
3. The first order parses and executes; the oracle caches the basket with `publishTime = commit + 5`.
4. The second order reuses the cache; feed 0 is stored at `commit + 1 < commit + 5`, so the call reverts `PletherOracle__StoredFeedBehind(feed0, commit + 1, commit + 5)`.
5. The batch rolls back: `nextExecuteId` unchanged, the first order's position undone, neither trader has a position, engine mark unchanged.

## Impact Details

Batched execution is unavailable whenever basket components carry different publish times: keepers pay gas for the reverted batch and must fall back to one transaction per order, which costs more per order and drains the FIFO queue more slowly. **No funds are frozen or lost** — orders stay pending and execute one at a time via `executeOrder`, which accepts the same data. Frequency is a hypothesis (not measured): unequal component publish times are probably the common case for the multi-feed basket, so multi-order batching may be broadly unavailable. The PoC uses mocked Pyth data.

**Funds at risk:** none (no freeze, no loss). The cost is keeper gas inefficiency and slower queue drainage.

## Recommendation

Check each feed against the time it was actually included at, as the fresh-parse path already does. Store the per-feed publish times (or their hash) in `BatchOrderPriceCache` and check `_requireStoredFeedCoverage(pythFeedIds[i], cachedPublishTime[i])`. A smaller change is to check against `cache.oldestPublishTime` (already stored at `:875`) — it still proves every stored feed is at least as new as the oldest component, but is weaker than a per-feed check, so confirm it matches the intended coverage guarantee. Alternatively, treat `StoredFeedBehind` on the reuse path as "cache not reusable" and fall back to a fresh parse instead of reverting the batch.

## References
- Source (in scope): `packages/perps/src/PletherOracle.sol`, `packages/perps/src/OrderRouterExecutionSidecar.sol` @ `a82d4fefb02768e5b3bddcc97dee371eee24fea8`
- Documented whole-batch revert (NOT claimed): WHITEPAPER.md:1028-1030, :1038-1042

## Link to Proof of Concept

https://gist.github.com/Akashneelesh/b40e06594ff05d316361662a8327dc6f

## Proof of Concept

## Proof of Concept

**PoC gist (secret; PoC files, run steps, expected output):** https://gist.github.com/Akashneelesh/b40e06594ff05d316361662a8327dc6f

Runs in the repository's Foundry suite at the in-scope commit. `DS1_StoredFeedBehind.t.sol` inherits `PocBase`; only Pyth and USDG mocked; no in-scope contract modified.

**Setup** (run from the repository root):
```bash
forge test --root packages/perps --match-path "test/perps/audit/DS1_StoredFeedBehind.t.sol" -vv
```

**Output:**
```
[PASS] test_Attack_BatchReuseRevertsWhenOneComponentIsBehindCachePublishTime() (gas: 4407605)
[PASS] test_Control_SingleOrderExecutionSucceedsWithUnequalComponents() (gas: 2500904)
Suite result: ok. 2 passed; 0 failed; 0 skipped
```

The attack test asserts the exact revert data (`StoredFeedBehind(feed0, commit + 1, commit + 5)`), that `nextExecuteId` and both position sizes are unchanged, and that the engine mark is unchanged. The control runs the same update data through single-order `executeOrder` and succeeds. PoC file: `packages/perps/test/perps/audit/DS1_StoredFeedBehind.t.sol` (dig copy under `dig/L1-batch-cache-stored-feed-behind/v1/`).

