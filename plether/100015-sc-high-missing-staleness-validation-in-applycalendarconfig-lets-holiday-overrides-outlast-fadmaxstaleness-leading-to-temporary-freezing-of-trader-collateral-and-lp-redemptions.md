# \#100015 [SC-High] Missing staleness validation in applyCalendarConfig lets holiday overrides outlast fadMaxStaleness, leading to temporary freezing of trader collateral and LP redemptions

**Submitted on Nov 16th 2026 at 03:50:55 UTC by @AT0M_I for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #100015
- **Report Type:** Smart Contract
- **Report severity:** High
- **Target:** https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/MarginClearinghouse.sol
- **Impacts:**
  - Temporary freezing of funds for at least 1 hour

## Description

## Brief/Intro

`CfdEngine.applyCalendarConfig` accepts any set of all-day UTC holiday overrides (FAD days) without checking that the resulting frozen window fits inside `fadMaxStaleness` (default 72h). A normal two-day override, Sunday plus Monday, is needed when the FX market stays shut through a holiday Monday. It stretches the regular weekend freeze to about 74h, so the last live mark expires inside the freeze. From then until the market reopens, closes and liquidations revert `PletherOracle__StalePrice`. Traders who hold a position cannot withdraw even idle collateral, and LP epoch settlement reverts, so maturing redemptions go unfunded. In production this freezes funds for every open position at the same time: 2h 0m 1s even when the operator follows the documented procedure, and up to 4h on a daylight-saving Friday. Nobody can end it early, because both configs that could fix it are behind a 72h timelock.

## Vulnerability Details

**1. No validation of freeze length (root cause).** `CfdEngine.applyCalendarConfig` (`src/CfdEngine.sol:689-704`) replaces the FAD-day set, and its docstring says "The admin is responsible for validation and timelock policy." `CfdEngineAdmin.proposeCalendarConfig` (`src/CfdEngineAdmin.sol:150-153`) states: "Only the runway is validated … while timestamps may be arbitrary." No code path compares the resulting continuous frozen window (the recurring weekend plus override days, `MarketCalendarLib.marketStatus`) with `fadMaxStaleness` (`src/CfdEngine.sol:140`).

**2. A stale mark blocks every exit.** During a freeze, `PletherOracle._frozenBasket` (`src/PletherOracle.sol:729-743`) replays `engine.lastMarkPrice()/lastMarkTime()`. It reverts `PletherOracle__StalePrice` once the mark is older than `fadMaxStaleness` (selected by `OracleFreshnessPolicyLib.getPolicy`). A stale frozen mark cannot be refreshed until the market reopens (`ORACLE_SYNCHRONIZATION.md:78`). Close commits still succeed, but `router.executeOrder` and `router.executeLiquidation` revert.

**3. Idle collateral is locked too.** `MarginClearinghouse._withdraw` calls `CfdEngine.checkWithdraw` (`src/CfdEngine.sol:730-748`), which calls `CfdEngineSettlementSidecar.validateWithdraw` (`:255-285`). For any account holding a position, that calls `_liveMark` (`:381-392`) and reverts `CfdEngine__MarkPriceStale`. This blocks withdrawal of collateral that is not backing the position.

**4. LP redemptions are not settled.** `HousePool._settleLpEpoch` requires a fresh mark whenever any position is open (`markFreshnessRequired = hasOpenPositions`; pool-reconcile mode uses `fadMaxStaleness` when frozen). It reverts `HousePool__MarkPriceStale`, so redemptions that mature in the window stay unfunded until reopen.

**Why a Sunday + Monday override is a normal admin action.**
- `packages/perps/SECURITY.md:1337-1338`: "unscheduled provider closures and holiday hours still require governance-configured UTC-day overrides."
- `WHITEPAPER.md:1120`: "Governance may also configure all-day UTC holiday overrides and their runway."
- Pyth's FX market hours (https://docs.pyth.network/pro/market-hours) say FX "continues during most US holidays", and most FX venues close for Christmas Day and New Year's Day.
- When such a holiday falls on a Monday (25 Dec 2028 and 1 Jan 2029 are both Mondays), the FX week does not reopen on Sunday at 17:00 New York time. Overrides work in whole UTC days, so covering the closure takes both Sunday (for Sunday 22:00–24:00 UTC) and Monday. The freeze then runs from Friday 22:00 UTC to Tuesday 00:00 UTC.

**Why this is not the documented "over-stale blocks execution" behaviour.**
- `WHITEPAPER.md:1112/1820` describe what happens once a market is over-stale. They do not say a routine holiday override should create that state.
- The repo-root `SECURITY.md:73` gives the reason the limit is 72h: "The 72-hour tolerance covers weekends plus holiday Mondays." The PoC shows a holiday Monday is not covered.
- `packages/perps/SECURITY.md:1332` states the design priority: "frozen-oracle windows prioritize close liveness". The PoC shows closes blocked.
- `ACCOUNTING_SPEC.md:553` says the protocol "keeps eligible queued-redemption settlement live during `oracleFrozen`". The PoC shows queued redemptions not settled.
- The project's own `test/perps/spec/oracle/FadStaleness.t.sol` reaches excess staleness only by lowering `fadMaxStaleness` to 1h. No test covers override days pushing a default-config freeze past 72h.

