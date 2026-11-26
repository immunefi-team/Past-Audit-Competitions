# \#98416 [SC-High] Calendar override days adjacent to the weekend exceed fadMaxStaleness: frozen closes, liquidations, LP epoch settlement and withdrawals revert for 3–24 h (temporary freezing of funds)

**Submitted on Nov 7th 2026 at 20:16:41 UTC by @Magnum for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #98416
- **Report Type:** Smart Contract
- **Report severity:** High
- **Target:** https://github.com/immunefi-team/audit-comp-plether/tree/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src
- **Impacts:**
  - Temporary freezing of funds for at least 24 hours

## Description

### Brief/Intro

Two independently timelocked engine settings are never cross-checked: `EngineCalendarConfig.fadDayTimestamps` (governance holiday override days, each freezing the oracle for a full UTC day) and `EngineFreshnessConfig.fadMaxStaleness` (maximum age of the cached mark while the oracle is frozen, default 3 days). The standard weekend freeze (Friday 17:00 NY → Sunday 17:00 NY) is 48 h. One override day directly after the weekend makes the freeze continuous until the following UTC midnight — 75 h in DST — and two override days bracketing the weekend (the canonical FX holiday pair Good Friday + Easter Monday) make it 96 h. The frozen basket is the engine's cached mark, and `PletherOracle._frozenBasket` reverts once that mark is older than `fadMaxStaleness`. No live tick can be installed while the calendar says frozen, so from `lastMarkTime + 3 days` until the calendar reopens every frozen-mode action reverts: frozen voluntary close execution, single and batch liquidation, Router `settleLpEpoch`, direct `HousePool.settleLpEpoch` and `updateMarkPrice`. Traders cannot exit, under-collateralised positions cannot be liquidated, LP redemptions matured for the frozen epochs cannot be funded. SECURITY.md promises that "frozen-oracle windows prioritize close liveness" and the program's own containment assumption states "Closes, liquidations, and already-funded claims remain available"; both are false for the whole stale window. The outage is 10,919 s (~3 h) with a single Monday override and 86,999 s (~24.2 h) with Friday + Monday overrides (PoC values).

### Vulnerability Details

**1. The frozen basket is the cached mark, bounded only by `fadMaxStaleness`.**

`PletherOracle._frozenBasket` (src/PletherOracle.sol:729-743):
```solidity
basket.price = engine.lastMarkPrice();
basket.publishTime = engine.lastMarkTime();
...
if (OracleFreshnessPolicyLib.isStale(basket.publishTime, maxStaleness, block.timestamp)) {
    revert PletherOracle__StalePrice(mode, bytes32(0), basket.publishTime, maxStaleness, block.timestamp);
}
```
`maxStaleness` for every mode while `oracleFrozen` is `engine.fadMaxStaleness()` (src/libraries/OracleFreshnessPolicyLib.sol:61-71; `CfdEngine.fadMaxStaleness = 3 days`, src/CfdEngine.sol:140).

**2. Every frozen-mode consumer routes through `_frozenBasket`, and the live path is unreachable while frozen.**

- Frozen order execution: `_updateAndGetSnapshot` / historical order basket → `_frozenBasket` (src/PletherOracle.sol:576, 592).
- Liquidations and mark refresh: `_resolveCurrentBasket` returns `_frozenBasket` whenever `policy.oracleFrozen` (src/PletherOracle.sol:749-756), *before* looking at any supplied Pyth payload. A fresh signed tick is therefore rejected until the calendar reopens.
- LP settlement: `OrderRouter.settleLpEpoch` goes through the same oracle path; the direct `HousePool.settleLpEpoch(0,0)` route is permissionless while frozen but reverts `HousePool__MarkPriceStale` (selector `0x40497d8e`) on the same stale mark.

**3. The calendar can keep the oracle frozen longer than 3 days.**

`MarketCalendarLib.marketStatus` (src/libraries/MarketCalendarLib.sol:30-58) returns `(true, true)` for any UTC day flagged in `fadDayOverrides`; weekend logic covers Friday from 17:00 NY through Sunday 17:00 NY. Combinations:

| Calendar | Last live mark | Frozen until | Frozen length | Stale-frozen outage (default 3 d) |
|---|---|---|---|---|
| Plain weekend (DST) | Fri 20:59 UTC | Sun 21:00 UTC | 48 h | none |
| Weekend + Monday override | Fri 20:59 UTC | Tue 00:00 UTC | 75 h | ~3 h (Mon 20:59 → Tue 00:00) |
| Friday override + weekend | Thu 23:59 UTC | Sun 21:00/22:00 UTC | 69-70 h | none |
| Friday + Monday overrides (Good Friday + Easter Monday) | Thu 23:59 UTC | Tue 00:00 UTC | 96 h | ~24 h (Sun 23:59 → Tue 00:00) |

**4. Nothing validates the pair.** `CfdEngine.applyCalendarConfig` (src/CfdEngine.sol:689-708) accepts any set of days; `CfdEngineAdmin.proposeFreshnessConfig` (src/CfdEngineAdmin.sol:207-212) only requires `fadMaxStaleness != 0`. Neither checks the other, and SECURITY.md:1337-1338 instructs governance to use UTC-day overrides for "unscheduled provider closures and holiday hours" without mentioning the staleness budget. Overrides are a legitimate, documented trusted action (program page: "a legitimate, documented action by a trusted role that triggers a bug harming users or the protocol" is in scope); the harmful outcome is produced by the code, not by misuse.

**5. Nobody can shorten the outage.** Both `EngineCalendarConfig` and `EngineFreshnessConfig` sit behind the 72-hour `CfdEngineAdmin` timelock; the guardian/EmergencyPauseCoordinator can only pause. `updateMarkPrice` with a freshly signed Pyth payload is rejected because `_resolveCurrentBasket` returns the frozen basket before reading the payload (PoC step "fresh tick refused").

**Why this is not a known issue.**
- SECURITY.md Known Limitations (L1332) state the opposite guarantee: "frozen-oracle windows prioritize close liveness". L1337-1338 and WHITEPAPER.md L1120 instruct governance to use UTC-day overrides for holidays without any constraint relative to `fadMaxStaleness`.
- WHITEPAPER.md L1112 lists a regime row "Frozen beyond max staleness: blocked / wait for valid data". That row documents the state machine's output, not that the protocol's own calendar forces that state while every operational instruction is followed. ORACLE_SYNCHRONIZATION.md L77-79 gives the operational mitigation — "install a final live mark before the scheduled freeze" — and the PoC does exactly that (final mark 2 minutes before the freeze); the outage still occurs, because the freeze is longer than the budget by construction. The whitepaper and ORACLE_SYNCHRONIZATION.md are not among the known-issue sources listed on the program page.
- "Setting a parameter to a value the code allows is a trusted action": no single parameter is set to a bad value. In the Monday case both are at their documented defaults (`fadMaxStaleness = 3 days`, one holiday override as instructed); the defect is the missing cross-validation between two independently valid settings and the built-in weekend calendar.
- Cecuro MED-20 and SC Audit Studio SC-M1 (both fixed) were accepted as Medium for the same impact class — closes/liquidations unavailable inside a frozen window — with different root causes (divergence limits). MED-30 / LOW-73 and the AI-scan calendar-digest finding concern stop-loss triggering and monitor digests. No report, scan entry or SECURITY.md line covers the frozen interval exceeding the frozen mark budget.
- The project's own tests (`test/perps/spec/oracle/FadStaleness.t.sol`) exercise override days only on a Wednesday and reach "excess staleness" only by artificially ageing the mark or shrinking the budget to 3600 s; the weekend-adjacent override combination is untested.

### Impact Details

During the stale-frozen window (deployed configuration, Arbitrum One):

