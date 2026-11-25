# \#100009 [SC-Medium] Legal Pyth Timestamp Skew Causes Atomic Reverts in Batched Order Execution

**Submitted on Nov 16th 2026 at 02:39:51 UTC by @xRakan for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #100009
- **Report Type:** Smart Contract
- **Report severity:** Medium
- **Target:** https://github.com/immunefi-team/audit-comp-plether/tree/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src
- **Impacts:**
  - Griefing (e.g. no profit motive for an attacker, but damage to the users or the protocol)

## Description

## Summary

PletherOracle accepts historical baskets whose component publish times differ by up to five seconds. The fresh path verifies each stored feed against that feed's parsed timestamp, but the batch-cache reuse path checks every feed against the basket's newest timestamp. A later FIFO order that reuses a legally skewed basket can therefore fail with PletherOracle__StoredFeedBehind, reverting the complete public batch.

The demonstrated loss is gas paid by the keeper who submits that batch. All order, Engine, Pyth, token, and fee-transfer state rolls back; the orders remain pending and public single-order execution recovers them. No trader-token theft, forced expiry, or fund freeze is claimed.

## Root Cause

The cache-reuse condition can be satisfied by a later order whose commit time precedes the oldest component time. Once reuse is selected, the coverage loop nevertheless compares each feed to cache.publishTime.

