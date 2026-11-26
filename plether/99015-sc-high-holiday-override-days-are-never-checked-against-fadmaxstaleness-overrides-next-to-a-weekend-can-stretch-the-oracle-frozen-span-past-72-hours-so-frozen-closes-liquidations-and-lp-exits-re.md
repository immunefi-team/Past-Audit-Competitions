# \#99015 [SC-High] Holiday override days are never checked against fadMaxStaleness: overrides next to a weekend can stretch the oracle-frozen span past 72 hours, so frozen closes, liquidations and LP exits revert for up to 24 hours before reopening

**Submitted on Nov 11th 2026 at 12:52:31 UTC by @abarbatei for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #99015
- **Report Type:** Smart Contract
- **Report severity:** High
- **Target:** https://github.com/immunefi-team/audit-comp-plether/tree/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src
- **Impacts:**

## Description

## Brief/Intro

During a scheduled FX closure the perps engine runs in "oracle-frozen" mode. Every exit path (voluntary close, liquidation, frozen LP epoch settlement, margin withdrawal from an account with a position) replays the Engine's last stored mark. Each one accepts that mark only while it is at most `fadMaxStaleness` old (default 3 days). Governance adds holiday closures as whole-UTC-day overrides, and SECURITY.md says holiday hours require them.

Nothing connects these two settings. `proposeCalendarConfig` checks only the runway and `proposeFreshnessConfig` checks only that the limits are nonzero, and no document says that override days lengthen the freeze. An override is a whole UTC day. Next to the weekly closure, alone or with another override day, it stretches the continuous frozen span to 69 to 96 hours, and several common sets exceed 72 hours even with a perfect final mark. Once the stored mark is older than 72 hours, every frozen-window exit reverts until the calendar reopens. A new signed payload is ignored while frozen, and both settings sit behind 72-hour timelocks, so the problem cannot be fixed once it becomes visible.

The first case is close. Christmas 2026 and New Year 2027 fall on Fridays. If the operator also marks Christmas Eve or New Year's Eve (Thursdays), each of those weekends ends with 22 hours in which no trader can close, no keeper can liquidate and no LP redemption can be funded. That holds even with the procedure followed exactly (timelocked overrides, a fresh live mark in the last second before the freeze), as a Foundry test in the repository's suite shows against a control.

This is a configuration-safety insight about two governance settings the code treats as independent, not a loss of funds.

## Insight Details

### What the documentation promises

- "frozen-oracle windows prioritize close liveness over live-market-style freshness guarantees" (SECURITY.md:1332).
- "unscheduled provider closures and holiday hours still require governance-configured UTC-day overrides" (SECURITY.md:1337-1338; WHITEPAPER.md:1120).
- "`oracleFrozen` keeps eligible synchronized LP settlement live" (SECURITY.md:1387-1388), and frozen exits trade "price precision for exit liveness" (WHITEPAPER.md:1799-1801).
- The regime table lists `Frozen beyond max staleness | Blocked | Blocked | Blocked | Wait for valid data` as a separate failure regime (WHITEPAPER.md:1111-1112, 1820).
- The runbook: "Install a final live mark before the scheduled freeze and verify its allowed age. A stale frozen mark cannot be repaired from Pyth storage or a new live payload during the freeze." (ORACLE_SYNCHRONIZATION.md:78-80).

None of these says that the override calendar sets the length of the freeze, that it can exceed `fadMaxStaleness`, or that the limit must be raised in advance. The repository's other rationale for 72 hours, "covers weekends plus holiday Mondays" (root SECURITY.md:73, spot `PythAdapter`), does not hold here.

### What the code does

