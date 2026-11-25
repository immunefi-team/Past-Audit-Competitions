# \#100017 [SC-Medium] Batch execution reverts when a historical basket's feeds have different publish times, although each order executes alone

**Submitted on Nov 16th 2026 at 03:57:56 UTC by @Kirkeelee for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #100017
- **Report Type:** Smart Contract
- **Report severity:** Medium
- **Target:** https://github.com/immunefi-team/audit-comp-plether/tree/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src
- **Impacts:**
  - Contract fails to deliver promised returns, but doesn't lose value

## Description

## Brief/Intro

In a batch, the first order resolves a historical basket and later orders reuse it. The first resolution checks each stored feed against that feed's own publish time. Reuse checks every stored feed against the basket's newest publish time. When the six feeds were not all published in the same second, which is normal for live Pyth payloads, reuse reverts with `PletherOracle__StoredFeedBehind`. The Router has already chosen reuse and does not fall back to a fresh parse, so the whole batch reverts, including orders that already executed. Each order executes fine on its own with the same prices.

## Vulnerability Details

The first historical resolution parses each feed's post-commit update, stores it, and checks each feed against its own time:

```solidity
// PletherOracle.sol:621-626
pyth.updatePriceFeeds{value: pythFee}(pythUpdateData);
for (uint256 i; i < parsedFeeds.length; ++i) {
    _requireStoredFeedCoverage(parsedFeeds[i].id, parsedFeeds[i].price.publishTime);
}
```

The batch cache keeps only the newest (`publishTime`) and oldest times. Reuse checks every feed against the newest:

```solidity
// PletherOracle.sol:262-266
if (!policy.oracleFrozen && _canReuseHistoricalBatchBasket(request.commitTime, cache)) {
    for (uint256 i; i < pythFeedIds.length; ++i) {
        _requireStoredFeedCoverage(pythFeedIds[i], cache.publishTime);
    }
```

If five feeds were published at `T-1` and one at `T`, the first order passes, but reuse requires all six stored feeds to be at `T` and reverts on the first feed at `T-1`.

The Router decides on reuse with its own copy of the eligibility check (`_canReuseHistoricalBatchBasket`, OrderRouterExecutionSidecar.sol:1074-1093). That copy checks commit time and the oldest/newest times, but not stored coverage, so it never sends the order to a fresh parse. ORACLE_SYNCHRONIZATION.md:59-62 says reuse requires cache coverage and that "otherwise the Router requests a new historical resolution".

Suggested fix: include the coverage check in the Router's reuse decision and fall back to a fresh historical resolution when it fails, or keep each feed's publish time in the cache and check each feed against its own.

## Impact Details

- Any batch with two or more reuse-eligible orders reverts whenever the first basket's feeds have different publish times. Live Pyth payloads routinely do (ORACLE_SYNCHRONIZATION.md:145-146).
- Orders already executed earlier in the batch roll back, and the keeper's gas is spent.
- Retrying the same batch fails the same way until newer updates for the lagging feeds are stored; sent one by one, the orders execute.

## References

- https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/PletherOracle.sol#L596-L626
- https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/PletherOracle.sol#L262-L274
- https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/OrderRouterExecutionSidecar.sol#L1074-L1093
- https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/ORACLE_SYNCHRONIZATION.md#L59-L62


## Proof of Concept

## Proof of Concept


Save as `test/scripts/ReviewBatchCachePoC.t.sol` in the repository root (next to `MockUSDGPerpsLifecycle.t.sol`, whose release-deployment setup it copies) and run from the repository root:

```bash
forge test --match-path test/scripts/ReviewBatchCachePoC.t.sol -vv
```

- `test_Recheck_K0082_ValidUnequalTimesBreakBatchButSinglesExecute`: two traders commit opens at the same time. Each feed gets one post-commit update: five at `T-1`, one at `T`, all at the same 1.20 basket. The batch reverts with `StoredFeedBehind(feed, T-1, T)` and the first order's position is rolled back. Then both orders execute one by one with the same updates.

