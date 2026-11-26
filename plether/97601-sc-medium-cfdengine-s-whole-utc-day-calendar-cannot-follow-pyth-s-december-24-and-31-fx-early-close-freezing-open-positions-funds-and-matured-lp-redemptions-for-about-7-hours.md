# \#97601 [SC-Medium] CfdEngine's whole-UTC-day calendar cannot follow Pyth's December 24 and 31 FX early close, freezing open positions' funds and matured LP redemptions for about 7 hours

**Submitted on Nov 2nd 2026 at 14:11:50 UTC by @xylz for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #97601
- **Report Type:** Smart Contract
- **Report severity:** Medium
- **Target:** https://github.com/immunefi-team/audit-comp-plether/tree/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src
- **Impacts:**
  - Temporary freezing of funds for at least 1 hour

## Description

## Summary

Pyth's FX schedule closes its feeds at 17:00 New York on December 24 and 31 and reopens them at New York midnight (22:00Z to 05:00Z). CfdEngine freezes the oracle only for the weekly Friday-to-Sunday window and for whole UTC days set as overrides, so no override matches this closure. On the deployed calendar the protocol stays live without prices for those 7 hours: accounts with an open position cannot withdraw or close, and while positions are open, matured LP redemptions stay unfunded. The documented override leaves 2 to 22 hours of this freeze in 2026, and freezes the protocol for 17 to 39 hours while Pyth publishes.

- Impact: Temporary freezing of funds for at least 1 hour
- Severity: Medium
- Attacker needs: none; the freeze follows Pyth's published schedule. The next dates are 2026-12-24 and 2026-12-31, both Thursdays
- Funds at risk: the whole clearinghouse balance of every account with an open position and, while any position is open, every Junior or Senior redemption that matures in the closure. In the test, a trader's 49,934.80 USDG and a 99,772.15 USDG Junior redemption stay frozen for 25,140 s on each date; one settlement call (`HousePool.sol:809-819`) serves both tranches
- Reproduce: `forge test --match-path test/perps/HolidayCalendarMismatch.t.sol -vv` (from `packages/perps`, audit repository commit a82d4fefb02768e5b3bddcc97dee371eee24fea8)

## Affected assets

| Contract (all in `packages/perps/src`) | Address (Arbitrum Sepolia v2.0.0) |
|---|---|
| CfdEngine | 0xca18a6783d7542f9d2f28f4f259772b74c4d6baa |
| CfdEnginePlanner | 0x802a0755413fb966b397155b5c5cc0198e6bed3f |
| CfdEngineAdmin | 0xfa444c18c90f1b5f768eddbb56cd1d92710a8327 |
| PletherOracle | 0x4f2c62a0e03657f52c633c19c11f88d8f0897bb2 |
| HousePool | 0xeb5604c393843b2f3c347ec24fa9ddad6ab2131e |
| MarginClearinghouse | 0xb5b851f627d6df3cf3f096e23e20a58884d7b89a |

Source: the audit repository at commit a82d4fefb02768e5b3bddcc97dee371eee24fea8, whose `packages/perps/src` equals that of the v2.0.0 release (source commit 1209ca8a):

https://github.com/immunefi-team/audit-comp-plether/tree/a82d4fefb02768e5b3bddcc97dee371eee24fea8

The runtime code of each address above matches the release manifest's code hash. At block 325,052,696 the deployment has `fadMaxStaleness` 259,200 s, a 3,600 s runway, a 60 s live mark limit and a 50 bps frozen close spread. It has no override on 2026-12-24, 12-25, 12-31 or 2027-01-01 and no pending calendar or freshness proposal; block 325,069,998 reads the same. Its six feeds are Pyth's EUR/USD, USD/JPY, GBP/USD, USD/CAD, USD/SEK and USD/CHF.

## Preconditions

- Attacker: none. No action triggers the freeze.
- Victim: holds an open position, or has a Junior or Senior redemption that matures between 22:00Z and 05:00Z while any position is open.
- External: by its published schedule, Pyth publishes no FX price during its holiday closure. Pyth's feed metadata gives all six basket feeds the schedule `America/New_York;O,O,O,O,0000-1700,C,1700-2400;1224/0000-1700,1231/0000-1700` (read 2026-11-01). Monday to Thursday are open all day, Friday until 17:00, and December 24 and 31 only until 17:00:

https://hermes.pyth.network/v2/price_feeds?asset_type=fx

- The closure is taken from that schedule. Pyth's publisher agent drops updates outside the schedule's hours, and a holiday entry replaces that day's weekly hours:

https://github.com/pyth-network/pyth-agent/blob/5e8d070da07b64bbca8515baa2f5a58751023e05/src/agent/state/exporter.rs#L216-L233

- I did not observe a past closure: on 2026-11-01 Pyth's Benchmarks and Hermes history endpoints returned 401 without an API key.
- Owner: honest; overrides and freshness limits change only through the 72-hour timelocks. The override cases in the Severity section need an override the deployment does not have, so the severity does not rest on them.
- The PoC uses the repository's two-feed MockPyth basket, at 1.00 unless a test moves it, and a keeper always submits the newest print that exists under the schedule; the calendar reads no price.
- Program terms: SECURITY.md's Known Limitations are declared known issues; the Severity section answers them.

## Root cause

1. The recurring frozen window runs from Friday 17:00 to Sunday 17:00 New York. `packages/perps/src/libraries/MarketCalendarLib.sol:38-49`:

```solidity
        if (dayOfWeek == 5) {
            uint256 marketBoundary = newYorkMarketBoundary(timestamp, dayOfWeek);
            fadWindow = secondOfDay >= marketBoundary - FAD_LEAD;
            oracleFrozen = secondOfDay >= marketBoundary;
        } else if (dayOfWeek == 6) {
            fadWindow = true;
            oracleFrozen = true;
        } else if (dayOfWeek == 0) {
            uint256 marketBoundary = newYorkMarketBoundary(timestamp, dayOfWeek);
            fadWindow = secondOfDay < marketBoundary + FAD_LAG;
            oracleFrozen = secondOfDay < marketBoundary;
        }
```

Monday to Thursday are frozen only by an override, which returns `(true, true)` for its whole UTC day (`MarketCalendarLib.sol:51-53`).

2. Overrides are UTC day numbers. `CfdEngine.sol:698-703` stores `config.fadDayTimestamps[i] / 86_400`, and `CfdEngine.sol:901-906` looks up `timestamp / 86_400`. Pyth's closure covers 22:00Z to 24:00Z of one UTC day and 00:00Z to 05:00Z of the next.

3. While live, every exit of an account with a position, and LP settlement while any position is open, needs a fresh Pyth print. A withdrawal from an account with a position needs a mark at most 60 s old (`CfdEngineSettlementSidecar.sol:274-281`). A delayed close needs a print within 15 s after its commit (`ORACLE_SYNCHRONIZATION.md:48-50`). LP settlement needs the Router and a mark published in the current LP epoch (`HousePool.sol:809-819`).

4. While frozen, exits use the cached mark up to `fadMaxStaleness`, and that mark cannot be refreshed until the freeze ends (`packages/perps/ORACLE_SYNCHRONIZATION.md:72-80`). `PletherOracle.sol:754-756` returns the cached basket before it reads any payload.

The broken rule: the frozen regime is meant for times when the feeds are offline (`SECURITY.md:821-824`). A freeze can only start or end at a UTC midnight or the weekly 17:00 New York boundary. Every calendar the owner can set therefore leaves part of Pyth's holiday closure live without prices or freezes hours in which Pyth publishes. `PletherOracle.sol:509-511` describes the weekly interval as following "Pyth FX hours"; the holiday entries of those hours have no matching interval.

## Attack path

No attacker is needed. On the deployed calendar:

1. Thursday 2026-12-24 21:59:59Z: a keeper installs Pyth's last print before the closure.
2. 22:00Z: Pyth's schedule closes the FX feeds, so no newer print exists; the protocol stays live.
3. From 22:01:00Z the mark is older than 60 s. Withdrawals from accounts with a position revert, closes stay pending, and LP settlement and mark refreshes revert.
4. Friday 05:00:00Z: Pyth's schedule reopens the feeds; with the first print, every path works again.

The same happens on 2026-12-31 into 2027-01-01.

## Impact

Measured by the PoC; in each window every exit below fails, and at its end all succeed:

