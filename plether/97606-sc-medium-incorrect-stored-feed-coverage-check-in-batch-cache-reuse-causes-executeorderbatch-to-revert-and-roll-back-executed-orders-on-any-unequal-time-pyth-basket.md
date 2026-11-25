# \#97606 [SC-Medium] Incorrect stored-feed coverage check in batch cache reuse causes executeOrderBatch to revert and roll back executed orders on any unequal-time Pyth basket

**Submitted on Nov 2nd 2026 at 14:17:18 UTC by @L4rp for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #97606
- **Report Type:** Smart Contract
- **Report severity:** Medium
- **Target:** https://github.com/immunefi-team/audit-comp-plether/tree/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src
- **Impacts:**
  - Contract fails to deliver promised returns, but doesn't lose value

## Description

Summary

OrderRouter.executeOrderBatch is the documented path for keepers to execute consecutive FIFO orders in one transaction. For the first order whose commit precedes a fresh Pyth tick, PletherOracle parses the unique post-commit basket and caches it; ORACLE_SYNCHRONIZATION.md promises that every later compatible order in the batch then reuses that proven basket at zero additional Pyth fee (“proven reuse costs zero”).

That promise holds only when all six basket components share one publish time. The parse branch synchronizes Pyth storage per feed to each feed’s own tick, but the reuse branch requires every stored feed to be at or after the basket’s newest component time. Any feed whose first-after-commit tick is earlier than the newest one therefore fails _requireStoredFeedCoverage, the oracle reverts PletherOracle__StoredFeedBehind, the execution sidecar does not catch that selector, and the whole executeOrderBatch transaction reverts — undoing the first order that had already executed and finalized its receipt. Retrying does not help: storage is in the same state every time. The condition is the normal case for six independent FX feeds, not an edge case.

The repository’s own regression test for this path passes only because it pushes a newer payload for every feed into mock Pyth storage one line before calling the batch — a third-party storage push that production does not guarantee.

Root cause

Reuse branch checks coverage against the newest component time (PletherOracle.sol)
solidity
// packages/perps/src/PletherOracle.sol:262-274
if (!policy.oracleFrozen && _canReuseHistoricalBatchBasket(request.commitTime, cache)) {
    // Cache contents are caller-supplied; independently prove storage coverage, not price authenticity.
    for (uint256 i; i < pythFeedIds.length; ++i) {
        _requireStoredFeedCoverage(pythFeedIds[i], cache.publishTime);   // <-- cache.publishTime == max_i P_i
    }
    basket = BasketPrice({
        price: cache.price,
        confidence: cache.confidence,
        publishTime: cache.publishTime,
        oldestPublishTime: cache.oldestPublishTime,
        pythFee: 0
    });
    reusedBasket = true;
}
Parse branch synchronizes storage per feed to that feed’s own time (PletherOracle.sol)
solidity
// packages/perps/src/PletherOracle.sol:613-626
try pyth.parsePriceFeedUpdatesUnique{value: pythFee}(pythUpdateData, pythFeedIds, minPublishTime, maxPublishTime)
    returns (PythStructs.PriceFeed[] memory parsedFeeds) {
    basket = _computeBasketPriceFromFeeds(PriceMode.OrderExecution, parsedFeeds, maxComponentPublishTimeDivergence);
    pyth.updatePriceFeeds{value: pythFee}(pythUpdateData);
    for (uint256 i; i < parsedFeeds.length; ++i) {
        _requireStoredFeedCoverage(parsedFeeds[i].id, parsedFeeds[i].price.publishTime);   // <-- per-feed P_i
    }

After this step stored feed i is at max(prior_i, P_i). For any feed with P_i < max_j P_j, the reuse check in (1) can only pass if some third party already pushed that feed to Pyth storage at or after the newest component time. Plether’s own live-mode path uses parsePriceFeedUpdates, which never writes storage.

The sidecar re-bubbles everything except HistoricalParserFailure (OrderRouterExecutionSidecar.sol)
solidity
// packages/perps/src/OrderRouterExecutionSidecar.sol:1019-1030
} catch (bytes memory reason) {
    if (reason.length != 68 || _selector(reason) != IPletherOracle.PletherOracle__HistoricalParserFailure.selector) {
        assembly ("memory-safe") { revert(add(reason, 32), mload(reason)) }   // <-- StoredFeedBehind aborts the batch
    }
    return (OrderV3Types.PendingReason.OracleFailure, result, state.oracleCache);
}

Because the oracle call is made from the batch frame rather than from the per-order rollback frame, the revert unwinds every item already executed in the same call.

