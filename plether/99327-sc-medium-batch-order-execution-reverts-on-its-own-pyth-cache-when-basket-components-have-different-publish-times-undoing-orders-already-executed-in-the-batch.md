# \#99327 [SC-Medium] Batch order execution reverts on its own Pyth cache when basket components have different publish times, undoing orders already executed in the batch

**Submitted on Nov 13th 2026 at 07:12:28 UTC by @tomX0 for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #99327
- **Report Type:** Smart Contract
- **Report severity:** Medium
- **Target:** https://github.com/immunefi-team/audit-comp-plether/tree/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src
- **Impacts:**
  - Contract fails to deliver promised returns, but doesn't lose value

## Description

## Summary

When a keeper calls `executeOrderBatch`, the Oracle parses one signed historical Pyth basket for the first order and caches it, so later orders in the batch can reuse it instead of paying for another parse. Before it reuses the cache, the Oracle checks that every feed in Pyth storage has reached `cache.publishTime`.

The problem is that `cache.publishTime` is the newest component time in the basket, while the parse that built the cache only writes each feed's own time into storage. Components are allowed to be up to 5 seconds apart. Whenever they aren't equal, the lagging feed sits behind the newest time, so the Router's own cache, written a moment earlier in the same transaction, fails the check. The Oracle reverts with `PletherOracle__StoredFeedBehind`, the Router re-raises it, and the whole batch reverts, including the first order that had already executed.

Nothing is wrong with the signed data. The same payload executes both orders through `executeOrder`, and the same batch succeeds if someone happens to have pushed a newer tick for the lagging feed beforehand. The reuse check is simply comparing against the wrong timestamp.

## Target

- **Asset:** `packages/perps/src` (the "all other contracts, libraries, router contracts and interfaces" asset), commit `a82d4fefb02768e5b3bddcc97dee371eee24fea8`
- **Files:** `PletherOracle.sol` and `OrderRouterExecutionSidecar.sol`
- **Function:** `OrderRouter.executeOrderBatch`, through `PletherOracle.updateBatchOrderExecutionPrice`

## Vulnerability details

### Where it goes wrong

The historical parse records the newest component as the basket time and keeps the oldest separately:

```solidity
// PletherOracle.sol:874-875
basket.publishTime = uint64(maxPublishTime);
basket.oldestPublishTime = uint64(minPublishTime);
```

Both values are copied into the batch cache (`PletherOracle.sol:288-297`), so `cache.publishTime` is the newest component time.

The parse writes the payload to Pyth and checks each feed only against its own publish time:

```solidity
// PletherOracle.sol:623-626
pyth.updatePriceFeeds{value: pythFee}(pythUpdateData);
for (uint256 i; i < parsedFeeds.length; ++i) {
    _requireStoredFeedCoverage(parsedFeeds[i].id, parsedFeeds[i].price.publishTime);
}
```

Pyth only replaces a stored price with a strictly newer one. Right after this parse, the lagging feed's storage holds the lagging time, unless something newer was already there.

When the next order reuses the cache, every feed is checked against the newest time:

```solidity
// PletherOracle.sol:262-266
if (!policy.oracleFrozen && _canReuseHistoricalBatchBasket(request.commitTime, cache)) {
    // Cache contents are caller-supplied; independently prove storage coverage, not price authenticity.
    for (uint256 i; i < pythFeedIds.length; ++i) {
        _requireStoredFeedCoverage(pythFeedIds[i], cache.publishTime);
    }
```

```solidity
// PletherOracle.sol:650-651
if (storedPublishTime < requiredPublishTime) {
    revert PletherOracle__StoredFeedBehind(feedId, storedPublishTime, requiredPublishTime);
```

So the parse proves `stored >= ownTime` for each feed, and the reuse check then demands `stored >= newestTime` from every feed. For a basket with unequal times the two disagree on the lagging feed, and the cache fails a check that its own parse just satisfied.

### Why the whole batch reverts

The Router decides to reuse without looking at Pyth storage. It only checks commit times against the cache (`OrderRouterExecutionSidecar.sol:1074-1093`, mirrored at `PletherOracle.sol:974-991`):

```solidity
// OrderRouterExecutionSidecar.sol:1083-1086
if (
    commitTime < cache.minReusableCommitTime || commitTime >= cache.oldestPublishTime
        || commitTime >= cache.publishTime
) {
    return false;
}
```

Any later order committed between the first order's commit and the oldest component time is sent down the reuse path with a zero fee. That includes an order committed in the same second as the first one, which is ordinary traffic on Arbitrum.

When the Oracle call fails, the Router re-raises everything except the 68-byte `HistoricalParserFailure`:

```solidity
// OrderRouterExecutionSidecar.sol:1019-1027
} catch (bytes memory reason) {
    if (
        reason.length != 68
            || _selector(reason) != IPletherOracle.PletherOracle__HistoricalParserFailure.selector
    ) {
        assembly ("memory-safe") {
            revert(add(reason, 32), mload(reason))
        }
    }
```

`StoredFeedBehind` is 100 bytes, so it is re-raised. Oracle preparation runs in the outer batch frame rather than the per-order frame, so the revert takes down the entire call, including orders that were already finalized.

### How it was introduced

At commit `6d6622f` the basket time was the oldest component (`basket.publishTime = uint64(minPublishTime);`). The coverage loop already compared against `cache.publishTime`, and because that was the oldest time, the parse's own storage write always satisfied it.

PR #120 (commit `81919218`) fixed MED-18 and MED-23 from the Cecuro audit by switching the basket time to the newest component and adding `oldestPublishTime` to the reuse rule. The coverage loop wasn't updated, so it now compares against the newest time. The interface NatSpec still describes `PriceSnapshot.publishTime` as the "Earliest component publish time" (`interfaces/IPletherOracle.sol:31`), which is the meaning the loop was written for.

## Steps to reproduce

No privileged role and no attacker are needed. Honest data is enough.

1. Trader A commits an order at `t0`.
2. Trader B commits an order before the oldest component time of the next Pyth update, for example in the same second as A.
3. The first signed Pyth update after `t0` has components published at different times, for example feed0 at `t0 + 1` and feed1 at `t0 + 2`. That is inside the 5 second limit. Pyth's unique-parse rule (`prevPublishTime < commit + 1 <= publishTime`) makes this the only payload that can price order A, so the keeper has no equal-time alternative.
4. A keeper calls `executeOrderBatch(idB, payload)`:
   - For order A, the parse succeeds and Pyth storage becomes feed0 = `t0 + 1`, feed1 = `t0 + 2`. The Engine mark is updated, A is executed and finalized, and the cache becomes `{publishTime: t0 + 2, oldestPublishTime: t0 + 1}`.
   - For order B, `commitTime < oldestPublishTime`, so the Router reuses the cache. The Oracle checks feed0 against `t0 + 2`, finds `t0 + 1` and reverts with `StoredFeedBehind(feed0, t0 + 1, t0 + 2)`.
   - The Router re-raises the error and the transaction reverts. A is `Pending` again, the mark update is undone, and the keeper's gas is spent.

