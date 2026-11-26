# \#97939 [SC-Medium] PR #120 (MED-23 fix) makes the Router's own mixed-time batch cache fail its stored-feed coverage check, reverting the whole `executeOrderBatch` instead of falling back to a fresh historical resolution

**Submitted on Nov 4th 2026 at 00:38:04 UTC by @Naresh_Kandula for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #97939
- **Report Type:** Smart Contract
- **Report severity:** Medium
- **Target:** https://github.com/immunefi-team/audit-comp-plether/tree/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src
- **Impacts:**
  - Contract fails to deliver promised returns, but doesn't lose value

## Description

## Severity

**Low**: *Contract fails to deliver promised returns, but doesn't lose value.* If the judge reads the impact as efficiency only, we accept **Insight**.

Asset: `packages/perps/src/PletherOracle.sol` (in scope), reached through `OrderRouter.executeOrderBatch` → `OrderRouterExecutionSidecar`.

**Why this row.** The documented batch behaviour is "reuse when coverage holds, otherwise request a new historical resolution" (ORACLE_SYNCHRONIZATION.md L62-L63; SECURITY.md L649). For an honest, Router-produced cache built from a valid first-post-commit payload, the documented fallback is not delivered: the call reverts and no order in it executes. No value is lost, and every order stays executable.

**Why not higher.**
- *Griefing / temporary freezing ≥ 1 h (Medium):* the trigger is Pyth component timing, not an attacker action. Progress resumes at once with a different call shape (PoC tests C and E), so no funds or orders are held for any meaningful time.
- *Theft of gas:* only the batching keeper's own gas is spent.
- *Critical/High rows:* no value moves.

**Program rule on fixed findings.** The behaviour is **introduced by the published MED-23 fix** (details below). The program accepts a report on a fixed finding when it shows the fix introduces a new bug, and that is the claim here.

## Summary

PR #120 (`81919218`) implemented Cecuro's MED-23 remediation by changing `BasketPrice.publishTime` from the **earliest** to the **newest** component publish time. The batch-reuse path was not updated:

1. The first historical parse synchronizes Pyth storage, and checks it, per feed against **that feed's own tick**.
2. The resulting basket is cached with `publishTime` = the **newest** tick.
3. For the next compatible FIFO order, the reuse branch demands that **every** feed's stored tick be ≥ that newest tick, and `revert`s otherwise. The reuse predicate never considers coverage, so reuse is selected and then can only revert. It cannot decline and fall back.

**Reachability change.** Before #120 (`6d6622f2`), `cache.publishTime` was the earliest component. After a successful first parse every feed was stored at ≥ its own tick ≥ the earliest, so **a Router-produced cache always passed coverage**. The documented `StoredFeedBehind` / "failed cache coverage" revert (ORACLE_SYNCHRONIZATION.md L109) was reachable only through caller-supplied caches on the lower-level Oracle helper. Since #120, the Router's **own honest cache** fails coverage whenever the first-post-commit components are not all on the same second.

## Root cause (permalinks at a82d4fef)

Base: `https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/`