The Router cannot opt out of reuse. OrderRouterExecutionSidecar._canReuseHistoricalBatchBasket (L1074-1093) applies the identical predicate as the oracle (L974-990): minReusableCommitTime <= commit < oldestPublishTime, cache.publishTime <= now, cache.publishTime <= commit + orderSettlementWindow. Whenever reuse is predicted, the Router sends fee = 0 and the oracle takes the reuse branch; there is no path to re-parse instead.

Documentation acknowledges the revert but not its trigger: packages/perps/ORACLE_SYNCHRONIZATION.md:109 lists StoredFeedBehind as “Reverts the transaction, including earlier batch progress”, describing it as a defensive check against a lagging Pyth endpoint — not as something that fires on every unequal-time basket.

Attack / loss scenario

Preconditions: two or more orders in the FIFO whose commits precede the same Pyth basket tick (the whole point of batching), and a basket whose six first-after-commit component ticks are not all equal. Pyth FX feeds publish independently, so unequal component times are the expected state.

Alice commits order A at t0; Bob commits order B at t0 + 3.
The first post-commit ticks are: feed 0 at t0 + 4, feeds 1–5 at t0 + 5 (within the 5 s component divergence limit, so the basket is valid).
A keeper calls executeOrderBatch(2, payload) at t0 + 6 with that payload.
Order A: parse branch runs, basket priced at the t0 + 5 basket time, updatePriceFeeds syncs storage (feed 0 → t0 + 4, others → t0 + 5), A executes and its receipt is finalized.
Order B: _canReuseHistoricalBatchBasket is true (t0 + 3 < oldest = t0 + 4). Reuse branch demands stored_i >= cache.publishTime = t0 + 5 for all i; feed 0 is at t0 + 4 → PletherOracle__StoredFeedBehind(feed0, t0+4, t0+5).
The sidecar re-bubbles the revert; the entire transaction reverts. The -vvvv trace shows order A fully settled — engine mark updated, receipt finalized, keeper bounty credited — before the revert unwinds all of it. Order A is back to Pending, nextExecuteId is still 1, and parseUniqueCallCount is 0.
Every retry of the batch fails identically until a third party happens to push feed 0 forward in Pyth storage. The keeper’s only recourses are to execute one order per transaction (executeOrder, paying 2 × pythFee per order instead of once per batch), or to pad the payload with an extra, newer update for the lagging feed and pay for it.

No attacker is required; no capital is required. The condition is produced by Pyth’s normal publishing cadence.

Impact

Batched execution — the protocol’s documented zero-marginal-cost path for draining the FIFO — fails deterministically for ordinary baskets, and fails after consuming the gas for the first order’s full execution and receipt finalization.
Keepers pay strictly more than the design intends (one parse fee per order rather than per batch), and the FIFO drains more slowly than documented under load, which is when batching matters.
No funds move and no order is lost: single-order execution continues to work, so this is a loss of promised functionality, not of value. Classified as Low on that basis.
The regression test that is supposed to guard this behaviour (test_BatchReusesBasketWhenEveryComponentFollowsBothCommits) masks it by pre-seeding storage, so the defect would not be caught by CI.

Deployment status (disclosed for triage)

The in-scope asset is the pinned commit a82d4fe of immunefi-team/audit-comp-plether. Per README.md and packages/perps/DEPLOYMENT.md, the perps system is deployed on Arbitrum Sepolia (testnet) with a 30-second mark-update worker; the earlier Ethereum mainnet deployments referenced in the repository are the spot product and are documented as retired. I did not identify a mainnet perps deployment. The defect is present unconditionally in the in-scope code and does not depend on any configuration value: basketMaxConfidenceRatioBps, maxComponentPublishTimeDivergence, and orderSettlementWindow only affect which baskets are valid, not the reuse coverage bound.

## Proof of Concept

Step-by-step

Two traders commit orders 3 seconds apart (ALICE at fc, BOB at fc+3).
Build the only valid historical Pyth payload for both: feed 0’s first post-commit tick at fc+4, feeds 1–5 at fc+5 (unequal component times, within the 5 s divergence limit).
Do not pre-push any newer payload into Pyth storage (production has no such third party).
Call router.executeOrderBatch(2, payload).
Order 1 parses, executes, finalizes its receipt, and syncs storage per feed (feed 0 → fc+4, others → fc+5).
Order 2 qualifies for cache reuse; the reuse branch requires every stored feed ≥ fc+5; feed 0 is at fc+4 → PletherOracle__StoredFeedBehind(feed0, fc+4, fc+5).
The sidecar does not catch that selector → the whole transaction reverts, rolling back order 1.
Retrying produces the identical revert every time. Controls: the same orders execute one at a time; an equal-time basket batch succeeds; re-adding the repo test’s storage pre-seed makes the batch pass.