1. **The calendar.** `MarketCalendarLib.marketStatus` freezes from Friday 17:00 New York time (21:00 UTC in daylight time, 22:00 UTC in standard time) until Sunday 17:00 New York time. An override returns `(true, true)` for its whole UTC day (MarketCalendarLib.sol:51-53). The runway extends FAD only, not the freeze (MarketCalendarLib.sol:54-57; CfdEngine.sol:887-907).
2. **Frozen replay.** When frozen, `OracleFreshnessPolicyLib.getPolicy` selects `fadMaxStaleness` for every mode (OracleFreshnessPolicyLib.sol:59-83). `_resolveCurrentBasket` and `_resolveOrderExecutionBasket` return `_frozenBasket` before reading any payload, and `getUpdateFee` quotes zero (PletherOracle.sol:462-469, 582-594, 754-755). `_frozenBasket` reverts `PletherOracle__StalePrice` once `block.timestamp - lastMarkTime > maxStaleness` (PletherOracle.sol:729-742).
3. **Every exit uses that rule.** Router closes, liquidations, `updateMarkPrice` and `settleLpEpoch` go through the oracle. Permissionless frozen `HousePool.settleLpEpoch` uses the same limit (CfdEngineProtocolLens.sol:74-88) and reverts `HousePool__MarkPriceStale` (HousePool.sol:833, 1475-1482). The margin-withdrawal guard reverts `CfdEngine__MarkPriceStale` (CfdEngineSettlementSidecar.sol:278-281, 381-410). While the calendar says frozen, nothing can advance the mark.

### Why the two configurations are uncoupled

- `proposeCalendarConfig`: "Only the runway is validated ... timestamps may be arbitrary" (CfdEngineAdmin.sol:149-166). `finalizeCalendarConfig` and `CfdEngine.applyCalendarConfig` add no check (CfdEngineAdmin.sol:173-181; CfdEngine.sol:685-706).
- `proposeFreshnessConfig`: only nonzero values, "no admin-side maximum" (CfdEngineAdmin.sol:201-216).
- The two use "independent single-slot queues", each with `TIMELOCK_DELAY = 72 hours` (CfdEngineAdmin.sol:12-20; SECURITY.md:95-96).
- `fadMaxStaleness = 3 days` (CfdEngine.sol:140) leaves 24 hours of headroom over the 48-hour weekend. A Friday override moves the start from 21:00/22:00 UTC back to 00:00 UTC (69-70 h in total). A Monday bridged through Sunday adds 26 to 27 hours (74-75 h), and a Thursday-plus-Friday pair adds 46 hours (94 h).

### Span arithmetic (measured on-chain by the test)

`test_FrozenSpansCreatedByOverrideDays` applies each set through the owner's propose/finalize path, steps time in 15-minute increments and pins both edges to the second with `isOracleFrozen()`. The no-exit time is `(reopening - lastMarkTime) - fadMaxStaleness`. With a perfect final mark it equals `span - 72 h`.

| Override days (UTC) | Continuous frozen span | No-exit window, perfect final mark |
|---|---|---|
| none (weekend 2026-12-18) | Fri 22:00 to Sun 22:00 = **48 h** | 0 |
| Fri 2026-12-25 | Fri 00:00 to Sun 12-27 22:00 = **70 h** | 0, only if the final mark is at most 2 h old at Thu 24:00 |
| Thu 2026-12-24 + Fri 12-25 | Thu 00:00 to Sun 12-27 22:00 = **94 h** | **22 h** (Sun 00:00 to 22:00) |
| Thu 2026-12-31 + Fri 2027-01-01 | Thu 00:00 to Sun 01-03 22:00 = **94 h** | **22 h** |
| Fri 2027-03-26 (Good Friday, daylight time) | Fri 00:00 to Sun 03-28 21:00 = **69 h** | 0, only if the final mark is at most 3 h old |
| Fri 03-26 + Sun 03-28 + Mon 03-29 | Fri 00:00 to Tue 03-30 00:00 = **96 h** | **24 h** (all of Monday) |
| Sun 2027-03-28 + Mon 03-29 (daylight time) | Fri 03-26 21:00 to Tue 00:00 = **75 h** | **3 h** |
| Sun 2028-12-24 + Mon 12-25 | Fri 12-22 22:00 to Tue 12-26 00:00 = **74 h** | **2 h** (Mon 22:00 to 24:00) |
| Sun 2028-12-31 + Mon 2029-01-01 | Fri 12-29 22:00 to Tue 01-02 00:00 = **74 h** | **2 h** |
| Mon 2028-12-25 only | 48 h, then **live** Sun 22:00-24:00, then 24 h | 0 if a fresh tick exists Sunday evening, else 2 h |
| Fri 2027-03-26 + Mon 03-29, no Sunday | 69 h, then **live** Sun 21:00-24:00, then 24 h | 0 if a fresh tick exists Sunday evening, else 24 h |

