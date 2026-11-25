# \#98583 [SC-High] Two holiday override days before a weekend push the frozen mark past `fadMaxStaleness`, blocking closes, liquidations, margin withdrawals and LP settlement for 22 hours

**Submitted on Nov 8th 2026 at 20:50:10 UTC by @Rareone for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #98583
- **Report Type:** Smart Contract
- **Report severity:** High
- **Target:** https://github.com/immunefi-team/audit-comp-plether/tree/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src
- **Impacts:**
  - Temporary freezing of funds for at least 1 hour

## Description


**Target:** Plether Perps, `packages/perps`
**Chain:** Arbitrum One
**Severity:** Medium (temporary freezing of funds for at least 1 hour; 22 hours in the PoC)
**Component:** `CfdEngine` / `CfdEngineAdmin` (calendar and freshness config), `MarketCalendarLib`, `PletherOracle._frozenBasket`



## Summary

Governance is expected to add holiday closures to the FX calendar as whole UTC-day overrides. Each override day puts the oracle in frozen mode for the full day. While frozen, every price-dependent action replays the last engine mark, and only as long as that mark is younger than `fadMaxStaleness`. At release that is the constructor default of 3 days.

Nothing ties these two settings together. If two override days fall on the Thursday and Friday before a weekend (Christmas Eve plus Christmas Day in 2026, or Christmas plus Boxing Day in 2025), there is no live window between Thursday 00:00 UTC and the Sunday 17:00 New York reopen. That is 94 hours of continuous freeze. The last mark anyone can install is from Wednesday 23:59:59 UTC, so it goes over-stale at Sunday 00:00 UTC and stays over-stale for the 22 hours until reopen.

In those 22 hours, at release parameters and with the keeper doing everything right:
- traders cannot close positions;
- nobody can liquidate;
- accounts with an open position cannot withdraw free margin;
- matured LP redemptions cannot be settled through either settlement route;
- no one can repair the mark, because live payloads are ignored while frozen.

Three controls show this only happens because of the override run. A normal weekend, the same holidays with a 5-day limit, and a single Friday override all show zero blocked hours.


## Vulnerability details

**1. An override day freezes the whole UTC day.**

```solidity
// MarketCalendarLib.sol:30-58 (abridged)
function marketStatus(uint256 timestamp, bool todayOverride, bool tomorrowOverride, uint256 fadRunwaySeconds)
    internal pure returns (bool fadWindow, bool oracleFrozen)
{
    (uint256 dayOfWeek, uint256 secondOfDay) = _dayAndSecond(timestamp);
    if (dayOfWeek == 5) {            // Friday: frozen from the 17:00 New York boundary
        ...
        oracleFrozen = secondOfDay >= marketBoundary;
    } else if (dayOfWeek == 6) {     // Saturday
        fadWindow = true;
        oracleFrozen = true;
    } else if (dayOfWeek == 0) {     // Sunday: frozen until the 17:00 New York boundary
        ...
        oracleFrozen = secondOfDay < marketBoundary;
    }
    if (todayOverride) {
        return (true, true);         // the whole UTC day is frozen
    }
    // the runway before an override day only sets FAD, not frozen
}
```

With overrides on Thursday and Friday, `oracleFrozen` is true from Thursday 00:00 UTC straight through the weekend to Sunday 17:00 New York (22:00 UTC in winter). No hour in that stretch is live.

**2. While frozen, every priced action replays the engine mark and fails once it is older than `fadMaxStaleness`.**

```solidity
// PletherOracle.sol:729-743
function _frozenBasket(PriceMode mode, uint256 maxStaleness) internal view returns (BasketPrice memory basket) {
    basket.price = engine.lastMarkPrice();
    basket.publishTime = engine.lastMarkTime();
    ...
    if (OracleFreshnessPolicyLib.isStale(basket.publishTime, maxStaleness, block.timestamp)) {
        revert PletherOracle__StalePrice(mode, bytes32(0), basket.publishTime, maxStaleness, block.timestamp);
    }
    ...
}
```

The engine's withdraw guard (`CfdEngine__MarkPriceStale`) and `HousePool.settleLpEpoch` (`HousePool__MarkPriceStale`) apply the same frozen-age rule. The release value is the constructor default:

```solidity
// CfdEngine.sol:140
uint256 public fadMaxStaleness = 3 days;
```

DEPLOYMENT.md's release parameter table does not override it.

**3. The mark cannot be refreshed during the freeze.** ORACLE_SYNCHRONIZATION.md:78-79 says:

> "A stale frozen mark cannot be repaired from Pyth storage or a new live payload during the freeze."

The PoC confirms it: `updateMarkPrice` with a fresh live tick reverts `PletherOracle__StalePrice` for the whole over-stale window. The only mitigation the docs give is "Install a final live mark before the scheduled freeze and verify its allowed age". Here that doesn't help. The latest possible mark is Wednesday 23:59:59, and its allowed age runs out 22 hours before the market reopens.

**4. Nothing checks the calendar against the staleness limit.** `proposeCalendarConfig` validates only the runway (`CfdEngineAdmin.sol:148-153`: "Only the runway is validated … timestamps may be arbitrary"). `proposeFreshnessConfig` has "no admin-side maximum" and no reference to the calendar (`CfdEngineAdmin.sol:200-205`). `finalizeCalendarConfig` / `applyCalendarConfig` write the days without computing the longest contiguous frozen run.

**Why this is in scope: a documented trusted-role action that triggers a bug.** The program puts in scope "a legitimate, documented trusted-role action that triggers a bug harming users/protocol". This is that case.

- **The action is legitimate and documented.** Holiday overrides are the protocol's own mechanism for closures:
  - WHITEPAPER.md:1120: "Governance may also configure all-day UTC holiday overrides and their runway."
  - SECURITY.md:1337-1338: "holiday hours still require governance-configured UTC-day overrides"

  Marking Christmas Eve and Christmas Day (or New Year's Eve and New Year's Day) is that documented action, done through the normal 72-hour timelock.
- **Governance does nothing wrong.** It doesn't set a harmful or out-of-range value. It doesn't touch `fadMaxStaleness` at all, which stays at the shipped release default. The PoC uses the release default and a plain two-day holiday.
- **The bug is in the code's reaction.** `proposeCalendarConfig` / `finalizeCalendarConfig` accept a calendar whose contiguous frozen run (94 hours) is longer than the staleness limit the same contract enforces (72 hours). They validate only the runway, and nothing compares the two. The protocol then enters "Frozen beyond max staleness" (WHITEPAPER.md:1112), where closes and liquidations are blocked, and has no way out until the market reopens.
- **The docs promise otherwise.** No document warns that a standard two-day holiday before a weekend leads there, or that `fadMaxStaleness` must be raised for it. The operator guidance (ORACLE_SYNCHRONIZATION.md:78, "Install a final live mark before the scheduled freeze and verify its allowed age") can't be met for this holiday shape, because the latest possible mark is already too old 22 hours before reopen.

---

## Impact

Measured in the PoC at release parameters, probing every hour from Thursday 00:00 to Sunday 23:00 UTC:

| Action | Blocked | First blocked | Last blocked | Revert |
|---|---|---|---|---|
| Voluntary full close (commit ok, execution reverts) | 22 h | Sun 00:00Z | Sun 21:00Z | `PletherOracle__StalePrice` |
| `updateMarkPrice`, empty or live payload | 22 h | Sun 00:00Z | Sun 21:00Z | `PletherOracle__StalePrice` |
| `executeLiquidation` on an under-margined account | 22 h | Sun 00:00Z | Sun 21:00Z | `PletherOracle__StalePrice` |
| `withdrawMargin`, account with an open position | 22 h | Sun 00:00Z | Sun 21:00Z | `CfdEngine__MarkPriceStale` |
| `OrderRouter.settleLpEpoch` (matured Junior redeem) | 22 h | Sun 00:00Z | Sun 21:00Z | `PletherOracle__StalePrice` |
| `HousePool.settleLpEpoch` direct route | 22 h | Sun 00:00Z | Sun 21:00Z | `HousePool__MarkPriceStale` |
| `withdrawMargin`, flat account | 0 h | — | — | — |

