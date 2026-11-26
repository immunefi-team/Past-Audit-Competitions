# \#97640 [SC-Medium] Whole-UTC-day holiday overrides cannot match Pyth's New York-time FX closures, freezing the market while Pyth is live and letting losing traders close at a stale mark against LPs

**Submitted on Nov 2nd 2026 at 15:11:24 UTC by @risinghorse for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #97640
- **Report Type:** Smart Contract
- **Report severity:** Medium
- **Target:** https://github.com/immunefi-team/audit-comp-plether/tree/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src
- **Impacts:**
  - Contract fails to deliver promised returns, but doesn't lose value

## Description

## Brief/Intro

Plether's documented way to handle FX holidays is a governance "override day", and override days can only be whole UTC days. On an override day the oracle is treated as frozen for all 24 UTC hours, and every price path replays the last stored Engine mark. The basket's Pyth FX feeds, however, close on New York time: the Dec 24 and Dec 31 holiday closures run from 22:00 UTC to 05:00 UTC the next morning. No UTC day fits inside such a closure, so any override either freezes the protocol while Pyth is publishing live prices, or leaves part of the closure in live mode with no data. In the frozen-while-live case, a trader whose position has moved against them can close at the stale pre-closure mark, paying only the fixed 50 bps spread. Opens are blocked, so this is a one-sided option at LPs' expense. For example, a 1.2% adverse move on $1M of notional leaks about $7,000 from the HousePool.

## Vulnerability Details

### 1. Overrides are whole UTC days

```solidity
// packages/perps/src/CfdEngine.sol:698-699
for (uint256 i; i < config.fadDayTimestamps.length; i++) {
    uint256 day = config.fadDayTimestamps[i] / 86_400;
```

```solidity
// packages/perps/src/libraries/MarketCalendarLib.sol:51-53
if (todayOverride) {
    return (true, true);
}
```

`_marketStatus` looks the override up by `block.timestamp / 86_400` (`CfdEngine.sol:897-907`). `proposeCalendarConfig` validates only the runway (`CfdEngineAdmin.sol:155-166`), so there is no way to set an intraday start or end. The recurring weekly window, by contrast, does follow the 17:00 New York boundary (`MarketCalendarLib.sol:38-49`).

### 2. Pyth's holiday closures run on New York time

The deployed EUR/USD feed (`0xa995d00b…c30b`, `script/DeployPerpsArbitrumSepolia.s.sol:147`) publishes this schedule on Hermes:

```
America/New_York;O,O,O,O,0000-1700,C,1700-2400;1224/0000-1700,1231/0000-1700
```

On Dec 24 and Dec 31, Pyth stops at 17:00 New York (22:00 UTC) and resumes at 00:00 New York (05:00 UTC). An override day therefore always covers hours in which Pyth is live, misses part of the closure, or both.

### 3. While frozen, every price path replays the stale Engine mark

```solidity
// packages/perps/src/PletherOracle.sol:588-593
if (!policy.oracleFrozen) {
    return _resolveHistoricalOrderBasket(refundRecipient, pythUpdateData, request);
}

basket = _frozenBasket(PriceMode.OrderExecution, policy.maxStaleness);
ok = true;
```

`_resolveCurrentBasket` short-circuits the same way for mark refresh, liquidation and LP reconcile (`PletherOracle.sol:754-755`). `_frozenBasket` returns `engine.lastMarkPrice()`, bounded only by `fadMaxStaleness` = 3 days (`PletherOracle.sol:729-743`). Live Pyth data cannot refresh the mark during an override.

### 4. Frozen voluntary closes fill at the stale price with only a fixed spread

```solidity
// packages/perps/src/PletherOracle.sol:934-939
// The fixed frozen-close spread replaces the Pyth adverse-confidence shift for voluntary
// close/reduce execution while the oracle is frozen. Basket confidence-width validation still
// applies when the basket is built, and opens (if called directly) retain the adverse shift.
if (oracleFrozen && request.isClose) {
    return price;
}
```

