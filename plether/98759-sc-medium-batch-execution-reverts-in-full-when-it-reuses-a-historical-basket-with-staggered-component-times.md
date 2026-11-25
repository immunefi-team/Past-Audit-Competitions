# \#98759 [SC-Medium] Batch execution reverts in full when it reuses a historical basket with staggered component times

**Submitted on Nov 10th 2026 at 02:29:24 UTC by @redbaron for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #98759
- **Report Type:** Smart Contract
- **Report severity:** Medium
- **Target:** https://github.com/immunefi-team/audit-comp-plether/tree/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src
- **Impacts:**
  - Contract fails to deliver promised returns, but doesn't lose value

## Description

## Brief/Intro

**The reuse.** `executeOrderBatch` prices the first order with a fresh historical parse and keeps the parsed basket in an in-memory cache. A later FIFO order whose commit precedes every component of that basket reuses the cache instead of paying a new Pyth fee. `ORACLE_SYNCHRONIZATION.md` promises that when reuse is not possible "the Router requests a new historical resolution."

**The mismatch.** The Router decides reuse with `_canReuseHistoricalBatchBasket`, which looks only at commit and publish times. The Oracle's reuse branch additionally requires the **stored** Pyth publish time of every feed to be at least `cache.publishTime`, which is the basket's **newest** component time. The fresh parse that built the cache stored each feed at its **own** publish time. When the components are staggered, which `maxComponentPublishTimeDivergence = 5` exists to allow, the lagging feed is below the basket maximum and the Oracle reverts with `PletherOracle__StoredFeedBehind`.

**The consequence.** The Router swallows only the 68-byte `HistoricalParserFailure`. It re-raises `StoredFeedBehind`, so the whole `executeOrderBatch` reverts, including the earlier order that was already priced and settled. Nothing is wrong with the payload: the same data prices both orders when they are executed one per call.

**Measured in the PoC** (repo fixture, production `OrderRouter`, execution sidecar and `PletherOracle`): two orders, components at t+4 and t+5, Pyth storage not ahead. The batch reverts with `StoredFeedBehind(feed1, t+4, t+5)` and the queue head stays at order 1. The inverted stagger reverts the same way. Executed one by one, both orders fill.

## Vulnerability Details

### 1. The Router predicts reuse from times only

```solidity
// packages/perps/src/OrderRouterExecutionSidecar.sol:1074-1093
function _canReuseHistoricalBatchBasket(IPletherOracle oracle, CfdTypes.Order memory order,
    IPletherOracle.BatchOrderPriceCache memory cache) private view returns (bool) {
    if (oracle.isOracleFrozen() || !cache.hasHistoricalBasket) {
        return false;
    }
    uint64 commitTime = order.commitTime;
    if (
        commitTime < cache.minReusableCommitTime || commitTime >= cache.oldestPublishTime
            || commitTime >= cache.publishTime
    ) {
        return false;
    }
    if (cache.publishTime > block.timestamp) {
        return false;
    }
    return uint256(cache.publishTime) <= uint256(commitTime) + oracle.orderSettlementWindow();
}
```

When this returns true the Router sends no Pyth fee and calls the Oracle with the cache. It never checks the storage coverage that the Oracle will require.

### 2. The Oracle requires storage coverage of the basket maximum

```solidity
// packages/perps/src/PletherOracle.sol:262-266
if (!policy.oracleFrozen && _canReuseHistoricalBatchBasket(request.commitTime, cache)) {
    // Cache contents are caller-supplied; independently prove storage coverage, not price authenticity.
    for (uint256 i; i < pythFeedIds.length; ++i) {
        _requireStoredFeedCoverage(pythFeedIds[i], cache.publishTime);
    }
```

```solidity
// packages/perps/src/PletherOracle.sol:645-652
function _requireStoredFeedCoverage(bytes32 feedId, uint256 requiredPublishTime) internal view {
    uint256 storedPublishTime = pyth.getPriceUnsafe(feedId).publishTime;
    if (storedPublishTime < requiredPublishTime) {
        revert PletherOracle__StoredFeedBehind(feedId, storedPublishTime, requiredPublishTime);
    }
}
```

