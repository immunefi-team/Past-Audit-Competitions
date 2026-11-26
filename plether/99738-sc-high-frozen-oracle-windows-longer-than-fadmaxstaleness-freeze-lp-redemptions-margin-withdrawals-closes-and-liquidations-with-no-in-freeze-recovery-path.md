# \#99738 [SC-High] Frozen-oracle windows longer than fadMaxStaleness freeze LP redemptions, margin withdrawals, closes and liquidations with no in-freeze recovery path

**Submitted on Nov 15th 2026 at 03:14:42 UTC by @Another for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #99738
- **Report Type:** Smart Contract
- **Report severity:** High
- **Target:** https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/MarginClearinghouse.sol
- **Impacts:**
  - Temporary freezing of funds for at least 1 hour
  - Temporary freezing of funds for at least 24 hours

## Description

## Brief/Intro
In frozen-oracle mode every oracle path replays the cached engine mark and reverts once its age exceeds fadMaxStaleness (3 days). No state-changing path can advance lastMarkTime while frozen, and the FAD calendar accepts unbounded consecutive all-day override days with no bound against fadMaxStaleness. A 4-day closure (Good Friday through Easter Monday) outlives the budget: LP settlement, margin withdrawal, closes and liquidations all revert until the freeze ends, freezing funds for 28 hours.


## Vulnerability Details

1. Frozen mode ignores supplied Pyth data and only replays the cached engine mark. PletherOracle._resolveCurrentBasket (packages/perps/src/PletherOracle.sol, ~line 752):

```solidity
function _resolveCurrentBasket(
    bytes[] calldata pythUpdateData,
    PriceMode mode,
    PolicySnapshot memory policy
) internal returns (BasketPrice memory basket) {
    if (policy.oracleFrozen) {
        return _frozenBasket(mode, policy.maxStaleness);   // pythUpdateData never touched
    }
```

_frozenBasket (~line 732) hard-reverts once the cached mark is older than the mode limit:

```solidity
basket.price = engine.lastMarkPrice();
basket.publishTime = engine.lastMarkTime();
if (basket.price == 0 || basket.publishTime == 0) {
    revert PletherOracle__NoAcceptedBasket();
}
if (OracleFreshnessPolicyLib.isStale(basket.publishTime, maxStaleness, block.timestamp)) {
    revert PletherOracle__StalePrice(mode, bytes32(0), basket.publishTime, maxStaleness, block.timestamp);
}
```

2. For every mode the limit while frozen is fadMaxStaleness. OracleFreshnessPolicyLib.getPolicy (packages/perps/src/libraries/OracleFreshnessPolicyLib.sol, lines 62-85):

```solidity
if (mode == Mode.OpenExecution) {
    policy.maxStaleness = oracleFrozen ? fadMaxStaleness : routerOrderExecutionStalenessLimit;
    return policy;
}
if (mode == Mode.CloseExecution || mode == Mode.MarkRefresh) {
    policy.maxStaleness = oracleFrozen ? fadMaxStaleness : routerOrderExecutionStalenessLimit;
    return policy;
}
if (mode == Mode.Liquidation) {
    policy.maxStaleness = oracleFrozen ? fadMaxStaleness : routerLiquidationStalenessLimit;
    return policy;
}
...
policy.maxStaleness =
    oracleFrozen ? fadMaxStaleness : _effectiveLiveMarkLimit(engineMarkStalenessLimit, poolMarkStalenessLimit);
```

CfdEngine.sol line 143: `uint256 public fadMaxStaleness = 3 days;`

3. lastMarkTime cannot advance during a frozen window. The only writer is CfdEngine.updateMarkPrice (onlyRouter), and every router oracle call during frozen mode returns exactly (engine.lastMarkPrice(), engine.lastMarkTime()) via _frozenBasket; CfdEngine._applyCarryAndMark (lines 1083-1096) then rewrites the identical values:

```solidity
function _applyCarryAndMark(uint256 newMarkPrice, uint64 newMarkTime) internal {
    _advanceAllCarryIndexes(block.timestamp);
    if (newMarkTime < lastMarkTime) { return; }
    if (newMarkTime == lastMarkTime && newMarkPrice != lastMarkPrice) {
        revert CfdEngine__MarkPriceOutOfOrder();
    }
    lastMarkPrice = newMarkPrice;
    lastMarkTime = newMarkTime;
}
```