Files
packages/perps/test/perps/spec/oracle/StoredFeedBehindBatchReuse.t.sol — uses the repository’s existing OracleSynchronizationTestBase fixture and MockPyth; no contract or existing test file is modified.

Five tests: the failing scenario, a retry loop showing the failure is persistent, and three controls (same orders execute individually; equal-time basket batch succeeds; re-adding the repository test’s storage pre-seed makes the failing scenario pass).

solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

// PoC for finding D-1: executeOrderBatch cache reuse checks every stored Pyth feed against the NEWEST component
// publish time of the cached basket, so any historical basket whose components carry unequal publish times
// reverts the whole batch (including the already-executed first order) with PletherOracle__StoredFeedBehind
// as soon as a second compatible order tries to reuse it.
//
// Copy to packages/perps/test/perps/spec/oracle/ and run:
//   forge test --root packages/perps --match-path "*StoredFeedBehindBatchReuse*" -vvv
//
// This file is the repo's own OracleSynchronization.t.sol::test_BatchReusesBasketWhenEveryComponentFollowsBothCommits
// with ONE line removed: the `baseMockPyth.updatePriceFeeds{value: FEE}(newer)` pre-seed that pushes every feed into
// Pyth storage at a time >= the newest component BEFORE the batch runs. Production has no such third party.

import {OracleSynchronizationTestBase} from "../../shared/OracleSynchronizationTestBase.sol";
import {OrderV3Types} from "@plether/perps/OrderV3Types.sol";
import {IPletherOracle} from "@plether/perps/interfaces/IPletherOracle.sol";
import {MockPyth} from "@plether/test-utils/MockPyth.sol";