On its own, a Monday override does not create one long span: Sunday evening UTC stays under live policy for 2 to 3 hours. If the provider publishes then, a keeper refresh resets the mark. If the closure runs through the holiday, nothing can execute in that live window, and Monday starts with a mark that expires before Tuesday. Bridging with a Sunday override gives the 74, 75 and 96-hour rows.

Which days a holiday needs depends on the provider's hours. Every plausible choice next to a weekend lands at or over the 72-hour budget. Because overrides are whole UTC days, covering an early close on 24 December means freezing all of Thursday (94 h); the alternative is leaving Thursday evening under live policy with no data.

## Why it matters

- **Traders cannot de-risk before the reopening gap.** Frozen closes are the documented risk-reduction path while protection triggers are disabled. Over Christmas 2026 with Thursday overrides, nobody can close in the last 22 hours before Sunday's reopening. Free collateral cannot be withdrawn from an account with an open position either.
- **Liquidations stop.** Accounts below the frozen (FAD) requirement, including those pushed there by four days of carry, wait for the reopening. They are then liquidated at the gapped price instead of the frozen mark with its conservative shift, and LPs carry the gap. In the demo, with a 1.035 reopening tick, the underwater LONG is liquidated there instead of at 1.02.
- **LP exits stop.** Matured redemptions cannot be funded by either the permissionless frozen path or the Router path (contrary to SECURITY.md:1387-1388).
- **There is no reactive fix.** New payloads are ignored. Raising `fadMaxStaleness` or removing the override takes 72 hours, and under the defaults a proposal activates exactly when a final mark published at proposal time expires. The raise must therefore be proposed by the time the final live mark is published, before any symptom appears. In the demo, a raise proposed at the first stale second becomes finalizable on Wed 2026-12-30 00:00:07 UTC, 50 hours after the reopening.

## Recommendation

**A. Couple the settings in `CfdEngineAdmin`.** Reject a calendar or freshness finalization that leaves the longest continuous frozen span, plus a budget for the final mark's age, above `fadMaxStaleness`:

```solidity
import {MarketCalendarLib} from "@plether/perps/libraries/MarketCalendarLib.sol";

uint256 public constant FINAL_MARK_AGE_BUDGET = 4 hours;
uint256 public activeFadMaxStaleness = 3 days;  // mirrors the engine value
uint256 public activeMaxFrozenSpan = 49 hours;  // recurring closure on the November DST weekend
error CfdEngineAdmin__FrozenSpanExceedsStaleness(uint256 frozenSpan, uint256 fadMaxStaleness);

function finalizeCalendarConfig() external onlyOwner {
    _requireTimelockReady(calendarConfigActivationTime);
    ICfdEngineAdminHost.EngineCalendarConfig memory config = _pendingCalendarConfig;
    uint256 span = maxFrozenSpan(config.fadDayTimestamps);
    _requireCovered(span, activeFadMaxStaleness);
    // ... existing body ...
    activeMaxFrozenSpan = span;
}

function finalizeFreshnessConfig() external onlyOwner {
    _requireTimelockReady(freshnessConfigActivationTime);
    ICfdEngineAdminHost.EngineFreshnessConfig memory config = pendingFreshnessConfig;
    _requireCovered(activeMaxFrozenSpan, config.fadMaxStaleness);
    // ... existing body ...
    activeFadMaxStaleness = config.fadMaxStaleness;
}

function _requireCovered(uint256 span, uint256 limit) internal pure {
    if (span + FINAL_MARK_AGE_BUDGET > limit) revert CfdEngineAdmin__FrozenSpanExceedsStaleness(span, limit);
}

/// Longest continuous frozen interval of the override set plus the recurring closure. Frozen state only
/// changes on whole UTC hours (00:00, 21:00, 22:00), so an hourly walk is exact.
function maxFrozenSpan(uint256[] memory dayTs) public pure returns (uint256 longest) {
    longest = 49 hours;
    for (uint256 i; i < dayTs.length; ++i) {
        uint256 dayStart = (dayTs[i] / 1 days) * 1 days;
        uint256 start = dayStart;
        while (_frozenAt(start - 1 hours, dayTs)) start -= 1 hours;
        uint256 end = dayStart + 1 days;
        while (_frozenAt(end, dayTs)) end += 1 hours;
        if (end - start > longest) longest = end - start;
    }
}

function _frozenAt(uint256 t, uint256[] memory dayTs) private pure returns (bool frozen) {
    uint256 day = t / 1 days;
    (, frozen) = MarketCalendarLib.marketStatus(t, _isOverride(dayTs, day), _isOverride(dayTs, day + 1), 0);
}

function _isOverride(uint256[] memory dayTs, uint256 day) private pure returns (bool) {
    for (uint256 i; i < dayTs.length; ++i) if (dayTs[i] / 1 days == day) return true;
    return false;
}
```