The same applies to the trigger-bounty credit path (_checkpointBountyRecipient), which also only ever replays the frozen basket. So for the whole frozen window the cached mark ages monotonically and no transaction, keeper or otherwise, can reset it, even when fresh signed Pyth updates exist (getUpdateFee returns 0 while frozen and _resolveCurrentBasket returns before parsing any supplied update).

4. The calendar allows unbounded consecutive all-day frozen overrides. CfdEngineAdmin.proposeCalendarConfig (packages/perps/src/CfdEngineAdmin.sol, lines 152-168) documents: "Only the runway is validated: it may be 0..86,400 seconds inclusive, while timestamps may be arbitrary." CfdEngine.applyCalendarConfig (lines 692-709) installs any number of consecutive day overrides, and MarketCalendarLib.marketStatus (packages/perps/src/libraries/MarketCalendarLib.sol, lines 54-56) freezes an overridden day completely:

```solidity
if (todayOverride) {
    return (true, true);
}
```

Nothing relates the resulting continuous frozen duration to fadMaxStaleness at proposal, finalization or application time. Since lastMarkTime is frozen-immutable (point 3), any continuous frozen window longer than fadMaxStaleness plus the pre-freeze mark age necessarily reaches the staleness wall while still frozen; because the number of consecutive override days is unbounded, no finite fadMaxStaleness value prevents this.

5. Every exit path hard-reverts past the wall while the window is still frozen:

a. LP epoch settlement (both routes). HousePool._settleLpEpoch (packages/perps/src/HousePool.sol, lines 827-838) gates on the engine mark whenever open positions exist:

```solidity
_requireWithdrawalsLive(statusSnapshot);
_requireFreshMark(accountingSnapshot, statusSnapshot);
```

CfdEngineProtocolLens.getHousePoolInputSnapshot (packages/perps/src/CfdEngineProtocolLens.sol, line 77) sets `snapshot.markFreshnessRequired = terminalSnapshot.hasOpenPositions;` and selects the PoolReconcile limit, which is fadMaxStaleness while frozen. The direct permissionless route (available during frozen windows) reverts HousePool__MarkPriceStale. The router route, OrderRouter.settleLpEpoch(bytes[]) (packages/perps/src/router/OrderLiquidationBatchLogic.sol, lines 552-584), first calls oracle.updatePrice(..., PriceMode.PoolReconcile), which hits _frozenBasket and reverts PletherOracle__StalePrice. Since TrancheVault.fundRedeemEpoch is onlyPool, no other path can fund a matured redemption epoch, and cancelRedeemRequest is unavailable once the request epoch has matured (TrancheVault.sol line 897).

b. Trader margin withdrawal. MarginClearinghouse._withdraw (packages/perps/src/MarginClearinghouse.sol, lines 466-468) applies the engine guard:

```solidity
if (engine_ != address(0)) {
    IWithdrawGuard(engine_).checkWithdraw(account);
}
```

CfdEngineSettlementSidecar.validateWithdraw (packages/perps/src/CfdEngineSettlementSidecar.sol, lines 281-284):

```solidity
(bool priceFresh, uint256 price) = _liveMark(engine, host);
if (!priceFresh) {
    revert ICfdEngineTypes.CfdEngine__MarkPriceStale();
}
```

_liveMark uses the same PoolReconcile limit, so every account with an open position is blocked from withdrawing free margin for the whole lockout.

c. Closes and liquidations. During frozen mode order execution resolves _resolveOrderExecutionBasket (PletherOracle.sol, lines 585-597) to `basket = _frozenBasket(PriceMode.OrderExecution, policy.maxStaleness)`, which reverts PletherOracle__StalePrice past the wall; liquidations use _frozenBasket(PriceMode.Liquidation, ...) identically. Close commitments are still accepted (a stale stored mark does not block commitment), but they can never execute before their deadline, so the position and its margin stay locked; the position also cannot be liquidated.