| Test | Calendar | Exits fail | Seconds |
|---|---|---|---|
| `test_attack_deployedCalendarDec24` | deployed, no override | Thu 22:01:00Z to Fri 05:00:00Z | 25,140 |
| `test_attack_deployedCalendarDec31` | deployed, no override | Thu 22:01:00Z to Fri 05:00:00Z | 25,140 |
| `test_attack_overrideDec25` | override on Dec 25 | Thu 22:01:00Z to 24:00:00Z | 7,140 |
| `test_attack_overrideDec24` | override on Dec 24 | Fri 00:00:00Z to 05:00:00Z | 18,000 |
| `test_attack_overrideDec24AndDec25` | both days | Sun 00:00:00Z to 22:00:00Z | 79,200 |

In each window the trader's withdrawal of 1,000 USDG reverts with `CfdEngine__MarkPriceStale`. The trader's full close stays `Pending` (`HistoricalPriceUnavailable`) while live, or reverts with `PletherOracle__StalePrice` once the frozen mark is too old. The Junior LP's matured redemption is not funded, and the keeper's refresh reverts. An account with no position still withdraws.

`test_control_weeklyScheduleNoEarlyClose` runs the same dates and calls with Pyth's weekly schedule only, and every call succeeds. `test_negative_holidayHoursCalendarRemovesTheFreeze` installs the recommended calendar rule, and every call succeeds through the closure at the cached mark. Of 17,856 calendar answers in December 2026 and January 2027, only 124 differ from the unmodified planner's, all inside the two closures and their FAD shoulders (21:30Z to 05:15Z).

## Severity

The PoC shows the program's row "Temporary freezing of funds for at least 1 hour": on the deployed calendar the trader's whole balance and the matured redemption are frozen for 25,140 s on each date.

The frozen regime applies "once feeds are genuinely offline" (`SECURITY.md:824`). The Known Limitations say "execution remains dependent on fresh keeper infrastructure and oracle publication outside frozen windows" (`SECURITY.md:1341`) but not which hours those are. Here Pyth's schedule takes the feeds offline, and only the calendar's whole-day granularity keeps the protocol out of the frozen regime. Pyth's 17:00 close is an intraday boundary of its current schedule, not the "future legal or provider schedule change" of `SECURITY.md:1339-1340`.

The documented remedy is that "holiday hours still require governance-configured UTC-day overrides" (`SECURITY.md:1337-1338`). In 2026 each override set that touches the closure still freezes exits, for 7,140 s, 18,000 s or 79,200 s. The overridden days freeze the protocol while Pyth publishes: 22 hours on December 24, 17 on December 25 and 39 for both. Raising `fadMaxStaleness` above 94 hours, which has no admin maximum (`CfdEngineAdmin.sol:201-216`), keeps exits open, and the 39 hours stay frozen (`test_negative_raisedFrozenLimitFreezesLiveTrading`).

In those hours a position on the losing side of the live move closes at the cached mark plus the fixed 50 bps spread. In `test_attack_overrideDec25StaleExit` the basket rises 100 bps on December 25, and a keeper's new print leaves the engine mark at 1.00. Against the same sequence without the override, a 200,000 USDG LONG and SHORT pair gains 1,000.80 USDG, Junior LPs lose 1,000.00 USDG and the treasury 0.80 USDG; no other balance changes.

The December 24 and two-day overrides measure the same, and the gain is the move less the spread (49 bps: -19.61 USDG, 51 bps: +20.41 USDG). At the 1,895,600 and 1,600,600 USDG pair the deployed caps allow on this 2,000,000 USDG pool, a 100 bps move costs Junior 9,581.67 USDG.

Taken literally, "frozen-oracle windows prioritize close liveness over live-market-style freshness guarantees" and "a fixed spread deliberately does not increase with mark staleness inside the permitted frozen window" (`SECURITY.md:1332`, `SECURITY.md:1334`) cover this exit price. Both describe the regime defined above for feeds that are offline, when no newer price exists. The override applies that regime while Pyth publishes, so the newer price is public and the spread becomes an option against Junior.

These cases need an override the deployment does not have, so I do not base the severity on them. They show that the documented remedy cannot replace the fix below, which removes both outcomes.

