# \#98623 [SC-High] Owner-configured override days are UTC-day aligned, so a closure spanning a weekend keeps the oracle frozen for over 3 days: the stored mark passes fadMaxStaleness and closes, liquidations, LP settlement and refresh all revert StalePrice until it ends

**Submitted on Nov 9th 2026 at 04:14:18 UTC by @enowk for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #98623
- **Report Type:** Smart Contract
- **Report severity:** High
- **Target:** https://github.com/immunefi-team/audit-comp-plether/tree/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src
- **Impacts:**
  - Temporary freezing of funds for at least 24 hours

## Description

## Brief/Intro

While the oracle is frozen (weekends and owner-configured override days), every price-dependent action replays the Engine's last stored mark, and that replay is accepted only while the mark is younger than `fadMaxStaleness`, which defaults to 3 days. A normal weekend freeze lasts 48 hours (Friday 22:00 UTC to Sunday 22:00 UTC in standard time, 21:00 in daylight-saving time), so the 3-day limit was sized with room to spare.

Override days do not follow that boundary: each one freezes its whole UTC day, 00:00 to 24:00. When a closure declared through override days touches a weekend, the frozen span grows past 3 days while the code applies no check that the stored mark can survive it:

- **Sunday and Monday override (a two-day closure).** The freeze runs from Friday 22:00 UTC straight through Monday 24:00 UTC, 74 hours, with no live interval at all. The last mark that can be installed is the one before the Friday close. It passes 3 days of age at Monday 22:00 UTC, and from then until Tuesday 00:00 UTC every exit path reverts with `PletherOracle__StalePrice`: delayed closes cannot execute, liquidations revert, LP epoch settlement reverts, and refreshing the mark is impossible because frozen policy ignores live data. The operator cannot prevent this by following the documented instruction to "install a final live mark before the scheduled freeze"; the calendar itself makes the minimum achievable mark age exceed the limit.
- **Monday override only.** The oracle is live for 2-3 hours on Sunday night. If no one refreshes the mark in that window (mark refresh pays no bounty), the same 2-hour outage occurs on Monday. With a Friday and Monday override pair, the whole Monday (24 hours) is unreachable.

The two parameters, override days and `fadMaxStaleness`, are validated independently (both behind separate 72-hour timelocks), and neither the Engine nor the documentation ties one to the other or mentions the Sunday-night refresh window.

## Vulnerability Details

**1. Frozen replay uses the stored mark and a 3-day age limit.**

