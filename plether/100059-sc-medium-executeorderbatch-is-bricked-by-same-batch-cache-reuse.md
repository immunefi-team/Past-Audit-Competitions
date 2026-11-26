# \#100059 [SC-Medium] executeOrderBatch is bricked by same-batch cache reuse

**Submitted on Nov 16th 2026 at 07:02:34 UTC by @niroh for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #100059
- **Report Type:** Smart Contract
- **Report severity:** Medium
- **Target:** https://github.com/immunefi-team/audit-comp-plether/tree/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src
- **Impacts:**
  - Theft of gas
  - Smart contract unable to operate due to lack of token funds

## Description

## Brief/Intro
The gas-optimized batch execution route (`executeOrderBatch`) of the Plether OrderRouter can fail as a whole whenever an order batch contains a "clustered" second order whose fill would normally reuse the historical price basket parsed for the first order. The oracle caches the newest (maximum) component publish time of the parsed price basket, but stores each feed in Pyth storage at its own, potentially older publish time. When the reuse path later verifies that *every* feed in storage covers the cached (maximum) time, the lagging feeds fail the check and the resulting revert is not classified by the execution sidecar - so the entire batch transaction reverts, including orders that had already been settled earlier in the batch. Keepers lose their gas on every attempt, and the batch route's fee savings are unavailable until keepers change how they build payloads. No user funds are lost or frozen.

## Vulnerability Details
The flaw lives in [`packages/perps/src/PletherOracle.sol`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/PletherOracle.sol) and interacts with the batch loop in [`packages/perps/src/OrderRouterExecutionSidecar.sol`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/OrderRouterExecutionSidecar.sol). It is a **logic asymmetry between two coverage checks** (state-consistency / uncaught-revert class of bug).

**Parse path - cache is stamped with the MAX component time (PletherOracle.sol:866-870):**
```solidity
866 basket.publishTime     = uint64(maxPublishTime);
868 basket.oldestPublishTime = uint64(minPublishTime);
```
Component publish times may diverge by up to 5 seconds (`maxComponentPublishTimeDivergence = 5`, PletherOracle.sol:90). After parsing, the oracle calls `pyth.updatePriceFeeds` (PletherOracle.sol:616-628), which stores each feed at *its own* parsed publish time. The parse-path coverage check then verifies each feed only against *its own* time - trivially true - and sets the batch cache regardless of the cross-feed spread.

**Reuse branch - demands full-basket coverage against the MAX time (PletherOracle.sol:262-267, 645-653):**
```solidity
262 for (uint256 i; i < pythFeedIds.length; ++i) {
263     _requireStoredFeedCoverage(pythFeedIds[i], cache.publishTime);
264 }
```
```solidity
645 function _requireStoredFeedCoverage(...) {
    // reverts PletherOracle__StoredFeedBehind when stored < required
```

For a cache produced by a fresh parse with divergent component times, at least one feed is stored at `minPublishTime < cache.publishTime`, so the reuse branch always reverts.

**The revert is unclassified and rolls back the whole batch (OrderRouterExecutionSidecar.sol:1014-1027, 285-330):**
```solidity
1019 } catch (bytes memory reason) {
1020     if (
1021         reason.length != 68
1022             || _selector(reason) != IPletherOracle.PletherOracle__HistoricalParserFailure.selector
1023     ) {
            assembly ("memory-safe") {
                revert(add(reason, 32), mload(reason))
```
The sidecar only catches the 68-byte `HistoricalParserFailure`; `PletherOracle__StoredFeedBehind` rebubbles. Because the batch loop finalizes prefix items via `executeOrderItemFromSidecar` *before* the failing `_prepareBatchOracle` call (OrderRouterExecutionSidecar.sol:285-330), the atomic revert undoes those already-executed settlements, the engine mark update, and the bounty ledger changes.

**Triggering state:** the batch cache is per-call memory and is never persisted, so *every* reuse is a same-batch reuse of a just-parsed cache. Any batch whose first parse has component publish-time divergence > 0 and whose later order is reuse-eligible (same or earlier `commitTime` within the cache coverage - clustered commits, the exact case the reuse feature targets) deterministically bricks the whole batch. Since the six FX feeds publish asynchronously, divergence of 1-5 seconds is routine in real Pyth payloads.