`cache.publishTime` is the basket's newest component (`basket.publishTime = uint64(maxPublishTime)`, `PletherOracle.sol:875`).

### 3. The fresh parse stores each feed only at its own time

```solidity
// packages/perps/src/PletherOracle.sol:618-626
basket = _computeBasketPriceFromFeeds(PriceMode.OrderExecution, parsedFeeds, maxComponentPublishTimeDivergence);
pyth.updatePriceFeeds{value: pythFee}(pythUpdateData);
for (uint256 i; i < parsedFeeds.length; ++i) {
    _requireStoredFeedCoverage(parsedFeeds[i].id, parsedFeeds[i].price.publishTime);
}
```

The fresh path checks each feed against **its own** parsed time, so it passes. Then it emits a cache whose `publishTime` is the maximum (`PletherOracle.sol:288-296`). The Oracle therefore produces a cache that its own reuse branch refuses whenever the components are not all equal.

This is the only `updatePriceFeeds` call in the Oracle. The live path keeps its basket in the Oracle's own storage and never writes Pyth storage (`ORACLE_SYNCHRONIZATION.md:19-27`). Storage is ahead of the cached basket only if a third party pushes all six FX feeds, SEK/USD included, to Pyth on Arbitrum within the few seconds between the tick and the keeper's call. The repo's own reuse test (`OracleSynchronization.t.sol:434`, `test_BatchReusesBasketWhenEveryComponentFollowsBothCommits`) staggers one component exactly like this. It passes only because it first calls `baseMockPyth.updatePriceFeeds(newer)`, which hides the revert. Our `test_Masking_PassesOnlyIfStorageWasPushedAhead` reproduces that.

### 4. The Router re-raises the error and the batch reverts

```solidity
// packages/perps/src/OrderRouterExecutionSidecar.sol:1019-1027
} catch (bytes memory reason) {
    if (
        reason.length != 68
            || _selector(reason) != IPletherOracle.PletherOracle__HistoricalParserFailure.selector
    ) {
        assembly ("memory-safe") {
            revert(add(reason, 32), mload(reason))
        }
    }
```

`StoredFeedBehind` is 100 bytes with a different selector, so it bubbles up and the full `executeOrderBatch` reverts. The earlier order's settlement is rolled back with it.

### 5. Precondition

- Order B takes the reuse branch: `commitTime(B) >= commitTime(A)` and `commitTime(B) < oldestPublishTime` of A's basket. B is committed before the first post-commit tick of the slowest component, for example in the same second as A or during a quiet gap. One trader can create this on purpose by committing two orders in one block.
- The first post-commit ticks of the components fall on different seconds. The protocol allows up to 5 s (`maxComponentPublishTimeDivergence = 5`, `PletherOracle.sol:90`). The direction of the stagger does not matter, because every feed is compared with the basket maximum.

### Relation to prior audits and known issues

- **Not in the known-issues sections.** "Router and keeper limitations" and "Known Limitations" (`SECURITY.md:1328-1378`) do not mention batch reuse or cache coverage. Prior router/oracle findings (Cecuro MED-7, MED-18, MED-20, MED-23, LOW-61, LOW-70) concern component selection, equal-time mark rewrites, the frozen divergence limit and single-order progress on unavailable history.
- **Not a generic preparation failure.** `SECURITY.md:676-678` says that other Oracle/Pyth preparation failures revert the complete batch. Here the payload is valid and the fresh path accepts it. The batch fails only because the Router mispredicts reuse of a cache that the Oracle built and cannot reuse, which contradicts `ORACLE_SYNCHRONIZATION.md:62-63`: "Reuse for a later order requires all components to follow that order's commit, fit its historical window and satisfy cache coverage. **Otherwise the Router requests a new historical resolution.**"

## Impact Details

**Low, "Contract fails to deliver promised returns, but doesn't lose value".** Batch reuse is a documented feature: one fee per basket and batched throughput. Under the precondition, the batch fails in full instead.

| Execution | Result (same orders, same payload) |
|---|---|
| `executeOrderBatch(2)`, components t+4 / t+5 | Reverts `StoredFeedBehind(feed1, t+4, t+5)`; queue head stays at order 1 |
| `executeOrderBatch(2)`, inverted stagger | Reverts `StoredFeedBehind` |
| One order per call | Both orders fill |
| Batch, after a third party pushes newer prices to Pyth storage | Both orders fill |

