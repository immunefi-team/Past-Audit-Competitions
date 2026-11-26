# \#98556 [SC-Medium] MED-23 fix introduces a new bug: batch cache reuse checks Pyth storage against the newest component time, so `executeOrderBatch` reverts on honest baskets with unequal component times

**Submitted on Nov 8th 2026 at 17:27:11 UTC by @VivaxiLabs for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #98556
- **Report Type:** Smart Contract
- **Report severity:** Medium
- **Target:** https://github.com/immunefi-team/audit-comp-plether/tree/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src
- **Impacts:**
  - Contract fails to deliver promised returns, but doesn't lose value

## Description

## Brief/Intro

Cecuro MED-23 was remediated with this change: *"The live basket's progress marker uses the newest component timestamp"*. The change was merged in PR #120 (commit `8191921`, which Cecuro also links under MED-28), and it switched `basket.publishTime` from the **oldest** component publish time to the **newest** one.

The batch cache-reuse coverage check, added earlier in PR #103 (`64c6ea9`), was not updated. It still requires **every** stored Pyth feed to cover `cache.publishTime`, which now means the newest component time. Its natspec ("verifies all stored feeds cover its timestamp") was also written when "timestamp" meant the oldest component. The synchronization step that precedes reuse only proves that each feed covers **its own** component time.

As a result, a cache that the Router built and synchronized itself, one step earlier in the same call, fails this check whenever the six components of that basket were not published in the same second. The protocol tolerates up to 5 s of component divergence for exactly this case.

The Router has already chosen reuse and forwarded no fee, and its `catch` re-throws every error except `HistoricalParserFailure`. The whole `executeOrderBatch` call therefore reverts, instead of doing what the documentation promises: *"Reuse for a later order requires … satisfy cache coverage. Otherwise the Router requests a new historical resolution."*

Cecuro MED-23 is marked *Acknowledged / Addressed*. Under the program's known-issue rule, this report shows that the fix at the frozen commit **introduces a new bug**. The bug does not exist at the audited commit `6d6622f2`, where `basket.publishTime` was `uint64(minPublishTime)` (L691 at that commit).

## Vulnerability Details

**1. The basket timestamp is now the newest component.** `packages/perps/src/PletherOracle.sol#L874-L875`:

```solidity
basket.publishTime = uint64(maxPublishTime);       // audited commit 6d6622f2: uint64(minPublishTime)
basket.oldestPublishTime = uint64(minPublishTime); // field added by PR #120
```

**2. The first historical parse only proves per-component storage coverage.** `PletherOracle.sol#L613-L626`:

```solidity
try pyth.parsePriceFeedUpdatesUnique{value: pythFee}(pythUpdateData, pythFeedIds, minPublishTime, maxPublishTime)
...
    pyth.updatePriceFeeds{value: pythFee}(pythUpdateData);
    for (uint256 i; i < parsedFeeds.length; ++i) {
        _requireStoredFeedCoverage(parsedFeeds[i].id, parsedFeeds[i].price.publishTime); // stored_i >= component_i
    }
```

**3. Reuse requires every feed to cover the newest component.** `PletherOracle.sol#L262-L266`:

```solidity
if (!policy.oracleFrozen && _canReuseHistoricalBatchBasket(request.commitTime, cache)) {
    // Cache contents are caller-supplied; independently prove storage coverage, not price authenticity.
    for (uint256 i; i < pythFeedIds.length; ++i) {
        _requireStoredFeedCoverage(pythFeedIds[i], cache.publishTime); // stored_i >= max_j component_j
    }
```

At the audited commit, `cache.publishTime` was the minimum component time, so condition 3 always followed from condition 2. After PR #120, condition 3 is strictly stronger than what step 2 establishes.

The only Pyth storage write in Plether is L623; live paths use `parsePriceFeedUpdates` (L769), which does not write storage. Pyth only stores an update that is newer than the stored one. So after step 2, a lagging feed's storage sits exactly at its own component time, unless a third party happens to push a newer update for that feed. The repository's `MockPyth._applyUpdate` applies the same "only if newer" rule.

Reuse then reverts with `PletherOracle__StoredFeedBehind(laggingFeed, component, newest)`. For the Router's own cache, this is the only way the check can fail.

**4. The Router chooses reuse without checking coverage, and cannot fall back.** `packages/perps/src/OrderRouterExecutionSidecar.sol`:

- `_canReuseHistoricalBatchBasket` (L1074-L1093) checks commit time, `oldestPublishTime`, `publishTime` and the settlement window, but not storage coverage. The Router selects reuse and forwards `fee = 0` (L997).
- The `catch` (L1019-L1027) only tolerates a 68-byte `PletherOracle__HistoricalParserFailure`. Every other error is re-thrown with `revert(add(reason, 32), mload(reason))`.

**5. The intended behaviour is successful reuse, and the repository test masks the bug.** `test/perps/spec/oracle/OracleSynchronization.t.sol#L434-L453` (`test_BatchReusesBasketWhenEveryComponentFollowsBothCommits`) builds exactly this unequal-time basket: component 0 at +4, the others at +5. It asserts that the later order *"reuses the signed basket"*.

It passes only because it first writes a **newer** price for every feed into Pyth storage (L445-L446), a pattern whose own comment (L404) presents as optional: *"Stored Pyth prices may already be newer"*. Nothing in the protocol guarantees that precondition. The other fixtures use equal component times.

`ORACLE_SYNCHRONIZATION.md` itself asks for unequal component times to be exercised separately (L123-L124), and says real target-chain payloads have them (L145).

**6. Why this is not the documented "batch-atomic oracle failure".** `ORACLE_SYNCHRONIZATION.md#L109` and `SECURITY.md#L676-L678` / `#L748-L749` document the **consequence** of an oracle-preparation failure: the complete batch reverts, and keepers needing a durable-progress boundary should bound `maxOrderId`.

These passages describe genuine failures, such as a caller-supplied cache that storage does not back. `updateBatchOrderExecutionPrice` is public, hence the comment *"Cache contents are caller-supplied"*.

They do not say that the Router's own freshly synchronized cache fails coverage. Instead, `ORACLE_SYNCHRONIZATION.md#L61-L63` states that the Router reuses only a cache that satisfies coverage, and otherwise requests a new resolution.

The defect is that the min→max change made the Router **manufacture** this failure from honest inputs. The rollback of the earlier orders is the documented consequence, not the bug itself.

**Step-by-step scenario.** No attacker is required.

1. `t0`: ALICE commits order 1. `t0+3`: BOB commits order 2.
2. The first post-commit Pyth tick is published at `t0+5` for five feeds and at `t0+4` for one feed. The divergence is 1 s, within the default 5 s.
3. `t0+6`: a keeper calls `executeOrderBatch(2, payload)` with one signed message per feed, which is the minimal payload `parsePriceFeedUpdatesUnique` needs.
4. Order 1 is parsed and synchronized. Storage now holds feed 0 at `t0+4` and the others at `t0+5`. Order 1 executes, and the cache becomes `{oldestPublishTime: t0+4, publishTime: t0+5}`.
5. Order 2: its commit `t0+3` is earlier than the oldest component, so the Router reuses the cache with `fee = 0`. The Oracle requires `stored(feed0) = t0+4 >= t0+5` and reverts with `StoredFeedBehind(feed0, t0+4, t0+5)`. The error is re-thrown and the call reverts.
6. Both orders remain pending. The **same payload** executes both orders when they are submitted one by one, at the cost of two historical parses.

**Preconditions and likelihood.** The condition is ordinary but not constant:

- At least two FIFO orders in the batch, where the later commit precedes the oldest component of the earlier order's first post-commit basket. Pyth FX feeds typically update sub-second during market hours, so in practice this means the two commits fall within the same publish second. That happens in busy periods, and anyone can cause it by committing right after another trader.
- At least one of the six feeds has no update in that first second, so its first post-commit component lands in a later second than the others. This is the case the protocol's 5 s divergence allowance exists for.
- Nobody has independently pushed a newer update for the lagging feed into Arbitrum Pyth storage before the batch.

## Impact Details

When the conditions above hold, batch execution of that FIFO segment fails. A keeper that simulates first sees the revert and must take one of these paths instead:

- execute the orders one by one with `executeOrder`, which costs one historical parse plus one Pyth storage update **per order** instead of one shared parse;
- lower `maxOrderId`;
- or wait until every lagging feed has published again and append those messages.

Without simulation, the transaction reverts and all progress in it is discarded.

Batch reuse exists to share one historical resolution across orders that can use it. In this case the documented fallback (*"Otherwise the Router requests a new historical resolution"*) is not delivered: the Router neither reuses nor re-resolves.

