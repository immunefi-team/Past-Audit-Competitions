# \#98319 [SC-Medium] Stale batch cache check on mixed-timestamp price baskets breaks batch execution and forces per-order keeper retries"

**Submitted on Nov 7th 2026 at 00:25:01 UTC by @Martelle88622 for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #98319
- **Report Type:** Smart Contract
- **Report severity:** Medium
- **Target:** https://github.com/immunefi-team/audit-comp-plether/tree/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src
- **Impacts:**
  - Block stuffing
  - Griefing (e.g. no profit motive for an attacker, but damage to the users or the protocol)

## Description

## Brief/Intro

This is a regression from a post-audit fix. It is not an old design choice.

Before PR #120, the batch cache stored the earliest component time of the price basket. The reuse check used that same earliest time. The two matched, so mixed-timestamp baskets worked.

PR #120 (`81919218`, October 21 2026) changed the stored basket time to the newest component time. It added a separate field for the oldest component time. But it left the reuse coverage check pointing at the newest time. That leftover is the whole bug. The check now demands coverage that the first price parse never proved.

The project's own release note for PR #120 still describes the old rule. It says reuse requires the oldest component time to follow the later order's commit. The shipped code checks the newest time instead.

What happens in practice:

When several orders run in one batch, the first order parses a historical price basket and keeps it in memory. Later orders in the same batch reuse that basket instead of paying for another price parse.

The cache keeps two times. One is the newest component time. It becomes the basket's publish time. The other is the oldest component time. It decides which later orders may reuse the basket.

The first parse stores each price feed at its own publish time. Suppose feed A is stored at second 1 and feed B at second 2. The cache records second 2 as its publish time. Before reuse, the code asks every feed to be stored at or after second 2. Feed A is only at second 1. The check fails and the whole batch reverts.

Nothing is wrong with the prices. Both components are fresh and the gap is inside the allowed spread. The same two orders run normally when every component carries the same timestamp. Each order also runs normally on its own. This case worked before PR #120 and broke after it.

## Vulnerability Details

Some words first, so this reads on its own:

- Pyth is the outside price service. A basket is built from several price feeds. Each feed has its own publish time.
- A delayed order is committed first and executed later by a keeper. The keeper may run one order or a batch of orders. `executeOrder` and `executeOrderBatch` have no caller gate, so any account can call them.
- A batch parses one historical basket and reuses it for later orders to save a price update fee. Reuse is allowed only when the later order was committed before the basket's proven coverage starts.
- `publishTime` is the basket time. It now holds the newest component time. `oldestPublishTime` holds the oldest component time. The cache keeps both.
- Amounts in the test use 6 decimals for money and 8 decimals for prices. The test setup uses a 15 second settlement window and a 5 second component spread limit.

What is wrong, in the order it happens:

1. `PletherOracle.sol:875` sets `basket.publishTime` to the newest component time and `basket.oldestPublishTime` to the oldest component time.
2. The first historical parse updates stored prices and checks each feed against its own parsed time (`PletherOracle.sol:618-628`). It never proves that every feed is stored at the newest component time.
3. The batch cache is saved with `publishTime` set to the newest time (`PletherOracle.sol:287-297`).
4. On reuse, the code asks every feed to be stored at or after `cache.publishTime` (`PletherOracle.sol:260-268`). With mixed component times, this is false for the older feeds.
5. The check reverts with `PletherOracle__StoredFeedBehind`. The batch preparation catches only `HistoricalParserFailure`, so this error passes through and rolls back the whole batch (`OrderRouterExecutionSidecar.sol:1009-1030`).
6. Reuse eligibility only asks that the later order was committed before `cache.oldestPublishTime` (`OrderRouterExecutionSidecar.sol:1074-1090`). Two orders committed in the same second pass this check, then fail the coverage check.

The mismatch is simple. The cache claims all feeds cover the newest time. The parse only proved that each feed covers its own time. The interface documents `publishTime` as the newest component time and `oldestPublishTime` as the reuse cutoff (`IPletherOracle.sol:118-119`). The release note says reuse follows the oldest component time. But the coverage check uses the newest.

## Why this is a regression (before versus after PR #120)

Before PR #120 (parent commit `2da338a8`), the code was consistent for mixed timestamps:

- `PletherOracle.sol:691` set `basket.publishTime` to the earliest component time. The interface said the same. It documented `publishTime` as the earliest component publish time.
- The reuse check asked every stored feed to cover that earliest time. The first parse had stored each feed at its own time. Every stored time is at or after the earliest. So the check passed for mixed baskets.
- The reuse eligibility rule used the earliest time as the cutoff. A later order could reuse the basket only if committed before the earliest proven component. That is the rule the PR #120 release note still describes.