This is not a malicious-attacker scenario; it triggers on ordinary clustered execution, though a keeper payload composition can also trigger it deliberately.

## Impact Details
Mapped to the bounty's medium categories:

- **Theft of gas (medium):** every reuse-eligible batch deterministically reverts after performing the full parse, storage update, and prefix-order execution work; keepers lose the gas and the Pyth fee funding on each attempt. Retries with the same payload shape fail identically.
- **Smart contract unable to operate (medium, scoped to the batch route):** the documented gas-efficient batch execution path is unusable in the common clustered-commit case. Liveness is preserved via fallbacks, but each costs more:
  - per-order `executeOrder`/single-item batches pay the full Pyth fee for every order (no cache reuse), inflating execution cost across the whole book;
  - keepers can append a newer, non-unique update for the lagging feed so the reuse coverage check passes (verified against the repo's MockPyth; plausible but untested against real Pyth);
  - payloads where all six feeds attest the same publish second also work - but that is generally not controllable since FX feeds publish asynchronously.

**Worst case:** the batch route is effectively dead for any batch containing clustered commits with divergent component times, until keepers adopt a workaround. **Realistic average case:** repeated reverting batch attempts, keeper gas waste, and elevated per-order execution fees. No funds are lost, frozen, or mis-priced; order A remains Pending and can be executed via the fallback routes.

## Proof of Concept

## Proof of Concept
Repo: `immunefi-team-audit-comp-plether`, frozen commit `a82d4fef…`
Setup:
```bash
git checkout a82d4fef
forge install
forge test --match-contract BatchReuseBrickTest -vvv
```
The PoC runs in the repo's Foundry suite against the repo `MockPyth` (no fork needed). *Note: Pyth behaviour is modelled with the repo's MockPyth, which stores per-feed publish times and only overwrites on newer updates - matching real Pyth's `updatePriceFeeds`. No fork test against real Pyth was run.*

`test/perps/spec/oracle/BatchReuseBrickTest.t.sol`:
```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

import {OracleSynchronizationTestBase} from "../../shared/OracleSynchronizationTestBase.sol";
import {OrderV3Types} from "@plether/perps/OrderV3Types.sol";
import {IPletherOracle} from "@plether/perps/interfaces/IPletherOracle.sol";
import {MockPyth} from "@plether/test-utils/MockPyth.sol";

contract BatchReuseBrickTest is OracleSynchronizationTestBase {

    /// Two orders committed in the same second; one valid signed payload whose components
    /// publish 2s apart (within the 5s divergence limit). No pre-populated newer storage.
    function _divergentPayload(uint64 commit) internal pure returns (bytes[] memory data) {
        data = _payload(commit + 3, commit);
        MockPyth.FeedUpdate[] memory updates = abi.decode(data[0], (MockPyth.FeedUpdate[]));
        updates[0].price.publishTime = commit + 1; // lagging FX component
        data[0] = abi.encode(updates);
    }

    function test_SameCommitBatchRevertsOnDivergentComponents() public {
        uint64 commit = uint64(block.timestamp);
        _commit(ALICE, 0);
        _commit(BOB, 0); // same commitTime
        _advance(commit + 6);

        bytes[] memory data = _divergentPayload(commit);
        uint64 markBefore = engine.lastMarkTime();
        uint256 balBefore = address(this).balance;

        vm.expectRevert(
            abi.encodeWithSelector(
                IPletherOracle.PletherOracle__StoredFeedBehind.selector, bytes32(uint256(1)), commit + 1, commit + 3
            )
        );
        router.executeOrderBatch{value: 4 * FEE}(2, data);

        // Whole batch rolled back: order A (which executed first) is still pending, no mark, no ETH spent.
        assertEq(engine.lastMarkTime(), markBefore);
        (uint256 size,,,,,,) = engine.positions(ALICE);
        assertEq(size, 0, "prefix order A rolled back");
        assertEq(uint256(router.lifecycleBook().lifecycleStatus(1)), uint256(OrderV3Types.LifecycleStatus.Pending));
        assertEq(address(this).balance, balBefore);
    }

    /// Control: the same payload with equal component publish times succeeds and reuses the cache.
    function test_Control_EqualComponentTimesBatchSucceeds() public {
        uint64 commit = uint64(block.timestamp);
        _commit(ALICE, 0);
        _commit(BOB, 0);
        _advance(commit + 6);

        OrderV3Types.BatchResult memory result = router.executeOrderBatch{value: 4 * FEE}(2, _payload(commit + 3, commit));
        assertEq(result.terminalCount, 2);
        assertEq(baseMockPyth.parseUniqueCallCount(), 1, "second order reused the cache");
    }

    /// Fallback: each order executes alone (single-item batches) with the same divergent payload.
    function test_Fallback_SingleOrderBatchesSucceed() public {
        uint64 commit = uint64(block.timestamp);
        _commit(ALICE, 0);
        _commit(BOB, 0);
        _advance(commit + 6);
        bytes[] memory data = _divergentPayload(commit);
        OrderV3Types.BatchResult memory r1 = router.executeOrderBatch{value: 2 * FEE}(1, data);
        assertEq(r1.terminalCount, 1);
        OrderV3Types.BatchResult memory r2 = router.executeOrderBatch{value: 2 * FEE}(2, data);
        assertEq(r2.terminalCount, 1);
        assertEq(baseMockPyth.parseUniqueCallCount(), 2, "no reuse, two fees");
    }

    /// Mitigation check: keeper appends a newer (non-unique) update for the lagging feed in the same payload.
    function test_Workaround_AppendNewerLaggingUpdate() public {
        uint64 commit = uint64(block.timestamp);
        _commit(ALICE, 0);
        _commit(BOB, 0);
        _advance(commit + 6);
        bytes[] memory data = _divergentPayload(commit);
        MockPyth.FeedUpdate[] memory u = abi.decode(data[0], (MockPyth.FeedUpdate[]));
        MockPyth.FeedUpdate[] memory w = new MockPyth.FeedUpdate[](u.length + 1);
        for (uint256 i; i < u.length; ++i) w[i] = u[i];
        w[u.length] = MockPyth.FeedUpdate(bytes32(uint256(1)), MockPyth.MockPrice(100_000_000, 100_000, -8, commit + 3, commit + 1));
        data[0] = abi.encode(w);
        OrderV3Types.BatchResult memory r = router.executeOrderBatch{value: 4 * FEE}(2, data);
        assertEq(r.terminalCount, 2);
        assertEq(baseMockPyth.parseUniqueCallCount(), 1);
    }
}
```

**Expected output:**
- `test_SameCommitBatchRevertsOnDivergentComponents` passes with the expected revert `PletherOracle__StoredFeedBehind(feedId(1), commit+1, commit+3)`; assertions confirm order A's settlement, the engine mark, and the caller's ETH are all rolled back to pre-call state.
- `test_Control_EqualComponentTimesBatchSucceeds` passes with `terminalCount == 2` and a single parse, proving the batch route and reuse mechanism work when components share a publish second.
- `test_Fallback_SingleOrderBatchesSucceed` passes with two parses (two Pyth fees), showing the per-order fallback route.
- `test_Workaround_AppendNewerLaggingUpdate` passes with one parse, showing a keeper payload workaround.


**Explanation:** order A performs a fresh parse of a payload with component publish times spanning `commit+1 .. commit+3`. `updatePriceFeeds` stores feed 1 at `commit+1`; the cache is stamped with `publishTime = commit+3`. Order B shares the commit time and is reuse-eligible; the reuse branch then demands every stored feed cover `commit+3`, and feed 1 (stored at `commit+1`) reverts. The sidecar does not classify that revert, so the whole `executeOrderBatch` transaction reverts atomically - including order A's already-finalized settlement. The control test shows the same flow succeeds with equal component times, isolating the coverage-check asymmetry as the cause. Note the report's original claim that keepers can *only* fall back to single orders or equal-time payloads is overstated: the appended-newer-update workaround also works, though it requires extra keeper logic and was not verified against real Pyth.