Orders are delayed rather than mispriced, no funds are lost, and the single-order path remains available.

This matches **Low — "Contract fails to deliver promised returns, but doesn't lose value"**.

## Recommendation

Require exactly what synchronization proves: every feed covers the **oldest** component, and at least one stored feed covers the **newest**. This keeps the "storage reached the cached tick" property the check exists for. `PletherOracle.sol`, reuse branch:

```diff
             // Cache contents are caller-supplied; independently prove storage coverage, not price authenticity.
+            uint256 newestStored;
             for (uint256 i; i < pythFeedIds.length; ++i) {
-                _requireStoredFeedCoverage(pythFeedIds[i], cache.publishTime);
+                _requireStoredFeedCoverage(pythFeedIds[i], cache.oldestPublishTime);
+                uint256 stored = pyth.getPriceUnsafe(pythFeedIds[i]).publishTime;
+                if (stored > newestStored) {
+                    newestStored = stored;
+                }
             }
+            if (newestStored < cache.publishTime) {
+                revert PletherOracle__StoredFeedBehind(bytes32(0), newestStored, cache.publishTime);
+            }
```

An alternative that matches the documented behaviour exactly: make `OrderRouterExecutionSidecar._canReuseHistoricalBatchBasket` return `false` when any stored feed is behind `cache.publishTime`, so that the order is re-resolved instead of reverting the batch.

Either way:

- update the natspec at `PletherOracle.sol#L231-L232` and the row at `ORACLE_SYNCHRONIZATION.md#L109`;
- add a reuse test with unequal component times that does **not** pre-seed newer prices in Pyth storage.

**Verification of the fix.**

- With the diff applied to `src/PletherOracle.sol`, the PoC batch executes both orders with a single historical parse. The PoC's `expectRevert`s no longer trigger, and both controls still pass.
- The repository's oracle specification suite (`test/perps/spec/oracle`) passes 241/241 before and after the patch.
- Using a patched copy, these still revert: a cache whose oldest component is not covered, a fabricated cache claiming a tick newer than every stored feed, and a later order whose commit is not before every component.

## References

- Basket timestamps: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/PletherOracle.sol#L874-L875
- Historical parse and synchronization coverage: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/PletherOracle.sol#L613-L626
- Reuse coverage check: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/PletherOracle.sol#L262-L266
- Natspec of the reuse path: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/PletherOracle.sol#L229-L236
- `_requireStoredFeedCoverage`: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/PletherOracle.sol#L645-L653
- Router reuse decision and re-throw: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/OrderRouterExecutionSidecar.sol#L997-L1030
- `_canReuseHistoricalBatchBasket`: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/OrderRouterExecutionSidecar.sol#L1074-L1093
- Documented fallback: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/ORACLE_SYNCHRONIZATION.md#L61-L63
- Failure table: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/ORACLE_SYNCHRONIZATION.md#L109
- Unequal component times expected: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/ORACLE_SYNCHRONIZATION.md#L145
- Repository test that masks the bug: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/test/perps/spec/oracle/OracleSynchronization.t.sol#L434-L453
- Cecuro AI audit (MED-23 remediation comment): https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/audits/plether-ai-audit-report-2026-10-26.pdf


## Proof of Concept

## Proof of Concept

The PoC runs inside the repository's own Foundry suite, at the frozen commit, with **no in-scope contract modified**. It reuses the repository's `OracleSynchronizationTestBase`, which deploys the real `OrderRouter`, its sidecars, `CfdEngine`, `MarginClearinghouse`, `HousePool` and `PletherOracle`. Only Pyth, which is out of scope, is the repository's `MockPyth`, driven with signed-style `(publishTime, prevPublishTime)` payloads.

**Setup.**

```bash
git clone --recurse-submodules https://github.com/immunefi-team/audit-comp-plether.git
cd audit-comp-plether
git checkout --detach a82d4fefb02768e5b3bddcc97dee371eee24fea8
git submodule update --init --recursive
# save the file below as packages/perps/test/perps/poc/L01_BatchCacheCoverage.t.sol
forge test --root packages/perps --match-path test/perps/poc/L01_BatchCacheCoverage.t.sol -vv
```

**Expected output** (Forge 1.5.1 as required by the program, fresh clone at the frozen commit, default 256 fuzz runs; first compile ≈ 1.5 min):