PR #120 changed the stored time but not the check:

- `PletherOracle.sol:874-875` set `basket.publishTime` to the newest component time and added `basket.oldestPublishTime` for the oldest. The interface meaning changed with it (`IPletherOracle.sol:118-119`).
- The release note kept the old words. It still says reuse requires the oldest component time to follow the later order's commit. The eligibility rule was correctly moved to the new `oldestPublishTime` field (`PletherOracle.sol:974-991`, `OrderRouterExecutionSidecar.sol:1083-1086`).
- But the coverage loop was not moved. It still reads `cache.publishTime` (`PletherOracle.sol:264-266`). That field now means the newest time. The same mixed basket that passed before now fails on every feed older than the newest component.

So the regression is precise. One line was correct when `publishTime` meant earliest. It became wrong when PR #120 redefined the field to mean newest. Everything around it was updated. That line was not. The same PoC file proves this directly. Run it unchanged in a clean checkout of the parent commit `2da338a8`. The three batch revert tests fail with `next call did not revert as expected`. The equal-time control and the single-order test still pass. The batch-of-one test fills, but its mark time is the oldest time instead of the newest. The mixed basket that breaks every batch after PR #120 runs in one batch before it. No audit or scan covers this because all of them predate PR #120. The AI audit is at commit `6d6622f2` from October 16. The AI scan is from October 15. The new behavior only exists after commit `81919218` from October 21.

## Impact Details

- A normal mixed-time price basket breaks the batch execution path. Keepers must fall back to single-order calls. They pay a separate price update and transaction for each order. The test proves two single calls run fine on the same payload that breaks the batch.
- The batch path is the normal way to clear several queued orders with one update. Repeated failure slows queue progress and raises keeper cost. This matters most when many orders are waiting. The test proves both orders stay queued and no fee is consumed when the batch reverts.
- No money is lost or frozen. Single-order execution still works, and the failed call changes no state. Still, this is a regression in a fix whose purpose was to improve historical price handling. Ordinary price data that batches handled before now breaks every batch it touches. The repair is small. Move one check to the field PR #120 itself created (`oldestPublishTime`).
- Severity is Medium because this breaks an execution path rather than taking funds. It is not High or Critical. No other impact is claimed.

## References

- Commit: `a82d4fefb02768e5b3bddcc97dee371eee24fea8`
- Regression evidence (all verifiable with `git show` at the pinned checkout):
  - PR #120 is commit `81919218` from October 21 2026. Its parent is `2da338a8`. It is a post-audit fix. It came after the AI audit (`6d6622f2`, October 16) and the AI scan (October 15). So no included audit or scan can cover it.
  - Before: `git show 81919218~1:packages/perps/src/PletherOracle.sol` shows `basket.publishTime = uint64(minPublishTime)` at line 691. The interface documented `publishTime` as the earliest component publish time. The coverage check against that earliest time passed for mixed baskets. The PoC confirms it. Run the unchanged PoC file in a clean checkout of `2da338a8`. The three revert tests fail with `next call did not revert as expected` because the batches run. The equal-time control and the single-order test still pass. The batch-of-one fills with the mark time at the oldest component time. Observed: 2 passed, 5 failed, with `test_EqualComponentTimesBatchSucceeds` and `test_SingleOrdersBypassTheStaleReuseCheck` passing.
  - After: `git show 81919218 -- packages/perps/src/PletherOracle.sol` adds `basket.oldestPublishTime = uint64(minPublishTime)` and changes `basket.publishTime` to `uint64(maxPublishTime)`. The interface changes to newest (`IPletherOracle.sol:118-119`). The coverage loop (`PletherOracle.sol:264-266`) still reads `cache.publishTime`, which now means the newest time.
  - The release note diff for `packages/perps/ORACLE_SYNCHRONIZATION.md` in commit `81919218` still states the old rule. It says reuse requires the oldest component time to follow the later order's commit. The shipped code contradicts its own release note.