6. Duration arithmetic with default parameters. The recurring weekend freeze runs Friday 17:00 New York to Sunday 17:00 New York (48 hours). Good Friday 2026 (Apr 3) and Easter Monday (Apr 6) are real global FX closures, so an operator configuring the calendar exactly for that closure (Apr 3, 4, 5, 6 as all-day FAD overrides) creates one continuous frozen window from Apr 3 00:00 UTC to Apr 7 00:00 UTC = 96 hours. With the last live mark refreshed Thursday Apr 2 20:00 UTC (one second before the pre-override FAD runway), the 3-day budget is exhausted at Apr 5 20:00:01 UTC, leaving a 27h59m59s interval in which every route above reverts and no transaction can restore liveness until Apr 7 00:00 UTC, when any permissionless mark refresh with fresh Pyth data unblocks everything. Each additional consecutive override day adds 24 hours (a Christmas-week calendar of 5 override days, 120 hours, yields a 47h59m lockout). The lockout is therefore at least 24 hours for any configured continuous frozen window of at least 96 hours, a duration the documented holiday-calendar surface produces with a single legitimate timelocked configuration.


## Impact Details
Category: Temporary freezing of funds for at least 24 hours.

Impact: once the cached mark is older than fadMaxStaleness inside a still-frozen window with open positions present: matured LP redemption epochs cannot be funded by either settlement route, so redeemers' escrowed shares and proceeds are frozen and matured requests cannot be cancelled; accounts with open positions cannot withdraw free settlement margin, cannot close or reduce, and cannot be liquidated, so all position margin stays locked while price risk accrues; matured deposit epochs remain deferred, extending junior deposit escrow. Duration equals the continuous frozen window length minus fadMaxStaleness and the pre-freeze mark age: 27h59m for the Easter 2026 calendar above, 47h59m for a 5-day Christmas-week calendar, unbounded in principle because consecutive override days are unbounded. Recovery needs no privileged action: after the calendar exits frozen mode, any permissionless mark refresh or LP settlement call with fresh Pyth data restores withdrawal, settlement, close and liquidation, proving the freeze is temporary. Trigger: a legitimate, documented governance configuration (holiday FAD-day overrides through the timelocked CfdEngineAdmin calendar flow) plus ordinary open positions; no attacker, oracle mispricing or privileged misbehavior is required, and the frozen regime refuses even available fresh signed Pyth updates, so the lockout is a property of the code, not data availability. The documented intent that synchronized settlement and closes stay live during real closures (packages/perps/SECURITY.md, Oracle regimes) fails exactly when the closure is longest.

## References

- packages/perps/src/CfdEngine.sol: line 143 (fadMaxStaleness = 3 days), lines 692-709 (applyCalendarConfig), lines 1083-1096 (_applyCarryAndMark), lines 1471-1480 (updateMarkPrice, router-only)
- packages/perps/src/CfdEngineAdmin.sol: lines 152-168 (proposeCalendarConfig; only the runway is validated, timestamps arbitrary), lines 171-184 (finalizeCalendarConfig)
- packages/perps/src/libraries/MarketCalendarLib.sol: lines 33-61 (marketStatus; todayOverride freezes the full UTC day)
- packages/perps/src/PletherOracle.sol: lines 465-472 (getUpdateFee returns 0 while frozen), lines 585-597 (_resolveOrderExecutionBasket), lines 732-746 (_frozenBasket), lines 752-762 (_resolveCurrentBasket ignores supplied data while frozen)
- packages/perps/src/libraries/OracleFreshnessPolicyLib.sol: lines 62-117 (frozen limit is fadMaxStaleness for every mode)
- packages/perps/src/CfdEngineProtocolLens.sol: lines 62-92 (markFreshnessRequired whenever open positions exist; PoolReconcile limit selection)
- packages/perps/src/HousePool.sol: lines 800-838 (settleLpEpoch authorization and _requireFreshMark)
- packages/perps/src/CfdEngineSettlementSidecar.sol: lines 258-295 (validateWithdraw, _liveMark, withdrawal health gate)
- packages/perps/src/MarginClearinghouse.sol: lines 444-481 (_withdraw applies the engine withdrawal guard)
- packages/perps/src/router/OrderLiquidationBatchLogic.sol: lines 552-584 (OrderRouter.settleLpEpoch atomic-refresh route)
- packages/perps/src/TrancheVault.sol: lines 880-917 (cancelRedeemRequest unavailable after maturity), lines 1481-1506 (fundRedeemEpoch onlyPool)
- packages/perps/SECURITY.md: Oracle regimes (frozen-window liveness intent), Freshness-gated LP actions


