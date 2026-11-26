# \#99045 [SC-Medium] # Inconsistent Pyth Coverage Check in `_requireStoredFeedCoverage` Blocks Batch Basket Cache Reuse in `executeOrderBatch`

**Submitted on Nov 11th 2026 at 16:33:43 UTC by @dikanevn for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #99045
- **Report Type:** Smart Contract
- **Report severity:** Medium
- **Target:** https://github.com/immunefi-team/audit-comp-plether/tree/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src
- **Impacts:**
  - Contract fails to deliver promised returns, but doesn't lose value

## Description

## 1. Summary

- **Protocol:** Plether
- **Repository / Commit:** `https://github.com/immunefi-team/audit-comp-plether` (`a82d4fefb02768e5b3bddcc97dee371eee24fea8`)
- **Asset:** `packages/perps/src/PletherOracle.sol`, `packages/perps/src/OrderRouterExecutionSidecar.sol`
- **Severity:** `Low` — `Contract fails to deliver promised returns, but doesn't lose value` (breaks batch order clearing and promised gas savings without loss or permanent locking of funds)
- **Vulnerability Type (CWE):** CWE-840 (Business Logic Errors), CWE-754 (Improper Check for Unusual or Exceptional Conditions)
- **Vulnerable Locations:**
  - `packages/perps/src/PletherOracle.sol:262-274` (`updateBatchOrderExecutionPrice`), `:645-653` (`_requireStoredFeedCoverage`), `:974-991` (`_canReuseHistoricalBatchBasket`)
  - `packages/perps/src/OrderRouterExecutionSidecar.sol:996-1030` (`_prepareBatchOracle`), `:1074-1093` (`_canReuseHistoricalBatchBasket`)

### 1.1. Core Issue

During batch order execution (`executeOrderBatch`), `PletherOracle` attempts to reuse the price basket computed for the first order (`BatchOrderPriceCache`), but verifies Pyth storage coverage for every feed against the basket's upper bound `cache.publishTime` (`maxPublishTime`). Because the protocol explicitly permits component timestamp divergence of up to 5 seconds (`maxComponentPublishTimeDivergence`), any lagging feed that does not yet have a newer record in Pyth storage (`storedPublishTime < cache.publishTime`, specifically when the historical VAAs themselves represent the newest storage state) consistently reverts with `PletherOracle__StoredFeedBehind` (100 bytes). The router only catches `PletherOracle__HistoricalParserFailure` (68 bytes) and rethrows the revert, crashing the entire batch of 2+ orders.

---

## 2. Root Cause Analysis

### 2.1. The `maxPublishTime` vs. Per-Component Check Paradox in `PletherOracle.sol`

