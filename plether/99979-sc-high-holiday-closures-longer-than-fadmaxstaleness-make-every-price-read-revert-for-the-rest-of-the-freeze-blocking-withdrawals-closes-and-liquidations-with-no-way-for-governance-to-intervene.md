# \#99979 [SC-High] Holiday closures longer than `fadMaxStaleness` make every price read revert for the rest of the freeze, blocking withdrawals, closes and liquidations with no way for governance to intervene in time

**Submitted on Nov 15th 2026 at 23:56:53 UTC by @Afriauditor for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #99979
- **Report Type:** Smart Contract
- **Report severity:** High
- **Target:** https://github.com/immunefi-team/audit-comp-plether/tree/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src
- **Impacts:**
  - Temporary freezing of funds for at least 24 hours

## Description


# Summary

While the oracle is frozen, every price read reuses the cached Engine mark and reverts with `PletherOracle__StalePrice` once that mark is older than `fadMaxStaleness` (3 days by default). Frozen mode ignores Pyth entirely, so nobody can refresh the mark, and no fresher data exists anyway. A holiday override freezes the whole UTC day, so a two-day holiday adjoining the weekend (Christmas and Boxing Day falling on Thursday and Friday) produces a continuous frozen span of about four days against a three-day limit. Nothing validates the two settings against each other, and both sit behind a 72-hour timelock, so a clash noticed close to the holiday cannot be fixed before it bites. For the last 23 hours before the Sunday reopen, every price-dependent action reverts: free-cash withdrawals by any account with a position, mark refreshes, close execution and every liquidation. Longer closures extend the dead period hour for hour.

# Description

**The frozen-mode age check.** `_frozenBasket` serves every frozen-mode read from `engine.lastMarkPrice()` / `engine.lastMarkTime()` and reverts if that timestamp is older than the caller's `maxStaleness`:

```solidity
// PletherOracle.sol:729-743
basket.price = engine.lastMarkPrice();
basket.publishTime = engine.lastMarkTime();
...
if (OracleFreshnessPolicyLib.isStale(basket.publishTime, maxStaleness, block.timestamp)) {
    revert PletherOracle__StalePrice(mode, bytes32(0), basket.publishTime, maxStaleness, block.timestamp);
}
```

In frozen mode `maxStaleness` is `engine.fadMaxStaleness()` (`PletherOracle.sol:695`), which defaults to 3 days (`CfdEngine.sol:140`). Frozen mode never parses Pyth payloads, so the cached mark cannot advance until the market reopens. The age check therefore has nothing to protect against — there is no fresher price anyone could supply — but it still fires.

**Holiday overrides extend the freeze.** `MarketCalendarLib` treats an override day as frozen for the entire UTC day (`:51-53`). A normal weekend freezes from Friday 17:00 to Sunday 17:00 New York (≈48 h). A Thursday–Friday override extends the span backwards to Thursday 00:00 UTC, giving roughly 4 days frozen. The cached mark's last update is the final live tick before the freeze, so by Saturday evening it is older than 3 days.

**What reverts.** Every frozen-mode read goes through `_frozenBasket`, so once the mark ages out:

- Withdrawals for any account holding a position revert `CfdEngine__MarkPriceStale` (`CfdEngineSettlementSidecar.sol:278-281`), including withdrawal of free balance unrelated to the position.
- `updateMarkPrice` replays revert `StalePrice`, so the mark cannot be re-stamped.
- Close execution and both liquidation entrypoints resolve their price through the same path and revert.
- LP epoch settlement reverts `HousePool__MarkPriceStale` (`HousePool.sol:817`).

**Why governance cannot fix it in time.** The two settings are validated independently. `proposeCalendarConfig` checks only that the FAD runway is ≤ 24 hours (`CfdEngineAdmin.sol:158-160`); the freshness config checks only that `fadMaxStaleness` is non-zero (`:210-212`). Neither checks that the staleness limit covers the longest frozen span the calendar can produce. Both proposals take 72 hours to activate. A clash discovered inside the final three days before the holiday cannot be corrected before the freeze begins, and once frozen the owner has no mark-refresh path either.

# Impact

**Temporary freezing of funds.** Every account with an open position cannot withdraw any balance, including free cash that backs nothing, for the dead period. With a two-day holiday and default settings that period is ~23 hours; a three-day holiday or any longer closure pushes it past 24 hours. The condition is calendar-driven and deterministic — no attacker and no market move is required.

**Liquidations are blocked exactly when they are needed.** Positions that cross the 3% FAD margin on carry alone during the dead period cannot be liquidated, and their shortfall at Monday's reopen lands on LPs. This compounds the known weekend-gap exposure with a window in which the protocol's only loss-containment tool is switched off.

**No remediation path.** The owner cannot shorten the freeze, refresh the mark, or activate a config change inside the window. The only exit is the scheduled Sunday reopen.

**Rubric:** Temporary freezing of funds for at least 1 hour (Medium) at the tested two-day holiday; Temporary freezing of funds for at least 24 hours (High) for any closure one day longer.

