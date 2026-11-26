# \#99311 [SC-Medium] Batch order execution reverts completely when the reused price basket is not covered by stored Pyth feeds

**Submitted on Nov 13th 2026 at 03:59:03 UTC by @Krish00 for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #99311
- **Report Type:** Smart Contract
- **Report severity:** Medium
- **Target:** https://github.com/immunefi-team/audit-comp-plether/tree/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src
- **Impacts:**
  - Contract fails to deliver promised returns, but doesn't lose value

## Description

Brief/Intro
When `executeOrderBatch` processes multiple orders, it can reuse the price data from the first order for the remaining orders in the batch.

However, the oracle requires all stored price feeds to be at least as recent as the newest feed in the reused data. Since different feeds are updated a second or two apart, this requirement can fail and trigger a `StoredFeedBehind` error.

As a result, the entire batch transaction reverts, including orders that were already processed earlier in the same batch..

Vulnerability Details
The issue occurs because the router reuses previously resolved oracle price data without checking whether all stored feeds meet the required timestamp.

The relevant code paths are:

1. **Historical price reuse:** In `OrderRouterExecutionSidecar._canReuseHistoricalBatchBasket` (around line 1074), the router checks whether the previous price basket can be reused. However, it does not verify that all stored feeds are sufficiently up to date.

2. **Oracle timestamp validation:** In `PletherOracle.updateBatchOrderExecutionPrice` (around lines 262–266), the oracle requires every stored feed to have a timestamp greater than or equal to `cache.publishTime`.

3. **Initial price resolution:** The `_resolveHistoricalOrderBasket` function (around lines 619–626) initially checks each feed against its own timestamp. It does not guarantee that every feed meets the newest timestamp when the basket is reused for another order.

4. **Error handling:** In `OrderRouterExecutionSidecar` (around lines 1020–1028), the router only handles the 68-byte parser error. However, the `StoredFeedBehind` error is 100 bytes, so it is not handled by this fallback logic. Instead, the error causes the entire batch transaction to revert.

5. **Documentation mismatch:** The `ORACLE_SYNCHRONIZATION.md` document, under the "Batch reuse" section, describes that the router should request a fresh historical resolution instead of continuing with an outdated batch basket.

This means the existing reuse logic does not properly handle differences in publication times between oracle feeds.
Impact Details
When this issue occurs, the entire batch transaction fails, even if some orders were successfully processed before the failure.

The practical impact is:

1. The keeper loses the gas spent on the reverted transaction.
2. Previously processed orders are rolled back along with the rest of the batch.
3. Keepers may need to execute orders individually instead of batching them.
4. Individual execution requires additional transactions and may require paying the Pyth update fee for each execution.
5. An order committed shortly after another order can trigger this condition when the keeper attempts to execute both together.

No direct loss of user funds has been demonstrated. The issue affects batch execution reliability, increases keeper costs, and prevents the router from behaving as described in its oracle synchronization documentation.

Based on the demonstrated impact, this report is classified as Low severity.
References
1. OrderRouterExecutionSidecar.sol
https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/OrderRouterExecutionSidecar.sol

2. PletherOracle.sol
https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/PletherOracle.sol

3. ORACLE_SYNCHRONIZATION.md
https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/ORACLE_SYNCHRONIZATION.md

## Link to Proof of Concept

https://gist.github.com/unknownshadow2011/19c8a824879e2c71e8e564e5fb3f76b4

## Proof of Concept

**Test file:** `BatchCacheCoverage.t.sol`

The complete test code is available in the attached Gist.

Place the file in:

`packages/perps/test/perps/poc/`

Run the test using repository commit:

`a82d4fefb02768e5b3bddcc97dee371eee24fea8`

The test uses the repository's existing `OracleSynchronizationTestBase` and `MockPyth` to simulate oracle price updates. Since Pyth is outside the scope of this issue, using a mock for testing is appropriate.

No in-scope contract has been changed.

**Command to run:**

```bash
cd packages/perps
FOUNDRY_MATCH_CONTRACT=BatchCacheCoveragePoC FOUNDRY_VERBOSITY=2 forge test
```

### Test Scenario

Test: `test_PoC_BatchReuseRevertsWholeBatchWithoutThirdPartyStorageRefresh`

1. Alice commits an order at time C. Bob commits another order at C+3.
2. At C+6, a keeper calls `executeOrderBatch` with a valid price payload.
   - Feed 0 has a publish time of C+4.
   - The other five feeds have a publish time of C+5.
   - The difference between feed timestamps is one second, within the allowed five-second limit.
3. The router processes Alice's order and resolves the price data. It caches the basket with `publishTime = C+5`. However, Pyth storage still contains feed 0 with a timestamp of C+4.
4. Bob's order uses the cached basket. The oracle checks the stored feeds and finds that feed 0 is older than the required timestamp.
5. The oracle reverts with `StoredFeedBehind(feed0, C+4, C+5)`.
6. The router does not handle this error, so the entire batch transaction reverts. Alice's order is also rolled back, even though it was processed first.

### Additional Tests

I added two more tests to check whether the problem is specifically related to cached price data reuse.

**1. Individual order execution**

Test: `test_PoC_SingleExecutionsSucceedWithSamePayload`

The same price payload is used to execute both orders separately through two `executeOrder` calls.

Both executions succeed, and both orders reach the `Executed` status.

**2. Batch execution with matching feed timestamps**

Test: `test_Control_BatchSucceedsWhenAllComponentsShareOneSecond`

The same scenario is tested again, but all six feeds have the same publish time of C+5.

This time, the batch executes both orders successfully. The second order reuses the cached basket, and the historical price data is parsed only once.

### Test Results

All three tests passed at repository commit `a82d4fe`, when rechecked on 13 November 2026.

The attached screenshot shows the successful test results.
