# \#97635 [SC-Medium] Inconsistent storage-coverage check on batch price reuse makes executeOrderBatch revert on the Router's own valid cache whenever basket component times differ

**Submitted on Nov 2nd 2026 at 15:03:32 UTC by @risinghorse for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #97635
- **Report Type:** Smart Contract
- **Report severity:** Medium
- **Target:** https://github.com/immunefi-team/audit-comp-plether/tree/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src
- **Impacts:**
  - Contract fails to deliver promised returns, but doesn't lose value

## Description

## Brief/Intro

In `executeOrderBatch`, the first order of a commit group gets a fresh historical Pyth parse, and later orders in the group reuse the cached basket. The two paths check Pyth storage differently. The fresh path checks each feed against its own publish time, while the reuse path checks every feed against the basket's newest component time. When component times differ, a lagging feed's stored time is below that newest time, so the reuse check reverts with `PletherOracle__StoredFeedBehind`. The Router has already predicted reuse and sent no fee, so the error escapes and the whole batch reverts, including orders already executed earlier in the same call. Retrying with the same payload fails the same way, even though a fresh parse would have succeeded. No funds are at risk, but the keeper batch path fails deterministically for ordinary order groups, contrary to the documentation.

## Vulnerability Details

### 1. The two oracle paths check storage coverage differently

The fresh path stores the payload, then checks each feed against its own time:

```solidity
// packages/perps/src/PletherOracle.sol:623-626
pyth.updatePriceFeeds{value: pythFee}(pythUpdateData);
for (uint256 i; i < parsedFeeds.length; ++i) {
    _requireStoredFeedCoverage(parsedFeeds[i].id, parsedFeeds[i].price.publishTime);
}
```

The reuse path checks every feed against `cache.publishTime`, the newest component time:

```solidity
// packages/perps/src/PletherOracle.sol:262-266
if (!policy.oracleFrozen && _canReuseHistoricalBatchBasket(request.commitTime, cache)) {
    // Cache contents are caller-supplied; independently prove storage coverage, not price authenticity.
    for (uint256 i; i < pythFeedIds.length; ++i) {
        _requireStoredFeedCoverage(pythFeedIds[i], cache.publishTime);
    }
```

```solidity
// packages/perps/src/PletherOracle.sol:645-653
function _requireStoredFeedCoverage(
    bytes32 feedId,
    uint256 requiredPublishTime
) internal view {
    uint256 storedPublishTime = pyth.getPriceUnsafe(feedId).publishTime;
    if (storedPublishTime < requiredPublishTime) {
        revert PletherOracle__StoredFeedBehind(feedId, storedPublishTime, requiredPublishTime);
    }
}
```

The cache keeps only the newest and oldest component times (`IPletherOracle.sol:118-127`, filled at `PletherOracle.sol:874-875` and `289-296`). Pyth's `updatePriceFeeds` stores a message only if it is newer than what is already stored. After the fresh parse, a lagging feed with no newer message in the payload is therefore stored below `cache.publishTime`, and the reuse check fails.

### 2. The Router predicts reuse without the coverage check and sends no fee

```solidity
// packages/perps/src/OrderRouterExecutionSidecar.sol:996-998
uint256 fee = 0;
if (!_canReuseHistoricalBatchBasket(oracle, order, state.oracleCache)) {
    fee = oracle.getOrderExecutionFee(pythUpdateData);
```

The Router-side `_canReuseHistoricalBatchBasket` (`OrderRouterExecutionSidecar.sol:1074-1093`) repeats only the oracle's time conditions (`PletherOracle.sol:974-991`). It never tests storage coverage, so the oracle gets no fee and cannot fall back to a paid fresh parse.

### 3. The revert escapes and takes the whole batch with it

- The catch around the batch oracle call absorbs only the 68-byte `PletherOracle__HistoricalParserFailure`. Anything else is re-raised (`OrderRouterExecutionSidecar.sol:1019-1027`). `StoredFeedBehind(bytes32,uint256,uint256)` is 100 bytes.
- `_prepareBatchOracle` runs outside the per-item try frame (`OrderRouterExecutionSidecar.sol:292-296`), and `OrderRouter._delegateExecutionSidecar` bubbles the revert (`OrderRouter.sol:545-554`).
- The transaction therefore reverts, including predecessor orders that were already finalized in the same call.

### 4. The documentation promises a fallback that does not exist

- `ORACLE_SYNCHRONIZATION.md:61-63`: reuse requires the components to "satisfy cache coverage. Otherwise the Router requests a new historical resolution."
- `SECURITY.md:649`: batch execution may reuse a parsed basket for later FIFO orders within its proven coverage.
- `ORACLE_SYNCHRONIZATION.md:109`: a "failed cache coverage" reverts the transaction, including earlier batch progress.

The Router never requests a new resolution here, because it never checks coverage. The repository's own unequal-time reuse test, `test_BatchReusesBasketWhenEveryComponentFollowsBothCommits` (`OracleSynchronization.t.sol:434-453`), passes only because line 446 first pushes newer prices into Pyth storage.

## Impact Details

- **Batch reverts.** `executeOrderBatch` reverts deterministically for a natural order group, such as two orders committed in the same second, whenever the first post-commit component times differ and that group is the last one covered by the payload. The revert also rolls back every predecessor already executed in the same call.
- **Keeper cost.** Keepers that do not simulate first waste gas; Pyth fees are returned on revert. To work around it, they must:
  - fall back to `executeOrder` per order;
  - stop the batch before the reuse-eligible order;
  - or add newer updates for the lagging feeds to the payload.