- **Traders:** committed closes execute into `PletherOracle__StalePrice`; new closes can be committed but cannot execute; positions cannot be reduced for the whole window. Orders committed in the window expire unexecuted (60 s window). With Good Friday + Easter Monday overrides the lock-out is ~24 h.
- **LPs:** `settleLpEpoch` reverts on both routes, so matured redemption epochs cannot be funded and deposits cannot activate; LP funds are frozen for the same period.
- **Protocol/LPs:** no liquidation is possible. At release parameters (0.1 % maintenance, 3 % FAD margin) every position that becomes under-collateralised against the Monday reopening gap stays open and unliquidatable until Tuesday, and the write-off above pledge is borne by LPs. The documented FAD rationale (elevated margin around closures) depends on liquidations being executable in the frozen regime; here they are not.
- **Trader free settlement:** `clearinghouse.withdraw` for any account with an open position runs the engine withdraw guard, whose `_liveMark` uses the same 3-day frozen budget; it reverts `CfdEngine__MarkPriceStale` for the whole window (PoC step 4). Cash that is not backing the position is frozen too.
- **Containment assumption on the program page** ("Closes, liquidations, and already-funded claims remain available") does not hold for closes and liquidations.

Realistic occurrence of the Monday case: the FX market is closed globally on 25 December and 1 January (Pyth FX feeds otherwise trade through most US holidays), and both fall on a Monday in 2028 (Christmas) and 2029 (New Year), again in 2034/2035. With the default budget and a single documented holiday override each of those weekends produces a 2-3 h outage. No attacker is required.

### Recommended fix

1. Validate the pair at finalization: reject a calendar config whose longest contiguous frozen interval (weekend + adjacent override days + FAD runway) exceeds `fadMaxStaleness`, and reject a freshness config whose `fadMaxStaleness` is shorter than the longest configured frozen interval.
2. Alternatively, let frozen-mode paths accept a freshly signed live basket when the cached mark has expired (fall through to `_resolveCurrentBasket` with `PoolReconcile`/`Liquidation` staleness), or let governance extend `fadMaxStaleness` without the 72-hour timelock while the calendar is frozen.
3. Document the constraint next to the override guidance in SECURITY.md.

### References

- src/PletherOracle.sol L729-743 (`_frozenBasket`), L576, L592 (frozen order execution), L749-756 (`_resolveCurrentBasket` frozen early return)
- src/libraries/OracleFreshnessPolicyLib.sol L57-71 (`maxStaleness = fadMaxStaleness` while frozen, all modes)
- src/CfdEngine.sol L140 (`fadMaxStaleness = 3 days`), L689-708 (`applyCalendarConfig`, no validation), L711-716 (`applyFreshnessConfig`), L897-907 (`_marketStatus`)
- src/CfdEngineAdmin.sol L155-178 (calendar proposal/finalization), L207-212 (freshness validation: nonzero only)
- src/libraries/MarketCalendarLib.sol L30-58 (`marketStatus`, override days)
- src/HousePool.sol L797-822 (`settleLpEpoch`, `HousePool__MarkPriceStale`)
- packages/perps/SECURITY.md L1332 ("frozen-oracle windows prioritize close liveness"), L1337-1338 (holiday overrides)

## Link to Proof of Concept

https://gist.github.com/Magnat79A/03e90c8bfe9bbd0fa294ed6fa94098d7

## Proof of Concept

### Proof of Concept

Foundry, competition commit a82d4fefb02768e5b3bddcc97dee371eee24fea8, stock `BasePerpTest` fixture, production `OrderRouter → PletherOracle → CfdEngine → HousePool` path, calendar override set through the real 72-hour `CfdEngineAdmin` timelock. Default `fadMaxStaleness = 3 days`. Dates: Friday 2024-10-18 (DST, freeze 21:00 UTC), override days Friday 2024-10-18 and/or Monday 2024-10-21 (UTC).

Steps
1. Save the file below as `packages/perps/test/perps/poc/PoC_FrozenStalenessOverride.t.sol`.
2. Run: `FOUNDRY_PROFILE=quick forge test --root packages/perps --match-path "test/perps/poc/PoC_FrozenStalenessOverride.t.sol" -vvv`
3. Observe:
   - `test_PoC_OverrideDayExtendsFreezePastFadMaxStaleness` (Monday override): oracle frozen on Monday; a frozen close executes at Monday 01:00 UTC while the mark is < 3 days old; at Monday 22:00 UTC (mark 3 d 1 h old) a committed close reverts `PletherOracle__StalePrice(OrderExecution, 0x0, lastMark, 259200, now)`, `executeLiquidation` reverts `PletherOracle__StalePrice(Liquidation, …)`, Router `settleLpEpoch` reverts, direct `pool.settleLpEpoch(0,0)` reverts `HousePool__MarkPriceStale`, `clearinghouse.withdraw(TRADER, 1 USDG)` reverts `CfdEngine__MarkPriceStale`, and `updateMarkPrice` with a fresh signed tick reverts; at Tuesday 00:00:10 UTC the calendar is live and a tick restores service. Logged outage: **10,919 s**.
   - `test_PoC_TwoOverrideDays_FreezeExceeds24h` (Friday + Monday overrides): frozen on all four days; logged frozen-stale window **86,999 s (> 24 h)**; at Monday 12:00 UTC close execution, liquidation and LP settlement all revert.