On the first order of a batch, the cache is empty (`hasHistoricalBasket == false`), and `updateBatchOrderExecutionPrice` calls `_resolveOrderExecutionBasket` → `_resolveHistoricalOrderBasket` → `_computeBasketPriceFromFeeds` (`PletherOracle.sol:805-876`):
- the basket permits publish-time divergence across feeds up to `maxComponentPublishTimeDivergence` (`5 seconds`);
- `basket.publishTime` is set to `uint64(maxPublishTime)` (the newest component's timestamp), while `basket.oldestPublishTime` is set to `uint64(minPublishTime)`;
- after `pyth.updatePriceFeeds{value: pythFee}(pythUpdateData)`, each feed is checked against its **own** `publishTime` (`PletherOracle.sol:624-626`):
  ```solidity
  for (uint256 i; i < parsedFeeds.length; ++i) {
      _requireStoredFeedCoverage(parsedFeeds[i].id, parsedFeeds[i].price.publishTime);
  }
  ```
- the resulting `nextCache` stores `publishTime: basket.publishTime` (`maxPublishTime`) and `oldestPublishTime: basket.oldestPublishTime` (`minPublishTime`).

On the second order of the batch, the cache reuse branch is entered (`PletherOracle.sol:262-266`):

```solidity
if (!policy.oracleFrozen && _canReuseHistoricalBatchBasket(request.commitTime, cache)) {
    // Cache contents are caller-supplied; independently prove storage coverage, not price authenticity.
    for (uint256 i; i < pythFeedIds.length; ++i) {
        _requireStoredFeedCoverage(pythFeedIds[i], cache.publishTime);
    }
    ...
}
```

Meanwhile, `_canReuseHistoricalBatchBasket` (`PletherOracle.sol:974-991`) allows cache reuse even when component timestamps diverge:

```solidity
function _canReuseHistoricalBatchBasket(
    uint64 commitTime,
    BatchOrderPriceCache calldata cache
) internal view returns (bool) {
    if (!cache.hasHistoricalBasket) {
        return false;
    }
    if (
        commitTime < cache.minReusableCommitTime || commitTime >= cache.oldestPublishTime
            || commitTime >= cache.publishTime
    ) {
        return false;
    }
    if (cache.publishTime > block.timestamp) {
        return false;
    }
    return uint256(cache.publishTime) <= uint256(commitTime) + orderSettlementWindow;
}
```

Whenever the basket has any valid divergence and the lagging feed has no newer update in Pyth storage yet (for example, `FEED_A = 1004`, `FEED_B = 1002`, when the historical payload itself sets the newest storage state since `updatePriceFeeds` does not roll storage backwards; an already-recorded newer tick merely masks the defect), Pyth storage for `FEED_B` holds `1002`, whereas the loop on line 265 requires `storedPublishTime >= cache.publishTime (1004)` from **all** feeds. Inside `_requireStoredFeedCoverage` (`1002 < 1004`), execution reverts with `PletherOracle__StoredFeedBehind(FEED_B, 1002, 1004)`. Notably, the cache is specifically designed for target FIFO orders (`commitTime < cache.oldestPublishTime` and `commitTime >= cache.minReusableCommitTime`), meaning the predicate admits the primary intended batch-cache workflow into the reuse branch; the branch itself succeeds when storage already covers `cache.publishTime`, and conditionally reverts inside only when this divergent payload is the head of storage.

### 2.2. Error Propagation in `OrderRouterExecutionSidecar.sol`

In `OrderRouterExecutionSidecar._prepareBatchOracle` (`OrderRouterExecutionSidecar.sol:1020-1027`), the `catch` block filters only for `PletherOracle__HistoricalParserFailure`:
```solidity
if (
    reason.length != 68
        || _selector(reason) != IPletherOracle.PletherOracle__HistoricalParserFailure.selector
) {
    assembly ("memory-safe") {
        revert(add(reason, 32), mload(reason))
    }
}
```
The error `PletherOracle__StoredFeedBehind(bytes32,uint256,uint256)` is 100 bytes long with a different selector, so the router propagates it outward and reverts the entire `executeOrderBatch` transaction.

*(In the protocol's base test helpers such as `OracleSynchronizationTestBase.sol:90-97`, all feeds were assigned the same `tick`, so cache reuse under feed divergence was not exercised.)*

### 2.3. Why This Revert Is Not "By Design" (Under Batch-Atomic Execution) and How It Differs from Known Issue `MED-7`

- **Batch-atomicity and NatSpec describe the coverage check, not a rejection of reuse when `oldestPublishTime != publishTime`; the revert stems from predicate inconsistency:** According to the specification (`packages/perps/SECURITY.md:748-749`, `OrderRouter.sol:251-252`), oracle preparation is intentionally `batch-atomic` so the transaction reverts on genuine oracle-data delivery/payment failures. Furthermore, the NatSpec on `updateBatchOrderExecutionPrice` (`PletherOracle.sol:230-232`) explicitly states: `A reused basket first verifies all stored feeds cover its timestamp, pays no Pyth fee, and leaves the cache unchanged`. The difference is that while the coverage check against the basket timestamp is documented, refusing reuse whenever `oldestPublishTime != publishTime` is not—even though the protocol officially allows component divergence up to `5 seconds`. As a result, a completely valid basket with permissible `<= 5s` divergence (and valid orders, as confirmed by single-order `executeOrder`) reverts due to the mismatch between the two predicates.
- **Difference from `MED-7`:** Known issue `MED-7` concerns a caller's ability to pick an older signed price within the feed's allowed window. Here, no attacker is selecting an older price—on the contrary, **a legitimate keeper cannot execute a batch of 2+ orders using a fresh historical payload that is itself the head of Pyth storage**, due to requiring `storedPublishTime >= maxPublishTime` on the lagging basket component.

---

## 3. Practical Impact

- **Failure of batch clearing (`executeOrderBatch`):** Any batch of 2 or more orders falling within the cache reuse window (`commitTime` strictly earlier than the cached tick) reverts with `PletherOracle__StoredFeedBehind` under natural Pyth publication asynchrony (`1–5` seconds). Reproduction boundary: the failure occurs as long as the lagging feed in storage has not yet caught up to `cache.publishTime` (`storedPublishTime < cache.publishTime`); if the slow feed already has `publishTime >= cache.publishTime` in storage, the cache is reused without error. A separate keeper workaround of "push latest first" only works when a newer tick has already been published off-chain—if the slow feed is still at `1002` at execution time, there is nothing newer to push.
- **No loss or freezing of funds, and why this is `Low` rather than `Medium`:** Funds are neither lost nor locked; single `executeOrder(orderId)` calls (and batches of length 1) work normally because the cache is never read for reuse, and batch execution only fails while the lagging feed in storage remains below `cache.publishTime`. Keepers are forced to clear the queue one order at a time, losing gas savings and the benefits of batched execution.

---

## Proof of Concept

## 4. Proof of Concept

The test uses the repository's standard `OrderRouterPythTestBase` fixture, which deploys `LegacyOrderRouterHarness` via `_deployLegacyOrderRouter(...)` (see `packages/perps/test/perps/shared/OrderRouterTestBase.sol:193`), and verifies two complementary scenarios:
1. `test_PoC_BatchRevertsWhileSingleOrderSucceeds` — primary PoC: proves that under an allowed 2-second feed divergence (`1004` / `1002`), calling `executeOrderBatch(/* maxOrderId = */ 2)` (where `2` is the inclusive `maxOrderId`, not batch length) reverts with `PletherOracle__StoredFeedBehind(FEED_B, 1002, 1004)` and leaves the queue untouched (`nextExecuteId == 1`, position `0`), whereas calling single-order `executeOrder(order1)` with the exact same inputs succeeds cleanly.
2. `test_Control_SynchronousFeeds_BatchExecutesCleanly` — control test: proves that with synchronous feeds (`1004` / `1004`), both orders execute cleanly in a single batch (`terminalCount == 2`, position `20_000e18`), isolating timestamp divergence as the sole cause of the failure.

### 4.1. Step-by-Step Reproduction Scenario

1. **Creating two orders in the same commit window:** At baseline `commitTime = 1000`, Alice submits two consecutive LONG orders via `router.commitOrder` (`order1` and `order2`, both size `10_000e18` and margin `500e6`). The execution queue is at `nextExecuteId == 1`, Alice has `2` pending orders, and no open position yet (`size == 0`).
2. **Execution window opens with natural Pyth feed divergence:** Time advances by 5 seconds (`vm.warp(commitTime + 5)`, `block.number + 1`), and `mockPyth` is configured with two historical prices exhibiting a protocol-allowed 2-second spread (within `maxComponentPublishTimeDivergence = 5 seconds`):
   - `FEED_A`: `publishTime = 1004` (`commitTime + 4`);
   - `FEED_B`: `publishTime = 1002` (`commitTime + 2`).
3. **Starting the batch and populating the cache on the first order:** A keeper calls `router.executeOrderBatch(2, _pythUpdateData())`. On `order1`, `state.oracleCache` is still empty (`hasHistoricalBasket == false`), so `PletherOracle.updateBatchOrderExecutionPrice` enters `_resolveOrderExecutionBasket` → `_resolveHistoricalOrderBasket`. The basket is built cleanly (`1004 - 1002 = 2 <= 5` seconds), `pyth.updatePriceFeeds` updates Pyth storage (`FEED_A = 1004`, `FEED_B = 1002`), each feed passes its per-component check `_requireStoredFeedCoverage(parsedFeeds[i].id, parsedFeeds[i].price.publishTime)`, and `nextCache` records `publishTime = 1004` (`maxPublishTime`) and `oldestPublishTime = 1002` (`minPublishTime`).
4. **Entering the reuse branch on the second order and reverting in `_requireStoredFeedCoverage`:** Moving to `order2`, `_canReuseHistoricalBatchBasket` in both `OrderRouterExecutionSidecar` and `PletherOracle` returns `true` (`commitTime < oldestPublishTime`, `commitTime < publishTime`, and the gap fits within `orderSettlementWindow`). The router forwards `fee = 0`, and inside the cache reuse branch the oracle checks **every** feed not against its own timestamp, but against the basket's upper bound `cache.publishTime` (`1004`). Because the lagging `FEED_B` in Pyth storage has `storedPublishTime = 1002 < 1004`, `_requireStoredFeedCoverage(FEED_B, 1004)` deterministically reverts with `PletherOracle__StoredFeedBehind(FEED_B, 1002, 1004)`.
5. **Router rethrows the revert and aborts the batch while `executeOrder` works:** Since the `catch` block in `OrderRouterExecutionSidecar._prepareBatchOracle` only intercepts the `68`-byte `PletherOracle__HistoricalParserFailure`, the `100`-byte `PletherOracle__StoredFeedBehind(bytes32,uint256,uint256)` with a different selector bubbles up and reverts the entire `executeOrderBatch(2)` transaction (`nextExecuteId == 1`, `pendingOrderCounts == 2`, `size == 0`). Immediately afterward, a single call `router.executeOrder(order1, _pythUpdateData())` with the exact same prices (`FEED_A = 1004`, `FEED_B = 1002`) succeeds (`LifecycleStatus.Executed`), advancing `nextExecuteId == 2` and opening a `10_000e18` position.
6. **Control run with synchronous feeds (`test_Control_SynchronousFeeds_BatchExecutesCleanly`):** With equal timestamps (`FEED_A = 1004`, `FEED_B = 1004`), both orders execute in a single `executeOrderBatch(2)` call (`terminalCount == 2`, `nextOrderId == 0`, `nextExecuteId == 0`, final position `20_000e18`), confirming that the caching mechanism itself works and that permissible feed divergence is what triggers the revert.

### 4.2. Instructions to Run the PoC

#### 4.2.1. Clone the repository and check out the target commit

```bash
git clone https://github.com/immunefi-team/audit-comp-plether.git
cd audit-comp-plether
git checkout a82d4fefb02768e5b3bddcc97dee371eee24fea8
git submodule update --init --recursive
```

#### 4.2.2. Create the PoC test file

```bash
cat << 'EOF' > packages/perps/test/perps/spec/oracle/BatchOracleFeedCoveragePoC.t.sol
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

import {CfdTypes} from "@plether/perps/CfdTypes.sol";
import {OrderV3Types} from "@plether/perps/OrderV3Types.sol";
import {IPletherOracle} from "@plether/perps/interfaces/IPletherOracle.sol";
import {OrderRouterPythTestBase} from "../../shared/OrderRouterTestBase.sol";

contract BatchOracleFeedCoveragePoCTest is OrderRouterPythTestBase {

    function test_PoC_BatchRevertsWhileSingleOrderSucceeds() public {
        _startRecordingLogs();

        uint64 commitTime = 1000;
        vm.warp(commitTime);

        vm.startPrank(alice);
        uint64 order1 = router.commitOrder(CfdTypes.Side.LONG, 10_000e18, 500e6, 0, false);
        uint64 order2 = router.commitOrder(CfdTypes.Side.LONG, 10_000e18, 500e6, 0, false);
        vm.stopPrank();

        assertEq(order1, 1);
        assertEq(order2, 2);

        vm.warp(commitTime + 5);
        vm.roll(block.number + 1);

        // Permissible 2-second divergence (protocol limit = 5s)
        mockPyth.setUniquePrice(FEED_A, 100_000_000, 100_000, -8, commitTime + 4, commitTime);
        mockPyth.setUniquePrice(FEED_B, 100_000_000, 100_000, -8, commitTime + 2, commitTime);

        assertEq(router.nextExecuteId(), 1, "nextExecuteId is order 1");
        assertEq(router.pendingOrderCounts(alice), 2, "Alice has 2 pending orders");
        (uint256 sizeBefore,,,,,,) = engine.positions(alice);
        assertEq(sizeBefore, 0, "Alice position is 0 before execution");

        // 1. Batch execution executeOrderBatch(2) (maxOrderId = 2) reverts with StoredFeedBehind on cache reuse for order 2
        vm.expectRevert(
            abi.encodeWithSelector(
                IPletherOracle.PletherOracle__StoredFeedBehind.selector,
                FEED_B,
                uint256(commitTime + 2),
                uint256(commitTime + 4)
            )
        );
        router.executeOrderBatch(/* maxOrderId = */ 2, _pythUpdateData());

        // Queue did not advance; both orders remain unexecuted
        assertEq(router.nextExecuteId(), 1, "Batch reverted: nextExecuteId remained at 1");
        assertEq(router.pendingOrderCounts(alice), 2, "Batch reverted: Alice still has 2 pending orders");
        (uint256 sizeAfterBatchRevert,,,,,,) = engine.positions(alice);
        assertEq(sizeAfterBatchRevert, 0, "Batch reverted: Alice position remains 0");

        // 2. With the exact same prices, single executeOrder(order1) succeeds because the cache is not reused
        OrderV3Types.ExecutionResult memory singleResult = router.executeOrder(order1, _pythUpdateData());
        assertEq(uint8(singleResult.status), uint8(OrderV3Types.LifecycleStatus.Executed));
        assertEq(router.nextExecuteId(), 2, "Order 1 executed, next is order 2");
        assertEq(router.pendingOrderCounts(alice), 1, "Alice now has only 1 pending order");
        (uint256 sizeAfterSingle,,,,,,) = engine.positions(alice);
        assertEq(sizeAfterSingle, 10_000e18, "Order 1 opened position to 10,000");
    }

    function test_Control_SynchronousFeeds_BatchExecutesCleanly() public {
        _startRecordingLogs();

        uint64 commitTime = 1000;
        vm.warp(commitTime);

        vm.startPrank(alice);
        uint64 order1 = router.commitOrder(CfdTypes.Side.LONG, 10_000e18, 500e6, 0, false);
        uint64 order2 = router.commitOrder(CfdTypes.Side.LONG, 10_000e18, 500e6, 0, false);
        vm.stopPrank();

        assertEq(order1, 1);
        assertEq(order2, 2);

        vm.warp(commitTime + 5);
        vm.roll(block.number + 1);

        // Control: identical 2 orders with synchronous feeds (1004 / 1004) execute cleanly in batch via cache reuse
        mockPyth.setUniquePrice(FEED_A, 100_000_000, 100_000, -8, commitTime + 4, commitTime);
        mockPyth.setUniquePrice(FEED_B, 100_000_000, 100_000, -8, commitTime + 4, commitTime);

        OrderV3Types.BatchResult memory batchResult = router.executeOrderBatch(/* maxOrderId = */ 2, _pythUpdateData());
        assertEq(batchResult.terminalCount, 2, "Both orders executed in batch");
        assertEq(batchResult.nextOrderId, 0, "Queue is empty after batch");
        assertEq(router.nextExecuteId(), 0, "nextExecuteId is 0 when queue is empty");
        assertEq(router.pendingOrderCounts(alice), 0, "No pending orders left");
        (uint256 sizeAfterBatch,,,,,,) = engine.positions(alice);
        assertEq(sizeAfterBatch, 20_000e18, "Both orders opened total position 20,000");
    }
}
EOF
```

#### 4.2.3. Run the test

```bash
cd packages/perps
forge test --match-contract BatchOracleFeedCoveragePoCTest -vv
```

### 4.3. Terminal Output from Running the PoC (Commit `a82d4fef`)

```text
Ran 2 tests for test/perps/spec/oracle/BatchOracleFeedCoveragePoC.t.sol:BatchOracleFeedCoveragePoCTest
[PASS] test_Control_SynchronousFeeds_BatchExecutesCleanly() (gas: 5806534)
[PASS] test_PoC_BatchRevertsWhileSingleOrderSucceeds() (gas: 7342045)
Suite result: ok. 2 passed; 0 failed; 0 skipped; finished in 15.37ms (14.59ms CPU time)

Ran 1 test suite in 16.08ms (15.37ms CPU time): 2 tests passed, 0 failed, 0 skipped (2 total tests)
```

Both tests confirm the defect: with synchronous feeds, `executeOrderBatch(2)` reuses the cache and clears both orders normally, whereas under a valid 2-second spread (`1004` / `1002`, within the 5-second limit), the batch consistently reverts with `PletherOracle__StoredFeedBehind` even though single-order `executeOrder` succeeds with the exact same price inputs.

---

## 5. Remediation

### 5.1. Security Caveat: Why the Threshold Must Not Be Relaxed to `cache.oldestPublishTime`

`BatchOrderPriceCache` is not stored in oracle storage and is not protected by a signature or hash—it is a `caller-supplied` argument (`// Cache contents are caller-supplied; independently prove storage coverage, not price authenticity.`). The check `_requireStoredFeedCoverage(pythFeedIds[i], cache.publishTime)` does not authenticate `cache.price` (when `storedPublishTime >= cache.publishTime`, a direct caller can already supply an arbitrary price; the router avoids this by only forwarding the oracle's previous return value). Instead, it enforces a different invariant: the timestamp that becomes `basket.publishTime` is no newer than Pyth storage for every feed.

If the threshold were lowered to `cache.oldestPublishTime`, a caller could claim a `cache.publishTime` that the lagging feed in storage does not cover (by passing an already-covered `oldestPublishTime` alongside a newer, uncovered `publishTime` within the settlement window). Therefore, the `cache.publishTime` threshold must not be weakened.

### 5.2. Option 1 (Recommended Minimal Fix): Disable Cache Reuse on Timestamp Divergence in `_canReuseHistoricalBatchBasket` (in Both Contracts)

The router does not call the oracle's `_canReuseHistoricalBatchBasket`—it maintains its own copy of the predicate in `OrderRouterExecutionSidecar.sol:1074-1093`, which governs `fee` calculation in `_prepareBatchOracle` (`OrderRouterExecutionSidecar.sol:996-1005`):
```solidity
uint256 fee = 0;
if (!_canReuseHistoricalBatchBasket(oracle, order, state.oracleCache)) {
    fee = oracle.getOrderExecutionFee(pythUpdateData); // 2 * pyth.getUpdateFee
    ...
}
oracle.updateBatchOrderExecutionPrice{value: fee}(...);
```
If only `PletherOracle.sol` is patched:
1. The router still treats the cache as reusable and sends `value: 0`.
2. The oracle falls back to `_resolveOrderExecutionBasket` → `_resolveHistoricalOrderBasket` (`PletherOracle.sol:604-608`).
3. There, `msg.value < 2 * pythFee` triggers `PletherOracle__InsufficientFee`.
4. That selector does not equal `PletherOracle__HistoricalParserFailure`, so the `catch` block in `OrderRouterExecutionSidecar.sol:1019-1027` aborts the entire batch again *(in mocks, `getUpdateFee("")` is often `0`, so tests without a nonzero fee will pass and fail to catch this production revert)*.

Therefore, rejecting reuse upon divergence (`cache.oldestPublishTime != cache.publishTime`) must be added to **both** contracts—`PletherOracle.sol` and `OrderRouterExecutionSidecar.sol`:

```diff
--- a/packages/perps/src/PletherOracle.sol
+++ b/packages/perps/src/PletherOracle.sol
@@ -978,6 +978,9 @@ contract PletherOracle is IPletherOracle, ReentrancyGuard {
         if (!cache.hasHistoricalBasket) {
             return false;
         }
+        if (cache.oldestPublishTime != cache.publishTime) {
+            return false;
+        }
         if (
             commitTime < cache.minReusableCommitTime || commitTime >= cache.oldestPublishTime
                 || commitTime >= cache.publishTime
```

```diff
--- a/packages/perps/src/OrderRouterExecutionSidecar.sol
+++ b/packages/perps/src/OrderRouterExecutionSidecar.sol
@@ -1079,6 +1079,9 @@ contract OrderRouterExecutionSidecar is IOrderRouterErrors {
         if (oracle.isOracleFrozen() || !cache.hasHistoricalBasket) {
             return false;
         }
+        if (cache.oldestPublishTime != cache.publishTime) {
+            return false;
+        }
         uint64 commitTime = order.commitTime;
         if (
             commitTime < cache.minReusableCommitTime || commitTime >= cache.oldestPublishTime
```

In essence, this option disables cache reuse whenever basket timestamps diverge: when `oldestPublishTime != cache.publishTime`, both `_canReuseHistoricalBatchBasket` functions return `false`, the router attaches `getOrderExecutionFee`, the oracle goes through `_resolveOrderExecutionBasket`, and the batch transaction succeeds without reverting (while preserving the `cache.publishTime` check and cache reuse for synchronous feeds). For a `Low` severity defect, this fix is sufficient.

### 5.3. Option 2 (Alternative): Graceful Fallback in `updateBatchOrderExecutionPrice` + Sidecar Sync and `msg.value` Refund

Instead of reverting with `PletherOracle__StoredFeedBehind`, coverage can be checked via a helper `_hasStoredFeedCoverage(cache.publishTime)` returning `bool` (`storedPublishTime >= requiredPublishTime` for all feeds), falling back to `_resolveOrderExecutionBasket` on a miss.

For such a fallback to work properly across all Pyth storage states under a nonzero `pythFee`, changing **only** the reuse branch is insufficient—three coordinated changes are required:
1. In `PletherOracle.updateBatchOrderExecutionPrice`, replace the reverting `_requireStoredFeedCoverage` loop with `_hasStoredFeedCoverage(cache.publishTime)`:
   ```diff
   --- a/packages/perps/src/PletherOracle.sol
   +++ b/packages/perps/src/PletherOracle.sol
   @@ -259,11 +259,11 @@ contract PletherOracle is IPletherOracle, ReentrancyGuard {
    
            BasketPrice memory basket;
            bool reusedBasket;
   -        if (!policy.oracleFrozen && _canReuseHistoricalBatchBasket(request.commitTime, cache)) {
   +        if (
   +            !policy.oracleFrozen && _canReuseHistoricalBatchBasket(request.commitTime, cache)
   +                && _hasStoredFeedCoverage(cache.publishTime)
   +        ) {
                // Cache contents are caller-supplied; independently prove storage coverage, not price authenticity.
   -            for (uint256 i; i < pythFeedIds.length; ++i) {
   -                _requireStoredFeedCoverage(pythFeedIds[i], cache.publishTime);
   -            }
                basket = BasketPrice({
   ```
2. In `OrderRouterExecutionSidecar._canReuseHistoricalBatchBasket`, add `if (cache.oldestPublishTime != cache.publishTime) return false;` so that on divergence the router attaches `fee` in case the oracle falls back to `_resolveOrderExecutionBasket` (otherwise, with a nonzero `pythFee`, the `else` branch reverts with `PletherOracle__InsufficientFee`).
3. In `PletherOracle.updateBatchOrderExecutionPrice`, refund unused `msg.value` when `reusedBasket == true` (and reconcile `oracleFundingTotal` / refund accounting in `OrderRouterExecutionSidecar`): if the oracle kept its original `_canReuseHistoricalBatchBasket` while the sidecar disallowed reuse on divergence, then whenever the lagging feed has already updated in storage, the router would send `fee > 0`, the oracle would take the cache branch (`reusedBasket = true`), return `basket.pythFee = 0` (`PletherOracle.sol:272`), and **retain the received ETH without refunding it** (per `PletherOracle.sol:233-234`, `285-287`, no refund is issued on cache reuse), after which the final `_refundEth(host, state.executor, msg.value - state.oracleFundingTotal)` in `OrderRouterExecutionSidecar.sol:326` would attempt to refund ETH that already left to the oracle.

*(Note: disabling reuse on divergence in `_canReuseHistoricalBatchBasket` across both contracts effectively reduces divergent-basket behavior to Option 1.)*

### 5.4. Note on `OrderRouterExecutionSidecar.sol` and Full Caching Under Divergence

Adding the `PletherOracle__StoredFeedBehind` selector to `catch` and setting `PendingReason.OracleFailure` is incorrect: a valid order would erroneously remain stuck in `pending`. Cache misses must be handled either by consistently disabling reuse in both `PletherOracle` and `OrderRouterExecutionSidecar` (Option 1), or by implementing a graceful fallback with unused `fee` refunds (Option 2). True reuse of a divergent basket without re-parsing is only possible if `BatchOrderPriceCache` stores individual component publish times or a hash commitment to them.