[Frozen PletherOracle.sol, lines 262–265](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/PletherOracle.sol#L262-L265)


```solidity
if (!policy.oracleFrozen && _canReuseHistoricalBatchBasket(request.commitTime, cache)) {
    // Cache contents are caller-supplied; independently prove storage coverage, not price authenticity.
    for (uint256 i; i < pythFeedIds.length; ++i) {
        _requireStoredFeedCoverage(pythFeedIds[i], cache.publishTime);
    }
```

Basket construction now defines publishTime as the maximum component time and separately records the minimum as oldestPublishTime.

[Frozen PletherOracle.sol, lines 874–875](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/PletherOracle.sol#L868-L876)


```solidity
basket.publishTime = uint64(maxPublishTime);
basket.oldestPublishTime = uint64(minPublishTime);
```

The fresh resolution path uses each parsed feed's own publish time for coverage.

[Frozen PletherOracle.sol, lines 613–625](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/PletherOracle.sol#L613-L626)


```solidity
pyth.updatePriceFeeds{value: pythFee}(pythUpdateData);
for (uint256 i; i < parsedFeeds.length; ++i) {
    _requireStoredFeedCoverage(parsedFeeds[i].id, parsedFeeds[i].price.publishTime);
}
```

At the audited commit, the same cache check was used, but basket publishTime was the minimum component timestamp:

[Audited PletherOracle.sol, lines 233–241](https://github.com/immunefi-team/audit-comp-plether/blob/6d6622f2871dbdfc4904c20da2e1f7662f5763b6/packages/perps/src/PletherOracle.sol#L233-L241)  
[Audited PletherOracle.sol, minimum timestamp assignment](https://github.com/immunefi-team/audit-comp-plether/blob/6d6622f2871dbdfc4904c20da2e1f7662f5763b6/packages/perps/src/PletherOracle.sol#L679-L692)

The semantic change was introduced by [commit 819192182833597641504b38cce7751bbcacecb8](https://github.com/immunefi-team/audit-comp-plether/commit/819192182833597641504b38cce7751bbcacecb8), titled “fix(perps): synchronize historical Pyth and enforce signed live basket.” It introduced the newest/oldest distinction to preserve the newest mark timestamp while constraining later-order reuse to commits before the oldest component. The coverage loop was not updated to use the oldest or per-feed timestamp.

OrderRouter.executeOrderBatch is public. Its Oracle preparation occurs before the per-item rollback frame, so an uncaught StoredFeedBehind therefore reverts earlier work in the batch:

[Pinned OrderRouter.executeOrderBatch, lines 249–262](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/OrderRouter.sol#L249-L262)  
[Pinned batch atomicity description](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/SECURITY.md#L746-L750)

## Pinned Source Excerpts

The historical parser and update path uses the post-commit window and the five-second component bound. The fresh path then verifies each component against its own time. See [the complete pinned function](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/PletherOracle.sol#L600-L653).

The Router's local cache is initialized within the batch and only extended from successful Oracle returns. A user cannot provide an external cache to the Router. See [pinned Router cache preparation](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/OrderRouterExecutionSidecar.sol#L982-L1017) and [cache eligibility](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/OrderRouterExecutionSidecar.sol#L1074-L1093).

## Attack Path and Public Reachability

1. An ordinary trader submits two valid orders that are eligible to reuse one historical basket. The PoC uses two accounts and the launch minimum open notional of 1,000 USDG.
2. A permissionless keeper submits a batch spanning both orders. The batch must contain unique post-commit feed updates whose component times differ by one second, within the configured five-second limit.
3. The first order resolves and synchronizes the historical basket. Its cache records the maximum and minimum component times.
4. The second order meets the cache-reuse conditions. Every stored feed is then required to reach the maximum timestamp, so a feed stored at the older component time throws PletherOracle__StoredFeedBehind (selector 0xf2aa6d49).
5. The full transaction reverts. All earlier order progress and Pyth updates roll back; the submitting keeper pays transaction gas.

The trader controls order submissions, not Pyth timestamps. The PoC does not demonstrate that a trader can force a keeper to batch.

### Authorization and Runtime Gates

The PoC uses two ordinary test accounts to create the order prefix and an unprivileged keeper address to call the public Router. It does not impersonate an owner, guardian, oracle publisher, or protocol contract. The Router's batch cache is internal memory, populated only after a successful Oracle response.

The checked-in [six-feed live-watermark fixture](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/test/fixtures/oracle-sync/live-watermark-uneven.json) contains signed live-feed batches with one component at 1790091390 and five at 1790091391. This shows that one-second component skew exists in signed Pyth data. It is a live-watermark fixture, not an authenticated historical-order payload.


## Measured Impact and Matched Controls

| Scenario | Observed result |
| --- | --- |
| Equal component timestamps, two-order batch | Both orders settle |
| One-second component skew, one-order batch | Fresh resolution succeeds |
| One-second component skew, two-order batch | Reverts with PletherOracle__StoredFeedBehind (0xf2aa6d49) |
| Retry the same pending orders with public single-order execution | Both orders settle |

The revert rolls back both queue entries' execution, positions, the Engine mark, Pyth storage updates and counters, settlement-token transfers, Pyth/Oracle ETH movements, and the forwarded call value. Only the keeper's transaction gas is spent. The failed two-feed call consumes 2,161,220 local EVM gas. This is a local execution measurement, not a USD fee estimate; it excludes a live Arbitrum gas price and L1 data fee.

The reverting call retains no Pyth fee. Outside an oracle freeze, each fresh historical execution quotes one parse plus one storage update. For per-payload fee q, a successful two-order cache-reusing batch uses 2q, while two single-order retries use 4q; the failed batch retains none of that fee. The checked-in fixture documentation reports q = 0 at its captured blocks, so no extra oracle-fee loss is claimed for that configuration.

## Deployment Assumptions and Feasibility Limits

The pinned Oracle defaults to a fifteen-second historical order window and a five-second maximum component divergence. The tested one-second skew is within both limits. Users can create the queue arrangement, but Pyth publishers control feed timestamps. Inclusion of two user transactions at a particular second is not guaranteed.

The historical-order PoC uses MockPyth with synthetic update bytes and previous timestamps. It does not verify signatures or claim that the live fixture bytes form an authenticated historical-order payload. The fixture is a live-watermark example, not a captured uneven historical-order payload. The official Pyth interface describes per-feed latest storage, updates only when a feed's incoming time is newer, and a historical parser that does not store its returned prices. See the immutable [Pyth getter](https://github.com/pyth-network/pyth-crosschain/blob/4d7e5ba3fa35f294706d41e1cc299b0b090eca7d/target_chains/ethereum/contracts/contracts/pyth/Pyth.sol#L172-L181), [Pyth update interface](https://github.com/pyth-network/pyth-sdk-solidity/blob/d7dd6e149936552198c12fac1273997cefc03ceb/IPyth.sol#L80-L87), and [historical parse interface](https://github.com/pyth-network/pyth-sdk-solidity/blob/d7dd6e149936552198c12fac1273997cefc03ceb/IPyth.sol#L115-L137). The test double models those per-feed storage semantics but does not authenticate signed bytes.

The failure does not force order expiry or freeze funds. The queue remains live and the public executeOrder entrypoint successfully recovers both orders. A caller can also simulate the batch before submitting it.

## Prior-Audit Differentiation and Eligibility

Cecuro MED-20 concerns frozen close orders applying the live divergence limit instead of the frozen policy; the [listed Cecuro report](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/audits/plether-ai-audit-report-2026-10-26.pdf) records that path as addressed. Cecuro MED-7 concerns a caller selecting an older still-valid component for a live basket. Neither finding uses the historical batch-cache reuse path or this stored-feed coverage threshold.

The frozen repository's [Oracle synchronization failure table](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/ORACLE_SYNCHRONIZATION.md#L103-L110) discloses that failed cache coverage reverts the whole transaction. That is the strongest eligibility counterargument. It describes the generic failure result, but not that a basket accepted by fresh resolution under the five-second divergence rule can fail reuse because publishTime now denotes the newest component while coverage still compares every feed to that newest time.

The proposed impact maps to official Medium row 31, Griefing, because the caller of a qualifying public batch can irrecoverably spend transaction gas. No trader/LP tokens are stolen or lost, and no order is permanently frozen. The impact remains gas-only; acceptance depends on whether the failed keeper transaction is considered damage beyond ordinary batch-retry overhead.

## Mitigation

The minimal fix checks each stored feed against the oldest timestamp carried by the validated cache:


```solidity
_requireStoredFeedCoverage(pythFeedIds[i], cache.oldestPublishTime);
```

This restores the audited minimum-timestamp coverage rule. Each parsed component time is at or after the basket minimum, and the Router creates the cache from a successful parse inside the same batch. The Router's commit-time eligibility, fifteen-second order window, five-second divergence bound, fresh per-feed checks, and Pyth parsing/authentication calls remain unchanged. Exact per-feed cache thresholds would require adding a timestamp for every configured feed to the cache struct; this one-line fix avoids that interface change.

The tested unified diff is:

```diff
diff --git a/packages/perps/src/PletherOracle.sol b/packages/perps/src/PletherOracle.sol
index 6dc346ce..ad288de1 100644
--- a/packages/perps/src/PletherOracle.sol
+++ b/packages/perps/src/PletherOracle.sol
@@ -262,7 +262,7 @@ contract PletherOracle is IPletherOracle, ReentrancyGuardTransient {
         if (!policy.oracleFrozen && _canReuseHistoricalBatchBasket(request.commitTime, cache)) {
             // Cache contents are caller-supplied; independently prove storage coverage, not price authenticity.
             for (uint256 i; i < pythFeedIds.length; ++i) {
-                _requireStoredFeedCoverage(pythFeedIds[i], cache.publishTime);
+                _requireStoredFeedCoverage(pythFeedIds[i], cache.oldestPublishTime);
             }
             basket = BasketPrice({
                 price: cache.price,
```

## References
- [Official Immunefi Plether Scope](https://immunefi.com/audit-competition/plether/scope/).
- [Frozen cache-reuse path](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/PletherOracle.sol#L260-L299).
- [Fresh per-feed coverage](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/PletherOracle.sol#L610-L627).
- [Newest and oldest timestamp assignment](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/PletherOracle.sol#L868-L876).
- [Audited source at commit 6d6622f2](https://github.com/immunefi-team/audit-comp-plether/blob/6d6622f2871dbdfc4904c20da2e1f7662f5763b6/packages/perps/src/PletherOracle.sol#L231-L241).
- [Commit introducing the timestamp semantics](https://github.com/immunefi-team/audit-comp-plether/commit/819192182833597641504b38cce7751bbcacecb8).
- [Six-feed live-watermark fixture](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/test/fixtures/oracle-sync/live-watermark-uneven.json).
- [Cecuro audit report](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/audits/plether-ai-audit-report-2026-10-26.pdf).
- [Oracle synchronization failure policy](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/ORACLE_SYNCHRONIZATION.md#L103-L110).
- [Pyth per-feed getter](https://github.com/pyth-network/pyth-crosschain/blob/4d7e5ba3fa35f294706d41e1cc299b0b090eca7d/target_chains/ethereum/contracts/contracts/pyth/Pyth.sol#L172-L181) and [Pyth update/parse interface](https://github.com/pyth-network/pyth-sdk-solidity/blob/d7dd6e149936552198c12fac1273997cefc03ceb/IPyth.sol#L80-L137).


## Proof of Concept

## Proof of Concept
## Clean-Checkout Reproduction

1. Check out commit a82d4fefb02768e5b3bddcc97dee371eee24fea8 and initialize the repository's pinned Foundry dependencies.
2. Create packages/perps/test/perps/PythBatchCacheCoveragePoC.t.sol and paste the complete source below without edits.
3. From the repository root, run with Foundry 1.8.3 and Solidity 0.8.35:

```sh
FOUNDRY_FFI=false FOUNDRY_OUT=out-report FOUNDRY_CACHE_PATH=cache-report FOUNDRY_FUZZ_SEED=0x11c3a6d94f7e258b forge test --offline --use 0.8.35 --root packages/perps --match-path test/perps/PythBatchCacheCoveragePoC.t.sol -vvv
```

## Full Inline Foundry PoC

The source below is identical to PythBatchCacheCoveragePoC.t.sol and was run at the frozen commit.

```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

import {BasePerpTest} from "./BasePerpTest.sol";
import {CfdTypes} from "@plether/perps/CfdTypes.sol";
import {OrderV3Types} from "@plether/perps/OrderV3Types.sol";
import {PletherOracle} from "@plether/perps/PletherOracle.sol";
import {IPletherOracle} from "@plether/perps/interfaces/IPletherOracle.sol";
import {IOrderRouterAdminHost} from "@plether/perps/interfaces/IOrderRouterAdminHost.sol";
import {MockPyth} from "@plether/test-utils/MockPyth.sol";
import {console2} from "forge-std/console2.sol";

/*
 * ## Proof Explanation
 *
 * test_ControlsAndSkewedBatchRollback proves the public Router failure:
 * 1. Two ordinary accounts commit 1,000 USDG notional open orders with the same commit second.
 * 2. Equal-time feeds settle both; a legal one-second skew settles one order by the fresh path.
 * 3. The same skew in a two-order batch returns PletherOracle__StoredFeedBehind.
 * 4. Assertions verify rollback of both queue entries, both positions, Engine mark, Pyth storage/counters,
 *    settlement-token custody, Oracle/Pyth ETH, and the call value.
 *
 * test_PublicSingleOrderCallsRecoverAfterBatchFailure then makes that failed call and settles both pending
 * orders through the public single-order entrypoint. It measures the failed local EVM call's gas and verifies
 * that only the two successful single-order parses/updates retain the deliberately configured mock fee.
 *
 * test_SixFeedTimestampVectorFromExistingSignedFixture uses the six production feed IDs and the 1390/1391
 * timestamp vector present in the checked-in uneven fixture. Its MockPyth payload and previous timestamps are
 * synthetic; the test does not claim to verify the fixture's signed bytes.
 */
/// @dev Independent frozen-source reproduction. All Router calls are public; KEEPER has no configured role.
contract PythBatchCacheCoveragePoCTest is BasePerpTest {
    address internal constant KEEPER = address(0xC001);
    address internal constant TRADER_A = address(0xA001);
    address internal constant TRADER_B = address(0xB002);
    uint256 internal constant MIN_NOTIONAL = 1_000e6;
    uint256 internal constant SIZE_DELTA = MIN_NOTIONAL * 1e12;
    uint256 internal constant LIVE_TIME = 1_796_083_200 + 9 hours;

    function _riskParams() internal pure override returns (CfdTypes.RiskParams memory) {
        return CfdTypes.RiskParams({
            vpiFactor: 0.01e18,
            maxSkewRatio: 0.4e18,
            maintMarginBps: 10,
            initMarginBps: 20,
            fadMarginBps: 300,
            baseCarryBps: 500,
            minBountyAssets: 1e6,
            bountyBps: 10,
            keeperShareBps: 5000,
            protocolShareBps: 0
        });
    }

    function _initialJuniorDeposit() internal pure override returns (uint256) { return 0; }
    function _initialJuniorSeedDeposit() internal pure override returns (uint256) { return 1_000_000e6; }
    function _initialSeniorSeedDeposit() internal pure override returns (uint256) { return 1e6; }

    function setUp() public override {
        super.setUp();
        IOrderRouterAdminHost.RouterConfig memory config = _routerConfig();
        config.minOpenNotionalAssets = MIN_NOTIONAL;
        config.adverseConfidenceMultiplierBps = 2_500;
        config.maxPendingOrders = 5;
        config.positionProtectionTriggerBountyAssets = 200_000;
        _setRouterConfig(config);
        _setSeniorCapacity(40_000_000e6, 8_000);
        vm.warp(LIVE_TIME);
        vm.roll(vm.getBlockNumber() + 10);
        _fundJunior(address(0x5151), 1_000_000e6);
        _fundSenior(address(0x6161), 2_000_000e6);
        _installLiveMark();
        vm.deal(address(this), 1 ether);
        vm.deal(KEEPER, 1 ether);
    }

    function _now() internal view returns (uint256) { return vm.getBlockTimestamp(); }

    function _payload(uint256 timeA, uint256 timeB, uint256 previous) internal pure returns (bytes[] memory data) {
        MockPyth.FeedUpdate[] memory updates = new MockPyth.FeedUpdate[](2);
        updates[0] = MockPyth.FeedUpdate({
            id: bytes32(uint256(1)),
            price: MockPyth.MockPrice(100_000_000, 0, -8, timeA, previous)
        });
        updates[1] = MockPyth.FeedUpdate({
            id: bytes32(uint256(2)),
            price: MockPyth.MockPrice(100_000_000, 0, -8, timeB, previous)
        });
        data = new bytes[](1);
        data[0] = abi.encode(updates);
    }

    function _installLiveMark() internal {
        bytes[] memory data = _payload(_now(), _now(), _now() - 1);
        router.updateMarkPrice(data);
    }

    function _request(address trader) internal returns (OrderV3Types.OrderRequest memory request) {
        request.clientOrderId = bytes32(uint256(uint160(trader)));
        request.side = CfdTypes.Side.LONG;
        request.sizeDelta = SIZE_DELTA;
        request.marginDelta = 300e6;
        request.targetPrice = 1;
        request.bounds.submitBy = uint64(_now() + router.maxExecutionWindowSeconds());
        request.bounds.executionWindowSeconds = uint32(router.maxExecutionWindowSeconds());
        request.bounds.allowedExecutionModes = 7;
        request.bounds.expectedConfigHash = router.lifecycleBook().currentExecutionConfigHash();
        request.bounds.maxExecutionBountyAssets = type(uint256).max;
        request.bounds.maxExecutionNotionalAssets = type(uint256).max;
        request.bounds.maxGrossAccountDebitAssets = type(uint256).max;
        request.bounds.maxActionChargeAssets = type(uint256).max;
        request.bounds.maxExplicitFeesAssets = type(uint256).max;
        request.bounds.maxPostPositionSize = type(uint256).max;
        request.bounds.maxPostLeverageBps = type(uint32).max;
    }

    function _commit(address trader) internal returns (uint64 id) {
        _fundTrader(trader, 650e6);
        OrderV3Types.OrderRequest memory request = _request(trader);
        vm.prank(trader);
        id = router.commitOrder(request);
    }

    function _storeTime(bytes32 feedId) internal view returns (uint256 publishTime) {
        (,,, publishTime,) = baseMockPyth.prices(feedId);
    }

    function _selector(bytes memory ret) internal pure returns (bytes4 selector) {
        if (ret.length >= 4) assembly { selector := mload(add(ret, 32)) }
    }

    function _commitPair() internal returns (uint64 first, uint64 second, uint64 commitTime) {
        commitTime = uint64(_now());
        first = _commit(TRADER_A);
        second = _commit(TRADER_B);
        assertEq(second, first + 1);
        assertEq(router.lifecycleBook().orderTiming(first).commitTimestamp, commitTime);
        assertEq(router.lifecycleBook().orderTiming(second).commitTimestamp, commitTime);
    }

    function _at(uint256 timestamp) internal {
        vm.warp(timestamp);
        vm.roll(vm.getBlockNumber() + 1);
    }

    function test_ControlsAndSkewedBatchRollback() public {
        assertEq(router.pletherOracle().maxComponentPublishTimeDivergence(), 5);
        assertEq(router.pletherOracle().orderSettlementWindow(), 15);
        baseMockPyth.setFee(1);
        (uint64 first, uint64 second, uint64 committedAt) = _commitPair();
        uint256 snap = vm.snapshotState();

        _at(committedAt + 2);
        vm.prank(KEEPER);
        OrderV3Types.BatchResult memory equalResult = router.executeOrderBatch{value: 2}(second, _payload(committedAt + 2, committedAt + 2, committedAt));
        assertEq(equalResult.terminalCount, 2, "equal-time two-order control");
        vm.revertToState(snap);

        _at(committedAt + 2);
        vm.prank(KEEPER);
        OrderV3Types.BatchResult memory singleResult = router.executeOrderBatch{value: 2}(first, _payload(committedAt + 2, committedAt + 1, committedAt));
        assertEq(singleResult.terminalCount, 1, "skewed one-order control");
        vm.revertToState(snap);

        _at(committedAt + 2);
        uint256 routerBalanceBefore = address(this).balance;
        uint256 pythBalanceBefore = address(baseMockPyth).balance;
        uint256 oracleBalanceBefore = address(pletherOracle).balance;
        uint256 storedA = _storeTime(bytes32(uint256(1)));
        uint256 storedB = _storeTime(bytes32(uint256(2)));
        uint256 updateCalls = baseMockPyth.updatePriceFeedsCallCount();
        uint256 parseCalls = baseMockPyth.parseUniqueCallCount();
        uint256 markTime = engine.lastMarkTime();
        uint256 markPrice = engine.lastMarkPrice();
        uint256 engineTokens = settlementToken.balanceOf(address(engine));
        uint256 clearinghouseTokens = settlementToken.balanceOf(address(clearinghouse));

        vm.prank(KEEPER);
        (bool ok, bytes memory reason) = address(router).call{value: 2}(
            abi.encodeWithSelector(router.executeOrderBatch.selector, second, _payload(committedAt + 2, committedAt + 1, committedAt))
        );
        console2.log("ordinary public batch call success", ok);
        console2.logBytes4(_selector(reason));
        assertFalse(ok, "legal 1-second skew must fail on the vulnerable revision");
        assertEq(_selector(reason), bytes4(keccak256("PletherOracle__StoredFeedBehind(bytes32,uint256,uint256)")));

        // Rollback checks span the entire order prefix, Router, Engine, Pyth storage and transferred ETH.
        assertEq(router.nextExecuteId(), first);
        assertEq(router.lifecycleBook().pendingIntent(first).account, TRADER_A);
        assertEq(router.lifecycleBook().pendingIntent(second).account, TRADER_B);
        (uint256 sizeA,,,,,,) = engine.positions(TRADER_A);
        (uint256 sizeB,,,,,,) = engine.positions(TRADER_B);
        assertEq(sizeA, 0);
        assertEq(sizeB, 0);
        assertEq(engine.lastMarkTime(), markTime);
        assertEq(engine.lastMarkPrice(), markPrice);
        assertEq(baseMockPyth.updatePriceFeedsCallCount(), updateCalls);
        assertEq(baseMockPyth.parseUniqueCallCount(), parseCalls);
        assertEq(_storeTime(bytes32(uint256(1))), storedA);
        assertEq(_storeTime(bytes32(uint256(2))), storedB);
        assertEq(address(baseMockPyth).balance, pythBalanceBefore);
        assertEq(address(pletherOracle).balance, oracleBalanceBefore);
        assertEq(address(this).balance, routerBalanceBefore);
        assertEq(settlementToken.balanceOf(address(engine)), engineTokens);
        assertEq(settlementToken.balanceOf(address(clearinghouse)), clearinghouseTokens);
    }

    function test_PublicSingleOrderCallsRecoverAfterBatchFailure() public {
        baseMockPyth.setFee(1);
        (uint64 first, uint64 second, uint64 committedAt) = _commitPair();
        _at(committedAt + 2);
        bytes[] memory skewed = _payload(committedAt + 2, committedAt + 1, committedAt);
        uint256 pythBalanceBefore = address(baseMockPyth).balance;
        uint256 gasBefore = gasleft();
        vm.prank(KEEPER);
        (bool batchOk, bytes memory reason) = address(router).call{value: 2}(
            abi.encodeWithSelector(router.executeOrderBatch.selector, second, skewed)
        );
        uint256 failedBatchGas = gasBefore - gasleft();
        console2.log("reverted batch local EVM gas", failedBatchGas);
        assertFalse(batchOk);
        assertEq(_selector(reason), bytes4(keccak256("PletherOracle__StoredFeedBehind(bytes32,uint256,uint256)")));
        assertEq(router.nextExecuteId(), first, "the failed batch leaves its full prefix pending");
        assertEq(address(baseMockPyth).balance, pythBalanceBefore, "reverted Pyth fees are returned");

        vm.prank(KEEPER);
        OrderV3Types.ExecutionResult memory firstResult = router.executeOrder{value: 2}(first, skewed);
        vm.prank(KEEPER);
        OrderV3Types.ExecutionResult memory secondResult = router.executeOrder{value: 2}(second, skewed);
        assertEq(uint8(firstResult.status), uint8(OrderV3Types.LifecycleStatus.Executed));
        assertEq(uint8(secondResult.status), uint8(OrderV3Types.LifecycleStatus.Executed));
        assertEq(router.nextExecuteId(), 0);
        (uint256 sizeA,,,,,,) = engine.positions(TRADER_A);
        (uint256 sizeB,,,,,,) = engine.positions(TRADER_B);
        assertEq(sizeA, SIZE_DELTA);
        assertEq(sizeB, SIZE_DELTA);
        assertEq(baseMockPyth.parseUniqueCallCount(), 2, "one historical parse per separate execution");
        assertEq(address(baseMockPyth).balance - pythBalanceBefore, 4, "two single orders pay four one-wei Pyth operations");
    }

    function test_SixFeedTimestampVectorFromExistingSignedFixture() public {
        // Feed IDs, production weights/base prices/inversions, and publish times are copied from
        // test/fixtures/oracle-sync/live-watermark-uneven.json. The JSON's signed bytes are not used
        // here; MockPyth supplies synthetic values and previous timestamps for the historical parser.
        bytes32[] memory ids = new bytes32[](6);
        ids[0] = 0xa995d00bb36a63cef7fd2c287dc105fc8f3d93779f062f09551b0af3e81ec30b;
        ids[1] = 0xef2c98c804ba503c6a707e38be4dfbb16683775f195b091252bf24693042fd52;
        ids[2] = 0x84c2dde9633d93d1bcad84e7dc41c9d56578b7ec52fabedc1f335d673df0a7c1;
        ids[3] = 0x3112b03a41c910ed446852aacf67118cb1bec67b2cd0b9a214c58cc0eaa2ecca;
        ids[4] = 0x8ccb376aa871517e807358d4e3cf0bc7fe4950474dbe6c9ffc21ef64e43fc676;
        ids[5] = 0x0b1e3297e69f162877b577b0d6a47a0d63b2392bc8499e6540da4187a63e28f8;
        uint256[] memory weights = new uint256[](6);
        weights[0] = 576e15; weights[1] = 136e15; weights[2] = 119e15;
        weights[3] = 91e15; weights[4] = 42e15; weights[5] = 36e15;
        uint256[] memory bases = new uint256[](6);
        bases[0] = 117_500_000; bases[1] = 638_000; bases[2] = 134_480_000;
        bases[3] = 72_880_000; bases[4] = 10_860_000; bases[5] = 126_100_000;
        bool[] memory inversions = new bool[](6);
        inversions[1] = true; inversions[3] = true; inversions[4] = true; inversions[5] = true;

        uint64 commitTime = 1_790_091_389;
        uint64 oldestTime = 1_790_091_390;
        uint64 newestTime = 1_790_091_391;
        vm.warp(newestTime);
        MockPyth sixFeedPyth = new MockPyth();
        sixFeedPyth.setAllPrices(ids, int64(100_000_000), int32(-8), commitTime);
        PletherOracle sixFeedOracle = new PletherOracle(
            address(engine), address(pool), address(sixFeedPyth), ids, weights, bases, inversions
        );
        MockPyth.FeedUpdate[] memory updates = new MockPyth.FeedUpdate[](6);
        for (uint256 i; i < updates.length; ++i) {
            uint256 timestamp = i == 0 ? oldestTime : newestTime;
            updates[i] = MockPyth.FeedUpdate({
                id: ids[i],
                price: MockPyth.MockPrice(100_000_000, 0, -8, timestamp, commitTime)
            });
        }
        bytes[] memory data = new bytes[](1);
        data[0] = abi.encode(updates);
        IPletherOracle.OrderExecutionRequest memory request = IPletherOracle.OrderExecutionRequest({
            commitTime: commitTime,
            targetPrice: type(uint256).max,
            side: CfdTypes.Side.LONG,
            isClose: false,
            revertOnHistoricalUnavailable: false
        });
        IPletherOracle.BatchOrderPriceCache memory cache;
        (bool parsed, IPletherOracle.PriceSnapshot memory snapshot, IPletherOracle.BatchOrderPriceCache memory next) =
            sixFeedOracle.updateBatchOrderExecutionPrice(address(this), data, request, cache);
        assertTrue(parsed);
        assertEq(snapshot.publishTime, newestTime);
        assertEq(next.oldestPublishTime, oldestTime);
        assertEq(sixFeedPyth.getPriceUnsafe(ids[0]).publishTime, oldestTime);
        assertEq(sixFeedPyth.getPriceUnsafe(ids[5]).publishTime, newestTime);
        (bool reused, bytes memory ret) = address(sixFeedOracle).call(
            abi.encodeWithSelector(
                sixFeedOracle.updateBatchOrderExecutionPrice.selector, address(this), data, request, next
            )
        );
        assertFalse(reused);
        assertEq(_selector(ret), bytes4(keccak256("PletherOracle__StoredFeedBehind(bytes32,uint256,uint256)")));
    }
}
```

## Actual Command and Output

This is the test-result portion of stdout from the exact clean-checkout command above. It includes the updated test contract name, revert selector, measured gas, and final summary.

```text
Ran 3 tests for test/perps/PythBatchCacheCoveragePoC.t.sol:PythBatchCacheCoveragePoCTest
[PASS] test_ControlsAndSkewedBatchRollback() (gas: 10698215)
Logs:
  ordinary public batch call success false
  0xf2aa6d49

[PASS] test_PublicSingleOrderCallsRecoverAfterBatchFailure() (gas: 8927092)
Logs:
  reverted batch local EVM gas 2161220

[PASS] test_SixFeedTimestampVectorFromExistingSignedFixture() (gas: 2854573)
Suite result: ok. 3 passed; 0 failed; 0 skipped; finished in 26.35ms (29.75ms CPU time)

Ran 1 test suite in 27.37ms (26.35ms CPU time): 3 tests passed, 0 failed, 0 skipped (3 total tests)
```