- `packages/perps/src/PletherOracle.sol` (cache creation `:288-296`, reuse coverage `:260-268`, per-component parse `:618-628`, basket times `:874-875`, reuse predicate `:974-991`, window and divergence defaults `:87,90`, batch entry `:247-299`)
- `packages/perps/src/OrderRouterExecutionSidecar.sol` (batch loop `:234-327`, oracle prep `:982-1037`, reuse predicate `:1074-1093`, single-order path `:961-980`)
- `packages/perps/src/OrderRouter.sol:233-263` (`executeOrder` and `executeOrderBatch` have no caller gate; `nonReentrant` is reentrancy protection only)
- `packages/perps/src/interfaces/IPletherOracle.sol:115-127` (`BatchOrderPriceCache` fields; `:118-119` document newest and oldest semantics), `packages/perps/src/interfaces/IPerpsKeeper.sol:29-46` (both entry points declared `external payable` with permissionless notes)
- AI audit report in the repo at `audit-comp-plether/audits/plether-ai-audit-report-2026-10-26.pdf`: MED-23, MED-7, and MED-20 concern live or frozen baskets and predate the cache. None covers the historical basket reuse check or `PletherOracle__StoredFeedBehind`. The pre-audit report and AI scan likewise predate the cache. They cover only live and frozen mixed-time handling, not batch reuse.
- `packages/perps/ORACLE_SYNCHRONIZATION.md`: reuse requires later orders to fall inside the cached basket's proven coverage. The code instead requires coverage at the newest time, which the first parse never proves.


## Link to Proof of Concept

https://gist.github.com/a-sklyarov2/b0f6e776da1aa12b01ea72cb66d7dabd

## Proof of Concept

## Proof of Concept

Runnable Foundry test at pinned commit `a82d4fefb02768e5b3bddcc97dee371eee24fea8`. No in-scope changes. Uses the repo's own mock price service to build the payload (allowed: mocks only for outside Pyth updates). Full file in the secret gist linked in the gist field below (filename `PocBatchCacheCoverage.t.sol`). The same file doubles as the regression proof. Run it unchanged at the parent of PR #120 (`2da338a8`). The three batch revert tests fail with `next call did not revert as expected` because the batches run. The mixed basket works before PR #120 and breaks after it.

To run: in a checkout at that commit, place the gist file at `packages/perps/test/perps/PocBatchCacheCoverage.t.sol`, then from the repo root run:

```bash
forge test --root packages/perps --match-path test/perps/PocBatchCacheCoverage.t.sol -vv
# [PASS] test_UnequalComponentTimesRevertSecondBatchOrder()
# [PASS] test_UnequalComponentTimesRevertWithDifferentSecondCommits()
# [PASS] test_MixedTimesBatchOfOneSucceeds()
# [PASS] test_QueueRecoversWithEqualTimePayload()
# [PASS] test_UnprivilegedKeeperTriggersTheRevert()
# [PASS] test_EqualComponentTimesBatchSucceeds()
# [PASS] test_SingleOrdersBypassTheStaleReuseCheck()
```

The test uses the normal trading setup and the repo's own mock price service to build the payload (allowed: mocks only for outside Pyth updates). It changes no protocol code and uses no stored-value cheating. No setup beyond the repo checkout is needed. The file is self-contained. The batch entry points have no caller gate (`OrderRouter.sol:233-263`; the sidecar only rejects direct calls to itself at `OrderRouterExecutionSidecar.sol:105-126`), and one test proves this by calling from an address with no role and no funds beyond the fee.

1. Two ordinary users commit one order each in the same second.
2. The test advances three seconds and builds a price payload where feed A is published at `commit + 1` and the other five feeds at `commit + 2`. The one second spread is inside the allowed five second spread. Every feed is inside its 15 second history window.
3. The batch call is expected to revert with `PletherOracle__StoredFeedBehind(feed A, commit + 1, commit + 2)`. The first order is rolled back with the second. The test asserts the queue head does not move, no fee is consumed, and neither trader holds a position afterwards.
4. A second revert test commits the two orders a full second apart. ALICE commits at second N and BOB at second N + 1. The shared payload puts feed A at N + 2 and the rest at N + 3. It reverts with the identical error and the same rollback assertions. This proves same-second commits are not required.
5. A batch-of-one test runs the same mixed payload for a single order in a batch call. It succeeds and the trader's position exists, so the success is a real fill and not a terminal failure. The test also reads the stored feed times directly. Feed A is stored at `commit + 1`, the other feeds at `commit + 2`, and the engine mark time is `commit + 2`. The mismatch the second order hits is therefore observed state, not inference. This proves the first parse handles the mixed basket fine, so the failure sits in the second order's reuse of the cached basket.
6. A recovery test reverts the two-order batch with the mixed payload, then retries the same two queued orders with an equal-time payload. Both fill and the queue drains. The failure bricks nothing. It stalls the queue until the keeper retries, order by order or with fresh timestamps.
7. A caller test repeats the revert from an address with no role and no funds beyond the fee. It reverts identically. This proves anyone can trigger the path.
8. The control test uses the same two orders and a payload where every feed is published at `commit + 2`. Both orders fill and the batch reports two terminal orders.
9. A final test runs the same mixed payload through two single `executeOrder` calls. Each call parses its own price at its own component times. Both orders run and the queue drains. This proves the defect sits in the batch cache. It also shows the only fallback: two separate keeper transactions and two price-update fees instead of one.