## Impact

### What the PoC shows

All of this is measured at release parameters.

- **Orders one second apart:** A commits at 1709805602 and B at 1709805603; feed0 is published at 1709805604 and feed1 at 1709805605. `executeOrderBatch` reverts with `StoredFeedBehind(feed0, 1709805604, 1709805605)`. Afterwards the queue head, both order statuses (`Pending`) and `engine.lastMarkTime()` are unchanged. The reverted call used 1,688,963 gas on the two-feed test basket.
- **Order A had really executed:** a `-vvvv` trace of the same test shows, in order, A's parse and `updatePriceFeeds`, `CfdEngine.updateMarkPrice`, `PositionOpened`, `OrderExecuted(orderId: 1)` and `OrderFinalized`, which paid the keeper a 108,000-atom (0.108 USDG) bounty. Only after that does B's reuse check revert, and all of it is undone.
- **Same-block commits:** the same revert happens when A and B commit in the same block, with feed0 at `t0 + 1` and feed1 at `t0 + 2`. Two traders committing in the same second is normal activity.
- **Controls:**
  - With equal component times, the same two orders execute in one batch with a single parse.
  - The same unequal payload executes both orders through two `executeOrder` calls.
  - The same unequal payload and batch succeed with a single parse once anyone has pushed a later feed0 tick to Pyth.

So whether the batch works depends on unrelated Pyth storage, not on the basket that prices the orders.

### Who is affected

**Keepers** using `executeOrderBatch`, the batch entrypoint listed in `CANONICAL_ENTRYPOINTS.md`, get a revert on valid data and lose the bounties they had already earned in that batch. A keeper that doesn't simulate first also loses the gas. To get the orders through, the keeper has to recognise this particular failure and switch to one of these workarounds:
- single `executeOrder` calls, paying one historical Pyth fee per order;
- capping `maxOrderId` at the first order;
- pushing a newer tick for the lagging feed first, which costs an extra Pyth fee.

**Plether's reference keeper** runs into the realistic trigger. From reading the `plether-app` backend:
- It batches every ready order that shares the cached first post-commit Hermes payload, up to `KEEPER_MAX_BATCH_SIZE` (default 20).
- Its reveal check requires the payload's oldest component time to equal each order's `commitTime + 1`, so the orders it batches together are exactly those committed in the same second.
- Its basket handling explicitly allows components up to 5 seconds apart.
- It simulates before sending, so it doesn't burn gas. But on failure it records the error and rebuilds the same batch on the next poll, so the queue head stalls until something pushes a newer tick for the lagging feed.

**Traders** whose orders were executed or cleaned up earlier in the batch have that progress undone, and their execution is delayed. The orders stay `Pending` and can still be executed inside their execution window. The PoC does not show any order expiring.

**Griefing** is limited. Nobody controls Pyth component times. But whenever the honest basket is unequal, anyone can turn an otherwise successful batch into a full revert by committing an order inside the window, subject to the normal minimum notional and margin rules.

### Why this is Low

- No funds are lost or frozen, and `executeOrder` works with the same payload.
- The reused price is the basket proven earlier in the same transaction, so no stale price is ever accepted.
- A delayed order is still filled at the first post-commit tick, so the delay doesn't change the fill price.
- The stall clears once a newer tick for the lagging feed reaches Pyth storage.

## Why this isn't known or intended

I went through the Cecuro AI audit, the SC Audit Studio pre-audit, the AI scan findings, the collaborative audit and the Cantina report in `audits/`. I also checked the excluded `SECURITY.md` sections, the repository tests, and the upstream `Plether-Fi/plether-core` history and open branches.