**Trusted-role scope.** The program keeps "a legitimate, documented trusted-role action that triggers a bug" in scope. The trigger here is the documented holiday-override procedure with the default `fadMaxStaleness`; no parameter is set to an extreme value. The bug is the missing check in the code, not admin misbehaviour. The program's privileged-address rule also says a report stays valid "where an in-scope impact arises from a logic error that a privileged address can trigger" (Scope tab, Privileged Addresses).

**Distinct from MED-30.** MED-30 says stop-losses have no automatic frozen trigger. This report does not rely on stop-losses. It shows that manual closes, liquidations, idle withdrawals and LP settlement are all blocked.

## Impact Details

Scenario: Sunday and Monday override, release parameters (maintenance margin 10 bps, initial margin 20 bps, `fadMaxStaleness` 259,200 s).

| Case | Last live mark | Mark stale from | Reopen | Funds frozen |
|---|---|---|---|---|
| Standard time, mark at Fri 21:00 UTC | Fri 21:00 | Mon 21:00:01 | Tue 00:00 | 3h |
| Standard time, operator procedure (mark Fri 21:59:59) | Fri 21:59:59 | Mon 22:00:00 | Tue 00:00 | 2h 0m 1s |
| Daylight saving, operator procedure (mark Fri 20:59:59) | Fri 20:59:59 | Mon 21:00:00 | Tue 00:00 | 3h 0m 1s |
| Daylight saving, mark at Fri 20:00 | Fri 20:00 | Mon 20:00:01 | Tue 00:00 | 4h |

During the window, a trader with a $100k position on $10k margin cannot close, cannot be liquidated, and cannot withdraw the remaining $40,000.00 of idle collateral. A position-free account can still withdraw during a stale freeze (tested with $5,000.00), which shows the block comes from holding a position. An LP redemption of about $100k that matures in the window is funded only after reopen. The amount is not reduced; it is delayed by about 3h. Every open position and every redemption maturing in the window is affected at the same time.

The impact matches "Temporary freezing of funds for at least 1 hour" (Medium). A three-day override (Sunday–Tuesday) freezes funds for 27h, which is the High duration floor. It is shown only as an upper bound and not claimed, because a three-day FX closure is unlikely.

## References