- The only extra charge is `notional * frozenCloseSpreadBps / 10_000` (`CloseAccountingLib.sol:122-124`), 50 bps in the release configuration (`script/DeployPerpsArbitrumSepolia.s.sol:137`).
- The same-block and MEV-boundary checks are skipped when the oracle is frozen (`OrderRouterExecutionSidecar.sol:447-452`), so a trader can commit and execute in the same block through the permissionless `executeOrder` (`OrderRouter.sol:240-247`).
- Opens revert with `OrderRouter__CloseOnlyWindow` (`OracleFreshnessPolicyLib.sol:59-60`; `router/OrderLiquidationBatchLogic.sol:332-333`), so nobody can take the other side: the option is one-sided.

### 5. An active override cannot be ended quickly

Calendar changes go through a 72-hour timelock (`CfdEngineAdmin.sol:20`, `173-181`). An override that turns out to overlap live trading cannot be removed in time.

### Documentation

`SECURITY.md:1337-1338` says holiday hours "still require governance-configured UTC-day overrides", so following the documented procedure creates this window. The frozen regime is meant for "genuine market closure" (`SECURITY.md:431`) and applies "once feeds are genuinely offline" (`SECURITY.md:824`), which day-granular overrides cannot guarantee for New York-aligned closures.

## Impact Details

- **Value from LPs to traders.** For each losing position closed during the frozen-while-live overlap, LPs lose (adverse live move against the stale mark − 50 bps) × notional. In the worked example, $1M of notional and a 1.2% move leak about $7,000.
- **Positions insolvent at the live price.** These can still close with their margin, because liquidations also run at the stale mark with only a 2.5 bps shift.
- **Size of the overlap at Christmas 2026.** A Dec 25 override freezes 17 hours of live Pyth trading. Because Dec 25, 2026 is a Friday, the stale Dec 24 mark is then replayed through the weekend while the real Friday close is public.
- **No safe configuration.**
  - Overriding Dec 24 instead freezes 22 hours of live trading.
  - Overriding both days freezes about 39 hours.
  - No override leaves 7 hours in live mode with no data, in which nothing executes or liquidates.
- **Limits.**
  - It requires an owner-only, timelocked calendar configuration, and the release deploys none.
  - It pays only on moves larger than the 50 bps spread during the overlap.
  - There is no theft of principal and no freeze of funds.

## Recommendation

1. Replace day-granular overrides with explicit, timelocked [start, end) timestamp windows, or accept New York-local windows in the same `MMDD/HHMM-HHMM` form as Pyth's schedule, so a 22:00–05:00 UTC closure can be frozen exactly. Validate that windows are ordered and bounded.
2. As defense in depth, while an admin override (not the recurring weekend window) is active, let a complete signed Pyth basket published after the override start end the frozen treatment for that operation, or reprice frozen closes once fresh data exists.
3. Add a fast, risk-reducing path to remove an active override early.
4. Document the overlap and the recommended handling of Pyth's short Dec 24 and Dec 31 closures in `SECURITY.md`.

## References

- Day flooring: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/CfdEngine.sol#L689-L706
- Override lookup: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/CfdEngine.sol#L897-L907
- Whole-day frozen status: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/libraries/MarketCalendarLib.sol#L38-L57
- Calendar proposal and timelock: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/CfdEngineAdmin.sol#L155-L181 and https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/CfdEngineAdmin.sol#L20
- Frozen order pricing: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/PletherOracle.sol#L582-L594 and https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/PletherOracle.sol#L925-L941
- Frozen current-price paths: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/PletherOracle.sol#L729-L756
- Frozen spread: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/libraries/CloseAccountingLib.sol#L122-L126
- Same-block and MEV checks skipped when frozen: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/OrderRouterExecutionSidecar.sol#L442-L452
- Close-only gate for opens: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/libraries/OracleFreshnessPolicyLib.sol#L59-L61 and https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/router/OrderLiquidationBatchLogic.sol#L332-L333
- Deployed feed ids and frozen spread: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/script/DeployPerpsArbitrumSepolia.s.sol#L137-L152
- Pyth EUR/USD schedule (Hermes): https://hermes.pyth.network/v2/price_feeds?query=EUR%2FUSD&asset_type=fx
- Documented holiday procedure and frozen-regime intent: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/SECURITY.md#L1337-L1338, https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/SECURITY.md#L431 and https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/SECURITY.md#L824