Mirror the check in both `propose*` functions; to raise both, finalize freshness first.

**B. Or derive the frozen limit from the span.** In `_frozenBasket` and the HousePool and sidecar freshness checks, accept the mark if `lastMarkTime + FINAL_MARK_AGE_BUDGET >= frozenRunStart`, where the Engine computes `frozenRunStart` by walking back over the calendar. Add an absolute cap on the run length.

**C. Runbook.** Add to ORACLE_SYNCHRONIZATION.md "Frozen operation" and to the SECURITY.md Known Limitations: *"Override days extend the continuous frozen span. If span + expected final-mark age exceeds `fadMaxStaleness`, propose a higher limit at least 72 hours before the final live mark and finalize it before the freeze. A Monday-only override leaves Sunday evening UTC under live policy."*

## Proof of Concept

## Proof of Concept

### Conditions under which this insight is valuable

- Holiday closures are configured as documented (SECURITY.md:1337-1338) with UTC-day overrides that touch the weekly closure or each other. Examples: Christmas Eve plus Christmas 2026, New Year's Eve plus New Year 2026/27, Easter 2027 bridged through Sunday, and Monday holidays bridged through Sunday.
- `fadMaxStaleness` is at its 72-hour default, or at any value below the longest span plus the final mark's age.
- Positions are open, or LP redemptions mature, during the tail of the closure.
- Nobody raised `fadMaxStaleness` 72 hours ahead; nothing prompts this today. The first instances are 24-27 December 2026 and 31 December 2026 to 3 January 2027.

### Test

`ProofHolidayFrozenSpanExceedsStaleness.t.sol` extends the suite's `HousePoolAsyncTestBase`; no source file changes. The owner uses only `CfdEngineAdmin` propose/finalize; traders, keepers and the LP use only public Router, HousePool, TrancheVault and MarginClearinghouse entry points. Suite helpers fund accounts, seed the LP and drive the Pyth mock. A LONG loses when the basket rises.

- `test_FrozenSpansCreatedByOverrideDays` produces the table above.
- `test_ChristmasThuFriOverride_ExitsBlocked22h`:
  1. Overrides for 2026-12-24/25 are proposed on 12-14 and finalized on 12-17.
  2. Two LONGs are opened: a well-margined one and a 4%-margin one.
  3. The final live mark (1.02) is installed with a fresh payload at Wed 23:59:59 UTC.
  4. At Sat 23:00 (age 71 h), on a snapshot: a margin withdrawal, a frozen close, a liquidation of the underwater LONG and frozen settlement of a matured Junior redemption all succeed.
  5. At Sun 00:00 and again at 21:58: all of them revert. A fresh-payload `updateMarkPrice` and Router `settleLpEpoch` also revert, and a raised `fadMaxStaleness` cannot be finalized before 12-30.
  6. After the 22:00 reopening, fresh data restores every path.
- `test_ControlSameWeekNoOverride_ExitsLive`: no overrides; every exit works at Sun 21:58 (mark 48 h old).
- `test_ChristmasSunMonOverride2028_ExitsBlocked2h`: Sun+Mon 2028-12-24/25, final mark at Fri 21:59:59. Exits work at Mon 21:59:59 (mark exactly 72 h old) and revert from Mon 22:00 until Tue 00:00.