"Incorrect data supplied by third party oracles" does not apply: Pyth's prices and its schedule are correct and published in advance.

By the schedule, the no-override and single-day windows recur whenever December 24 or 31 falls on a Monday to Thursday.

## Recommended fix

```diff
--- a/packages/perps/src/libraries/MarketCalendarLib.sol
+++ b/packages/perps/src/libraries/MarketCalendarLib.sol
@@ -19,6 +19,8 @@ library MarketCalendarLib {
     uint256 internal constant NEW_YORK_DST_START_UTC = 7 hours;
     /// @dev New York returns to standard time at 02:00 local daylight time (06:00 UTC).
     uint256 internal constant NEW_YORK_DST_END_UTC = 6 hours;
+    /// @dev UTC second-of-day of New York midnight under standard time (December and January).
+    uint256 internal constant NEW_YORK_STANDARD_MIDNIGHT = 5 hours;
 
     /// @notice Returns both recurring market-calendar regimes with one New York boundary calculation.
     /// @param timestamp Timestamp to classify.
@@ -48,6 +50,19 @@ library MarketCalendarLib {
             oracleFrozen = secondOfDay < marketBoundary;
         }
 
+        // Pyth FX holiday hours (1224/0000-1700, 1231/0000-1700 New York): the market closes at the 17:00 boundary
+        // and reopens at the next New York midnight. Friday and Saturday already follow the weekly rule.
+        (uint256 month, uint256 dayOfMonth) = _monthAndDay(timestamp / SECONDS_PER_DAY);
+        if (month == 12 && (dayOfMonth == 24 || dayOfMonth == 31) && dayOfWeek < 5) {
+            fadWindow = fadWindow || secondOfDay >= NEW_YORK_STANDARD_MARKET_BOUNDARY - FAD_LEAD;
+            oracleFrozen = oracleFrozen || secondOfDay >= NEW_YORK_STANDARD_MARKET_BOUNDARY;
+        } else if (
+            ((month == 12 && dayOfMonth == 25) || (month == 1 && dayOfMonth == 1)) && dayOfWeek >= 1 && dayOfWeek <= 5
+        ) {
+            fadWindow = fadWindow || secondOfDay < NEW_YORK_STANDARD_MIDNIGHT + FAD_LAG;
+            oracleFrozen = oracleFrozen || secondOfDay < NEW_YORK_STANDARD_MIDNIGHT;
+        }
+
         if (todayOverride) {
             return (true, true);
         }
```

This adds Pyth's two holiday entries to the recurring calendar, so the freeze starts and ends with Pyth's closure and no whole-day override is needed on these dates; overrides remain for other holidays.

`test_negative_holidayHoursCalendarRemovesTheFreeze` and the `StaleExit` tests install the same rule as a planner with `vm.etch`. With the diff itself applied to the pinned checkout, the seven calendar, FAD and frozen-oracle test files I ran (67 tests) pass. The four freeze tests with fewer than two overrides then fail at their first blocked-path assertion. `test_control_weeklyScheduleNoEarlyClose` fails too, because it installs no print before 22:00Z, so its cached mark is older than `fadMaxStaleness`. The `StaleExit` tests pass, as they set the override themselves.

## References

https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/libraries/MarketCalendarLib.sol#L30-L58
https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/CfdEngine.sol#L685-L706
https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/CfdEngine.sol#L887-L907
https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/PletherOracle.sol#L508-L515
https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/PletherOracle.sol#L749-L756
https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/CfdEngineSettlementSidecar.sol#L255-L285
https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol#L794-L821
https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/CfdEngineAdmin.sol#L149-L216
https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/SECURITY.md?plain=1#L819-L829
https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/SECURITY.md?plain=1#L1328-L1341
https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/ORACLE_SYNCHRONIZATION.md?plain=1#L70-L80


## Link to Proof of Concept

https://gist.github.com/Xylzlol/b7b179660110177a297dbc0d0f80cba9

## Proof of Concept

Files in the proof-of-concept link: `HolidayCalendarMismatch.t.sol` (the test) and `README.md` (setup and full output).

Requirements: Foundry; the audit repository at commit a82d4fefb02768e5b3bddcc97dee371eee24fea8 with its submodules.