## Proof of Concept

## Proof of Concept

Defaults: `frozenCloseSpreadBps` 50, `executionFeeBps` 4, `fadMaxStaleness` 3 days, `fadRunwaySeconds` 1 hour, FAD margin 3%, frozen liquidation shift 2.5 bps.

### Step by step

1. **Governance follows `SECURITY.md:1337-1338`.** It proposes `fadDayTimestamps = [1798156800]` (Friday 25 December 2026, 00:00 UTC, UTC day 20812) and finalizes it after the 72-hour timelock.
2. **Thursday 24 December, 22:00 UTC (17:00 New York).** Pyth FX stops publishing, per the `1224/0000-1700` schedule entry.
   - The last accepted basket, P0, is the Engine mark.
   - From 22:00 to 24:00 UTC the protocol is in live mode (FAD runway from 23:00) with no Pyth data, so no orders or liquidations can execute.
3. **Friday 25 December, 00:00 UTC.** The override day begins and `isOracleFrozen()` becomes true. Every execution, liquidation and mark refresh now resolves to `_frozenBasket`, which is P0.
4. **25 December, 05:00 UTC (00:00 New York).** Pyth resumes publishing under the Friday `0000-1700` entry, but the protocol stays frozen for the rest of the UTC day.
   - The recurring Friday freeze then starts at 22:00 UTC, so P0 is replayed until the Sunday 27 December 22:00 UTC reopen.
   - That is 17 hours of live, public Pyth prices the protocol cannot use, followed by a weekend in which the real Friday close is public.
5. **During the overlap.** The live basket trades 1.2% above P0 (USD has weakened). Alice holds a $1,000,000 LONG-USD position opened before the runway, so her live loss is $12,000.
   - She commits a full close, which is allowed in close-only mode.
   - In the same block she calls the permissionless `OrderRouter.executeOrder`. The same-block and MEV-boundary checks are skipped because the oracle is frozen.
6. **Fill.** The close fills at P0 with no adverse shift, so price PnL is about 0.
   - Alice pays the frozen spread of $5,000 and the execution fee of $400, plus or minus VPI.
   - Compared with closing at the live price, she avoids about $7,000 of loss, and more counting the adverse-confidence shift a live close would pay. The fee is due either way.
   - HousePool LPs lose the same amount compared with settlement at the live price.
7. **One-sided option.** Had the live move favoured Alice, she would simply hold and close after the freeze at the live price. Opens are blocked, so no one can take the other side of the stale price. The trade pays whenever the adverse move exceeds about 50 bps.
8. **Liquidations.** In the same window, liquidations also use P0 with only a 2.5 bps shift. Positions insolvent at the live price can still close with their margin instead of being liquidated.

### No configuration avoids the overlap

- Overriding Dec 24 instead freezes 00:00–22:00 UTC of live trading (22 hours) and leaves Dec 25 00:00–05:00 UTC live with no data.
- Overriding both days freezes about 39 hours of live trading.
- Configuring no override leaves a 7-hour live-mode gap with no data and no FAD margin.

### How to reproduce in a test

1. Finalize a calendar config with `fadDayTimestamps = [1798156800]`.
2. Install a mark P0 at 21:59 UTC on 24 December, then warp to 12:00 UTC on 25 December.
3. Assert that `isOracleFrozen()` is true, although a mock Pyth feed publishes a basket 1.2% above P0 at that time.
4. Commit and execute a full close of a $1M LONG in the same block. Assert that it fills at P0 and pays only the 50 bps spread plus the 4 bps fee.
5. Compare with the settlement the same close would get at the live price after the freeze.