- **Not a protocol-wide denial of service.** Execution is permissionless and the single-order path works. Fills are pinned to the commit-time basket, so a delay does not change fill prices.
- **Prices and accounting.** Unaffected: a reused basket would equal a freshly parsed one.
- **Classification.** Edge-case incorrectness and a spec deviation, with a limited liveness cost for the batch keeper path.

## Recommendation

1. Make the reuse coverage rule match the fresh-path rule, stored(feed_i) >= t_i. Store each component's publish time in `BatchOrderPriceCache` and check each feed against its own time on reuse. A lighter option for Router-originated caches is to check against `oldestPublishTime`.
2. Alternatively, keep the strict check but let the Router predict it. Expose an oracle view that includes the stored-feed coverage check; when it returns false, quote and pay for a fresh historical parse, which succeeds on the same payload.
3. Add a regression test identical to `test_BatchReusesBasketWhenEveryComponentFollowsBothCommits` but without the pre-push at line 446.
4. Reconcile `ORACLE_SYNCHRONIZATION.md:61-63` and `:109`, and define "cache coverage" precisely.

## References

- Reuse-path coverage check: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/PletherOracle.sol#L262-L266
- Fresh-path coverage check: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/PletherOracle.sol#L613-L626
- Coverage helper: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/PletherOracle.sol#L645-L653
- Cache fields: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/interfaces/IPletherOracle.sol#L118-L127 and https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/PletherOracle.sol#L289-L296
- Oracle reuse conditions: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/PletherOracle.sol#L974-L991
- Router reuse prediction and zero fee: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/OrderRouterExecutionSidecar.sol#L996-L1005 and https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/OrderRouterExecutionSidecar.sol#L1074-L1093
- Error handling: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/OrderRouterExecutionSidecar.sol#L1019-L1030
- Oracle step outside the item frame: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/OrderRouterExecutionSidecar.sol#L292-L302
- Revert bubbling: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/OrderRouter.sol#L545-L554
- Test relying on a pre-push: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/test/perps/spec/oracle/OracleSynchronization.t.sol#L434-L453
- Documentation: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/ORACLE_SYNCHRONIZATION.md#L61-L63, https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/ORACLE_SYNCHRONIZATION.md#L109 and https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/SECURITY.md#L649


## Proof of Concept

## Proof of Concept

Default parameters apply: `orderSettlementWindow` = 15 and `maxComponentPublishTimeDivergence` = 5 (`PletherOracle.sol:87,90`), with no override in the deploy scripts. Every commit uses `commitTime = uint64(block.timestamp)` (`OrderRouter.sol:170,465`).

### Step by step

1. **Commits.** At T = 1,000,000, Alice commits order #10 and Bob commits order #11 in the same second, so both have commitTime T. Arbitrum produces several blocks per second.
2. **Price ticks.** The first ticks after the commit are EUR, JPY, GBP, CAD and CHF at T+1, and SEK at T+2, each with `prevPublishTime <= T`.
   - The 1-second spread is under the 5-second divergence limit.
   - No third party has pushed these FX feeds into Arbitrum Pyth storage, so storage is at or below T+1. Plether's live paths parse without writing storage, which makes this ordinary.
3. **Keeper call.** The keeper fetches the first update after T for each feed from Hermes: EUR to CHF from the T+1 slot and SEK from the T+2 slot. At `block.timestamp` T+5 it calls `executeOrderBatch{value: 2 * getOrderExecutionFee(payload)}(11, payload)`. Orders #10 and #11 form the last commit group in the batch.
4. **Order #10 (fresh parse).** The cache is empty, so the Router pays the fee.
   - `parsePriceFeedUpdatesUnique` over [T+1, T+5] succeeds.
   - `updatePriceFeeds` stores EUR to CHF at T+1 and SEK at T+2.
   - The per-feed coverage checks pass: T+1 >= T+1 and T+2 >= T+2.
   - The cache becomes {minReusableCommitTime T, oldestPublishTime T+1, publishTime T+2}.
   - The Engine mark is set at T+2, and #10 executes and is finalized inside its item frame.
5. **Order #11 (reuse).** The Router's reuse prediction passes every condition (T >= T, T < T+1, T < T+2, T+2 <= T+5 and T+2 <= T+15), so it sends a zero fee. In the oracle's reuse branch, `_requireStoredFeedCoverage(EUR, T+2)` reads a stored time of T+1 and reverts with `PletherOracle__StoredFeedBehind(EUR, T+1, T+2)`.
6. **Whole batch reverts.** The revert data is 100 bytes, not 68, so the sidecar re-raises it outside the item frame. `OrderRouter` bubbles it, and the transaction reverts, including #10's finalized execution. Retrying with the same payload always reverts the same way. A fresh parse for #11 would have passed, because each feed would be checked only against its own time.

### When it happens

The reuse-eligible group must be the last commit group whose updates are in the payload. A later group, committed at or after T+1, brings messages at T+2 or later, which #10's `updatePriceFeeds` stores, repairing coverage. This is common for small batches, for example any two orders committed in the same second.

### How to reproduce with the existing test

1. Copy `test_BatchReusesBasketWhenEveryComponentFollowsBothCommits` (`packages/perps/test/perps/spec/oracle/OracleSynchronization.t.sol:434-453`). It already uses unequal component times: feed 0 at +4, the others at +5.
2. Delete the storage pre-push on line 446: `baseMockPyth.updatePriceFeeds{value: FEE}(newer);`.
3. Run the test. `executeOrderBatch` now reverts with `PletherOracle__StoredFeedBehind`: feed 0 is stored at +4 while +5 is required.
4. As a control, call `executeOrder` for each order separately with the same payload. Both orders execute.