**Fix.** Validate the pair whenever either config is proposed: require `fadMaxStaleness ≥ longest continuous frozen span implied by the calendar`. Alternatively, drop the absolute age limit in frozen mode, since by construction no fresher data can exist while frozen; the limit protects nothing there.



## Proof of Concept

## Proof of Concept
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

// Place at: packages/perps/test/perps/poc/HolidayStalenessFreeze.t.sol
// Run:      forge test --root packages/perps --match-path 'test/perps/poc/HolidayStalenessFreeze.t.sol' -vv
//
// Claim: a holiday override adjoining the weekend makes the continuous frozen span exceed
// fadMaxStaleness (3 days default). Once the cached Engine mark is older than that, every
// frozen-mode price read reverts StalePrice, and no path exists to refresh it until reopen.
//
// Fixture: FadStalenessTest (alice LONG 10_000e18 open on Wednesday; last live mark Fri 18:00:06 UTC).
// We add carol (thin margin, liquidatable under the 3% FAD rate) and declare the Monday and
// Tuesday after the weekend as override days, giving a frozen span of Fri 22:00 UTC -> Wed 00:00 UTC.
// The mark goes stale at Fri 18:00 + 3d = Mon 18:00 UTC. Dead window: Mon 18:00 -> Wed 00:00 = 30 h.
//
// Timeline note: the calendar config is finalized at Mon ~18:01 in this test only because the
// fixture's 72 h timelock must elapse after setUp. In production it would be finalized before
// the holiday. The stale check depends only on mark age and frozen status at call time, so the
// late finalize does not change the result.

import {FadStalenessTest} from "../spec/oracle/FadStaleness.t.sol";
import {CfdTypes} from "@plether/perps/CfdTypes.sol";
import {ICfdEngineAdminHost} from "@plether/perps/interfaces/ICfdEngineAdminHost.sol";
import {ICfdEngineTypes} from "@plether/perps/interfaces/ICfdEngineTypes.sol";
import {IPletherOracle} from "@plether/perps/interfaces/IPletherOracle.sol";

abstract contract HolidayStalenessBase is FadStalenessTest {
    address carol = address(0x333);

    // UTC day boundaries following the fixture's FRIDAY_18UTC (Fri 2026-03-06 18:00 UTC)
    uint256 constant MONDAY_00UTC    = 1_773_014_400; // Mon 2026-03-09 00:00 UTC (override day)
    uint256 constant TUESDAY_00UTC   = 1_773_100_800; // Tue 2026-03-10 00:00 UTC (override day)
    uint256 constant WEDNESDAY_00UTC = 1_773_187_200; // Wed 2026-03-11 00:00 UTC (reopen)
    uint256 constant MONDAY_19UTC    = MONDAY_00UTC + 19 hours;   // mark is > 3 d old here
    uint256 constant TUESDAY_NOON    = TUESDAY_00UTC + 12 hours;
    uint256 constant WEDNESDAY_01UTC = WEDNESDAY_00UTC + 1 hours; // live again

    uint64 internal lastLiveTick;

    /// @dev Override point: the control contract sets a 5-day staleness limit.
    function _fadMaxStaleness() internal pure virtual returns (uint256) { return 3 days; }

    function setUp() public virtual override {
        super.setUp(); // ends at FRIDAY_18UTC + 6, market still live (freeze is 22:00 UTC)

        // --- carol: thin margin, so she is liquidatable under the 3% FAD rate all weekend ---
        settlementToken.mint(carol, 10_000 * 1e6);
        vm.startPrank(carol);
        settlementToken.approve(address(clearinghouse), type(uint256).max);
        clearinghouse.deposit(carol, 10_000 * 1e6);
        vm.stopPrank();
        vm.deal(carol, 10 ether);

        uint256 t0 = FRIDAY_18UTC + 10;
        vm.warp(t0);
        mockPyth.setAllPrices(feedIds, int64(80_000_000), int32(-8), uint64(t0 + 6));
        // notional 10_000e18 * $0.80 = 8,000 USDG; initMargin 1.5% = 120; post 130 (1.63%) < 3% FAD
        vm.prank(carol);
        router.commitOrder(CfdTypes.Side.LONG, 10_000 * 1e18, 130 * 1e6, 0.8e8, false);
        vm.warp(t0 + 50);
        vm.roll(block.number + 1);
        router.executeOrder(2, _pythUpdateData()); // id 1 was alice's open in the fixture
        (uint256 carolSize,,,,,,) = engine.positions(carol);
        require(carolSize == 10_000 * 1e18, "setUp: carol not opened");
        lastLiveTick = uint64(t0 + 6); // the fill installed this tick as the Engine mark

        // --- declare Mon + Tue after the weekend as holiday override days; set staleness limit ---
        ICfdEngineAdminHost.EngineCalendarConfig memory cal = _engineCalendarConfig();
        cal.fadDayTimestamps = new uint256[](2);
        cal.fadDayTimestamps[0] = MONDAY_00UTC;
        cal.fadDayTimestamps[1] = TUESDAY_00UTC;
        engineAdmin.proposeCalendarConfig(cal);

        ICfdEngineAdminHost.EngineFreshnessConfig memory fresh = _engineFreshnessConfig();
        fresh.fadMaxStaleness = _fadMaxStaleness();
        engineAdmin.proposeFreshnessConfig(fresh);

        vm.warp(block.timestamp + 72 hours + 1); // -> Mon ~18:01 UTC
        engineAdmin.finalizeCalendarConfig();
        engineAdmin.finalizeFreshnessConfig();
    }

    function _assertFrozenAndStaleAt(uint256 ts) internal {
        vm.warp(ts);
        assertTrue(engine.isOracleFrozen(), "override day must be frozen");
        uint256 markAge = block.timestamp - engine.lastMarkTime();
        emit log_named_uint("mark age (s)", markAge);
        assertGt(markAge, 3 days, "mark must be older than default fadMaxStaleness");
    }
}