```
cp HolidayCalendarMismatch.t.sol <repo>/packages/perps/test/perps/
cd <repo>/packages/perps
forge test --match-path test/perps/HolidayCalendarMismatch.t.sol -vv
```

Expected output (abridged and reordered; the full output is in the README):

```
[PASS] test_attack_deployedCalendarDec24()
  ==== no override, 2026-12-24
  -- 22:01:00Z (protocol live)
     keeper mark refresh: FAILS, no Pyth print in the window (feed not found within range)
     trader withdraws 1,000 USDG: FAILS, CfdEngine__MarkPriceStale
     trader closes the position: FAILS, stays Pending (HistoricalPriceUnavailable)
     matured Junior redemption funded: FAILS, no Pyth print in the window (feed not found within range)
     flat account withdraws 1,000 USDG: ok
  -- 05:00:01Z, first print 05:00:00Z (protocol live)
     keeper mark refresh: ok
     trader withdraws 1,000 USDG: ok
     trader closes the position: ok
     matured Junior redemption funded: ok
  seconds with the trader's exits and the redemption blocked 25140
[PASS] test_attack_deployedCalendarDec31()
  seconds with the trader's exits and the redemption blocked 25140
[PASS] test_attack_overrideDec25()
  seconds with the trader's exits and the redemption blocked 7140
  seconds frozen while Pyth publishes (Fri 05:00Z-22:00Z) 61200
[PASS] test_attack_overrideDec24()
  seconds with the trader's exits and the redemption blocked 18000
  seconds frozen while Pyth publishes (Thu 00:00Z-22:00Z) 79200
[PASS] test_attack_overrideDec24AndDec25()
  -- Sun 00:00:00Z, mark age 259,201 s (protocol frozen)
     keeper mark refresh: FAILS, PletherOracle__StalePrice
     trader withdraws 1,000 USDG: FAILS, CfdEngine__MarkPriceStale
     trader closes the position: FAILS, PletherOracle__StalePrice
     matured Junior redemption funded: FAILS, HousePool__MarkPriceStale
  seconds with the trader's exits and the redemption blocked 79200
[PASS] test_attack_overrideDec25StaleExit()
  == Dec 25 override, basket +100 bp
     protocol frozen at the LONG's close true
     Pyth's newest basket price, engine mark after a keeper submits it 101000000 100000000
  == no override, basket +100 bp
     protocol frozen at the LONG's close false
     Pyth's newest basket price, engine mark after a keeper submits it 101000000 101000000
  -- Dec 25 override minus no override, both with the +100 bp move (USDG atoms; LP shares in share atoms)
     attacker LONG leg 1000800000
     attacker SHORT leg 0
     holder 0
     Junior tranche -1000000000
     Senior tranche 0
     pool assets -1000000000
     treasury -800000
     keeper 0
     other trader (open LONG) 0
     flat trader 0
     Junior LP shares 0
     Senior LP shares 0
[PASS] test_attack_overrideDec25BreakEvenMove()
  basket move (bp x 100), pair result with override, without, difference (USDG atoms) 4900
  -570144020
  -550536020
  -19608000
  basket move (bp x 100), pair result with override, without, difference (USDG atoms) 5100
  -530160020
  -550568020
  20408000
[PASS] test_attack_overrideDec25StaleExitAtCapSize()
  -- cap-size pair: override minus no override (USDG atoms; LP shares in share atoms)
     attacker LONG leg 9590810701
     attacker SHORT leg -1553940
     Junior tranche -9581674361
[PASS] test_control_deployedCapsAllowedSizes()
  attacker legs open: LONG size, SHORT size (tokens, 18 decimals) 1895600000000000000000000 1600600000000000000000000
[PASS] test_control_weeklyScheduleNoEarlyClose()
[PASS] test_negative_holidayHoursCalendarRemovesTheFreeze()
  calendar answers compared, and those that differ (all within 21:30Z-05:15Z on the two dates)
  17856 124
[PASS] test_negative_raisedFrozenLimitFreezesLiveTrading()
  seconds frozen while Pyth publishes (Thu 00:00Z-22:00Z, Fri 05:00Z-22:00Z) 140400
  trader balance change on a full close: frozen, live, difference (USDG atoms)
  -516714066
  -16714066
  -500000000
Suite result: ok. 15 passed; 0 failed; 0 skipped
```