- **No earlier finding covers it.** The closest are MED-18 and MED-23, and their fix (PR #120) is what introduced this. The program accepts reports on a previous root cause when the fix at `a82d4fe` introduces a new bug, which is what happened here. MED-7 and the excluded live-basket limitation are about which signed components price a live basket, not about batch liveness.
- **The excluded limitations don't mention it.** "Known Limitations", "Router and keeper limitations", "live basket component selection" and "Arbitrum and Pyth clock alignment" say nothing about batch cache reuse or coverage. The oracle limitation about depending on keepers and oracle publication doesn't apply either. Here a keeper has submitted a valid payload and the Oracle has just written it to storage. Nothing is waiting on publication.
- **The docs describe a fallback the code doesn't take.** `ORACLE_SYNCHRONIZATION.md` ("Batch reuse") says that reusing the basket for a later order requires cache coverage, and that otherwise the Router requests a new historical resolution. The Router never reads storage, so it never falls back; it reuses and reverts. `SECURITY.md` ("Delayed-order model") likewise allows reuse only within "proven coverage". Here the coverage was proven by the same transaction a moment earlier.
- **"Oracle failures revert the batch" is meant for real failures.** `SECURITY.md`, the failure table in `ORACLE_SYNCHRONIZATION.md` and the LOW-68 disposition all say that `StoredFeedBehind` reverts the whole call. I'm not disputing that rule. In this case nothing has actually failed: the basket is valid and storage has just been synced to it.
- **The repository's tests agree.** Every test that expects `StoredFeedBehind` (`OracleSynchronization.t.sol:154-171` and `:490-515`, plus the `OracleSynchronizationHandler` and `OracleEthConservationHandler` invariant handlers) first forces a skipped or failed storage write, or rolls storage back by hand. None of them expects it from an honest basket.
- **The one test of unequal-time reuse avoids the bug.** `test_BatchReusesBasketWhenEveryComponentFollowsBothCommits` (`OracleSynchronization.t.sol:434-453`) uses a basket with feed0 a second behind and expects reuse to work. It passes only because it pushes a newer payload into Pyth storage first (line 446). Without that line it hits this bug.
- **Unequal component times are expected.** `ORACLE_SYNCHRONIZATION.md` requires release qualification against real payloads with unequal component times, and the Oracle has a dedicated 5 second divergence limit.
- **It's not fixed or disclosed upstream.** Upstream `master` still compares against `cache.publishTime`, and I found no PR, commit or issue mentioning this.

## Recommended fix

Make the reuse check match what the parse proved. The smallest change restores the meaning the loop had before PR #120:

```diff
 // PletherOracle.updateBatchOrderExecutionPrice
             for (uint256 i; i < pythFeedIds.length; ++i) {
-                _requireStoredFeedCoverage(pythFeedIds[i], cache.publishTime);
+                _requireStoredFeedCoverage(pythFeedIds[i], cache.oldestPublishTime);
             }
```

I tested this locally:
- The regression test below passes.
- The two bug tests in the PoC no longer revert.
- The three controls still pass.
- All 24 tests in the repository's `OracleSynchronization.t.sol` still pass.

One caveat. `updateBatchOrderExecutionPrice` has no caller restriction, and its NatSpec already says it doesn't authenticate a caller-supplied cache. With the minimal fix, a hand-built cache could claim a newest time above every stored feed. The Router never passes a caller-supplied cache, so the protocol isn't affected. If you want to keep the stronger check for direct callers, either of these works:
- Store per-component publish times in `BatchOrderPriceCache` and check each feed against its own time.
- Implement the documented fallback. When stored coverage is short, the Router requests a fresh historical resolution for that order instead of reusing the cache. The single-execution control shows that the same payload resolves the later order.

Whichever fix you choose, a batch test with unequal component times that does not pre-advance Pyth storage would catch a regression. This one can be added to the PoC contract:

```solidity
function test_regression_unequalComponentBatchReusesCache() public {
    (uint64 idA, uint64 idB, uint64 t0) = _ordersOneSecondApart();
    bytes[] memory data = _unequalPayload(t0 + 2, t0 + 3, t0);
    uint256 fee = _batchFee(data);
    _pocEnsureEth(keeper, fee);
    uint256 parsesBefore = baseMockPyth.parseUniqueCallCount();
    vm.prank(keeper);
    OrderV3Types.BatchResult memory result = router.executeOrderBatch{value: fee}(idB, data);
    assertEq(result.terminalCount, 2, "both executed");
    assertEq(baseMockPyth.parseUniqueCallCount() - parsesBefore, 1, "second order reused the cache");
    assertEq(_status(idA), uint8(OrderV3Types.LifecycleStatus.Executed), "A executed");
    assertEq(_status(idB), uint8(OrderV3Types.LifecycleStatus.Executed), "B executed");
    assertEq(engine.lastMarkTime(), t0 + 3, "mark = newest component time");
}
```



## Proof of Concept

## Proof of concept

The PoC is a single self-contained Foundry file. It holds a release-configured fixture built on the repository's `BasePerpTest` and five tests. No in-scope contract is modified, and the only mocks are the repository's `MockPyth` and USDG, both of which are out of scope.

`setUp` applies the release values through the normal owner propose and finalize timelocks and funds Junior LP capital through the public ERC-7540 request, settle and claim flow. Every test re-checks the release values at the end with `_assertReleaseParams()`. After `setUp`, only fresh accounts (alice, bob, keeper) act, and only through public entrypoints. There is no `vm.store`, no `vm.mockCall` and no prank of a protocol address.

Signed Pyth data is modelled with explicit `MockPyth` payloads. The mock enforces Pyth's unique-parse window and its rule that storage only accepts newer prices. The fixture uses the repository's two-feed test basket instead of the six FX feeds. `BasePerpTest` deploys the production `OrderRouter` bytecode through its standard `LegacyOrderRouterHarness` wrapper. Every order goes through the production V3 `commitOrder(OrderRequest)` path.

### Running it

From the repository root at commit `a82d4fe`:

```bash
git submodule update --init --recursive
mkdir -p packages/perps/test/perps/poc
# save the file below as packages/perps/test/perps/poc/BatchReuseCoveragePoc.t.sol
FOUNDRY_PROFILE=quick forge test --root packages/perps \
  --match-path test/perps/poc/BatchReuseCoveragePoc.t.sol -vvv
```

### What each test checks

- `test_batchRevertsOnRoutersOwnCache_unequalComponentTimes`: two orders one second apart. It checks that the revert data is exactly `StoredFeedBehind(feed0, t0 + 2, t0 + 3)`, and that the queue head, both statuses and the Engine mark time are unchanged afterwards.
- `test_sameBlockCommits_batchReverts`: the same revert with both orders committed in one block, which is the normal-operation trigger.
- `test_control_equalComponentTimesBatchSucceeds`: equal component times give two executions with one parse.
- `test_control_singleExecutionWithSamePayloadSucceeds`: the same unequal payload executes both orders through `executeOrder`.
- `test_control_batchSucceedsWhenLaggingStorageAlreadyAhead`: with a newer feed0 tick already in Pyth, the same batch and payload execute both orders with one parse, still priced from the parsed basket.

### BatchReuseCoveragePoc.t.sol

```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

import {BasePerpTest} from "../BasePerpTest.sol";
import {SafeCast} from "@openzeppelin/contracts/utils/math/SafeCast.sol";
import {CfdTypes} from "@plether/perps/CfdTypes.sol";
import {OrderRouter} from "@plether/perps/OrderRouter.sol";
import {OrderV3Types} from "@plether/perps/OrderV3Types.sol";
import {PletherOracle} from "@plether/perps/PletherOracle.sol";
import {TrancheVault} from "@plether/perps/TrancheVault.sol";
import {ICfdEngineAdminHost} from "@plether/perps/interfaces/ICfdEngineAdminHost.sol";
import {ICfdEngineRiskParamsView} from "@plether/perps/interfaces/ICfdEngineRiskParamsView.sol";
import {ICfdEngineTypes} from "@plether/perps/interfaces/ICfdEngineTypes.sol";
import {IHousePool} from "@plether/perps/interfaces/IHousePool.sol";
import {IOrderRouterAdminHost} from "@plether/perps/interfaces/IOrderRouterAdminHost.sol";
import {IPletherOracle} from "@plether/perps/interfaces/IPletherOracle.sol";
import {MockPyth} from "@plether/test-utils/MockPyth.sol";

/// @notice Release-configured fixture on top of the repository's BasePerpTest.
/// @dev Release wrapper values are installed in setUp through the normal owner propose/finalize timelocks, Junior LP
///      capital is funded through the public ERC-7540 path, and every value is re-asserted by _assertReleaseParams().
///      After setUp, only fresh EOAs act, and only through public entrypoints. Pyth and USDG are the repository mocks.
///      No vm.store, no vm.mockCall, no prank of any protocol address.
abstract contract BatchReuseFixture is BasePerpTest {

    uint256 internal constant REL_VPI_FACTOR = 0.01e18;
    uint256 internal constant REL_MAX_SKEW_RATIO = 0.4e18;
    uint256 internal constant REL_MAINT_MARGIN_BPS = 10;
    uint256 internal constant REL_INIT_MARGIN_BPS = 20;
    uint256 internal constant REL_FAD_MARGIN_BPS = 300;
    uint256 internal constant REL_BASE_CARRY_BPS = 500;
    uint256 internal constant REL_MIN_BOUNTY_ASSETS = 1e6;
    uint256 internal constant REL_BOUNTY_BPS = 10;
    uint256 internal constant REL_KEEPER_SHARE_BPS = 5000;
    uint256 internal constant REL_PROTOCOL_SHARE_BPS = 0;
    uint256 internal constant REL_EXECUTION_FEE_BPS = 4;
    uint256 internal constant REL_SETTLEMENT_BUFFER_BPS = 25;
    uint256 internal constant REL_FROZEN_CLOSE_SPREAD_BPS = 50;
    uint256 internal constant REL_FAD_RUNWAY_SECONDS = 1 hours;
    uint256 internal constant REL_BASKET_MAX_CONF_RATIO_BPS = 10;
    uint256 internal constant REL_ADVERSE_CONF_MULTIPLIER_BPS = 2500;
    uint256 internal constant REL_ORDER_STALENESS = 60;
    uint256 internal constant REL_LIQUIDATION_STALENESS = 15;
    uint256 internal constant REL_ORDER_SETTLEMENT_WINDOW = 15;
    uint256 internal constant REL_MAX_COMPONENT_DIVERGENCE = 5;
    uint256 internal constant REL_MIN_OPEN_NOTIONAL_ASSETS = 1000e6;
    uint256 internal constant REL_MAX_PENDING_ORDERS = 5;
    uint256 internal constant REL_PROTECTION_TRIGGER_BOUNTY_ASSETS = 200_000;
    uint256 internal constant REL_MAX_EXECUTION_WINDOW_SECONDS = 60;
    uint256 internal constant REL_MAX_SENIOR_EXPOSURE_ASSETS = 40_000_000e6;
    uint256 internal constant REL_MAX_SENIOR_SHARE_BPS = 8000;
    uint256 internal constant REL_SENIOR_RATE_BPS = 800;
    uint256 internal constant REL_POOL_MARK_STALENESS = 60;
    uint256 internal constant REL_SENIOR_FROZEN_LP_FEE_BPS = 25;
    uint256 internal constant REL_JUNIOR_FROZEN_LP_FEE_BPS = 75;
    uint256 internal constant REL_JUNIOR_MAINTENANCE_FEE_APR_BPS = 0;
    uint256 internal constant REL_SEED_ASSETS = 1e6;
    uint256 internal constant ALL_EXECUTION_MODES = 7;
    uint256 internal constant DEFAULT_PRICE = 1.2e8;
    uint256 internal constant JUNIOR_LP_ASSETS = 1_000_000e6;

    address internal pocKeeper;
    address internal pocJuniorLp;
    uint256 internal pocPrice;
    uint256 private _clientNonce;

    function _riskParams() internal pure virtual override returns (CfdTypes.RiskParams memory) {
        return CfdTypes.RiskParams({
            vpiFactor: REL_VPI_FACTOR,
            maxSkewRatio: REL_MAX_SKEW_RATIO,
            maintMarginBps: REL_MAINT_MARGIN_BPS,
            initMarginBps: REL_INIT_MARGIN_BPS,
            fadMarginBps: REL_FAD_MARGIN_BPS,
            baseCarryBps: REL_BASE_CARRY_BPS,
            minBountyAssets: REL_MIN_BOUNTY_ASSETS,
            bountyBps: REL_BOUNTY_BPS,
            keeperShareBps: REL_KEEPER_SHARE_BPS,
            protocolShareBps: REL_PROTOCOL_SHARE_BPS
        });
    }

    // Zero so BasePerpTest.setUp never pranks the Router to fund LPs; LPs are funded publicly below.
    function _initialJuniorDeposit() internal pure virtual override returns (uint256) {
        return 0;
    }

    function _initialSeniorDeposit() internal pure virtual override returns (uint256) {
        return 0;
    }

    function _initialJuniorSeedDeposit() internal pure virtual override returns (uint256) {
        return REL_SEED_ASSETS;
    }

    function _initialSeniorSeedDeposit() internal pure virtual override returns (uint256) {
        return REL_SEED_ASSETS;
    }

    function setUp() public virtual override {
        super.setUp();
        assertEq(pool.seniorPrincipal(), REL_SEED_ASSETS, "release Senior seed");
        assertEq(pool.juniorPrincipal(), REL_SEED_ASSETS, "release Junior seed");
        assertTrue(pool.isTradingActive(), "trading active after seeds");

        pocKeeper = makeAddr("pocKeeper");
        pocJuniorLp = makeAddr("pocJuniorLp");

        _applyReleaseConfig();
        _warpToWeekdayHourUtc(9);
        baseMockPyth.setFee(1);

        _setPrice(DEFAULT_PRICE);
        _updateMark(pocKeeper, DEFAULT_PRICE);

        // Public LP funding: request -> epoch maturity -> permissionless settlement -> claim.
        _pocRequireActor(pocJuniorLp);
        settlementToken.mint(pocJuniorLp, JUNIOR_LP_ASSETS);
        vm.startPrank(pocJuniorLp);
        settlementToken.approve(address(juniorVault), JUNIOR_LP_ASSETS);
        uint256 requestId = juniorVault.requestDeposit(JUNIOR_LP_ASSETS, pocJuniorLp, pocJuniorLp);
        vm.stopPrank();
        uint256 start = pool.lpEpochStart(requestId);
        if (vm.getBlockTimestamp() < start) {
            vm.warp(start);
            vm.roll(vm.getBlockNumber() + 1);
        }
        _settleLpEpochPublic(pocKeeper);
        uint256 claimable = juniorVault.claimableDepositRequest(requestId, pocJuniorLp);
        vm.prank(pocJuniorLp);
        assertGt(juniorVault.claimDeposit(requestId, claimable, pocJuniorLp, pocJuniorLp), 0, "Junior shares");

        _setPrice(DEFAULT_PRICE);
        _updateMark(pocKeeper, DEFAULT_PRICE);
        assertFalse(engine.isFadWindow() || engine.isOracleFrozen(), "outside FAD / frozen windows");
        _assertReleaseParams();
    }

    function _applyReleaseConfig() internal {
        uint256 readyAt = vm.getBlockTimestamp();

        bool poolChange;
        IHousePool.PoolConfig memory poolConfig = _currentPoolConfig();
        if (
            poolConfig.maxSeniorExposureAssets != REL_MAX_SENIOR_EXPOSURE_ASSETS
                || poolConfig.maxSeniorShareBps != REL_MAX_SENIOR_SHARE_BPS
                || poolConfig.seniorRateBps != REL_SENIOR_RATE_BPS
                || poolConfig.markStalenessLimit != REL_POOL_MARK_STALENESS
                || poolConfig.seniorFrozenLpFeeBps != REL_SENIOR_FROZEN_LP_FEE_BPS
                || poolConfig.juniorFrozenLpFeeBps != REL_JUNIOR_FROZEN_LP_FEE_BPS
        ) {
            poolConfig.maxSeniorExposureAssets = REL_MAX_SENIOR_EXPOSURE_ASSETS;
            poolConfig.maxSeniorShareBps = REL_MAX_SENIOR_SHARE_BPS;
            poolConfig.seniorRateBps = REL_SENIOR_RATE_BPS;
            poolConfig.markStalenessLimit = REL_POOL_MARK_STALENESS;
            poolConfig.seniorFrozenLpFeeBps = REL_SENIOR_FROZEN_LP_FEE_BPS;
            poolConfig.juniorFrozenLpFeeBps = REL_JUNIOR_FROZEN_LP_FEE_BPS;
            pool.proposePoolConfig(poolConfig);
            readyAt = _max(readyAt, pool.poolConfigActivationTime());
            poolChange = true;
        }

        bool routerChange;
        IOrderRouterAdminHost.RouterConfig memory routerConfig = _routerConfig();
        if (
            routerConfig.minOpenNotionalAssets != REL_MIN_OPEN_NOTIONAL_ASSETS
                || routerConfig.adverseConfidenceMultiplierBps != REL_ADVERSE_CONF_MULTIPLIER_BPS
                || routerConfig.basketMaxConfidenceRatioBps != REL_BASKET_MAX_CONF_RATIO_BPS
                || routerConfig.maxPendingOrders != REL_MAX_PENDING_ORDERS
                || routerConfig.positionProtectionTriggerBountyAssets != REL_PROTECTION_TRIGGER_BOUNTY_ASSETS
        ) {
            routerConfig.minOpenNotionalAssets = REL_MIN_OPEN_NOTIONAL_ASSETS;
            routerConfig.adverseConfidenceMultiplierBps = REL_ADVERSE_CONF_MULTIPLIER_BPS;
            routerConfig.basketMaxConfidenceRatioBps = REL_BASKET_MAX_CONF_RATIO_BPS;
            routerConfig.maxPendingOrders = REL_MAX_PENDING_ORDERS;
            routerConfig.positionProtectionTriggerBountyAssets = REL_PROTECTION_TRIGGER_BOUNTY_ASSETS;
            routerAdmin.proposeRouterConfig(routerConfig);
            readyAt = _max(readyAt, routerAdmin.routerConfigActivationTime());
            routerChange = true;
        }

        bool riskChange;
        ICfdEngineAdminHost.EngineRiskConfig memory riskConfig = _engineRiskConfig();
        if (
            riskConfig.executionFeeBps != REL_EXECUTION_FEE_BPS
                || riskConfig.settlementBufferBps != REL_SETTLEMENT_BUFFER_BPS
                || riskConfig.frozenCloseSpreadBps != REL_FROZEN_CLOSE_SPREAD_BPS
                || keccak256(abi.encode(riskConfig.riskParams)) != keccak256(abi.encode(_riskParams()))
        ) {
            riskConfig.riskParams = _riskParams();
            riskConfig.executionFeeBps = REL_EXECUTION_FEE_BPS;
            riskConfig.settlementBufferBps = REL_SETTLEMENT_BUFFER_BPS;
            riskConfig.frozenCloseSpreadBps = REL_FROZEN_CLOSE_SPREAD_BPS;
            engineAdmin.proposeRiskConfig(riskConfig);
            readyAt = _max(readyAt, engineAdmin.riskConfigActivationTime());
            riskChange = true;
        }

        bool calendarChange;
        if (engine.fadRunwaySeconds() != REL_FAD_RUNWAY_SECONDS) {
            ICfdEngineAdminHost.EngineCalendarConfig memory calendarConfig;
            calendarConfig.fadDayTimestamps = new uint256[](0);
            calendarConfig.fadRunwaySeconds = REL_FAD_RUNWAY_SECONDS;
            engineAdmin.proposeCalendarConfig(calendarConfig);
            readyAt = _max(readyAt, engineAdmin.calendarConfigActivationTime());
            calendarChange = true;
        }

        bool maintenanceFeeChange;
        if (juniorVault.maintenanceFeeAprBps() != REL_JUNIOR_MAINTENANCE_FEE_APR_BPS) {
            juniorVault.proposeMaintenanceFeeConfig(REL_JUNIOR_MAINTENANCE_FEE_APR_BPS, engine.protocolTreasury());
            readyAt = _max(readyAt, juniorVault.maintenanceFeeConfigActivationTime());
            maintenanceFeeChange = true;
        }

        if (readyAt > vm.getBlockTimestamp()) {
            vm.warp(readyAt + 1);
            vm.roll(vm.getBlockNumber() + 1);
        }
        if (poolChange) pool.finalizePoolConfig();
        if (routerChange) routerAdmin.finalizeRouterConfig();
        if (riskChange) engineAdmin.finalizeRiskConfig();
        if (calendarChange) engineAdmin.finalizeCalendarConfig();
        if (maintenanceFeeChange) juniorVault.finalizeMaintenanceFeeConfig();
    }

    /// @dev Forward-only warp to the next Monday-Thursday `hourUtc`:00 UTC.
    function _warpToWeekdayHourUtc(
        uint256 hourUtc
    ) internal {
        uint256 nowTs = vm.getBlockTimestamp();
        uint256 day = nowTs / 1 days;
        uint256 target = day * 1 days + hourUtc * 1 hours;
        if (target < nowTs) {
            day += 1;
            target += 1 days;
        }
        while (((day + 4) % 7) == 0 || ((day + 4) % 7) >= 5) {
            day += 1;
            target += 1 days;
        }
        vm.warp(target);
        vm.roll(vm.getBlockNumber() + 1);
    }

    /// @notice Asserts the release values the finding depends on, plus "no privileged action pending".
    function _assertReleaseParams() internal view {
        CfdTypes.RiskParams memory rp = ICfdEngineRiskParamsView(address(engine)).riskParams();
        assertEq(keccak256(abi.encode(rp)), keccak256(abi.encode(_riskParams())), "rel: risk params");
        assertEq(engine.executionFeeBps(), REL_EXECUTION_FEE_BPS, "rel: executionFeeBps");
        assertEq(engine.fadRunwaySeconds(), REL_FAD_RUNWAY_SECONDS, "rel: fadRunwaySeconds");
        assertEq(router.minOpenNotionalAssets(), REL_MIN_OPEN_NOTIONAL_ASSETS, "rel: minOpenNotionalAssets");
        assertEq(router.maxPendingOrders(), REL_MAX_PENDING_ORDERS, "rel: maxPendingOrders");
        assertEq(router.maxExecutionWindowSeconds(), REL_MAX_EXECUTION_WINDOW_SECONDS, "rel: maxExecutionWindow");

        IPletherOracle oracle = router.pletherOracle();
        assertEq(address(oracle), address(pletherOracle), "rel: router oracle");
        assertEq(oracle.adverseConfidenceMultiplierBps(), REL_ADVERSE_CONF_MULTIPLIER_BPS, "rel: adverse multiplier");
        assertEq(oracle.basketMaxConfidenceRatioBps(), REL_BASKET_MAX_CONF_RATIO_BPS, "rel: basket confidence");
        assertEq(oracle.orderExecutionStalenessLimit(), REL_ORDER_STALENESS, "rel: order staleness");
        assertEq(oracle.liquidationStalenessLimit(), REL_LIQUIDATION_STALENESS, "rel: liquidation staleness");
        assertEq(oracle.orderSettlementWindow(), REL_ORDER_SETTLEMENT_WINDOW, "rel: order settlement window");
        assertEq(
            oracle.maxComponentPublishTimeDivergence(), REL_MAX_COMPONENT_DIVERGENCE, "rel: component divergence"
        );

        assertEq(pool.maxSeniorExposureAssets(), REL_MAX_SENIOR_EXPOSURE_ASSETS, "rel: maxSeniorExposureAssets");
        assertEq(pool.maxSeniorShareBps(), REL_MAX_SENIOR_SHARE_BPS, "rel: maxSeniorShareBps");
        assertTrue(pool.isTradingActive(), "rel: trading active");

        assertEq(pool.poolConfigActivationTime(), 0, "rel: no pending pool proposal");
        assertEq(routerAdmin.routerConfigActivationTime(), 0, "rel: no pending router proposal");
        assertEq(routerAdmin.oracleConfigActivationTime(), 0, "rel: no pending oracle proposal");
        assertEq(engineAdmin.riskConfigActivationTime(), 0, "rel: no pending risk proposal");
        assertEq(engineAdmin.calendarConfigActivationTime(), 0, "rel: no pending calendar proposal");
        assertEq(engineAdmin.freshnessConfigActivationTime(), 0, "rel: no pending freshness proposal");
        assertFalse(routerAdmin.paused(), "rel: router not paused");
        assertFalse(pool.paused(), "rel: pool not paused");
    }

    // ---------------------------------------------------------------- actors

    function _pocRequireActor(
        address actor
    ) internal view {
        require(actor != address(0) && actor != address(this), "fixture: owner or zero cannot act");
        require(
            actor != address(engine) && actor != address(engineAdmin) && actor != address(pool)
                && actor != address(seniorVault) && actor != address(juniorVault) && actor != address(clearinghouse)
                && actor != address(router) && actor != address(routerAdmin) && actor != router.executionSidecar()
                && actor != address(pletherOracle) && actor != address(baseMockPyth)
                && actor != address(settlementToken),
            "fixture: protocol address cannot act"
        );
    }

    function _pocEnsureEth(
        address actor,
        uint256 weiAmount
    ) internal {
        if (actor.balance < weiAmount) vm.deal(actor, weiAmount);
    }

    function _traderDeposit(
        address trader,
        uint256 assets
    ) internal {
        _pocRequireActor(trader);
        settlementToken.mint(trader, assets);
        vm.startPrank(trader);
        settlementToken.approve(address(clearinghouse), assets);
        clearinghouse.deposit(trader, assets);
        vm.stopPrank();
    }

    /// @notice Production V3 commit with permissive-but-valid bounds.
    function _commitOpen(
        address trader,
        CfdTypes.Side side,
        uint256 size,
        uint256 margin
    ) internal returns (uint64 orderId) {
        _pocRequireActor(trader);
        uint256 window = router.maxExecutionWindowSeconds();
        OrderV3Types.OrderRequest memory request;
        request.clientOrderId = bytes32(++_clientNonce);
        request.side = side;
        request.sizeDelta = size;
        request.marginDelta = margin;
        request.targetPrice = side == CfdTypes.Side.LONG ? 1 : type(uint256).max;
        request.isClose = false;
        request.bounds = OrderV3Types.ExecutionBounds({
            submitBy: SafeCast.toUint64(vm.getBlockTimestamp() + window),
            executionWindowSeconds: SafeCast.toUint32(window),
            allowedExecutionModes: uint8(ALL_EXECUTION_MODES),
            expectedConfigHash: router.lifecycleBook().currentExecutionConfigHash(),
            maxExecutionBountyAssets: type(uint256).max,
            maxExecutionNotionalAssets: type(uint256).max,
            maxGrossAccountDebitAssets: type(uint256).max,
            maxActionChargeAssets: type(uint256).max,
            maxExplicitFeesAssets: type(uint256).max,
            maxPostPositionSize: type(uint256).max,
            minPostSettlementBalanceAssets: 0,
            minPostPositionEquityAssets: 0,
            maxPostLeverageBps: type(uint32).max
        });
        vm.prank(trader);
        orderId = OrderRouter(address(router)).commitOrder(request);
    }

    // ---------------------------------------------------------------- oracle

    /// @notice Advances one block and one second.
    function _pocTick() internal {
        vm.roll(vm.getBlockNumber() + 1);
        vm.warp(vm.getBlockTimestamp() + 1);
    }

    function _feedRawPrice(
        uint256 i,
        uint256 basketPrice
    ) internal view returns (uint256) {
        PletherOracle oracle = PletherOracle(address(router.pletherOracle()));
        require(!oracle.inversions(i), "fixture: inverted components unsupported");
        return (basketPrice * oracle.basePrices(i)) / 1e8;
    }

    /// @notice Explicit MockPyth payload for every configured feed at a basket-level price.
    function _pythPayload(
        uint256 basketPrice,
        uint64 basketConfidence,
        uint256 publishTime,
        uint256 prevPublishTime
    ) internal view returns (bytes[] memory data) {
        bytes32[] memory ids = _basePythFeedIds();
        MockPyth.FeedUpdate[] memory updates = new MockPyth.FeedUpdate[](ids.length);
        for (uint256 i; i < ids.length; ++i) {
            updates[i] = MockPyth.FeedUpdate(
                ids[i],
                MockPyth.MockPrice(
                    SafeCast.toInt64(SafeCast.toInt256(_feedRawPrice(i, basketPrice))),
                    SafeCast.toUint64(_feedRawPrice(i, basketConfidence)),
                    -8,
                    publishTime,
                    prevPublishTime
                )
            );
        }
        data = new bytes[](1);
        data[0] = abi.encode(updates);
    }

    function _setPrice(
        uint256 price
    ) internal {
        pocPrice = price;
        bytes32[] memory ids = _basePythFeedIds();
        uint256 nowTs = vm.getBlockTimestamp();
        for (uint256 i; i < ids.length; ++i) {
            baseMockPyth.setPrice(
                ids[i], SafeCast.toInt64(SafeCast.toInt256(_feedRawPrice(i, price))), uint64(0), int32(-8), nowTs
            );
        }
    }

    /// @notice Public OrderRouter.updateMarkPrice with a fresh tick and the exact Pyth fee.
    function _updateMark(
        address keeper,
        uint256 price
    ) internal {
        _pocRequireActor(keeper);
        _pocTick();
        uint256 nowTs = vm.getBlockTimestamp();
        bytes[] memory data = _pythPayload(price, 0, nowTs, nowTs - 1);
        uint256 fee = router.pletherOracle().getUpdateFee(data);
        _pocEnsureEth(keeper, fee);
        vm.prank(keeper);
        router.updateMarkPrice{value: fee}(data);
        pocPrice = price;
    }

    /// @notice Permissionless LP epoch settlement (direct pool route, or the Router route with a fresh payload).
    function _settleLpEpochPublic(
        address keeper
    ) internal {
        _pocRequireActor(keeper);
        ICfdEngineTypes.TerminalNavSnapshot memory snapshot = engine.terminalNavSnapshot();
        if (!snapshot.hasOpenPositions || engine.isOracleFrozen()) {
            vm.prank(keeper);
            pool.settleLpEpoch(0, 0);
            return;
        }
        _pocTick();
        uint256 nowTs = vm.getBlockTimestamp();
        bytes[] memory data = _pythPayload(pocPrice, 0, nowTs, nowTs - 1);
        uint256 fee = router.pletherOracle().getUpdateFee(data);
        _pocEnsureEth(keeper, fee);
        vm.prank(keeper);
        router.settleLpEpoch{value: fee}(data);
    }

    function _minOpenSize(
        uint256 price
    ) internal view returns (uint256) {
        uint256 minNotional = router.minOpenNotionalAssets();
        return ((minNotional + price - 1) / price) * CfdTypes.SIZE_QUANTUM;
    }

    function _max(
        uint256 a,
        uint256 b
    ) internal pure returns (uint256) {
        return a > b ? a : b;
    }

}

/// @notice executeOrderBatch reverts on the Router's own, freshly proven batch cache when the first post-commit signed
///         basket has unequal component publish times. The whole batch unwinds, including the order that had already
///         executed. Controls show the payload is valid and that the outcome depends only on unrelated Pyth storage.
contract BatchReuseCoveragePocTest is BatchReuseFixture {

    address internal alice;
    address internal bob;
    address internal keeper;
    uint256 internal size;

    function setUp() public override {
        super.setUp();
        alice = makeAddr("alice");
        bob = makeAddr("bob");
        keeper = makeAddr("keeper");
        _traderDeposit(alice, 1_000e6);
        _traderDeposit(bob, 1_000e6);
        size = _minOpenSize(pocPrice);
    }

    /// alice commits at t0, bob at t0 + 1 (next block). The keeper arrives at t0 + 4.
    function _ordersOneSecondApart() internal returns (uint64 idA, uint64 idB, uint64 t0) {
        _pocTick();
        t0 = uint64(vm.getBlockTimestamp());
        idA = _commitOpen(alice, CfdTypes.Side.LONG, size, 50e6);
        _pocTick();
        idB = _commitOpen(bob, CfdTypes.Side.SHORT, size, 50e6);
        _pocTick();
        _pocTick();
        _pocTick();
        assertEq(vm.getBlockTimestamp(), uint256(t0) + 4, "keeper time");
    }

    /// Honest first post-commit basket: feed0 published at `lagging`, every other feed at `newest`.
    function _unequalPayload(
        uint64 lagging,
        uint64 newest,
        uint64 previous
    ) internal view returns (bytes[] memory data) {
        data = _pythPayload(pocPrice, 0, newest, previous);
        MockPyth.FeedUpdate[] memory updates = abi.decode(data[0], (MockPyth.FeedUpdate[]));
        updates[0].price.publishTime = lagging;
        data[0] = abi.encode(updates);
    }

    function _status(
        uint64 id
    ) internal view returns (uint8) {
        return uint8(router.lifecycleBook().lifecycleStatus(id));
    }

    /// Conservative keeper funding: one historical execution fee per candidate order; the Router refunds the excess.
    function _batchFee(
        bytes[] memory data
    ) internal view returns (uint256) {
        return router.pletherOracle().getOrderExecutionFee(data) * 2;
    }

    /// 1. Orders one second apart: the batch reverts with StoredFeedBehind raised against the Router's own cache, and
    ///    the already executed first order is rolled back.
    function test_batchRevertsOnRoutersOwnCache_unequalComponentTimes() public {
        (uint64 idA, uint64 idB, uint64 t0) = _ordersOneSecondApart();
        bytes32 feed0 = _basePythFeedIds()[0];
        bytes[] memory data = _unequalPayload(t0 + 2, t0 + 3, t0);
        uint256 fee = _batchFee(data);
        _pocEnsureEth(keeper, fee);

        uint256 headBefore = router.nextExecuteId();
        uint256 markTimeBefore = engine.lastMarkTime();
        uint256 gasBefore = gasleft();
        vm.prank(keeper);
        (bool ok, bytes memory ret) =
            address(router).call{value: fee}(abi.encodeCall(router.executeOrderBatch, (idB, data)));
        uint256 gasBurned = gasBefore - gasleft();

        assertFalse(ok, "batch must revert");
        assertEq(
            ret,
            abi.encodeWithSelector(IPletherOracle.PletherOracle__StoredFeedBehind.selector, feed0, t0 + 2, t0 + 3),
            "StoredFeedBehind(feed0, lagging, newest)"
        );
        assertEq(router.nextExecuteId(), headBefore, "head unchanged: executed predecessor rolled back");
        assertEq(_status(idA), uint8(OrderV3Types.LifecycleStatus.Pending), "A still pending");
        assertEq(_status(idB), uint8(OrderV3Types.LifecycleStatus.Pending), "B still pending");
        assertEq(engine.lastMarkTime(), markTimeBefore, "mark update rolled back");
        emit log_named_uint("commit A (t0)", t0);
        emit log_named_uint("commit B", t0 + 1);
        emit log_named_uint("feed0 publishTime (lagging)", t0 + 2);
        emit log_named_uint("feed1 publishTime (newest) = cache.publishTime", t0 + 3);
        emit log_named_uint("keeper gas consumed by the reverted batch", gasBurned);
        _assertReleaseParams();
    }

    /// 2. Normal operation, no adversary: two traders commit in the SAME block. The first post-commit basket has
    ///    feed0 at t0 + 1 and feed1 at t0 + 2. The batch covering both orders reverts.
    function test_sameBlockCommits_batchReverts() public {
        _pocTick();
        uint64 t0 = uint64(vm.getBlockTimestamp());
        uint64 idA = _commitOpen(alice, CfdTypes.Side.LONG, size, 50e6);
        uint64 idB = _commitOpen(bob, CfdTypes.Side.SHORT, size, 50e6);
        _pocTick();
        _pocTick();
        _pocTick();
        bytes32 feed0 = _basePythFeedIds()[0];
        bytes[] memory data = _unequalPayload(t0 + 1, t0 + 2, t0);
        uint256 fee = _batchFee(data);
        _pocEnsureEth(keeper, fee);

        vm.prank(keeper);
        (bool ok, bytes memory ret) =
            address(router).call{value: fee}(abi.encodeCall(router.executeOrderBatch, (idB, data)));
        assertFalse(ok, "batch must revert");
        assertEq(
            ret,
            abi.encodeWithSelector(IPletherOracle.PletherOracle__StoredFeedBehind.selector, feed0, t0 + 1, t0 + 2),
            "StoredFeedBehind(feed0, lagging, newest)"
        );
        assertEq(_status(idA), uint8(OrderV3Types.LifecycleStatus.Pending), "A still pending");
        assertEq(_status(idB), uint8(OrderV3Types.LifecycleStatus.Pending), "B still pending");
        emit log_named_uint("same-block commit time (t0)", t0);
        _assertReleaseParams();
    }

    /// Control 1: same orders and timing with EQUAL component times execute in one batch with one parse (cache reuse).
    function test_control_equalComponentTimesBatchSucceeds() public {
        (uint64 idA, uint64 idB, uint64 t0) = _ordersOneSecondApart();
        bytes[] memory data = _pythPayload(pocPrice, 0, t0 + 3, t0);
        uint256 fee = _batchFee(data);
        _pocEnsureEth(keeper, fee);
        uint256 parsesBefore = baseMockPyth.parseUniqueCallCount();
        vm.prank(keeper);
        OrderV3Types.BatchResult memory result = router.executeOrderBatch{value: fee}(idB, data);
        assertEq(result.terminalCount, 2, "both executed");
        assertEq(baseMockPyth.parseUniqueCallCount() - parsesBefore, 1, "second order reused the cache");
        assertEq(_status(idA), uint8(OrderV3Types.LifecycleStatus.Executed), "A executed");
        assertEq(_status(idB), uint8(OrderV3Types.LifecycleStatus.Executed), "B executed");
        _assertReleaseParams();
    }

    /// Control 2: the SAME unequal payload executes both orders through two single executeOrder calls, so the payload
    ///            is valid for both orders and only the batch reuse coverage check rejects it.
    function test_control_singleExecutionWithSamePayloadSucceeds() public {
        (uint64 idA, uint64 idB, uint64 t0) = _ordersOneSecondApart();
        bytes[] memory data = _unequalPayload(t0 + 2, t0 + 3, t0);
        uint256 fee = router.pletherOracle().getOrderExecutionFee(data);
        _pocEnsureEth(keeper, 2 * fee);
        vm.prank(keeper);
        OrderV3Types.ExecutionResult memory rA = router.executeOrder{value: fee}(idA, data);
        vm.prank(keeper);
        OrderV3Types.ExecutionResult memory rB = router.executeOrder{value: fee}(idB, data);
        assertEq(uint8(rA.status), uint8(OrderV3Types.LifecycleStatus.Executed), "A executed");
        assertEq(uint8(rB.status), uint8(OrderV3Types.LifecycleStatus.Executed), "B executed");
        assertEq(engine.lastMarkTime(), t0 + 3, "engine mark = newest component time");
        _assertReleaseParams();
    }

    /// Control 3: the SAME unequal payload and the SAME batch succeed (with cache reuse) once the lagging feed's Pyth
    ///            storage already holds a newer tick, pushed by anyone beforehand. The batch outcome depends on
    ///            unrelated Pyth storage, not on the basket that prices the orders.
    function test_control_batchSucceedsWhenLaggingStorageAlreadyAhead() public {
        (uint64 idA, uint64 idB, uint64 t0) = _ordersOneSecondApart();
        bytes32 feed0 = _basePythFeedIds()[0];
        bytes[] memory data = _unequalPayload(t0 + 2, t0 + 3, t0);

        bytes[] memory later = _pythPayload(pocPrice, 0, t0 + 4, t0 + 2);
        MockPyth.FeedUpdate[] memory all = abi.decode(later[0], (MockPyth.FeedUpdate[]));
        MockPyth.FeedUpdate[] memory onlyFeed0 = new MockPyth.FeedUpdate[](1);
        onlyFeed0[0] = all[0];
        later[0] = abi.encode(onlyFeed0);
        uint256 pythFee = baseMockPyth.getUpdateFee(later);
        _pocEnsureEth(keeper, pythFee);
        vm.prank(keeper);
        baseMockPyth.updatePriceFeeds{value: pythFee}(later);
        (,,, uint256 storedFeed0,) = baseMockPyth.prices(feed0);
        assertEq(storedFeed0, t0 + 4, "feed0 storage ahead");

        uint256 fee = _batchFee(data);
        _pocEnsureEth(keeper, fee);
        uint256 parsesBefore = baseMockPyth.parseUniqueCallCount();
        vm.prank(keeper);
        OrderV3Types.BatchResult memory result = router.executeOrderBatch{value: fee}(idB, data);
        assertEq(result.terminalCount, 2, "both executed");
        assertEq(baseMockPyth.parseUniqueCallCount() - parsesBefore, 1, "second order reused the unequal-time cache");
        assertEq(_status(idA), uint8(OrderV3Types.LifecycleStatus.Executed), "A executed");
        assertEq(_status(idB), uint8(OrderV3Types.LifecycleStatus.Executed), "B executed");
        assertEq(engine.lastMarkTime(), t0 + 3, "fill priced from the parsed historical basket");
        _assertReleaseParams();
    }

}
```

### Expected output

```
Ran 5 tests for test/perps/poc/BatchReuseCoveragePoc.t.sol:BatchReuseCoveragePocTest
[PASS] test_batchRevertsOnRoutersOwnCache_unequalComponentTimes() (gas: 4440203)
Logs:
  commit A (t0): 1709805602
  commit B: 1709805603
  feed0 publishTime (lagging): 1709805604
  feed1 publishTime (newest) = cache.publishTime: 1709805605
  keeper gas consumed by the reverted batch: 1688963

[PASS] test_control_batchSucceedsWhenLaggingStorageAlreadyAhead() (gas: 4579141)
[PASS] test_control_equalComponentTimesBatchSucceeds() (gas: 4542110)
[PASS] test_control_singleExecutionWithSamePayloadSucceeds() (gas: 4557471)
[PASS] test_sameBlockCommits_batchReverts() (gas: 4422759)
Logs:
  same-block commit time (t0): 1709805602

Suite result: ok. 5 passed; 0 failed; 0 skipped
```