contract HolidayStalenessFreezePoC is HolidayStalenessBase {

    function test_PoC_everyPriceReadRevertsUntilReopen() public {
        _assertFrozenAndStaleAt(MONDAY_19UTC);

        // 1. Free-cash withdrawal by a position holder reverts.
        vm.prank(alice);
        vm.expectRevert(ICfdEngineTypes.CfdEngine__MarkPriceStale.selector);
        clearinghouse.withdraw(alice, 100 * 1e6);

        // 2. The mark cannot be re-stamped: zero-fee frozen replay reverts.
        vm.expectRevert(IPletherOracle.PletherOracle__StalePrice.selector);
        router.updateMarkPrice(_pythUpdateData());

        // 3. Liquidation of a position under the FAD margin reverts.
        vm.expectRevert(IPletherOracle.PletherOracle__StalePrice.selector);
        router.executeLiquidation(carol, _pythUpdateData());

        // 4. Alice cannot exit by closing. Either commit is refused, or execution reverts on the price read.
        (uint256 sizeBefore,,,,,,) = engine.positions(alice);
        vm.prank(alice);
        try router.commitOrder(CfdTypes.Side.LONG, 10_000 * 1e18, 0, 0, true) returns (uint64 closeId) {
            vm.roll(block.number + 1);
            vm.expectRevert(IPletherOracle.PletherOracle__StalePrice.selector);
            router.executeOrder(closeId, _pythUpdateData());
            emit log_string("close: commit accepted, execution reverted StalePrice");
        } catch {
            emit log_string("close: commit itself reverted");
        }
        (uint256 sizeAfter,,,,,,) = engine.positions(alice);
        assertEq(sizeAfter, sizeBefore, "alice could not exit");

        // 5. Still blocked the next day.
        _assertFrozenAndStaleAt(TUESDAY_NOON);
        vm.expectRevert(IPletherOracle.PletherOracle__StalePrice.selector);
        router.executeLiquidation(carol, _pythUpdateData());

        // 6. Reopen: a live refresh clears everything, proving the block was the staleness clash.
        vm.warp(WEDNESDAY_01UTC);
        assertFalse(engine.isOracleFrozen(), "Wednesday must be live");
        mockPyth.setAllPrices(feedIds, int64(80_000_000), int32(-8), uint64(WEDNESDAY_01UTC + 6));
        vm.warp(WEDNESDAY_01UTC + 6);
        router.updateMarkPrice(_livePayload(80_000_000, uint64(WEDNESDAY_01UTC + 6), lastLiveTick));
        router.executeLiquidation(carol, _pythUpdateData());
        (uint256 carolAfter,,,,,,) = engine.positions(carol);
        assertEq(carolAfter, 0, "liquidation succeeds once live");

        // Dead-window arithmetic
        uint256 staleFrom = uint256(lastLiveTick) + 3 days;
        emit log_named_uint("stale from (unix)", staleFrom);
        emit log_named_uint("reopen (unix)", WEDNESDAY_00UTC);
        emit log_named_uint("dead window (hours)", (WEDNESDAY_00UTC - staleFrom) / 1 hours); // expect 30
        assertGt(WEDNESDAY_00UTC - staleFrom, 24 hours, "dead window exceeds 24 h");
    }
}

/// Control: identical holiday, fadMaxStaleness raised to cover the span. Nothing reverts.
contract HolidayStalenessControlPoC is HolidayStalenessBase {

    function _fadMaxStaleness() internal pure override returns (uint256) { return 5 days; }

    function test_Control_coveringStalenessLimitKeepsFrozenModeLive() public {
        vm.warp(MONDAY_19UTC);
        assertTrue(engine.isOracleFrozen());
        assertGt(block.timestamp - engine.lastMarkTime(), 3 days, "same mark age as the failing run");

        router.updateMarkPrice(_pythUpdateData());        // frozen replay succeeds
        router.executeLiquidation(carol, _pythUpdateData()); // liquidation succeeds
        (uint256 carolAfter,,,,,,) = engine.positions(carol);
        assertEq(carolAfter, 0, "control: liquidation works with a covering staleness limit");
    }
}