Each attack test builds the same holiday week on the deployed parameters and calls every exit path at the start and end of the window, each from the same state. `test_control_weeklyScheduleNoEarlyClose` removes only Pyth's holiday entry, and `test_negative_holidayHoursCalendarRemovesTheFreeze` changes only the calendar. The `StaleExit` tests run the same price path with the override, without it, without a move and with the recommended calendar rule, each from the same snapshot.

Core of the test, copied from `HolidayCalendarMismatch.t.sol` (the shared setup and the per-path balance checks are in the file):

```solidity
    /// Deployed calendar (no override). Every exit of an account with an open position, and the funding of a
    /// matured Junior redemption, fails from 22:01:00Z until Pyth's first print at 05:00:00Z.
    function test_attack_deployedCalendarDec24() public {
        _noOverride(DEC24_2026, "2026-12-24");
    }

    function test_attack_deployedCalendarDec31() public {
        _noOverride(DEC31_2026, "2026-12-31");
    }

    function _noOverride(
        uint256 day,
        string memory label
    ) internal {
        _holidayWeek(day, true, new uint256[](0), 259_200);
        _requestRedeemAt(day + 21 hours + 30 minutes); // matures at the 22:00Z LP epoch
        _markAt(_pythClose() - 1); // Pyth's last print, 21:59:59Z
        console2.log(string.concat("==== no override, ", label));
        Outcome memory o = _probe("22:00:59Z, mark age 60 s", _pythClose() + 59);
        assertTrue(o.withdraw, "withdraw still accepts a 60 s old mark");
        assertFalse(o.close || o.lpFunded || o.refresh, "no print after 21:59:59Z");
        _assertBlocked(_probe("22:01:00Z", _pythClose() + 60), "22:01:00Z");
        _assertBlocked(_probe("04:59:59Z", _pythReopen() - 1), "04:59:59Z");
        _assertOpen(_probe("05:00:01Z, first print 05:00:00Z", _pythReopen() + 1), "after the reopen");
        assertFalse(_frozenAt(_pythClose() + 60) || _frozenAt(_pythReopen() - 1), "protocol stays live");
        console2.log("seconds with the trader's exits and the redemption blocked", _pythReopen() - (_pythClose() + 60));
    }

    /// Override on Friday 2026-12-25, the holiday itself: Thursday 22:00Z-24:00Z stays live with no prints.
    function test_attack_overrideDec25() public {
        uint256[] memory days_ = new uint256[](1);
        days_[0] = DEC24_2026 + 1 days + 12 hours;
        _holidayWeek(DEC24_2026, true, days_, 259_200);
        _requestRedeemAt(day0 + 21 hours + 30 minutes);
        _markAt(_pythClose() - 1);
        console2.log("==== override on Friday 2026-12-25");
        _assertBlocked(_probe("Thu 22:01:00Z", _pythClose() + 60), "Thu 22:01:00Z");
        _assertBlocked(_probe("Thu 23:59:59Z", day0 + 1 days - 1), "Thu 23:59:59Z");
        _assertOpen(_probe("Fri 00:00:00Z, frozen", day0 + 1 days), "Fri 00:00:00Z");
        uint256 noon = day0 + 1 days + 12 hours;
        assertTrue(_frozenAt(noon) && _pythPublishesAt(noon), "frozen while Pyth publishes");
        console2.log("seconds with the trader's exits and the redemption blocked", day0 + 1 days - (_pythClose() + 60));
        console2.log("seconds frozen while Pyth publishes (Fri 05:00Z-22:00Z)", _weeklyClose() - _pythReopen());
    }

    /// Newest Pyth print at time t: one per second while Pyth publishes, otherwise its last print before the closure.
    function _newestPrint(
        uint256 t
    ) internal view returns (uint256 p) {
        p = t - 1;
        if (p >= _weeklyClose() && p < _weeklyReopen()) {
            p = _weeklyClose() - 1;
        }
        if (earlyClose && p >= _pythClose() && p < _pythReopen()) {
            p = _pythClose() - 1;
        }
    }
```