contract StoredFeedBehindBatchReuseTest is OracleSynchronizationTestBase {

    // Component 0 ticks one second earlier than the other five. Six FX feeds publish on independent cadences, so
    // unequal first-after-commit ticks are the ordinary production shape, not an edge case.
    uint64 internal constant LAG = 1;

    /// @dev Builds the repo test's exact two-order queue and unequal-time historical payload.
    ///      Order 1 (ALICE) commits at fc, order 2 (BOB) at fc+3; feed 0 ticks at fc+4, feeds 1..5 at fc+5.
    function _queueTwoOrdersWithUnequalBasket()
        internal
        returns (uint64 firstCommit, bytes[] memory historical, bytes32 laggingFeed)
    {
        firstCommit = uint64(block.timestamp);
        _commit(ALICE, 0);
        _advance(firstCommit + 3);
        _commit(BOB, 0);
        _advance(firstCommit + 6);

        historical = _pricedPayload(firstCommit + 5, firstCommit, 101_000_000);
        MockPyth.FeedUpdate[] memory updates = abi.decode(historical[0], (MockPyth.FeedUpdate[]));
        updates[0].price.publishTime = firstCommit + 5 - LAG;
        historical[0] = abi.encode(updates);
        laggingFeed = updates[0].id;
    }

    /// EXPECTED (per ORACLE_SYNCHRONIZATION.md "proven cache reuse consumes no Pyth fee"): both orders execute with
    ///          one parse, terminalCount == 2, nextOrderId == 0.
    /// ACTUAL:  the second order's reuse branch demands stored(feed0) >= fc+5 while the first order's own
    ///          synchronization left stored(feed0) == fc+4. The batch reverts with StoredFeedBehind(feed0, fc+4, fc+5)
    ///          and the already-finalized first order is rolled back with it.
    function test_PoC_UnequalTimeBasketRevertsWholeBatchOnReuse() public {
        (uint64 firstCommit, bytes[] memory historical, bytes32 laggingFeed) = _queueTwoOrdersWithUnequalBasket();

        uint256 markBefore = engine.lastMarkPrice();
        uint64 markTimeBefore = engine.lastMarkTime();
        assertEq(router.nextExecuteId(), 1, "two pending orders, head is order 1");

        vm.expectRevert(
            abi.encodeWithSelector(
                IPletherOracle.PletherOracle__StoredFeedBehind.selector,
                laggingFeed,
                uint256(firstCommit + 5 - LAG), // stored: what order 1's own updatePriceFeeds wrote for feed 0
                uint256(firstCommit + 5) // required: cache.publishTime == newest component
            )
        );
        router.executeOrderBatch{value: 2 * FEE}(2, historical);

        // Whole transaction rolled back: order 1 was NOT executed even though it resolved and applied cleanly.
        assertEq(router.nextExecuteId(), 1, "head still order 1 - first order's execution was reverted");
        assertEq(engine.lastMarkPrice(), markBefore, "mark update from order 1 rolled back");
        assertEq(engine.lastMarkTime(), markTimeBefore, "mark time rolled back");
        assertEq(baseMockPyth.parseUniqueCallCount(), 0, "parse rolled back");
    }

    /// The failure is deterministic and persists on retry: nothing in the protocol ever pushes feed 0 forward, so every
    /// batch attempt with the only valid historical payload keeps reverting until the orders expire.
    function test_PoC_RetryKeepsRevertingUntilThirdPartyPushesLaggingFeed() public {
        (uint64 firstCommit, bytes[] memory historical, bytes32 laggingFeed) = _queueTwoOrdersWithUnequalBasket();

        for (uint256 i; i < 3; ++i) {
            _advance(uint64(block.timestamp) + 2);
            vm.expectRevert(
                abi.encodeWithSelector(
                    IPletherOracle.PletherOracle__StoredFeedBehind.selector,
                    laggingFeed,
                    uint256(firstCommit + 5 - LAG),
                    uint256(firstCommit + 5)
                )
            );
            router.executeOrderBatch{value: 2 * FEE}(2, historical);
        }
        assertEq(router.nextExecuteId(), 1, "still nothing executed after repeated retries");
    }

    /// CONTROL 1: the identical orders and the identical payload execute fine one at a time via executeOrder, paying
    /// 2 * pythFee per order (4 * FEE total) instead of the 2 * FEE the batch is documented to cost.
    function test_Control_SameOrdersAndPayloadExecuteIndividually() public {
        (, bytes[] memory historical,) = _queueTwoOrdersWithUnequalBasket();

        OrderV3Types.ExecutionResult memory first = router.executeOrder{value: 2 * FEE}(1, historical);
        assertEq(uint256(first.status), uint256(OrderV3Types.LifecycleStatus.Executed), "order 1 executes alone");
        assertEq(router.nextExecuteId(), 2);

        OrderV3Types.ExecutionResult memory second = router.executeOrder{value: 2 * FEE}(2, historical);
        assertEq(uint256(second.status), uint256(OrderV3Types.LifecycleStatus.Executed), "order 2 executes alone");
        assertEq(router.nextExecuteId(), 0, "queue drained");

        assertEq(baseMockPyth.parseUniqueCallCount(), 2, "two full parses -> two Pyth fees paid instead of one");
        assertEq(engine.lastMarkPrice(), 101_000_000);
    }

    /// CONTROL 2: with EQUAL component times the same two-order batch reuses the basket and succeeds. Reuse only works
    /// when every component ticks at exactly the same second.
    function test_Control_EqualTimeBasketBatchReusesAndSucceeds() public {
        uint64 firstCommit = uint64(block.timestamp);
        _commit(ALICE, 0);
        _advance(firstCommit + 3);
        _commit(BOB, 0);
        _advance(firstCommit + 6);
        bytes[] memory historical = _pricedPayload(firstCommit + 5, firstCommit, 101_000_000);

        OrderV3Types.BatchResult memory result = router.executeOrderBatch{value: 2 * FEE}(2, historical);
        assertEq(result.terminalCount, 2);
        assertEq(result.nextOrderId, 0);
        assertEq(baseMockPyth.parseUniqueCallCount(), 1, "one parse, second order reused the basket");
    }

    /// CONTROL 3: the repo's own test passes only because a third party pre-seeds Pyth storage for every feed at a
    /// time >= the newest component. Re-adding that single line makes the failing scenario pass.
    function test_Control_RepoFixturePassesOnlyBecauseOfExternalPreSeed() public {
        (uint64 firstCommit, bytes[] memory historical,) = _queueTwoOrdersWithUnequalBasket();

        // The line removed from the repo test. Plether's own live path never writes Pyth storage; only an unrelated
        // Arbitrum user pushing all six feeds at/after fc+5 would do this.
        bytes[] memory newer = _pricedPayload(firstCommit + 6, firstCommit + 5, 102_000_000);
        baseMockPyth.updatePriceFeeds{value: FEE}(newer);

        OrderV3Types.BatchResult memory result = router.executeOrderBatch{value: 2 * FEE}(2, historical);
        assertEq(result.terminalCount, 2, "with the pre-seed the batch succeeds");
        assertEq(result.nextOrderId, 0);
        assertEq(baseMockPyth.parseUniqueCallCount(), 1);
    }

    function _pricedPayload(
        uint64 tick,
        uint64 previous,
        int64 price
    ) private pure returns (bytes[] memory data) {
        data = _payload(tick, previous);
        MockPyth.FeedUpdate[] memory updates = abi.decode(data[0], (MockPyth.FeedUpdate[]));
        for (uint256 i; i < updates.length; ++i) {
            updates[i].price.price = price;
        }
        data[0] = abi.encode(updates);
    }

}