1. **The basket time is the NEWEST component**
   - [`src/PletherOracle.sol#L34`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/PletherOracle.sol#L34): `/// @param publishTime Newest component publish time as a Unix timestamp`
   - [`#L874-L875`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/PletherOracle.sol#L874-L875):
     ```solidity
     basket.publishTime = uint64(maxPublishTime);
     basket.oldestPublishTime = uint64(minPublishTime);
     ```
   - Before #120, at the audited commit `6d6622f2` (`PletherOracle.sol:30`, `:691`), the field read `Earliest component publish time` and was set by `basket.publishTime = uint64(minPublishTime);`.

2. **The first parse syncs each feed only to its own tick, then caches the newest**
   - [`#L623-L626`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/PletherOracle.sol#L623-L626):
     ```solidity
     pyth.updatePriceFeeds{value: pythFee}(pythUpdateData);
     for (uint256 i; i < parsedFeeds.length; ++i) {
         _requireStoredFeedCoverage(parsedFeeds[i].id, parsedFeeds[i].price.publishTime);
     }
     ```
   - The cache is written at [`#L294`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/PletherOracle.sol#L294): `publishTime: basket.publishTime,`.

3. **The reuse branch requires every feed to be ≥ the newest tick, and reverts**
   - [`#L262-L266`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/PletherOracle.sol#L262-L266):
     ```solidity
     if (!policy.oracleFrozen && _canReuseHistoricalBatchBasket(request.commitTime, cache)) {
         // Cache contents are caller-supplied; independently prove storage coverage, not price authenticity.
         for (uint256 i; i < pythFeedIds.length; ++i) {
             _requireStoredFeedCoverage(pythFeedIds[i], cache.publishTime);
         }
     ```
   - [`_requireStoredFeedCoverage` #L645-L653](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/PletherOracle.sol#L645-L653) reverts with `PletherOracle__StoredFeedBehind(feedId, stored, required)`.
   - The predicate [`_canReuseHistoricalBatchBasket` #L974-L991](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/PletherOracle.sol#L974-L991) checks commit time, window and block time, but **not** storage coverage.

4. **The batch does not catch it**
   - [`src/OrderRouterExecutionSidecar.sol#L1009-L1030`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/OrderRouterExecutionSidecar.sol#L1009-L1030) catches only a 68-byte `PletherOracle__HistoricalParserFailure` and re-raises everything else (`revert(add(reason, 32), mload(reason))`). Oracle preparation sits in the outer batch frame, so the whole call reverts, including the head order the call had already priced.

**Trigger conditions:**
- A later FIFO order passes `_canReuseHistoricalBatchBasket`: its commit is no earlier than the cache's `minReusableCommitTime` and earlier than the oldest component tick, and the basket lies within its window.
- The supplied payload contains, for some feed, only a tick older than the basket's newest component, for example the first-post-commit ticks alone, with no newer update for the lagging feed.

Test D shows the tight shape: both orders committed in the **same second** T, components at T+1 and T+2.

## Documentation

The docs promise a fallback:

- [`ORACLE_SYNCHRONIZATION.md` L62-L63 ("Batch reuse")](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/ORACLE_SYNCHRONIZATION.md#L59-L64):
  > Reuse for a later order requires all components to follow that order's commit, fit its historical window and satisfy cache coverage. Otherwise the Router requests a new historical resolution.
- [`SECURITY.md` L649](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/SECURITY.md#L649):
  > batch execution may reuse a parsed historical basket only for later FIFO orders whose `commitTime` is strictly before the cached tick and falls within its proven coverage,

The docs also say a coverage failure reverts:

- [`ORACLE_SYNCHRONIZATION.md` L109 (Failure behavior table)](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/ORACLE_SYNCHRONIZATION.md#L109):
  > `PletherOracle__StoredFeedBehind(...)`, insufficient funding or failed cache coverage | Reverts the transaction, including earlier batch progress.
- SECURITY.md L676-L678 says the same about preparation failures in general.

The two statements are consistent only if "failed cache coverage" is an abnormal case: a forged or caller-supplied cache on the lower-level helper. Before #120 that was true for the Router, because its own cache could not fail coverage (see Summary). After #120 an honest Router cache fails coverage on a valid payload. The revert row now covers ordinary operation, and the Router never reaches the documented "requests a new historical resolution" path.

The project's own positive test hides this. [`test/perps/spec/oracle/OracleSynchronization.t.sol` L434](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/test/perps/spec/oracle/OracleSynchronization.t.sol#L434), `test_BatchReusesBasketWhenEveryComponentFollowsBothCommits`, builds exactly this mixed-time basket (T+4 / T+5) and passes only because it first writes a newer T+6 tick to Pyth storage out of band (`baseMockPyth.updatePriceFeeds{value: FEE}(newer)`).

## Impact

- **What happens:** a batch built from the head order's first-post-commit payload, as the docs and the project's reveal-window model describe, reverts entirely when its components are on different seconds. No order in that call executes, the keeper pays for a reverted transaction, and the reuse saving (one Pyth parse and fee for N orders) is lost for that payload.
- **Who:** keepers calling `executeOrderBatch` and, indirectly, the traders queued behind them, whose execution waits for a retry. Context only, since the keeper is out of scope: Plether's own keeper (`plether-app` `apps/backend/src/Plether/Keeper.hs`, `selectBatchCandidates` / `submitIntent`) batches the contiguous reveal-ready orders behind the queue head, up to `cfgKeeperMaxBatchSize`, with the head's single cached payload.
- **Workarounds (stated up front):**
  1. Execute with `maxOrderId` bounded to the head, or one order per call. The same payload is re-parsed per order (test C).
  2. Include a newer update for each lagging feed in the same payload. The unique parse still prices the first post-commit tick, `updatePriceFeeds` stores the newer one, and reuse then succeeds (test E).

  Either way progress resumes in the next transaction. **No funds are lost or frozen.**
- **Frequency:** this depends on how often the first-post-commit ticks of the six configured feeds land on different seconds while two or more orders are queued. We have not measured it on Hermes data. The 5 s `maxComponentPublishTimeDivergence` shows only that the protocol accepts such baskets, not how common they are.

## Known-issue check

- **Cecuro MED-23**, "Mixed-time basket can block trader protections during valid mark updates". Summary table: Acknowledged; client comment: "MED-23 — Addressed." The comment describes the change that causes this regression:
  > "The live basket's progress marker uses the newest component timestamp, so a newer component can advance it even when another component has not updated."

  The fix switched `BasketPrice.publishTime` to the newest component but did not update the batch-reuse consumer of that field.
  - The report-cited fix commits `2fca7941` / `7de2f742` are squashed into `81919218` (#120) in this repo.
  - `git log -S'basket.publishTime = uint64(maxPublishTime)'` returns only `81919218`.
- **Cecuro MED-7** (partly fixed; program known issue "a caller can still pick an older valid signed price per feed"). That issue is keeper *choice* of a priced component. Here the priced ticks are the ones the unique parse selects; any additional newer ticks in the payload are stored but not priced. The failure is a revert of an honest payload, not a price choice. Client comment:
  > "A caller can still choose an older valid signed component within the allowed window."
- **Cecuro MED-18** (Acknowledged; client comment "Addressed: The Engine rejects replacing its cached mark with a different clamped price at the same oracle timestamp…"). Unrelated: there is no equal-time mark conflict, and the mark is never written because the call reverts first.
- **ORACLE_SYNCHRONIZATION.md L109 / SECURITY.md L676-L678** document that a coverage failure reverts the batch. They do not document that the Router's own cache built from a valid first-post-commit payload fails coverage, which could not happen before #120 (see Documentation).
- **SECURITY.md "Known limitation: live basket component selection" (L687) and "Router and keeper limitations" (L1343)** do not mention reuse or coverage.
- **AI scan d4541b49, SC Audit Studio pre-audit, Cecuro, collaborative and Cantina texts:** grepping for `StoredFeedBehind`, `batch reuse`, `historical cache` and `reuse.*cache` finds no matching finding.

## Recommendation

Move storage coverage into the reuse **predicate**, so a basket that fails coverage is simply not reused and the Router performs the documented fresh historical resolution:

```solidity
function _canReuseHistoricalBatchBasket(uint64 commitTime, BatchOrderPriceCache calldata cache)
    internal view returns (bool)
{
    // ... existing checks ...
    for (uint256 i; i < pythFeedIds.length; ++i) {
        if (pyth.getPriceUnsafe(pythFeedIds[i]).publishTime < cache.publishTime) {
            return false; // fall back to a fresh unique parse, as ORACLE_SYNCHRONIZATION.md L63 specifies
        }
    }
    return uint256(cache.publishTime) <= uint256(commitTime) + orderSettlementWindow;
}
```

- The reverting loop at L264-L266 then becomes dead code, because both read the same storage in the same call, so it can be removed.
- The fallback is the authenticated unique parse, so caller-supplied-cache authentication is not weakened. Its second Pyth fee is within the documented per-order funding bound.
- Do **not** compare against `cache.oldestPublishTime`. That would weaken coverage for the newer components.
- Drop the out-of-band `updatePriceFeeds(newer)` pre-seed from `test_BatchReusesBasketWhenEveryComponentFollowsBothCommits`, or adopt tests B/D below as regression tests.

## Proof of Concept

## Proof of Concept

File: `packages/perps/test/perps/spec/oracle/Delta120ReuseCoverage.t.sol` (verbatim below). It extends the project's own `OracleSynchronizationTestBase`: the real Router, sidecars, Engine, HousePool and PletherOracle. Pyth is the project's `MockPyth`, whose unique-parse and storage-update semantics match Pyth's for explicit `FeedUpdate[]` payloads. No `vm.store` is used.

| Test | Shape | Result |
|---|---|---|
| A, control | equal component times | batch reuses the basket: 2 orders, 1 parse |
| B, bug | one component 1 s older (T+4 vs T+5) | whole batch reverts `StoredFeedBehind(feed 1, T+4, T+5)`; engine mark time unchanged |
| D, bug | both commits in the same second; ticks T+1 / T+2 | whole batch reverts `StoredFeedBehind(feed 1, T+1, T+2)` |
| C | the same payload as B in two calls | both orders execute, so the price data is valid and only the reuse gate fails |
| E, workaround | B's payload plus a newer tick for feed 1 | batch reuses: 2 orders, 1 parse |

```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

import {OracleSynchronizationTestBase} from "../../shared/OracleSynchronizationTestBase.sol";
import {OrderV3Types} from "@plether/perps/OrderV3Types.sol";
import {IPletherOracle} from "@plether/perps/interfaces/IPletherOracle.sol";
import {MockPyth} from "@plether/test-utils/MockPyth.sol";

/// @notice Batch historical-basket reuse checks stored-feed coverage against the NEWEST component time
///         (BasketPrice.publishTime since #120 / MED-23), while the first parse only synchronizes each feed to its
///         OWN tick. Any mixed-time basket that qualifies for reuse therefore reverts the whole batch instead of
///         falling back to a fresh historical resolution (ORACLE_SYNCHRONIZATION.md "Batch reuse").
contract Delta120ReuseCoverageTest is OracleSynchronizationTestBase {

    uint64 internal firstCommit;

    function _priced(uint64 tick, uint64 previous, int64 price) private pure returns (bytes[] memory data) {
        data = _payload(tick, previous);
        MockPyth.FeedUpdate[] memory updates = abi.decode(data[0], (MockPyth.FeedUpdate[]));
        for (uint256 i; i < updates.length; ++i) {
            updates[i].price.price = price;
        }
        data[0] = abi.encode(updates);
    }

    /// @dev ALICE commits at T, BOB at T+3, block time T+6. The signed payload holds the first post-commit tick for
    ///      every feed (prevPublishTime = T). If `mixed`, feed 1 ticked at T+4 and feeds 2..6 at T+5 (1 s apart,
    ///      well inside maxComponentPublishTimeDivergence = 5 s). BOB's commit (T+3) precedes every component, so
    ///      BOB qualifies for cache reuse.
    function _setup(bool mixed) private returns (bytes[] memory historical) {
        firstCommit = uint64(block.timestamp);
        _commit(ALICE, 0);
        _advance(firstCommit + 3);
        _commit(BOB, 0);
        _advance(firstCommit + 6);

        historical = _priced(firstCommit + 5, firstCommit, 101_000_000);
        if (mixed) {
            MockPyth.FeedUpdate[] memory updates = abi.decode(historical[0], (MockPyth.FeedUpdate[]));
            updates[0].price.publishTime = firstCommit + 4;
            historical[0] = abi.encode(updates);
        }
    }

    // A (control): equal component times -> batch reuses the basket and executes both orders with ONE parse.
    function test_Control_EqualComponentTimes_BatchReuses() public {
        bytes[] memory historical = _setup(false);
        OrderV3Types.BatchResult memory result = router.executeOrderBatch{value: 4 * FEE}(2, historical);
        assertEq(result.terminalCount, 2, "both orders executed");
        assertEq(baseMockPyth.parseUniqueCallCount(), 1, "second order reused the cached basket");
        assertEq(engine.lastMarkPrice(), 101_000_000);
    }

    // B (bug): one component 1 s older -> the reuse coverage check reverts the ENTIRE batch, head order included.
    function test_Bug_MixedComponentTimes_WholeBatchReverts() public {
        bytes[] memory historical = _setup(true);
        uint64 markTimeBefore = engine.lastMarkTime();

        vm.expectRevert(
            abi.encodeWithSelector(
                IPletherOracle.PletherOracle__StoredFeedBehind.selector,
                bytes32(uint256(1)),
                uint256(firstCommit + 4),
                uint256(firstCommit + 5)
            )
        );
        router.executeOrderBatch{value: 4 * FEE}(2, historical);

        assertEq(engine.lastMarkTime(), markTimeBefore, "no progress: head order's execution rolled back too");
    }

    // D (tight, realistic shape): ALICE and BOB commit in the SAME second T; components tick at T+1 (feed 1) and T+2
    //   (feeds 2..6). The honest first-post-commit payload still reverts the whole batch.
    function test_Bug_SameSecondCommits_OneSecondSkew_WholeBatchReverts() public {
        uint64 t = uint64(block.timestamp);
        _commit(ALICE, 0);
        _commit(BOB, 0);
        _advance(t + 3);

        bytes[] memory historical = _priced(t + 2, t, 101_000_000);
        MockPyth.FeedUpdate[] memory updates = abi.decode(historical[0], (MockPyth.FeedUpdate[]));
        updates[0].price.publishTime = t + 1;
        historical[0] = abi.encode(updates);

        vm.expectRevert(
            abi.encodeWithSelector(
                IPletherOracle.PletherOracle__StoredFeedBehind.selector, bytes32(uint256(1)), uint256(t + 1), uint256(t + 2)
            )
        );
        router.executeOrderBatch{value: 4 * FEE}(2, historical);
    }

    // E (in-call workaround, stated for completeness): a keeper that ALSO appends a newer tick for the lagging feed
    //   gets it stored by updatePriceFeeds (parse still prices the first post-commit tick), so coverage passes.
    function test_Workaround_AppendNewerTickForLaggingFeed() public {
        bytes[] memory historical = _setup(true);
        MockPyth.FeedUpdate[] memory first = abi.decode(historical[0], (MockPyth.FeedUpdate[]));
        MockPyth.FeedUpdate[] memory both = new MockPyth.FeedUpdate[](first.length + 1);
        for (uint256 i; i < first.length; ++i) {
            both[i] = first[i];
        }
        both[first.length] = MockPyth.FeedUpdate(
            first[0].id,
            MockPyth.MockPrice(first[0].price.price, first[0].price.conf, first[0].price.expo, firstCommit + 5, firstCommit + 4)
        );
        historical[0] = abi.encode(both);

        OrderV3Types.BatchResult memory result = router.executeOrderBatch{value: 4 * FEE}(2, historical);
        assertEq(result.terminalCount, 2, "both orders executed with the augmented payload");
        assertEq(baseMockPyth.parseUniqueCallCount(), 1, "reuse succeeded");
    }

    // C (the documented fallback would have worked): the SAME signed payload prices BOB's order on a fresh
    //   historical resolution. Executing the two orders in separate calls succeeds, so the revert in B is caused
    //   solely by the reuse coverage gate, not by missing or invalid price data.
    function test_SamePayload_FreshResolutionPricesBothOrders() public {
        bytes[] memory historical = _setup(true);

        OrderV3Types.BatchResult memory r1 = router.executeOrderBatch{value: 2 * FEE}(1, historical);
        assertEq(r1.terminalCount, 1, "head order executes alone");
        OrderV3Types.BatchResult memory r2 = router.executeOrderBatch{value: 2 * FEE}(2, historical);
        assertEq(r2.terminalCount, 1, "second order executes on a fresh parse of the same payload");
        assertEq(baseMockPyth.parseUniqueCallCount(), 2, "two parses (and two Pyth fees) instead of one");
        assertEq(engine.lastMarkPrice(), 101_000_000);
    }
}
```

Command (from the repository root):

```bash
cd packages/perps && forge test --root . --match-path test/perps/spec/oracle/Delta120ReuseCoverage.t.sol -vv
```

Output (compiler warnings omitted):

```
Ran 5 tests for test/perps/spec/oracle/Delta120ReuseCoverage.t.sol:Delta120ReuseCoverageTest
[PASS] test_Bug_MixedComponentTimes_WholeBatchReverts() (gas: 4380355)
[PASS] test_Bug_SameSecondCommits_OneSecondSkew_WholeBatchReverts() (gas: 4369111)
[PASS] test_Control_EqualComponentTimes_BatchReuses() (gas: 4237108)
[PASS] test_SamePayload_FreshResolutionPricesBothOrders() (gas: 4327116)
[PASS] test_Workaround_AppendNewerTickForLaggingFeed() (gas: 4252071)
Suite result: ok. 5 passed; 0 failed; 0 skipped; finished in 10.46ms (18.77ms CPU time)
Ran 1 test suite in 11.38ms (10.46ms CPU time): 5 tests passed, 0 failed, 0 skipped (5 total tests)
```

Reverting frame of test B at `-vvvv`, argument lists and ids elided. The second `updateBatchOrderExecutionPrice` call takes the reuse branch and makes no parse:

```
    │   │   ├─ [17860] PletherOracle::updateBatchOrderExecutionPrice(...)
    │   │   │   ├─ [1619] MockPyth::getPriceUnsafe(0x…01) [staticcall]
    │   │   │   └─ ← [Revert] PletherOracle__StoredFeedBehind(0x…01, 1709794804 [1.709e9], 1709794805 [1.709e9])
```

`1709794804` = T+4, feed 1's stored tick after the first parse. `1709794805` = T+5, `cache.publishTime`, the newest component.