```
Ran 4 tests for test/perps/poc/L01_BatchCacheCoverage.t.sol:L01_BatchCacheCoveragePoC
[PASS] testFuzz_PoC_AnyUnequalBasketRevertsBatch(uint256,uint256) (runs: 256, μ: 4377028, ~: 4376104)
[PASS] test_Control_EqualComponentTimesReuseSucceeds() (gas: 4231826)
[PASS] test_Control_SamePayloadExecutesOrderByOrder() (gas: 4338945)
[PASS] test_PoC_BatchRevertsOnHonestCacheWithUnequalComponentTimes() (gas: 4381096)
Suite result: ok. 4 passed; 0 failed; 0 skipped; finished in 1.05s (1.06s CPU time)
Ran 1 test suite in 1.05s (1.05s CPU time): 4 tests passed, 0 failed, 0 skipped (4 total tests)
```

With `FOUNDRY_FUZZ_RUNS=10000 ... --match-test testFuzz`:

```
[PASS] testFuzz_PoC_AnyUnequalBasketRevertsBatch(uint256,uint256) (runs: 10000, μ: 4377345, ~: 4376104)
```

**Test file:** `packages/perps/test/perps/poc/L01_BatchCacheCoverage.t.sol`

```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

import {OracleSynchronizationTestBase} from "../shared/OracleSynchronizationTestBase.sol";
import {OrderV3Types} from "@plether/perps/OrderV3Types.sol";
import {IPletherOracle} from "@plether/perps/interfaces/IPletherOracle.sol";
import {MockPyth} from "@plether/test-utils/MockPyth.sol";

/// @title  PoC — executeOrderBatch reverts when a reusable historical basket has unequal component times
/// @notice Uses the repository's own OracleSynchronizationTestBase (real OrderRouter, sidecars, CfdEngine,
///         MarginClearinghouse, HousePool and PletherOracle). Only Pyth is mocked (out of scope), with
///         signed-style payloads carrying (publishTime, prevPublishTime). No in-scope contract is modified.
/// @dev    Run from the repository root at commit a82d4fefb02768e5b3bddcc97dee371eee24fea8:
///             forge test --root packages/perps --match-path test/perps/poc/L01_BatchCacheCoverage.t.sol -vv
///
///         Scenario: ALICE commits at t0, BOB at t0 + 3. The first post-commit Pyth tick of five feeds is
///         published at `newest`, the sixth ("lagging") feed published one or more seconds earlier at `oldest`
///         (still after BOB's commit, and within the default 5 s component divergence). The keeper submits one
///         signed message per feed, which is exactly what `parsePriceFeedUpdatesUnique` needs.
contract L01_BatchCacheCoveragePoC is OracleSynchronizationTestBase {

    uint64 internal t0;

    function _queueTwoOrders() internal {
        t0 = uint64(block.timestamp);
        _commit(ALICE, 0); // order 1, commitTime t0
        _advance(t0 + 3);
        _commit(BOB, 0); // order 2, commitTime t0 + 3
    }

    /// @param lagging Index of the feed whose first post-commit tick is older than the others
    /// @param oldest  Publish time of the lagging feed (> t0 + 3, so BOB's order is eligible for reuse)
    /// @param newest  Publish time of every other feed
    function _historicalPayload(
        uint256 lagging,
        uint64 oldest,
        uint64 newest
    ) internal view returns (bytes[] memory data) {
        data = _payload(newest, t0); // prevPublishTime = t0: first tick after both commits
        MockPyth.FeedUpdate[] memory updates = abi.decode(data[0], (MockPyth.FeedUpdate[]));
        updates[lagging].price.publishTime = oldest;
        data[0] = abi.encode(updates);
    }

    /// @notice Main PoC: an honest batch of two valid orders reverts entirely.
    function test_PoC_BatchRevertsOnHonestCacheWithUnequalComponentTimes() public {
        _queueTwoOrders();
        _advance(t0 + 6);
        bytes[] memory historical = _historicalPayload(0, t0 + 4, t0 + 5);
        bytes32 laggingFeed = _synchronizationFeedIds()[0];

        // Order 1: unique parse + synchronization. Pyth storage now holds feed 0 at t0+4, others at t0+5.
        //          Order 1 executes; the cache is {oldestPublishTime: t0+4, publishTime: t0+5}.
        // Order 2: commit t0+3 < oldest t0+4, so the Router reuses the cache and forwards no fee. The Oracle then
        //          requires stored(feed i) >= cache.publishTime (t0+5) for EVERY feed -> feed 0 (t0+4) fails.
        vm.expectRevert(
            abi.encodeWithSelector(IPletherOracle.PletherOracle__StoredFeedBehind.selector, laggingFeed, t0 + 4, t0 + 5)
        );
        router.executeOrderBatch{value: 4 * FEE}(2, historical);

        // The revert is not caught by the Router: order 1, which was valid and had executed, is rolled back too.
        assertEq(router.nextExecuteId(), 1, "both orders are still pending");
        (uint256 aliceSize,,,,,,) = engine.positions(ALICE);
        assertEq(aliceSize, 0, "order 1 execution was rolled back");
        (uint256 bobSize,,,,,,) = engine.positions(BOB);
        assertEq(bobSize, 0, "order 2 never executed");
    }

    /// @notice Any lagging feed and any allowed component gap (1..5 s) reverts the batch.
    function testFuzz_PoC_AnyUnequalBasketRevertsBatch(
        uint256 laggingSeed,
        uint256 gapSeed
    ) public {
        uint256 lagging = bound(laggingSeed, 0, 5);
        uint64 gap = uint64(bound(gapSeed, 1, 5)); // within maxComponentPublishTimeDivergence (default 5 s)
        _queueTwoOrders();
        _advance(t0 + 4 + gap);
        bytes[] memory historical = _historicalPayload(lagging, t0 + 4, t0 + 4 + gap);
        bytes32 laggingFeed = _synchronizationFeedIds()[lagging];

        vm.expectRevert(
            abi.encodeWithSelector(
                IPletherOracle.PletherOracle__StoredFeedBehind.selector, laggingFeed, t0 + 4, t0 + 4 + gap
            )
        );
        router.executeOrderBatch{value: 4 * FEE}(2, historical);
        assertEq(router.nextExecuteId(), 1, "nothing executed");
    }

    /// @notice Control 1: the SAME signed payload is valid for both orders; only the batch reuse path fails.
    function test_Control_SamePayloadExecutesOrderByOrder() public {
        _queueTwoOrders();
        _advance(t0 + 6);
        bytes[] memory historical = _historicalPayload(0, t0 + 4, t0 + 5);

        OrderV3Types.ExecutionResult memory first = router.executeOrder{value: 2 * FEE}(1, historical);
        assertEq(uint256(first.status), uint256(OrderV3Types.LifecycleStatus.Executed));
        _advance(t0 + 7);
        OrderV3Types.ExecutionResult memory second = router.executeOrder{value: 2 * FEE}(2, historical);
        assertEq(uint256(second.status), uint256(OrderV3Types.LifecycleStatus.Executed));
        assertEq(router.nextExecuteId(), 0);
        assertEq(baseMockPyth.parseUniqueCallCount(), 2, "order-by-order pays a second historical parse");
    }

    /// @notice Control 2: with equal component times (what the repository fixtures use) batch reuse works.
    function test_Control_EqualComponentTimesReuseSucceeds() public {
        _queueTwoOrders();
        _advance(t0 + 6);
        bytes[] memory historical = _historicalPayload(0, t0 + 5, t0 + 5);

        OrderV3Types.BatchResult memory result = router.executeOrderBatch{value: 4 * FEE}(2, historical);
        assertEq(result.terminalCount, 2);
        assertEq(result.nextOrderId, 0);
        assertEq(baseMockPyth.parseUniqueCallCount(), 1, "second order reused the cached basket");
    }

}
```

**What each test proves.**

- `test_PoC_BatchRevertsOnHonestCacheWithUnequalComponentTimes`: an honest two-order batch reverts with `StoredFeedBehind(feed0, t0+4, t0+5)`. Order 1, which had executed, is rolled back. Neither trader has a position.
- `testFuzz_PoC_AnyUnequalBasketRevertsBatch`: for any lagging feed (0..5) and any component gap within the allowed divergence (1..5 s), the batch reverts.
- `test_Control_SamePayloadExecutesOrderByOrder`: the same signed payload executes both orders one by one, at the cost of two historical parses. The payload is valid; only the batch reuse path fails.
- `test_Control_EqualComponentTimesReuseSucceeds`: with equal component times, which is what the repository fixtures use, batch reuse works with a single parse.