- No order in the batch makes progress, and the keeper pays the gas of the reverted call.
- Workaround: execute one order per call. Each order then pays its own fresh Pyth fee, so the batch savings are lost and keeper throughput drops in the congested periods that batching is meant for.
- No funds are lost: the orders stay pending and are executable later. A trader can raise the frequency by placing a second order in the same pre-tick window, but the cost (real order fees) exceeds the keeper's lost gas, so this is not claimed as griefing.

### Preconditions

- Two or more FIFO orders committed before the first post-commit tick of the slowest component.
- Staggered first post-commit component times (≤ 5 s apart).
- Pyth storage not pushed ahead by a third party between the tick and execution, which is the normal case.

## Recommendation

Emit a reusable cache only when Pyth storage covers the basket's newest component time. Otherwise the next order takes the fresh path and pays its own fee, as documented:

```diff
-        if (!policy.oracleFrozen && !reusedBasket) {
+        if (!policy.oracleFrozen && !reusedBasket && _storageCovers(basket.publishTime)) {
             nextCache = BatchOrderPriceCache({
```

`_storageCovers` loops over `pythFeedIds` and checks `pyth.getPriceUnsafe(id).publishTime >= t` (full diff in the gist, `fix.diff`). With the fix:
- the two revert tests fail on purpose, because the batch now fills both orders;
- the three controls pass;
- the `OracleSynchronization` suites pass.

Alternatives: let the Router check coverage in `_canReuseHistoricalBatchBasket`, or require coverage of each feed's own cached time instead of the basket maximum.

## Link to Proof of Concept

https://gist.github.com/the-red-ace/c91b1c13f70812287bafef34c46db733

## Proof of Concept

## Proof of Concept

**Gist:** https://gist.github.com/the-red-ace/c91b1c13f70812287bafef34c46db733. It contains the test file, `fix.diff` and a README with the exact folder.

**How the tests run:** in the repo's own Foundry suite at a82d4fe, on the repo's `OracleSynchronizationTestBase` fixture. Production `OrderRouter`, execution sidecar and `PletherOracle` are used, and `MockPyth` stands in for Pyth with its unique-parse semantics (`prevPublishTime < min <= publishTime <= max`). No in-scope contract is modified. The keeper funds a fresh parse for every order (`4 * FEE`), so funding is not the cause.

```bash
git checkout a82d4fefb02768e5b3bddcc97dee371eee24fea8 && git submodule update --init --recursive
cp BatchReuseCoverage.t.sol packages/perps/test/perps/spec/oracle/
forge test --root packages/perps --match-contract BatchReuseCoverageTest -vv
```

| Test | What it shows |
|---|---|
| `test_BatchRevertsOnReuseWithStaggeredComponents` | Components t+4 / t+5: the batch reverts with `StoredFeedBehind(feed1, t+4, t+5)`; `nextExecuteId` stays 1, so the first order is rolled back too. |
| `test_InvertedStaggerAlsoReverts` | Feed 1 newest, the others lagging: also reverts. The direction does not matter. |
| `test_Control_SingleOrdersSucceed` | The same payload, one order per call: both fill. |
| `test_Control_BatchOfOneSucceeds` | The same payload prices a single order. |
| `test_Masking_PassesOnlyIfStorageWasPushedAhead` | The batch fills only after newer prices are pushed into Pyth storage, which is what the repo's own reuse test does. |

Expected output (reproduced on a clean a82d4fe checkout with only the gist file):
```
[PASS] test_BatchRevertsOnReuseWithStaggeredComponents()
[PASS] test_Control_BatchOfOneSucceeds()
[PASS] test_Control_SingleOrdersSucceed()
[PASS] test_InvertedStaggerAlsoReverts()
[PASS] test_Masking_PassesOnlyIfStorageWasPushedAhead()
Suite result: ok. 5 passed; 0 failed; 0 skipped
```

With `fix.diff` applied (`git apply fix.diff` from the repo root), the two revert tests fail with `next call did not revert as expected`, because the batch now executes. The three controls still pass.