Consequences:
- **Traders with open positions** can't close or withdraw free margin for 22 hours. That is a temporary freeze of their funds well past the 1-hour threshold, on top of the normal weekend.
- **LPs with matured redemptions** can't be funded through either settlement route for the same 22 hours.
- **The pool** can't liquidate anything for 22 hours, then faces the full Sunday reopen gap with every under-water position still open. I don't claim a loss from this, because the FX price is closed too. But it removes the protocol's documented risk-reduction path ("Oracle frozen, within limit: … Liquidation: stored mark with conservative confidence shift") for most of a day.

By the same arithmetic (not run in the PoC), a three-day override run from Wednesday to Friday freezes from Wednesday 00:00 to Sunday 22:00 UTC. That gives about 46 over-stale hours. I don't rely on it: a two-day closure before a weekend is the realistic case.

The condition recurs. Upcoming examples where governance would plausibly mark Thursday and Friday:
- 2026-12-24 / 2026-12-25 (Christmas Eve / Christmas Day);
- 2026-12-31 / 2027-01-01 (New Year's Eve / New Year's Day).








## Proof of Concept

## Proof of Concept


## Proof of concept

The PoC goes in `packages/perps/test/perps/P_HolidayStaleness.t.sol` (the file is attached). It uses the repo's `BasePerpTest` with the release risk parameters. Calendar and freshness configs go through the real `CfdEngineAdmin` 72-hour timelock. Positions are opened through the real router, and the only mock is the repo's standard `MockPyth`. Each hour, every action is attempted inside a snapshot and reverted, so the probes don't affect each other.

Run:

```bash
forge test --root packages/perps --match-path test/perps/P_HolidayStaleness.t.sol -vv
```

Result: 4/4 pass.

| Test | Override days | `fadMaxStaleness` | Frozen hours | Over-stale hours | Blocked hours (close / liq / withdraw / settle) |
|---|---|---|---|---|---|
| `test_HolidayTwoDayOverride_Release` | Thu 12-25 + Fri 12-26 | 3 days (release) | 94 | **22** | **22 / 22 / 22 / 22** |
| `test_Control_NoOverride_NormalWeekend` | none | 3 days | 48 | 0 | 0 |
| `test_Control_TwoDayOverride_FadMax5d` | Thu + Fri | 5 days | 94 | 0 | 0 |
| `test_Control_FridayOnlyOverride` | Fri only | 3 days | 70 | 0 | 0 |

Full PoC, `packages/perps/test/perps/P_HolidayStaleness.t.sol`. Drop it into the repo's `test/perps/` folder; it uses only the repo's own `BasePerpTest` and `MockPyth`:

```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

import {BasePerpTest} from "./BasePerpTest.sol";
import {CfdTypes} from "@plether/perps/CfdTypes.sol";
import {OrderV3Types} from "@plether/perps/OrderV3Types.sol";
import {ICfdEngineAdminHost} from "@plether/perps/interfaces/ICfdEngineAdminHost.sol";
import {IPletherOracle} from "@plether/perps/interfaces/IPletherOracle.sol";
import {MockPyth} from "@plether/test-utils/MockPyth.sol";
import {console2} from "forge-std/console2.sol";

/// GOVL-10: two consecutive override days (Thu 2025-12-25 + Fri 2025-12-26) before a weekend keep the oracle frozen
/// ~94h; the last live engine mark ages past fadMaxStaleness (3d) ~22h before the Sunday NY reopen.
contract P_HolidayStalenessTest is BasePerpTest {

    address internal constant TRADER = address(0x7A0001);
    address internal constant LIQ = address(0x7A0002);
    address internal constant FLAT = address(0x7A0003);
    address internal constant KEEPER = address(0x7106);

    uint256 internal constant THU = 1_766_620_800; // 2025-12-25 00:00 UTC
    uint256 internal constant NACT = 8;

    uint256 internal redeemId;
    uint256 internal liveOkCount; // anchor: number of probes that succeeded

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

    modifier self() {
        require(msg.sender == address(this));
        _;
    }

    // ------------------------------------------------------------ oracle helpers
    function _live(
        uint256 price
    ) internal view returns (bytes[] memory updateData) {
        bytes32[] memory ids = _basePythFeedIds();
        MockPyth.FeedUpdate[] memory ticks = new MockPyth.FeedUpdate[](ids.length);
        uint256 previous = engine.lastMarkTime();
        for (uint256 i; i < ids.length; ++i) {
            ticks[i] = MockPyth.FeedUpdate(
                ids[i], MockPyth.MockPrice(int64(uint64(price)), 0, -8, vm.getBlockTimestamp(), previous)
            );
        }
        updateData = new bytes[](1);
        updateData[0] = abi.encode(ticks);
    }

    function _refresh(
        uint256 price
    ) internal {
        if (vm.getBlockTimestamp() <= engine.lastMarkTime()) {
            vm.warp(uint256(engine.lastMarkTime()) + 1);
        }
        router.updateMarkPrice(_live(price));
    }

    function _payload() internal view returns (bytes[] memory d) {
        if (engine.isOracleFrozen()) {
            d = new bytes[](1);
            d[0] = "";
        } else {
            d = _live(1.01e8);
        }
    }

    // ------------------------------------------------------------ preparation
    function _prepare(
        uint256[] memory overrides,
        uint256 fadMax,
        uint256 installTs
    ) internal {
        // Monday 2025-12-15 12:00 UTC: propose calendar (+ freshness) configs, respect the 72h timelock.
        vm.warp(THU - 10 days + 12 hours);
        ICfdEngineAdminHost.EngineCalendarConfig memory cal = _engineCalendarConfig();
        cal.fadDayTimestamps = overrides;
        cal.fadRunwaySeconds = 1 hours;
        engineAdmin.proposeCalendarConfig(cal);
        bool changeFresh = fadMax != engine.fadMaxStaleness();
        if (changeFresh) {
            ICfdEngineAdminHost.EngineFreshnessConfig memory fr = _engineFreshnessConfig();
            fr.fadMaxStaleness = fadMax;
            engineAdmin.proposeFreshnessConfig(fr);
        }
        vm.warp(block.timestamp + 72 hours + 1);
        engineAdmin.finalizeCalendarConfig();
        if (changeFresh) engineAdmin.finalizeFreshnessConfig();
        assertEq(engine.fadMaxStaleness(), fadMax, "fadMax");
        assertEq(engine.fadRunwaySeconds(), 1 hours, "runway");

        _fundTrader(TRADER, 200_000e6);
        _fundTrader(LIQ, 10_000e6);
        _fundTrader(FLAT, 5_000e6);
        vm.deal(KEEPER, 10 ether);

        // Matured-but-unsettled LP redeem request (junior shares held by this contract).
        uint256 sh = juniorVault.balanceOf(address(this));
        require(sh > 0, "no junior shares");
        redeemId = juniorVault.requestRedeem(sh / 10, address(this), address(this));

        // Wednesday 2025-12-24 22:00 UTC, live: refresh mark, open positions through the real router.
        vm.warp(THU - 2 hours); // runway (FAD, close-only) starts at THU-1h, so open earlier
        require(!engine.isOracleFrozen() && !engine.isFadWindow(), "FAD/frozen at open time");
        _refresh(1e8);
        _openViaRouter(TRADER, 100, 5000e6);
        _openViaRouter(LIQ, 1000, 600e6);

        // Final live mark as late as possible before the freeze.
        vm.warp(installTs);
        require(!engine.isOracleFrozen(), "frozen at install time");
        _refresh(1.01e8);
        assertEq(uint256(engine.lastMarkTime()), installTs, "mark time");
    }

    function _openViaRouter(
        address a,
        uint256 lots,
        uint256 margin
    ) internal {
        vm.prank(a);
        uint64 id = router.commitOrder(CfdTypes.Side.LONG, lots * CfdTypes.SIZE_QUANTUM, margin, 1, false);
        vm.roll(vm.getBlockNumber() + 1);
        bytes[] memory d = _mockPythUpdateData(1e8);
        vm.prank(KEEPER);
        OrderV3Types.ExecutionResult memory r = router.executeOrder(id, d);
        require(r.status == OrderV3Types.LifecycleStatus.Executed, "open failed");
        (uint256 size,,,,,,) = engine.positions(a);
        require(size == lots * CfdTypes.SIZE_QUANTUM, "open size");
    }

    // ------------------------------------------------------------ action wrappers (self-only)
    function aCommitClose() external self returns (uint64 id) {
        vm.prank(TRADER);
        id = router.commitOrder(CfdTypes.Side.LONG, 100 * CfdTypes.SIZE_QUANTUM, 0, 0, true);
    }

    function aExecClose(
        uint64 id
    ) external self returns (uint256 status) {
        vm.roll(vm.getBlockNumber() + 1);
        vm.warp(vm.getBlockTimestamp() + 2);
        bytes[] memory d;
        if (engine.isOracleFrozen()) {
            d = new bytes[](1);
            d[0] = "";
        } else {
            d = _mockPythUpdateData(1.01e8);
        }
        vm.prank(KEEPER);
        OrderV3Types.ExecutionResult memory r = router.executeOrder(id, d);
        return uint256(r.status);
    }

    function aUpdEmpty() external self {
        vm.prank(KEEPER);
        router.updateMarkPrice(new bytes[](0));
    }

    function aUpdLive() external self {
        bytes[] memory d = _live(1.01e8);
        vm.prank(KEEPER);
        router.updateMarkPrice(d);
    }

    function aLiq() external self {
        bytes[] memory d = _payload();
        vm.prank(KEEPER);
        router.executeLiquidation(LIQ, d);
        (uint256 size,,,,,,) = engine.positions(LIQ);
        require(size == 0, "NOT_LIQUIDATED");
    }

    function aWdTrader() external self {
        vm.prank(TRADER);
        clearinghouse.withdrawMargin(1000e6);
    }

    function aWdFlat() external self {
        vm.prank(FLAT);
        clearinghouse.withdrawMargin(1000e6);
    }

    function aRouterSettle() external self {
        bytes[] memory d = _payload();
        vm.prank(KEEPER);
        router.settleLpEpoch(d);
    }

    function aPoolSettle() external self {
        vm.prank(KEEPER);
        pool.settleLpEpoch(0, 0);
    }

    // ------------------------------------------------------------ probing
    function _name(
        uint256 a
    ) internal pure returns (string memory) {
        if (a == 0) return "voluntaryClose";
        if (a == 1) return "updMark(empty)";
        if (a == 2) return "updMark(live)";
        if (a == 3) return "liquidation";
        if (a == 4) return "withdraw(trader,open pos)";
        if (a == 5) return "withdraw(flat)";
        if (a == 6) return "routerSettleLp";
        return "poolSettleLp(0,0)";
    }

    function _sel(
        bytes memory err
    ) internal view returns (string memory) {
        if (err.length < 4) return "EMPTY";
        bytes4 s;
        assembly { s := mload(add(err, 32)) }
        if (s == IPletherOracle.PletherOracle__StalePrice.selector) return "STALE";
        if (s == 0x08c379a0) return "Error(string)";
        return vm.toString(abi.encodePacked(s));
    }

    /// @return res "" when the action succeeded, otherwise a reason string
    function _try(
        uint256 a
    ) internal returns (string memory res) {
        if (a == 0) {
            try this.aCommitClose() returns (uint64 id) {
                try this.aExecClose(id) returns (uint256 st) {
                    return st == uint256(OrderV3Types.LifecycleStatus.Executed) ? "" : "exec-status-Failed";
                } catch (bytes memory e) {
                    return string.concat("exec:", _sel(e));
                }
            } catch (bytes memory e) {
                return string.concat("commit:", _sel(e));
            }
        }
        if (a == 1) {
            try this.aUpdEmpty() { return ""; } catch (bytes memory e) { return _sel(e); }
        }
        if (a == 2) {
            try this.aUpdLive() { return ""; } catch (bytes memory e) { return _sel(e); }
        }
        if (a == 3) {
            try this.aLiq() { return ""; } catch (bytes memory e) { return _sel(e); }
        }
        if (a == 4) {
            try this.aWdTrader() { return ""; } catch (bytes memory e) { return _sel(e); }
        }
        if (a == 5) {
            try this.aWdFlat() { return ""; } catch (bytes memory e) { return _sel(e); }
        }
        if (a == 6) {
            try this.aRouterSettle() { return ""; } catch (bytes memory e) { return _sel(e); }
        }
        try this.aPoolSettle() { return ""; } catch (bytes memory e) { return _sel(e); }
    }

    struct Summary {
        uint256[8] blockedHours;
        uint256[8] firstBlocked; // hour index +1, 0 = never
        uint256[8] lastBlocked;
        string[8] firstReason;
        uint256 frozenHours;
        uint256 overStaleHours;
    }

    function _hhmm(
        uint256 ts
    ) internal pure returns (string memory) {
        uint256 dow = ((ts / 86_400) + 4) % 7;
        string[7] memory n = ["Sun", "Mon", "Tue", "Wed", "Thu", "Fri", "Sat"];
        uint256 day = 1 + (ts - 1_764_547_200) / 86_400 - 0; // 2025-12-01 00:00 UTC = 1764547200
        uint256 hh = (ts % 86_400) / 3600;
        return string.concat(n[dow], " Dec", vm.toString(day), " ", vm.toString(hh), ":00Z");
    }

    function _scan(
        string memory label,
        uint256 loopStart,
        uint256 hoursN
    ) internal returns (Summary memory s) {
        console2.log("=== SCENARIO", label);
        console2.log("fadMaxStaleness", engine.fadMaxStaleness());
        uint256 markTime = engine.lastMarkTime();
        uint256 base = vm.snapshotState();
        string memory prevSig;
        for (uint256 h; h < hoursN; ++h) {
            uint256 ts = loopStart + h * 3600;
            vm.warp(ts);
            bool frozen = engine.isOracleFrozen();
            uint256 age = ts - markTime;
            bool over = age > engine.fadMaxStaleness();
            if (frozen) s.frozenHours++;
            if (frozen && over) s.overStaleHours++;
            string memory line = string.concat(
                _hhmm(ts), " frozen=", frozen ? "1" : "0", " ageH=", vm.toString(age / 3600), over ? " OVER" : "     "
            );
            string memory sig;
            for (uint256 a; a < NACT; ++a) {
                uint256 snap = vm.snapshotState();
                vm.warp(ts);
                string memory r = _try(a);
                vm.revertToState(snap);
                if (bytes(r).length == 0) {
                    liveOkCount++;
                    line = string.concat(line, " | ok");
                    sig = string.concat(sig, "ok,");
                } else {
                    line = string.concat(line, " | ", r);
                    sig = string.concat(sig, r, ",");
                    if (!frozen) continue; // live-mode failures (needs fresh tick) are not freeze/staleness blocking
                    s.blockedHours[a]++;
                    if (s.firstBlocked[a] == 0) {
                        s.firstBlocked[a] = h + 1;
                        s.firstReason[a] = r;
                    }
                    s.lastBlocked[a] = h + 1;
                }
            }
            sig = string.concat(frozen ? "F" : "L", sig);
            if (h == 0 || h == hoursN - 1 || keccak256(bytes(sig)) != keccak256(bytes(prevSig))) console2.log(line);
            prevSig = sig;
        }
        vm.revertToState(base);
        console2.log("columns: close | updEmpty | updLive | liq | wdTrader | wdFlat | routerSettle | poolSettle");
        console2.log("frozen hours probed", s.frozenHours);
        console2.log("frozen AND mark older than fadMaxStaleness hours", s.overStaleHours);
        for (uint256 a; a < NACT; ++a) {
            console2.log(_name(a));
            console2.log("  blocked hours", s.blockedHours[a]);
            if (s.firstBlocked[a] != 0) {
                console2.log("  first blocked:", _hhmm(loopStart + (s.firstBlocked[a] - 1) * 3600));
                console2.log("  last  blocked:", _hhmm(loopStart + (s.lastBlocked[a] - 1) * 3600));
                console2.log("  reason:", s.firstReason[a]);
            }
        }
    }

    function _days(
        uint256 n
    ) internal pure returns (uint256[] memory d) {
        d = new uint256[](n);
        for (uint256 i; i < n; ++i) d[i] = THU + i * 1 days + 12 hours; // floored to UTC day by the engine
    }

    // ------------------------------------------------------------ tests
    /// ATTACK scenario: overrides Thu+Fri, release fadMax=3d. Mark installed Wed 23:59:59.
    function test_HolidayTwoDayOverride_Release() public {
        _prepare(_days(2), 3 days, THU - 1);
        Summary memory s = _scan("TWO-DAY OVERRIDE (Thu+Fri), fadMax 3d", THU, 96);
        // 94h frozen Thu 00:00 -> Sun 21:59; hours probed frozen = 94 (Thu 00 .. Sun 21)
        assertEq(s.frozenHours, 94, "frozen hours");
        assertEq(s.overStaleHours, 22, "over stale hours");
        // hypothesis: close / liq / withdraw w/ position / router LP settle blocked for exactly the over-stale hours
        assertEq(s.blockedHours[0], 22, "close blocked");
        assertEq(s.blockedHours[3], 22, "liq blocked");
        assertEq(s.blockedHours[4], 22, "withdraw trader blocked");
        assertEq(s.blockedHours[6], 22, "router settle blocked");
    }

    /// CONTROL (i): no overrides, normal weekend; keeper installs the last live mark at Fri 21:59:59.
    function test_Control_NoOverride_NormalWeekend() public {
        _prepare(_days(0), 3 days, THU + 1 days + 22 hours - 1);
        Summary memory s = _scan("CONTROL no overrides", THU + 1 days + 22 hours, 50);
        assertEq(s.overStaleHours, 0, "no over-stale");
        assertEq(s.blockedHours[0], 0, "close not blocked");
        assertEq(s.blockedHours[3], 0, "liq not blocked");
        assertEq(s.blockedHours[4], 0, "wd not blocked");
        assertGt(s.frozenHours, 40, "frozen weekend actually scanned");
    }

    /// CONTROL (ii): same overrides, fadMaxStaleness 5 days via timelocked freshness config.
    function test_Control_TwoDayOverride_FadMax5d() public {
        _prepare(_days(2), 5 days, THU - 1);
        Summary memory s = _scan("CONTROL two-day override, fadMax 5d", THU, 96);
        assertEq(s.frozenHours, 94, "frozen hours");
        assertEq(s.overStaleHours, 0, "no over-stale");
        assertEq(s.blockedHours[0], 0, "close not blocked");
        assertEq(s.blockedHours[3], 0, "liq not blocked");
        assertEq(s.blockedHours[4], 0, "wd not blocked");
        assertEq(s.blockedHours[6], 0, "router settle not blocked");
    }

    /// CONTROL (iii): Friday-only override; mark installed Thu 23:59:59 (last live second before frozen Friday).
    function test_Control_FridayOnlyOverride() public {
        uint256[] memory d = new uint256[](1);
        d[0] = THU + 1 days;
        _prepare(d, 3 days, THU + 1 days - 1);
        Summary memory s = _scan("CONTROL Friday-only override", THU + 1 days, 72);
        console2.log("frozen hours", s.frozenHours);
        assertEq(s.frozenHours, 70, "frozen Fri 00:00 -> Sun 21:59");
        assertEq(s.overStaleHours, 0, "70h < 72h: never over-stale");
        assertEq(s.blockedHours[0], 0);
        assertEq(s.blockedHours[3], 0);
        assertEq(s.blockedHours[4], 0);
    }

}
```