Result
```
Ran 1 test for test/perps/poc/PoC_FrozenStalenessOverride.t.sol:PoC_FrozenStalenessTwoOverridesTest
[PASS] test_PoC_TwoOverrideDays_FreezeExceeds24h() (gas: 4003482)
Logs:
  frozen-stale window (s): 86999
Suite result: ok. 1 passed; 0 failed; 0 skipped

Ran 1 test for test/perps/poc/PoC_FrozenStalenessOverride.t.sol:PoC_FrozenStalenessOneOverrideTest
[PASS] test_PoC_OverrideDayExtendsFreezePastFadMaxStaleness() (gas: 5672921)
Logs:
  last live mark (unix): 1729285081
  fadMaxStaleness (s):   259200
  direct pool.settleLpEpoch reverted: 0x40497d8e
  frozen-stale window length (s): 10919
Suite result: ok. 1 passed; 0 failed; 0 skipped
```

Test file
```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

// PoC: a governance FAD/frozen override day adjacent to the weekend extends the frozen window beyond
// fadMaxStaleness (3 days). Once the cached mark is older than that, every frozen-mode path reverts with
// PletherOracle__StalePrice: frozen closes, liquidations and LP epoch settlement are all unavailable until the
// calendar reopens and a live tick arrives.
//
// Run: FOUNDRY_PROFILE=quick forge test --root packages/perps \
//        --match-path "test/perps/poc/PoC_FrozenStalenessOverride.t.sol" -vvv

import {BasePerpTest} from "../BasePerpTest.sol";
import {CfdTypes} from "@plether/perps/CfdTypes.sol";
import {ICfdEngineAdminHost} from "@plether/perps/interfaces/ICfdEngineAdminHost.sol";
import {IPletherOracle} from "@plether/perps/interfaces/IPletherOracle.sol";
import {OrderV3Types} from "@plether/perps/OrderV3Types.sol";
import {IHousePool} from "@plether/perps/interfaces/IHousePool.sol";
import {ICfdEngineTypes} from "@plether/perps/interfaces/ICfdEngineTypes.sol";
import {console} from "forge-std/console.sol";

abstract contract PoC_FrozenStalenessBase is BasePerpTest {

    address internal constant TRADER = address(0x5E1F);
    address internal constant KEEPER = address(0xCAFE);

    /// @dev Wednesday 2024-10-16 12:00 UTC (live market).
    uint256 internal constant WEDNESDAY_NOON = 1_729_080_000;
    /// @dev Friday 2024-10-18 21:00 UTC == 17:00 New York (EDT): weekend oracle freeze begins.
    uint256 internal constant FRIDAY_FREEZE = 1_729_285_200;
    /// @dev Monday 2024-10-21 00:00 UTC: governance FAD/frozen override day (UTC day).
    uint256 internal constant MONDAY_UTC = 1_729_468_800;
    /// @dev Friday 2024-10-18 00:00 UTC (used by the two-override variant: Friday + Monday, e.g. Good Friday + Easter Monday).
    uint256 internal constant FRIDAY_UTC = 1_729_209_600;
    bool internal twoOverrides;

    uint256 internal constant SIZE = 100_000e18; // 1000 lots == $100k notional at $1.00
    uint256 internal constant MARGIN = 5_000e6;

    function setUp() public virtual override {
        super.setUp();
        // Governance schedules Monday as an override day (72 h calendar timelock elapses during setup).
        vm.warp(WEDNESDAY_NOON - 5 days);
        ICfdEngineAdminHost.EngineCalendarConfig memory cfg = _engineCalendarConfig();
        if (twoOverrides) {
            cfg.fadDayTimestamps = new uint256[](2);
            cfg.fadDayTimestamps[0] = FRIDAY_UTC + 12 hours;
            cfg.fadDayTimestamps[1] = MONDAY_UTC + 12 hours;
        } else {
            cfg.fadDayTimestamps = new uint256[](1);
            cfg.fadDayTimestamps[0] = MONDAY_UTC + 12 hours;
        }
        _setCalendarConfig(cfg);

        vm.warp(WEDNESDAY_NOON);
        _fundTrader(TRADER, 10_000e6);
        vm.deal(TRADER, 1 ether);
        vm.deal(KEEPER, 1 ether);
    }

    function _frozenData() internal pure returns (bytes[] memory data) {
        data = new bytes[](1);
        data[0] = "";
    }

}

/// @notice Single Monday override day after a normal weekend: 75 h frozen, ~3 h of frozen-stale outage.
contract PoC_FrozenStalenessOneOverrideTest is PoC_FrozenStalenessBase {

    function test_PoC_OverrideDayExtendsFreezePastFadMaxStaleness() public {
        // Open a position on Wednesday through the router.
        vm.prank(TRADER);
        uint64 openId = router.commitOrder(CfdTypes.Side.LONG, SIZE, MARGIN, 0, false);
        router.executeOrder(openId, _mockPythUpdateData());

        // Keepers refresh the mark right up to the Friday freeze.
        vm.warp(FRIDAY_FREEZE - 2 minutes);
        router.updateMarkPrice(_mockPythUpdateData());
        uint64 lastMark = engine.lastMarkTime();
        console.log("last live mark (unix):", lastMark);
        console.log("fadMaxStaleness (s):  ", engine.fadMaxStaleness());

        // Sunday 17:00 NY would normally reopen, but Monday is an override day -> still frozen on Monday.
        vm.warp(MONDAY_UTC + 1 hours);
        assertTrue(engine.isOracleFrozen(), "Monday override keeps the oracle frozen");
        assertTrue(engine.isFadWindow(), "Monday override keeps FAD active");

        // Frozen close still works while the cached mark is within fadMaxStaleness.
        vm.prank(TRADER);
        uint64 closeIdOk = router.commitOrder(CfdTypes.Side.LONG, SIZE / 2, 0, 0, true);
        OrderV3Types.ExecutionResult memory okResult = router.executeOrder(closeIdOk, _frozenData());
        assertEq(uint8(okResult.status), uint8(OrderV3Types.LifecycleStatus.Executed), "frozen close executes");

        // Monday 22:00 UTC: frozen (override day) AND cached mark older than 3 days.
        vm.warp(MONDAY_UTC + 22 hours);
        assertTrue(engine.isOracleFrozen(), "still frozen on the override day");
        assertGt(block.timestamp - lastMark, engine.fadMaxStaleness(), "cached mark is now past fadMaxStaleness");

        // 1. Frozen voluntary close: commit succeeds, execution reverts on the stale frozen basket.
        vm.prank(TRADER);
        uint64 closeIdStale = router.commitOrder(CfdTypes.Side.LONG, SIZE / 2, 0, 0, true);
        vm.expectRevert(
            abi.encodeWithSelector(
                IPletherOracle.PletherOracle__StalePrice.selector,
                IPletherOracle.PriceMode.OrderExecution,
                bytes32(0),
                uint256(lastMark),
                engine.fadMaxStaleness(),
                block.timestamp
            )
        );
        router.executeOrder(closeIdStale, _frozenData());

        // 2. Liquidation path reverts the same way (no liquidation possible for the whole stale window).
        vm.prank(KEEPER);
        vm.expectRevert(
            abi.encodeWithSelector(
                IPletherOracle.PletherOracle__StalePrice.selector,
                IPletherOracle.PriceMode.Liquidation,
                bytes32(0),
                uint256(lastMark),
                engine.fadMaxStaleness(),
                block.timestamp
            )
        );
        router.executeLiquidation(TRADER, _frozenData());

        // 3. Router LP epoch settlement reverts too.
        vm.prank(KEEPER);
        vm.expectRevert();
        router.settleLpEpoch(_frozenData());
        // Direct HousePool entry (permissionless while frozen): check whether LP settlement is blocked there too.
        try pool.settleLpEpoch(0, 0) returns (IHousePool.LpEpochSettlementResult memory r) {
            console.log("direct pool.settleLpEpoch succeeded; redemptionsDeferred/entriesDeferred:", r.entriesDeferred);
        } catch (bytes memory reason) {
            console.log("direct pool.settleLpEpoch reverted:", vm.toString(reason));
        }

        // 4. Trader free-settlement withdrawal (unrelated to the position) is blocked: _liveMark is stale.
        vm.prank(TRADER);
        vm.expectRevert(ICfdEngineTypes.CfdEngine__MarkPriceStale.selector);
        clearinghouse.withdraw(TRADER, 1e6);
        // 5. Protection trigger path is frozen-blocked by design; cancel+commit close commits but cannot execute (as 1).

        // Even a fresh signed Pyth tick is refused while the calendar says frozen (frozen path ignores live data).
        bytes[] memory live = _mockPythUpdateData(1e8);
        vm.expectRevert();
        router.updateMarkPrice(live);

        // Tuesday 00:00 UTC: calendar reopens; a live tick restores service.
        vm.warp(MONDAY_UTC + 24 hours + 10);
        assertFalse(engine.isOracleFrozen(), "Tuesday is live again");
        router.updateMarkPrice(_mockPythUpdateData(1e8));
        console.log("frozen-stale window length (s):", (MONDAY_UTC + 24 hours) - (lastMark + engine.fadMaxStaleness()));
    }

}

/// @notice Friday + Monday override days around one weekend (e.g. Good Friday + Easter Monday): the frozen window
///         runs from Thursday 24:00 UTC to Tuesday 00:00 UTC (96 h). With fadMaxStaleness = 3 days the last live
///         mark expires Sunday ~24:00 UTC, so closes, liquidations and LP settlement are unavailable for ~24 h.
contract PoC_FrozenStalenessTwoOverridesTest is PoC_FrozenStalenessBase {

    function setUp() public override {
        twoOverrides = true;
        super.setUp();
    }

    function test_PoC_TwoOverrideDays_FreezeExceeds24h() public {
        vm.prank(TRADER);
        uint64 openId = router.commitOrder(CfdTypes.Side.LONG, SIZE, MARGIN, 0, false);
        router.executeOrder(openId, _mockPythUpdateData());

        // Thursday 23:50 UTC: last live refresh before the Friday override day starts (FAD runway already active).
        vm.warp(FRIDAY_UTC - 10 minutes);
        assertFalse(engine.isOracleFrozen(), "Thursday evening is still live");
        router.updateMarkPrice(_mockPythUpdateData());
        uint64 lastMark = engine.lastMarkTime();

        // Friday (override), Saturday, Sunday (weekend), Monday (override): frozen throughout.
        for (uint256 d; d < 4; ++d) {
            vm.warp(FRIDAY_UTC + d * 1 days + 12 hours);
            assertTrue(engine.isOracleFrozen(), "frozen through all four days");
        }
        uint256 staleFrom = lastMark + engine.fadMaxStaleness();
        uint256 reopen = MONDAY_UTC + 1 days;
        console.log("frozen-stale window (s):", reopen - staleFrom);
        assertGt(reopen - staleFrom, 24 hours, "closes/liquidations/LP settlement unavailable for more than 24 h");

        // Sample inside the stale window: Monday 12:00 UTC.
        vm.warp(MONDAY_UTC + 12 hours);
        vm.prank(TRADER);
        uint64 closeId = router.commitOrder(CfdTypes.Side.LONG, SIZE / 2, 0, 0, true);
        vm.expectRevert(
            abi.encodeWithSelector(
                IPletherOracle.PletherOracle__StalePrice.selector,
                IPletherOracle.PriceMode.OrderExecution,
                bytes32(0),
                uint256(lastMark),
                engine.fadMaxStaleness(),
                block.timestamp
            )
        );
        router.executeOrder(closeId, _frozenData());
        vm.prank(KEEPER);
        vm.expectRevert();
        router.executeLiquidation(TRADER, _frozenData());
        vm.prank(KEEPER);
        vm.expectRevert();
        router.settleLpEpoch(_frozenData());
    }

}
```