The fixture, _commit, _advance, _payload, FEE, ALICE, and BOB are the repository’s own (test/perps/shared/OracleSynchronizationTestBase.sol); the basket construction is copied from OracleSynchronization.t.sol:434-453 so the orders reach the reuse branch through the protocol’s normal path rather than by mocking state. The only difference between the failing test and the repository’s own test is the removal of the storage pre-seed at OracleSynchronization.t.sol:445-446; test_Control_RepoFixturePassesOnlyBecauseOfExternalPreSeed re-adds exactly that line and passes.

Reproduction

git clone https://github.com/immunefi-team/audit-comp-plether.git
cd audit-comp-plether
git checkout a82d4fefb02768e5b3bddcc97dee371eee24fea8
git submodule update --init --recursive
foundryup --version v1.5.1            # CI toolchain; solc 0.8.35 via foundry.toml
# place StoredFeedBehindBatchReuse.t.sol in packages/perps/test/perps/spec/oracle/
forge test --root packages/perps --match-path "*StoredFeedBehindBatchReuse*" -vvvv

Expected output

[PASS] test_PoC_UnequalTimeBasketRevertsWholeBatchOnReuse()
[PASS] test_PoC_RetryKeepsRevertingUntilThirdPartyPushesLaggingFeed()
[PASS] test_Control_SameOrdersAndPayloadExecuteIndividually()
[PASS] test_Control_EqualTimeBasketBatchReusesAndSucceeds()
[PASS] test_Control_RepoFixturePassesOnlyBecauseOfExternalPreSeed()
Suite result: ok. 5 passed; 0 failed; 0 skipped

Notes for triage / retest

Verify the asymmetry in one grep: grep -n "_requireStoredFeedCoverage" packages/perps/src/PletherOracle.sol — L265 passes cache.publishTime for all feeds; L625 passes parsedFeeds[i].price.publishTime per feed.
Verify the catch list: sed -n '1019,1030p' packages/perps/src/OrderRouterExecutionSidecar.sol — only HistoricalParserFailure (length 68) is converted to a pending reason.
Verify the masking pre-seed: sed -n '440,447p' packages/perps/test/perps/spec/oracle/OracleSynchronization.t.sol.
Provenance: the reuse-coverage check and the masking test were introduced together in upstream PR #103, “synchronize stored Pyth feeds during historical execution” (Plether-Fi/plether-core, merged 2026-10-10, commit 64c6ea90) — the V3 atomic oracle-synchronization feature, landed three weeks before this competition’s snapshot. git log -S_requireStoredFeedCoverage -- packages/perps/src/PletherOracle.sol in the upstream repository confirms this. It is new code with a regression test that assumes a storage state production does not guarantee, not a long-standing design decision. Upstream master (6909f261, 2026-10-29) carries no contract changes relative to the in-scope commit, so the defect is live at the current head as well.
Real Pyth updatePriceFeeds is also monotonic per feed (it never lowers a stored publish time), so the mock faithfully reproduces production storage state after the parse branch.
Not a listed known issue. The program’s known-issues list cites SECURITY.md “Known Limitations”, “Router and keeper limitations”, and “live basket component selection”; none mentions batch cache reuse or StoredFeedBehind. ORACLE_SYNCHRONIZATION.md:109 documents StoredFeedBehind as a guard against a Pyth endpoint lagging the parsed update — a lagging endpoint — not as a condition produced by the oracle’s own per-feed synchronization on every unequal-time basket, and the repository’s regression test asserts the opposite of the behaviour reported here.
This is distinct from Cecuro LOW-70 (history unavailable on single execution, no reuse involved), MED-20 / MED-23 (live divergence limits blocking frozen closes / protections), and MED-4 (FIFO filled with stale orders). None of those describes the reuse coverage bound or the batch-wide revert it causes.
Suggested fix (either is sufficient): check each feed against cache.oldestPublishTime in the reuse branch (the bound the new commit already had to precede), or carry per-feed publish times in BatchOrderPriceCache; alternatively, treat StoredFeedBehind on a reuse as “re-parse” instead of a batch-aborting revert.
Line numbers refer to commit a82d4fe.