### Command

From `packages/perps` at `a82d4fef`, with the file in `test/poc/hunt-in01/` (any `test/poc/<dir>/` works):

```
FOUNDRY_TEST=test/poc/hunt-in01 forge test --match-path 'test/poc/hunt-in01/ProofHolidayFrozenSpanExceedsStaleness.t.sol' -vv
```

### Output (condensed; Foundry 1.7.1, solc 0.8.35)

```
[PASS] test_ChristmasThuFriOverride_ExitsBlocked22h() (gas: 13989598)
  frozen span Thu 00:00 -> Sun 22:00 UTC, hours: 94
  Sat 2026-12-26 23:00 UTC (inside the limit):
      mark age, seconds: 255601
      margin withdrawal: ok; frozen close: Executed; frozen liquidation: executed; frozen LP settlement: funded
  Sun 2026-12-27 00:00 UTC (past the limit):
      mark age, seconds: 259208
      frozen close, liquidation, Router settleLpEpoch, updateMarkPrice: PletherOracle__StalePrice; HousePool.settleLpEpoch: HousePool__MarkPriceStale; margin withdrawal: CfdEngine__MarkPriceStale
  raised fadMaxStaleness proposed; earliest finalization (unix): 1798588807
  Sun 2026-12-27 21:58 UTC (still frozen): mark age 338283 s, same reverts
  minutes with no frozen exit (Sun 00:00 -> 22:00 UTC): 1320
  after reopening: mark refresh, close, liquidation and LP settlement all succeed with fresh data
[PASS] test_ControlSameWeekNoOverride_ExitsLive() (gas: 8346856)
  control, Sun 2026-12-27 21:58 UTC (no override): mark age 172681 s, all exits succeed
[PASS] test_ChristmasSunMonOverride2028_ExitsBlocked2h() (gas: 12325089)
  Mon 2028-12-25 21:59:59 UTC: mark age 259200 s, all exits succeed
  Mon 2028-12-25 22:00 UTC: mark age 259208 s, all exits revert
  minutes with no frozen exit (Mon 22:00 -> Tue 00:00 UTC): 120
[PASS] test_FrozenSpansCreatedByOverrideDays() (gas: 17850850)
  (every row of the table above)
Suite result: ok. 4 passed; 0 failed; 0 skipped
```

## References

Code:

- MarketCalendarLib.marketStatus: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/libraries/MarketCalendarLib.sol#L23-L58
- CfdEngine defaults: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/CfdEngine.sol#L136-L142
- CfdEngine isOracleFrozen: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/CfdEngine.sol#L887-L907
- CfdEngineAdmin queues and timelock: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/CfdEngineAdmin.sol#L10-L20
- CfdEngineAdmin calendar: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/CfdEngineAdmin.sol#L149-L181
- CfdEngineAdmin freshness: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/CfdEngineAdmin.sol#L201-L229
- OracleFreshnessPolicyLib: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/libraries/OracleFreshnessPolicyLib.sol#L49-L100
- PletherOracle frozen order basket: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/PletherOracle.sol#L582-L594
- PletherOracle _frozenBasket: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/PletherOracle.sol#L729-L756
- HousePool frozen settlement: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol#L794-L833
- HousePool freshness gate: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol#L1475-L1482
- CfdEngineProtocolLens: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/CfdEngineProtocolLens.sol#L74-L88
- Withdrawal guard: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/CfdEngineSettlementSidecar.sol#L274-L281

Documentation:

- SECURITY.md timelocks: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/SECURITY.md?plain=1#L95-L96
- SECURITY.md oracle limitations: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/SECURITY.md?plain=1#L1330-L1341
- SECURITY.md LP limitations: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/SECURITY.md?plain=1#L1383-L1388
- ORACLE_SYNCHRONIZATION.md: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/ORACLE_SYNCHRONIZATION.md?plain=1#L70-L80
- WHITEPAPER.md regimes: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/WHITEPAPER.md?plain=1#L1107-L1120
- WHITEPAPER.md limits: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/WHITEPAPER.md?plain=1#L1797-L1820