https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/PletherOracle.sol#L729-L745
```solidity
    function _frozenBasket(PriceMode mode, uint256 maxStaleness) internal view returns (BasketPrice memory basket) {
        basket.price = engine.lastMarkPrice();
        basket.publishTime = engine.lastMarkTime();
        ...
        if (OracleFreshnessPolicyLib.isStale(basket.publishTime, maxStaleness, block.timestamp)) {
            revert PletherOracle__StalePrice(mode, bytes32(0), basket.publishTime, maxStaleness, block.timestamp);
        }
```
In every frozen mode the limit is `fadMaxStaleness` ([`OracleFreshnessPolicyLib.sol` L61-L82](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/libraries/OracleFreshnessPolicyLib.sol#L61-L82)), whose default is 3 days ([`CfdEngine.sol` L140](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/CfdEngine.sol#L140)). HousePool applies the same limit to LP settlement through `_requireFreshMark` ([`HousePool.sol` L1475-L1481](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol#L1475-L1481)).

**2. An override day freezes its whole UTC day.**

https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/libraries/MarketCalendarLib.sol#L51-L53
```solidity
        if (todayOverride) {
            return (true, true);
        }
```
Override days are set by the owner through the 72-hour `CfdEngineAdmin.proposeCalendarConfig` / `finalizeCalendarConfig` path and applied by `applyCalendarConfig`, which floors each timestamp to a UTC day ([`CfdEngine.sol` L689-L710](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/CfdEngine.sol#L689-L710)). The recurring weekend freeze, by contrast, starts and ends at the 17:00 New York FX boundary.

**3. The mismatch.** With a Monday override the frozen spans are Friday 22:00 to Sunday 22:00 UTC (standard time) and Monday 00:00 to 24:00 UTC. The live gap between them is two hours (three under daylight-saving time). The last mark that can serve the Monday freeze is therefore either one installed in that gap, or the one installed before the Friday close. The latter reaches 3 days of age at Monday 22:00 UTC. Nothing in the Engine, the oracle, or the documentation ties the override day to the staleness limit or flags the gap.

**4. Nothing can repair a stale frozen mark during the freeze.** Frozen policy never parses live data (`_resolveCurrentBasket` returns `_frozenBasket` first), so `updateMarkPrice` with a valid Pyth payload also reverts. The PoC shows all of `getLatestPrice`, `executeOrder` on a close, `executeLiquidation`, `settleLpEpoch` and `updateMarkPrice` reverting until Tuesday 00:00 UTC. This matches [`ORACLE_SYNCHRONIZATION.md` L78-L80](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/ORACLE_SYNCHRONIZATION.md?plain=1#L78-L80): "A stale frozen mark cannot be repaired from Pyth storage or a new live payload during the freeze."

**Relation to the documentation.** The documentation covers the operator duty ("install a final live mark before the scheduled freeze and verify its allowed age") and says that holiday closures "require governance-configured UTC-day overrides" ([`SECURITY.md` L1337-L1338](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/SECURITY.md?plain=1#L1337-L1338)). It does not state that an override adjacent to a weekend produces a 70-96 hour span against a 72-hour limit, that the only refresh window is the Sunday night before the override, or that mark refresh pays no bounty so no keeper is paid to perform it.

## Impact Details

Program impact category: **Temporary freezing of funds for at least 1 hour (Medium)**. In the Sunday+Monday case the freeze is 2 hours (3 in daylight-saving time) and no operator action can avoid it once the calendar is applied; with a Friday+Monday pair and an unrefreshed Sunday gap it is 24 hours. During the outage no trader can close, no position can be liquidated, no LP epoch settles and the mark cannot be refreshed, so positions keep their price risk and carry and liquidations that should have happened are delayed, which also exposes LPs to write-offs.

Why this is a defect and not only operator duty:
- The protocol's own stated priority for frozen windows is "close liveness over live-market-style freshness guarantees" ([`SECURITY.md` L1333](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/SECURITY.md?plain=1#L1333)); here the parameters as shipped make close liveness fail on any weekend-adjacent two-day closure.
- The trigger is a legitimate, documented owner action (override days exist precisely for "unscheduled provider closures and holiday hours"). The program treats a documented trusted-role action that triggers a bug as in scope.
- `applyCalendarConfig` accepts a calendar whose shortest possible frozen span exceeds `fadMaxStaleness` without any check, and no documentation states the resulting spans (70-96 hours) or the Sunday-night refresh requirement.

If the project considers the parameter coordination an operator responsibility, the same report stands as an Insight (Documentation Improvements); the author accepts that judgment.

Hardening options:
1. Define override days by the FX boundary (17:00 New York) rather than UTC midnight, so an override extends the weekend freeze contiguously and predictably, and document the resulting maximum span.
2. Have `applyCalendarConfig` (or `proposeCalendarConfig`) reject or warn when the longest frozen span implied by the configured days, including adjacency to weekends, exceeds `fadMaxStaleness`.
3. Allow a live, policy-valid Pyth basket to refresh the stored mark during an override day, or let the frozen staleness limit extend automatically to the end of the active override.
4. At minimum, document the spans, the Sunday-night refresh requirement before a Monday override, and that mark refresh is unpaid so the operator must schedule it.

## References

- Frozen replay and age check: [`PletherOracle.sol` L729-L745](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/PletherOracle.sol#L729-L745); limit selection [`OracleFreshnessPolicyLib.sol` L61-L82](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/libraries/OracleFreshnessPolicyLib.sol#L61-L82); default [`CfdEngine.sol` L140](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/CfdEngine.sol#L140); LP settlement gate [`HousePool.sol` L1475-L1481](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol#L1475-L1481)
- Override day semantics: [`MarketCalendarLib.sol` L30-L60](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/libraries/MarketCalendarLib.sol#L30-L60), [`CfdEngine.sol` L689-L710](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/CfdEngine.sol#L689-L710), [L897-L907](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/CfdEngine.sol#L897-L907)
- Documentation addressed: [`ORACLE_SYNCHRONIZATION.md` L72-L80](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/ORACLE_SYNCHRONIZATION.md?plain=1#L72-L80), [`SECURITY.md` L1330-L1341](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/SECURITY.md?plain=1#L1330-L1341)
- Known-issue check: the Cecuro, SC Audit Studio and AI-scan reports do not discuss override-day alignment against the frozen staleness limit or the Sunday-night refresh window.


## Proof of Concept

## Proof of Concept

**When this matters:** whenever governance declares a closure with override days that touches a weekend. A Sunday+Monday pair (two-day closure) produces a 74-hour continuous freeze with no live interval, so the stored mark goes stale no matter what the operator does. A Monday-only override produces the same outage if the mark is not refreshed in the 2-3 hour Sunday-night live window; a Friday+Monday pair with that window missed makes the whole Monday unreachable. During the outage no close, liquidation, LP epoch settlement or mark refresh is possible.

Two runnable Foundry tests are attached. Both use the repository's own fixture (`BasePerpTest`) at the pinned commit, with no fork or RPC. Only Pyth (`MockPyth`) and USDG are mocked; nothing under `packages/perps/src` is changed. Release risk parameters are used; the frozen staleness limit is the contract default (3 days). The fixture clock starts Monday 2024-03-04, so the test weekend is 9-10 March 2024, when US daylight-saving time begins; the calendar library handles that (Friday boundary 22:00 UTC, Sunday boundary 21:00 UTC). Override days are applied through the Engine admin, the documented governance path.

Setup and run:
```bash
git clone https://github.com/immunefi-team/audit-comp-plether.git
cd audit-comp-plether
git checkout --detach a82d4fefb02768e5b3bddcc97dee371eee24fea8
git submodule update --init --recursive
mkdir -p packages/perps/test/poc
# save the two attached files as:
#   packages/perps/test/poc/OverrideDayNoGapPoC.t.sol
#   packages/perps/test/poc/OverrideDayStaleMarkPoC.t.sol
FOUNDRY_TEST=test/poc forge test --root packages/perps --match-path "test/poc/OverrideDay*.t.sol" -vv
```

Expected output:
```
[PASS] test_sundayAndMondayOverrides_noLiveGap_operatorCannotPrevent()
[PASS] test_mondayOverrideAfterWeekend_staleMarkBlocksExitsForTwoHours()
2 passed; 0 failed
```

**Test 1, `OverrideDayNoGapPoC.t.sol` (Sunday + Monday override, no live gap):**
1. The owner applies a calendar with Sunday 2024-03-10 and Monday 2024-03-11 as override days. A trader holds an open position.
2. Friday 21:59 UTC: a keeper installs the final live mark, the latest moment the documented instruction can be followed.
3. Friday 22:00 UTC onward the oracle is frozen. Sunday 22:00 UTC, when the market would normally have reopened, it is still frozen because of the Sunday override. A keeper calls `updateMarkPrice` with a valid live Pyth payload; the call is accepted but frozen policy replays the stored mark, so `lastMarkTime` stays at Friday 21:59. No refresh is possible anywhere in the span.
4. Monday 22:00:01 UTC: the mark is older than 3 days. `getLatestPrice` reverts `PletherOracle__StalePrice`; a committed close cannot be executed (`executeOrder` reverts `StalePrice`); `executeLiquidation` reverts `StalePrice`; `settleLpEpoch` reverts.
5. Tuesday 00:00:01 UTC: live again; the re-committed close executes.

**Test 2, `OverrideDayStaleMarkPoC.t.sol` (Monday override only, Sunday-night gap not used):**
1. Monday 2024-03-11 is an override day. Final live mark installed Friday 21:59 UTC.
2. Sunday 22:00 UTC: the oracle is live (reopened 21:00 UTC); nobody refreshes.
3. Monday 21:58 UTC: frozen replay still works (mark 71 h 59 min old).
4. Monday 22:00:01 UTC: same outage as test 1, including `updateMarkPrice` with a live payload reverting `StalePrice`.
5. Tuesday 00:00:01 UTC: live again; the close executes.

Attached files: `packages/perps/test/poc/OverrideDayNoGapPoC.t.sol`, `packages/perps/test/poc/OverrideDayStaleMarkPoC.t.sol`.