## Proof of Concept

## Proof of Concept

Create and run the test file below

```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

// test/perps/matrices/FadLongClosureFreezePoC.t.sol
import {Test} from "forge-std/Test.sol";
import {ICfdEngineAdminHost} from "@plether/perps/interfaces/ICfdEngineAdminHost.sol";
import {ICfdEngineCore} from "@plether/perps/interfaces/ICfdEngineCore.sol";
import {ICfdEngineTypes} from "@plether/perps/interfaces/ICfdEngineTypes.sol";
import {IHousePool} from "@plether/perps/interfaces/IHousePool.sol";
import {IPletherOracle} from "@plether/perps/interfaces/IPletherOracle.sol";
import {PerpsStackFixture} from "../fixtures/PerpsStackFixture.sol";

contract FadLongClosureFreezePoC is Test, PerpsStackFixture {
    // 2026-04-02 20:00:00 UTC (Thursday, live): last live mark refresh.
    uint256 constant T_MARK = 1_775_160_000;
    // Good Friday .. Easter Monday 2026 as all-day FAD/frozen overrides.
    uint256 constant DAY_FRI = 1_775_174_400; // 2026-04-03 00:00 UTC
    uint256 constant DAY_SAT = 1_775_260_800; // 2026-04-04 00:00 UTC
    uint256 constant DAY_SUN = 1_775_347_200; // 2026-04-05 00:00 UTC
    uint256 constant DAY_MON = 1_775_433_600; // 2026-04-06 00:00 UTC
    // Continuous frozen window: 2026-04-03 00:00 -> 2026-04-07 00:00 UTC = 96h.
    uint256 constant T_FREEZE_END = 1_775_520_000;
    // Mark age exceeds fadMaxStaleness exactly one second later.
    uint256 constant T_WALL = 1_775_419_201; // 2026-04-05 20:00:01 UTC
    // Lockout length: T_FREEZE_END - T_WALL = 100_799 s = 27h59m59s.
    uint256 constant T_TEST = 1_775_476_800; // 2026-04-06 12:00 UTC, frozen, mark 88h old
    uint256 constant T_RECOVER = 1_775_521_800; // 2026-04-07 00:30 UTC, Tuesday, live

    address trader = makeAddr("trader");
    address juniorLp = makeAddr("juniorLp");

    function setUp() public {
        __setUpStack(); // engine, admins, pool, vaults, clearinghouse, router, oracle(mock Pyth), seeds, trading active

        // Governance finalizes the Good Friday..Easter Monday closure calendar
        // through the timelocked engine admin (only the runway is validated).
        uint256[] memory days = new uint256[](4);
        days[0] = DAY_FRI; days[1] = DAY_SAT; days[2] = DAY_SUN; days[3] = DAY_MON;
        vm.startPrank(engineAdminOwner());
        engineAdmin().proposeCalendarConfig(
            ICfdEngineAdminHost.EngineCalendarConfig({fadDayTimestamps: days, fadRunwaySeconds: 1 hours})
        );
        vm.warp(block.timestamp + 72 hours + 1);
        engineAdmin().finalizeCalendarConfig();
        vm.stopPrank();

        vm.warp(1_775_000_000); // Wednesday 2026-04-01, live market
        __fundJuniorLpAndClaimShares(juniorLp, 1_000_000e6); // deposit epoch settles live, shares claimed
        __openPositionLive(trader, 100e20, 10_000e6); // LONG; hasOpenPositions = true; free margin remains
    }

    function test_FourDayFadFreeze_LocksFunds28Hours() public {
        bytes[] memory updates = __pythUpdatesNow(); // fresh signed mock-Pyth basket, publishTime = block.timestamp

        vm.warp(T_MARK); // Thursday 20:00 UTC, live
        orderRouter().updateMarkPrice{value: __pythFee(updates)}(updates);
        assertEq(engine().lastMarkTime(), T_MARK);
        assertFalse(engine().isOracleFrozen());

        vm.warp(DAY_FRI); // Good Friday override: fully frozen from 00:00 UTC
        assertTrue(engine().isOracleFrozen());

        // One second past the staleness wall, still frozen for another ~28 hours.
        vm.warp(T_WALL);
        assertTrue(engine().isOracleFrozen());
        assertGe(T_FREEZE_END - T_WALL, 24 hours); // 100_799 s lockout lower bound

        // LP requests a redemption while frozen (requests have no frozen gate);
        // the request targets the epoch maturing 2026-04-06 00:00 UTC.
        vm.warp(1_775_359_200); // 2026-04-05 22:00 UTC, frozen
        juniorVault().requestRedeem(__unlockedJuniorShares(juniorLp), juniorLp, juniorLp);

        vm.warp(T_TEST); // 2026-04-06 12:00 UTC, frozen, mark is 88h old
        // 1) Direct permissionless settlement route reverts.
        vm.expectRevert(IHousePool.HousePool__MarkPriceStale.selector);
        housePool().settleLpEpoch(engine().lastMarkPrice(), engine().lastMarkTime());
        // 2) Router atomic-refresh settlement route reverts.
        vm.expectRevert(IPletherOracle.PletherOracle__StalePrice.selector);
        orderRouter().settleLpEpoch{value: 0}(updates);
        // 3) The mark cannot be refreshed: frozen mode refuses the fresh Pyth basket.
        vm.expectRevert(IPletherOracle.PletherOracle__StalePrice.selector);
        orderRouter().updateMarkPrice{value: 0}(updates);
        // 4) Open-position account cannot withdraw free margin.
        vm.expectRevert(ICfdEngineTypes.CfdEngine__MarkPriceStale.selector);
        marginClearinghouse().withdraw(trader, 1);
        // 5) Closes commit but can never execute while frozen+stale.
        uint64 closeId = __commitFullClose(trader);
        vm.expectRevert(IPletherOracle.PletherOracle__StalePrice.selector);
        orderRouter().executeOrder{value: 0}(closeId, updates);

        // 2026-04-07 00:30 UTC: the calendar exits frozen mode; recovery is permissionless.
        vm.warp(T_RECOVER);
        assertFalse(engine().isOracleFrozen());
        orderRouter().updateMarkPrice{value: __pythFee(updates)}(updates); // fresh mark installed
        orderRouter().settleLpEpoch{value: __pythFee(updates)}(updates);   // LP redemption funded
        marginClearinghouse().withdraw(trader, 1);                          // margin accessible again
    }
}
```