```solidity
// SPDX-License-Identifier: UNLICENSED
pragma solidity 0.8.35;

import {IPletherOracle} from "@plether/perps/interfaces/IPletherOracle.sol";
import {IPositionProtectionActions} from "@plether/perps/interfaces/IPositionProtectionActions.sol";
import {PositionProtectionTypes} from "@plether/perps/interfaces/PositionProtectionTypes.sol";


import {DeployPerpsArbitrumSepolia, MockUSDG} from "../../script/DeployPerpsArbitrumSepolia.s.sol";
import {MockPyth} from "../mocks/MockPyth.sol";
import {CfdEngine} from "@plether/perps/CfdEngine.sol";
import {CfdTypes} from "@plether/perps/CfdTypes.sol";
import {EmergencyPauseCoordinator} from "@plether/perps/EmergencyPauseCoordinator.sol";
import {HousePool} from "@plether/perps/HousePool.sol";
import {MarginClearinghouse} from "@plether/perps/MarginClearinghouse.sol";
import {OrderLifecycleBook} from "@plether/perps/OrderLifecycleBook.sol";
import {OrderRouter} from "@plether/perps/OrderRouter.sol";
import {OrderV3Types} from "@plether/perps/OrderV3Types.sol";
import {PletherOracle} from "@plether/perps/PletherOracle.sol";
import {TerminalNavBookV2} from "@plether/perps/TerminalNavBookV2.sol";
import {TrancheVault} from "@plether/perps/TrancheVault.sol";
import {IHousePool} from "@plether/perps/interfaces/IHousePool.sol";
import {ICfdEngineTypes} from "@plether/perps/interfaces/ICfdEngineTypes.sol";
import {Test} from "forge-std/Test.sol";
import {console2} from "forge-std/console2.sol";

/// @dev Mirrors the Arbitrum precompile's block-number view inside the local test EVM.
contract PoCArbSys {
    function arbBlockNumber() external view returns (uint256) {
        return block.number;
    }
}

/// @notice Same setup as the repo's test/scripts/MockUSDGPerpsLifecycle.t.sol: runs the release deploy script
/// (script/DeployPerpsArbitrumSepolia.s.sol, release parameters) inside the test EVM. Only Pyth and USDG are mocks.
/// Adds 1,000,000 USDG of Junior and 1,000,000 USDG of Senior liquidity through the normal async LP flow.
abstract contract ReleaseStackPoCBase is Test {
    address internal constant RELEASE_PYTH = 0x0B73614636C855Bf23F342F307FB981A3e47f42B;
    address internal constant ARBSYS = address(100);
    uint256 internal constant DEPLOYER_KEY = 0xA11CE;
    /// @dev Monday 2024-03-04 10:00 UTC (FX market open, no FAD window) — same anchor as the repo tests.
    uint256 internal constant MONDAY_MORNING = 1_709_532_000;
    uint256 internal constant CAP_PRICE = 2e8;
    uint256 internal constant START_PRICE = 120_000_000; // basket mark 1.20 (8 decimals)
    uint256 internal constant SIZE_QUANTUM = 100e18; // positionSizeQuantum = 1e20 raw units
    uint256 internal constant JUNIOR_LP_ASSETS = 1_000_000e6;
    uint256 internal constant SENIOR_LP_ASSETS = 1_000_000e6;
    uint8 internal constant MODE_LIVE = 1; // allowedExecutionModes bit for ExecutionMode.Live (1 << (Live-1))
    uint8 internal constant MODES_ALL = 7; // Live | Fad | Frozen

    DeployPerpsArbitrumSepolia.DeployedContracts internal stack;
    MockUSDG internal usdg;
    CfdEngine internal engine;
    HousePool internal pool;
    TrancheVault internal seniorVault;
    TrancheVault internal juniorVault;
    MarginClearinghouse internal clearinghouse;
    OrderRouter internal router;
    OrderLifecycleBook internal lifecycleBook;
    TerminalNavBookV2 internal navBook;
    EmergencyPauseCoordinator internal coordinator;
    PletherOracle internal oracle;
    MockPyth internal pyth;

    /// Trusted roles: OWNER (deployer, owner of every perps contract) and GUARDIAN.
    address internal OWNER;
    address internal GUARDIAN;
    /// Untrusted actors (anyone can be a keeper).
    address internal KEEPER;
    address internal JUNIOR_LP;
    address internal SENIOR_LP;

    uint256 internal currentBasketPrice;
    uint256 internal _clientNonce;

    function setUp() public virtual {
        vm.chainId(421_614);
        vm.warp(MONDAY_MORNING);
        vm.etch(ARBSYS, address(new PoCArbSys()).code);
        vm.etch(RELEASE_PYTH, address(new MockPyth()).code);
        pyth = MockPyth(RELEASE_PYTH);
        vm.setEnv("TEST_PRIVATE_KEY", vm.toString(DEPLOYER_KEY));
        stack = DeployPerpsArbitrumSepolia(
            deployCode("DeployPerpsArbitrumSepolia.s.sol:DeployPerpsArbitrumSepolia")
        ).run();

        usdg = stack.settlementToken;
        engine = stack.engine;
        pool = stack.housePool;
        seniorVault = stack.seniorVault;
        juniorVault = stack.juniorVault;
        clearinghouse = stack.clearinghouse;
        router = stack.router;
        lifecycleBook = stack.lifecycleBook;
        navBook = stack.terminalNavBook;
        coordinator = stack.emergencyPauseCoordinator;
        oracle = PletherOracle(stack.pletherOracle);

        OWNER = vm.addr(DEPLOYER_KEY);
        GUARDIAN = makeAddr("guardian");
        KEEPER = makeAddr("keeper");
        JUNIOR_LP = makeAddr("juniorLP");
        SENIOR_LP = makeAddr("seniorLP");
        vm.label(OWNER, "OWNER");

        // Production bootstrap (DEPLOYMENT.md): guardian set, 1 USDG permanent seed per tranche, trading activated.
        vm.startPrank(OWNER);
        coordinator.setGuardian(GUARDIAN);
        usdg.mint(OWNER, 2e6);
        usdg.approve(address(pool), 2e6);
        pool.initializeSeedPosition(false, 1e6, OWNER);
        pool.initializeSeedPosition(true, 1e6, OWNER);
        pool.activateTrading();
        vm.stopPrank();

        _setBasketPrice(START_PRICE);
        router.updateMarkPrice(_updateData());

        // Ordinary LP liquidity through the async epoch flow (Junior first, then Senior), like the v2.0.0 testnet.
        _lpDeposit(juniorVault, JUNIOR_LP, JUNIOR_LP_ASSETS);
        _lpDeposit(seniorVault, SENIOR_LP, SENIOR_LP_ASSETS);
    }

    // ------------------------------------------------------------------ oracle -----------------------------------
    /// @notice Moves every release feed by the same normalized factor so the basket mark becomes `basketPrice`
    /// (8 decimals), publishTime = now. Inverse FX feeds keep their own units. Does NOT push it on-chain: pass
    /// `_updateData()` to a Router call (executeOrder / updateMarkPrice / settleLpEpoch / liquidation).
    function _setBasketPrice(uint256 basketPrice) internal {
        _setBasketPriceAt(basketPrice, vm.getBlockTimestamp());
    }

    function _setBasketPriceAt(uint256 basketPrice, uint256 publishTime) internal {
        currentBasketPrice = basketPrice;
        for (uint256 i; i < 6; ++i) {
            uint256 normalizedPrice = oracle.basePrices(i) * basketPrice / 1e8;
            uint256 feedPrice = oracle.inversions(i) ? 1e16 / normalizedPrice : normalizedPrice;
            pyth.setPrice(oracle.pythFeedIds(i), int64(uint64(feedPrice)), int32(-8), publishTime);
        }
    }

    /// @dev Non-encoded payload: MockPyth answers with the individually configured feed prices.
    function _updateData() internal pure returns (bytes[] memory data) {
        data = new bytes[](1);
        data[0] = "";
    }

    function _pushMark(uint256 basketPrice) internal {
        _setBasketPrice(basketPrice);
        vm.prank(KEEPER);
        router.updateMarkPrice(_updateData());
    }

    // ------------------------------------------------------------------ LPs --------------------------------------
    /// @notice requestDeposit -> warp to the target epoch -> keeper settles via Router (atomic oracle refresh) -> claim.
    function _lpDeposit(TrancheVault vault, address lp, uint256 assets) internal returns (uint256 shares) {
        usdg.mint(lp, assets);
        vm.startPrank(lp);
        usdg.approve(address(vault), assets);
        uint256 requestId = vault.requestDeposit(assets, lp, lp);
        vm.stopPrank();
        _settleEpoch(requestId);
        vm.prank(lp);
        shares = vault.claimDeposit(requestId, assets, lp, lp);
    }

    /// @notice requestRedeem (after the deposit cooldown) -> settle the target epoch -> returns requestId (claim it
    /// yourself with vault.claimRedeem(requestId, shares, lp, lp)).
    function _lpRequestRedeemAndSettle(TrancheVault vault, address lp, uint256 shares)
        internal
        returns (uint256 requestId)
    {
        uint256 cooldownEnd = vault.lastDepositTime(lp) + vault.DEPOSIT_COOLDOWN();
        if (vm.getBlockTimestamp() < cooldownEnd) vm.warp(cooldownEnd);
        vm.prank(lp);
        requestId = vault.requestRedeem(shares, lp, lp);
        _settleEpoch(requestId);
    }

    function _settleEpoch(uint256 epochId) internal {
        uint256 start = pool.lpEpochStart(epochId);
        if (vm.getBlockTimestamp() < start) vm.warp(start);
        vm.roll(vm.getBlockNumber() + 1);
        _setBasketPrice(currentBasketPrice);
        vm.prank(KEEPER);
        router.settleLpEpoch(_updateData());
    }

    // ------------------------------------------------------------------ traders ----------------------------------
    function _fundTrader(address trader, uint256 assets) internal {
        usdg.mint(trader, assets);
        vm.startPrank(trader);
        usdg.approve(address(clearinghouse), assets);
        clearinghouse.depositMargin(assets);
        vm.stopPrank();
    }

    /// @notice Permissive bounds (live mode only by default); tighten fields in your own request if the test needs to.
    function _bounds(uint256 maxPostSize) internal view returns (OrderV3Types.ExecutionBounds memory b) {
        b = OrderV3Types.ExecutionBounds({
            submitBy: uint64(vm.getBlockTimestamp() + 60),
            executionWindowSeconds: 60,
            allowedExecutionModes: MODE_LIVE,
            expectedConfigHash: lifecycleBook.currentExecutionConfigHash(),
            maxExecutionBountyAssets: type(uint128).max,
            maxExecutionNotionalAssets: type(uint128).max,
            maxGrossAccountDebitAssets: type(uint128).max,
            maxActionChargeAssets: type(uint128).max,
            maxExplicitFeesAssets: type(uint128).max,
            maxPostPositionSize: maxPostSize,
            minPostSettlementBalanceAssets: 0,
            minPostPositionEquityAssets: 0,
            maxPostLeverageBps: type(uint32).max
        });
    }

    /// @dev LONG gains when the basket falls: an opening LONG accepts any fill >= targetPrice, a closing LONG any
    /// fill <= targetPrice (and the reverse for SHORT). Permissive targets: 1 / CAP_PRICE.
    function _permissiveTarget(CfdTypes.Side side, bool isClose) internal pure returns (uint256) {
        bool wantsHigh = (side == CfdTypes.Side.LONG) != isClose;
        return wantsHigh ? 1 : CAP_PRICE;
    }

    function _commit(address trader, CfdTypes.Side side, uint256 sizeDelta, uint256 marginDelta, bool isClose)
        internal
        returns (uint64 orderId)
    {
        OrderV3Types.OrderRequest memory request;
        request.clientOrderId = keccak256(abi.encode("poc", trader, ++_clientNonce));
        request.side = side;
        request.sizeDelta = sizeDelta;
        request.marginDelta = marginDelta;
        request.targetPrice = _permissiveTarget(side, isClose);
        request.isClose = isClose;
        request.bounds = _bounds(isClose ? type(uint128).max : type(uint128).max);
        vm.prank(trader);
        orderId = router.commitOrder(request);
    }

    /// @notice Keeper executes `orderId` at basket price `fillPrice` after the minimum delay (7 s, 2 blocks).
    function _execute(uint64 orderId, uint256 fillPrice) internal returns (OrderV3Types.ExecutionResult memory r) {
        vm.warp(vm.getBlockTimestamp() + 7);
        vm.roll(vm.getBlockNumber() + 2);
        _setBasketPrice(fillPrice);
        vm.prank(KEEPER);
        r = router.executeOrder(orderId, _updateData());
    }

    function _open(address trader, CfdTypes.Side side, uint256 size, uint256 margin, uint256 fillPrice)
        internal
        returns (OrderV3Types.ExecutionResult memory r)
    {
        r = _execute(_commit(trader, side, size, margin, false), fillPrice);
    }

    function _close(address trader, CfdTypes.Side side, uint256 size, uint256 fillPrice)
        internal
        returns (OrderV3Types.ExecutionResult memory r)
    {
        r = _execute(_commit(trader, side, size, 0, true), fillPrice);
    }

    function _positionSize(address trader) internal view returns (uint256 size) {
        (size,,,,,,) = engine.positions(trader);
    }

    function _assertExecuted(OrderV3Types.ExecutionResult memory r, string memory what) internal pure {
        assertEq(uint8(r.status), uint8(OrderV3Types.LifecycleStatus.Executed), what);
    }
}



contract ReviewBatchCachePoC is ReleaseStackPoCBase {
    function _tick(uint256 dt) internal {
        vm.warp(vm.getBlockTimestamp() + dt);
        vm.roll(vm.getBlockNumber() + 1);
    }

    function test_Recheck_K0082_ValidUnequalTimesBreakBatchButSinglesExecute() public {
        address first = makeAddr("firstBatchTrader");
        address second = makeAddr("secondBatchTrader");
        _fundTrader(first, 10_000e6);
        _fundTrader(second, 10_000e6);
        uint64 orderA = _commit(first, CfdTypes.Side.LONG, 10_000e18, 1_000e6, false);
        uint64 orderB = _commit(second, CfdTypes.Side.LONG, 10_000e18, 1_000e6, false);
        _tick(7);
        vm.roll(vm.getBlockNumber() + 1);
        uint256 latest = vm.getBlockTimestamp();
        for (uint256 i; i < 6; ++i) {
            uint256 normalized = oracle.basePrices(i) * START_PRICE / 1e8;
            uint256 componentPrice = oracle.inversions(i) ? 1e16 / normalized : normalized;
            pyth.setPrice(oracle.pythFeedIds(i), int64(uint64(componentPrice)), int32(-8), i == 5 ? latest : latest - 1);
        }

        // All components are valid first-post-commit reports with only one second of dispersion.
        // Creation covers each component's own time; reuse incorrectly demands the newest time for all.
        vm.expectRevert(abi.encodeWithSelector(
            IPletherOracle.PletherOracle__StoredFeedBehind.selector,
            oracle.pythFeedIds(0), latest - 1, latest
        ));
        router.executeOrderBatch(orderB, _updateData());
        assertEq(_positionSize(first), 0, "batch prefix rolled back");
        _assertExecuted(router.executeOrder(orderA, _updateData()), "first single succeeds with identical feeds");
        _assertExecuted(router.executeOrder(orderB, _updateData()), "second single succeeds with identical feeds");
    }
}
```

Output (commit a82d4fef):

```
Ran 1 test for test/scripts/ReviewBatchCachePoC.t.sol:ReviewBatchCachePoC
[PASS] test_Recheck_K0082_ValidUnequalTimesBreakBatchButSinglesExecute() (gas: 8930944)
Suite result: ok. 1 passed; 0 failed; 0 skipped
```