Trimmed log from the main test (columns: close | updEmpty | updLive | liq | wdTrader | wdFlat | routerSettle | poolSettle):

```
Thu Dec25 0:00Z  frozen=1 ageH=0       | ok          | ok    | ok    | ok    | ok         | ok | ok    | ok
Sun Dec28 0:00Z  frozen=1 ageH=72 OVER | exec:STALE  | STALE | STALE | STALE | 0xe4c92389 | ok | STALE | 0x40497d8e
Sun Dec28 22:00Z frozen=0 ageH=94 OVER | ok          | STALE | ok    | ok    | 0xe4c92389 | ok | ok    | 0xbd1bc066
blocked hours 22   first blocked: Sun Dec28 0:00Z   last blocked: Sun Dec28 21:00Z
```

`0xe4c92389` is `CfdEngine__MarkPriceStale()` and `0x40497d8e` is `HousePool__MarkPriceStale()`. At the 22:00Z reopen, close, live mark refresh, liquidation and Router settlement all work again with a live tick. The remaining reverts in that row are normal live-mode rules, and they appear identically in all three controls:
- the empty-payload refresh and the position-holder withdrawal need a fresh live tick first;
- the direct pool route returns `0xbd1bc066` = `HousePool__Unauthorized()`, because with open positions in live mode it must be called through the Router.