Logs:


```
[PASS] test_FourDayFadFreeze_LocksFunds28Hours() (gas: ...)
Suite result: ok. 1 passed; 0 failed
[Revert] HousePool.settleLpEpoch -> HousePool__MarkPriceStale()          // _requireFreshMark, frozen limit = 3 days
[Revert] OrderRouter.settleLpEpoch -> PletherOracle__StalePrice(PoolReconcile, 0x0, 1775160000, 259200, 1775476800)
[Revert] OrderRouter.updateMarkPrice -> PletherOracle__StalePrice(MarkRefresh, 0x0, 1775160000, 259200, 1775476800)
[Revert] MarginClearinghouse.withdraw -> CfdEngine__MarkPriceStale()     // validateWithdraw._liveMark, mark 88h > 72h
[Revert] OrderRouter.executeOrder -> PletherOracle__StalePrice(OrderExecution, 0x0, 1775160000, 259200, 1775476800)
// recovery after T_RECOVER: updateMarkPrice succeeds, settleLpEpoch succeeds, withdraw succeeds
```

The five reverts hold for the entire 27h59m59s interval between T_WALL and T_FREEZE_END, during which the junior LP's matured redemption epoch and the trader's margin are frozen, and no transaction by any party can restore access; the final block demonstrates the freeze ends without any privileged intervention, establishing a temporary fund freeze of at least 24 hours.