- `src/CfdEngine.sol`: `fadDayOverrides` (#137), `fadMaxStaleness` (#140), `applyCalendarConfig` (#689-704), `checkWithdraw` (#730-748)
- `src/CfdEngineAdmin.sol`: `proposeCalendarConfig` (#150-153)
- `src/PletherOracle.sol`: `_frozenBasket` (#729-743)
- `src/CfdEngineSettlementSidecar.sol`: `validateWithdraw` (#255-285), `_liveMark` (#381-392)
- `src/HousePool.sol`: `_settleLpEpoch`
- `src/libraries/MarketCalendarLib.sol`, `src/libraries/OracleFreshnessPolicyLib.sol`
- `packages/perps/SECURITY.md:95-96` (72h timelock), `:1332`, `:1337-1338`; repo-root `SECURITY.md:73`
- `ORACLE_SYNCHRONIZATION.md:78`; `WHITEPAPER.md:1102, 1112, 1120, 1820`; `ACCOUNTING_SPEC.md:553`
- Source tree: https://github.com/immunefi-team/audit-comp-plether/tree/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src


## Link to Proof of Concept

https://gist.github.com/AT0MII/d443ff42cf55bff16c3c0d4b19f724f5

## Proof of Concept

## Proof of Concept

### Step-by-step (Sunday + Monday holiday override, default config, `fadMaxStaleness` = 259,200 s / 72h)

1. **Admin proposes the holiday override.** The owner calls `CfdEngineAdmin.proposeCalendarConfig` with FAD days for Sunday and Monday (for example, Christmas Day falling on a Monday). Only the runway is validated.
2. **The 72h timelock passes and the config is applied.** `CfdEngine.applyCalendarConfig` stores the days. Nothing checks that the merged frozen window (Fri 22:00 UTC weekend start through Tue 00:00 UTC) is about 74h, which is longer than `fadMaxStaleness`.
3. **A trader opens a position before the weekend.** It's a $100k long on $10k margin, with $50k deposited in total, so $40k is idle collateral.
4. **The last live mark lands on Friday.** The keeper pushes the final mark at Fri 21:00 UTC (or 21:59:59, following `ORACLE_SYNCHRONIZATION.md:78`). The oracle then freezes at Fri 22:00 UTC.
5. **The mark expires inside the freeze.** On Monday at 21:00:01 UTC (or 22:00:00) the mark becomes older than 72h. The market is still frozen because of the Monday override, so the mark cannot be refreshed.
6. **Every exit reverts until Tuesday 00:00 UTC:**
   - `router.executeOrder` for the trader's close reverts `PletherOracle__StalePrice` (via `_frozenBasket`).
   - `router.executeLiquidation` reverts `PletherOracle__StalePrice`.
   - `clearinghouse.withdrawMargin` for the full $40,000.00 idle balance reverts `CfdEngine__MarkPriceStale` (via `validateWithdraw` → `_liveMark`).
   - `HousePool.settleLpEpoch` reverts `HousePool__MarkPriceStale` while any position is open, so a matured LP redemption is not funded.
7. **Controls isolate the cause.** In the same stale window, an account with no position can still withdraw. On a normal weekend with no override, closes, withdrawals and LP settlement all succeed. With a fresh mark during the freeze, the redemption settles.
8. **The freeze ends at reopen.** At Tue 00:00 UTC the market is live, the mark refreshes, and the close succeeds. A close at exactly 72h of mark age succeeds, and at 72h + 1s it reverts.

Result: funds are frozen for 2h 0m 1s to 3h on standard time, and 3h 0m 1s to 4h on a daylight-saving Friday.

### Runnable PoC (local Foundry only, no `src` changes)

All files are additions under `packages/perps/test/perps/`. They use mock Pyth and mock USDG only, and set calendar overrides through the real 72h timelock path. Run from `packages/perps` at commit `a82d4fefb02768e5b3bddcc97dee371eee24fea8`:

```bash
forge test --match-path 'test/perps/{ShortFreezePoC,IdleCollateralFreezePoC,LpStaleFreezePoC,DstFreezePoC,ExtendedFreeze3DayPoC}.t.sol' -vv
```

Result: 19 passed, 0 failed.

Each claim on its own:

- 3h freeze (main claim): `forge test -vv --match-test test_ShortFreeze_FreezesFundsFor3h`
- 2h freeze even when the operator procedure is followed: `forge test -vv --match-test test_ShortFreeze_LateMarkStillFreezes2h`
- Window edges measured: `forge test -vv --match-test "test_ShortFreeze_BoundaryAt72h|test_ShortFreeze_ReopenWorks"`
- Liquidation and withdrawal blocked: `forge test -vv --match-test test_ShortFreeze_BlockedPathsInWindow`
- $40,000.00 idle collateral locked: `forge test -vv --match-test test_ShortFreeze_PositionedTraderCannotWithdrawFullIdle`
- LP redemption delayed: `forge test -vv --match-test test_StaleWindow_LpRedemptionStuckUntilReopen`
- Daylight-saving Friday, 3h and 4h windows: `forge test -vv --match-test "test_NovemberDstFriday"`
- Controls: `forge test -vv --match-test "NormalWeekend|NoPosition|FreshFrozenWeekend"`

| Suite | Key tests (all pass) |
|---|---|
| `ShortFreezePoC` (5) | `FreezesFundsFor3h` (mark age 262,800 s at Mon 22:00, close reverts `StalePrice`); `LateMarkStillFreezes2h` (mark Fri 21:59:59, 7,201 s frozen); `BoundaryAt72h` (close at exactly 72h succeeds, 72h + 1s reverts); `ReopenWorks` (close at Tue 00:00 succeeds); `BlockedPathsInWindow` (liquidation and withdrawal revert) |
| `IdleCollateralFreezePoC` (3) | `ShortFreeze_PositionedTraderCannotWithdrawFullIdle` (full $40,000.00 idle withdrawal reverts `MarkPriceStale` in the 2-day window); `ExtendedFreeze_PositionedTraderCannotWithdrawIdle` ($5,000.00 withdrawal reverts for the positioned trader and succeeds for a position-free account); `NormalWeekend_PositionedTraderCanWithdrawIdle` (control succeeds) |
| `LpStaleFreezePoC` (4) | `settleLpEpoch` reverts `HousePool__MarkPriceStale`; redemption unfunded until reopen; no-position control and fresh-weekend control settle |
| `DstFreezePoC` (5) | US transition boundaries; daylight-saving Friday with mark 20:59:59 freezes for 10,801 s; mark 20:00 freezes for 4h |
| `ExtendedFreeze3DayPoC` (2) | three-day override upper bound: mark age 313,200 s, close and liquidation revert, 27h frozen (not claimed); normal-weekend control |

The test files are attached (or linked in a secret gist).

## Recommendation

In `applyCalendarConfig` (or a helper it calls), reject any FAD-day set whose continuous frozen window, merged with the recurring weekend, exceeds `fadMaxStaleness` minus a safety margin. Alternatives:
- derive the effective frozen staleness limit from the scheduled window;
- let frozen-regime closes and withdrawals use the stored mark beyond `fadMaxStaleness`, with a spread that grows as the mark ages.
