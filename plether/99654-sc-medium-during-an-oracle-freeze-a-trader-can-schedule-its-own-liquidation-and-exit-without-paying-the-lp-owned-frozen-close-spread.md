# \#99654 [SC-Medium] During an oracle freeze, a trader can schedule its own liquidation and exit without paying the LP-owned frozen-close spread

**Submitted on Nov 14th 2026 at 17:00:51 UTC by @arcomus for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #99654
- **Report Type:** Smart Contract
- **Report severity:** Medium
- **Target:** https://github.com/immunefi-team/audit-comp-plether/tree/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src
- **Impacts:**
  - Contract fails to deliver promised returns, but doesn't lose value

## Description

## Brief/Intro

With the oracle frozen and the mark fixed, a trader can use `addMargin` to pick the second its position first hits the 3% frozen-market margin. Any keeper can then liquidate it at the stale mark plus a 2.5 bp shift, skipping the LP-owned 50 bp frozen-close spread. In the PoC at release parameters, a 400k position exits for 502.05 USDG instead of the 2,082.05 a voluntary frozen close costs, so LPs get 1,619.99 less and the treasury 160.00 less.

## Vulnerability Details

While frozen, the oracle reuses the engine's last mark (PletherOracle.sol:729-743), so apart from top-ups, equity only drifts down with carry. A voluntary close at that mark pays the 50 bp spread and 4 bp execution fee, computed only in the close math (CloseAccountingLib.sol:120-124), which `planLiquidation` never calls. A liquidation at the same mark applies a 2.5 bp adverse shift (PletherOracle.sol:742, :960-971), takes the 10 bp charge, half to the caller (LiquidationAccountingLib.sol:65-78), and returns the residual (CfdEnginePlanLib.sol:1529-1549). Live, that charge already exceeds the close fee, so the gap opens only while frozen. `addMargin` only checks the caller, the amount and an open position (CfdEngine.sol:501-529), so the trader decides when equity meets the 3% FAD margin (CfdEnginePlanLib.sol:1450); anyone can then liquidate (OrderLiquidationBatchLogic.sol:719-728).

```solidity
// CloseAccountingLib.sol:121-124, voluntary close math (planLiquidation never calls it)
state.executionFeeAssets = (notionalAssets * inputs.executionFeeBps) / 10_000;
if (inputs.oracleFrozen) {
    state.frozenSpreadAssets = (notionalAssets * inputs.frozenCloseSpreadBps) / 10_000;
}

// CfdEnginePlanLib.sol:1530-1549, _applyLiquidationSettlement at the same frozen mark
delta.settlementSeizedAssets = delta.pricePnlPledgeConsumedAssets + delta.lpLiquidationFeeAssets;
delta.residualAssets = delta.riskState.equityAssets - int256(delta.liquidationChargeAssets);
// ...
uint256 settlementAfterDebitsAssets = snap.accountBuckets.settlementBalanceAssets
    - delta.pricePnlPledgeConsumedAssets - delta.actionChargeCollectedAssets - delta.liquidationChargeAssets;
delta.settlementRetainedAssets =
    settlementAfterDebitsAssets + (delta.freshPayoutIsImmediate ? delta.freshTraderPayoutAssets : 0);
```

No audit item covers this. The listed Known Limitations do limit the spread to "oracle-frozen voluntary closes only" (SECURITY.md:1442, :1333), and text outside that section exempts frozen liquidations (SECURITY.md:843-845, ACCOUNTING_SPEC.md:156, :1188, WHITEPAPER.md:1074-1076, :1304). None covers a holder picking which exit it gets, and when: in the PoC nobody can liquidate a minute early, and 0.01 USDG of `addMargin` pushes the liquidation back 7 minutes. The stated reasons are liveness (SECURITY.md:845), which a spread capped at the residual keeps, and a "conservative" shift, here 2.5 bp against a 50 bp spread.

## Impact Details

This fits Low, "Contract fails to deliver promised returns, but doesn't lose value". The spread should "Compensate LPs for voluntary stale-mark exit" (WHITEPAPER.md:814), which a trader-timed frozen exit skips.

Measured in the PoC at release parameters (400k LONG, 12,003.00 equity, 10M Junior pool), a voluntary frozen close a minute before it turns liquidatable costs 2,082.05: 1,922.05 to the pool (1,920.00 net spread after the 79.99 VPI rebate, plus 2.05 carry) and 160.00 to the treasury. A neutral keeper's liquidation two minutes later costs 502.05: 302.05 to the pool, 200.00 to the keeper. LPs get 1,619.99 less and the trader saves 1,579.99.

Equity must start the freeze just above the 3% margin. A same-block top-up absorbs an adverse last live tick (the trader posts a 1 bp one and still saves 1,580.20), while a favourable one above weekend carry (about 4 USDG here) pushes the crossing past the reopen. Holding to the reopen skips the spread too (trader -84.13, pool +4.13), but the spread prices an exit during the freeze, which this route gets for 12.55 bp instead of 52.05.

The route comes back in every frozen window, weekends and calendar override days alike, and the skipped spread grows linearly with size.

## Recommendation

Have `_applyLiquidationSettlement` also seize the frozen-close spread when `snap.oracleFrozen`, from the margin left after price loss and capped at the residual (diff below). Underwater liquidations don't change; the 12 liquidation tests in the repo's oracle-regime suites still pass. Patched, the PoC's liquidation costs 420.50 more than the voluntary close, so timing one no longer pays.

```diff
--- a/packages/perps/src/libraries/CfdEnginePlanLib.sol
+++ b/packages/perps/src/libraries/CfdEnginePlanLib.sol
@@ -1527,8 +1527,10 @@
         _planLiquidationActionSettlement(snap, pos, delta);
 
         delta.freshTraderPayoutAssets = delta.priceGainAssets - delta.actionChargeWithheldAssets;
-        delta.settlementSeizedAssets = delta.pricePnlPledgeConsumedAssets + delta.lpLiquidationFeeAssets;
         delta.residualAssets = delta.riskState.equityAssets - int256(delta.liquidationChargeAssets);
+        uint256 frozenSpreadAssets = _frozenLiquidationSpreadAssets(snap, delta, pos, price);
+        delta.settlementSeizedAssets =
+            delta.pricePnlPledgeConsumedAssets + delta.lpLiquidationFeeAssets + frozenSpreadAssets;
         delta.residualPlan.mutation.positionMarginUnlockedAssets = pos.margin;
 
         if (delta.freshTraderPayoutAssets > 0) {
@@ -1544,12 +1546,36 @@
         }
 
         uint256 settlementAfterDebitsAssets = snap.accountBuckets.settlementBalanceAssets
-            - delta.pricePnlPledgeConsumedAssets - delta.actionChargeCollectedAssets - delta.liquidationChargeAssets;
+            - delta.pricePnlPledgeConsumedAssets - delta.actionChargeCollectedAssets - delta.liquidationChargeAssets
+            - frozenSpreadAssets;
         delta.settlementRetainedAssets =
             settlementAfterDebitsAssets + (delta.freshPayoutIsImmediate ? delta.freshTraderPayoutAssets : 0);
         return delta;
     }
 
+    /// @dev While the oracle is frozen a liquidation exits at the same stale mark as a voluntary frozen close, so it
+    ///      also assesses the LP-owned frozen-close spread. The spread is seized from the position margin left after
+    ///      price loss and is capped at the residual equity, so it never blocks a liquidation or creates a deficit
+    ///      (the same priority as the terminal full-close waiver).
+    function _frozenLiquidationSpreadAssets(
+        CfdEnginePlanTypes.RawSnapshot memory snap,
+        CfdEnginePlanTypes.LiquidationDelta memory delta,
+        CfdTypes.Position memory pos,
+        uint256 price
+    ) private pure returns (uint256 spreadAssets) {
+        if (!snap.oracleFrozen || delta.residualAssets <= 0) {
+            return 0;
+        }
+        spreadAssets = (((pos.size * price) / CfdMath.ASSETS_TO_TOKEN_SCALE) * snap.frozenCloseSpreadBps) / 10_000;
+        uint256 capAssets = pos.margin - delta.pricePnlPledgeConsumedAssets;
+        if (uint256(delta.residualAssets) < capAssets) {
+            capAssets = uint256(delta.residualAssets);
+        }
+        if (spreadAssets > capAssets) {
+            spreadAssets = capAssets;
+        }
+    }
+
     /// @notice Allocates still-unpaid liquidation carry and VPI clawback across price gain, reserves, and account cash.
     function _planLiquidationActionSettlement(
         CfdEnginePlanTypes.RawSnapshot memory snap,
```

## References

- https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/PletherOracle.sol#L729-L743
- https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/PletherOracle.sol#L934-L972
- https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/libraries/CloseAccountingLib.sol#L114-L126
- https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/libraries/LiquidationAccountingLib.sol#L61-L78
- https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/libraries/CfdEnginePlanLib.sol#L1450
- https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/libraries/CfdEnginePlanLib.sol#L1488-L1551
- https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/CfdEngine.sol#L501-L529
- https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/router/OrderLiquidationBatchLogic.sol#L706-L729
- https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/CfdEngineSettlementSidecar.sol#L875-L896
- https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/SECURITY.md#L1333-L1336
- https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/SECURITY.md#L1442
- https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/SECURITY.md#L837-L845
- https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/ACCOUNTING_SPEC.md#L152-L156
- https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/ACCOUNTING_SPEC.md#L1188
- https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/WHITEPAPER.md#L808-L814
- https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/WHITEPAPER.md#L1070-L1076
- https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/WHITEPAPER.md#L1304
- https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/DEPLOYMENT.md#L126-L158

## Link to Proof of Concept

https://gist.github.com/antonio-rosendo/4d5b3544ef59cc0f2a3e75df560b0edf

## Proof of Concept

## Proof of Concept

Runs in the repo's Foundry suite at the frozen commit, no in-scope contract modified. The fixture deploys at the DEPLOYMENT.md release values (some via the owner's 72 h timelock) and each scenario re-asserts them. Only USDG and Pyth are mocked; the router is the repo's `LegacyOrderRouterHarness` (production `OrderRouter` plus an overload the PoC never calls). Secret gist: https://gist.github.com/antonio-rosendo/4d5b3544ef59cc0f2a3e75df560b0edf

```
git clone https://github.com/immunefi-team/audit-comp-plether && cd audit-comp-plether
git checkout a82d4fefb02768e5b3bddcc97dee371eee24fea8 && git submodule update --init --recursive
export GIST=/absolute/path/to/this/gist
mkdir -p packages/perps/test/perps/poc/round4 && cp "$GIST"/FrozenWindowLiquidationExit.t.sol packages/perps/test/perps/poc/round4/
cd packages/perps
FOUNDRY_VIA_IR=false forge test --match-path test/perps/poc/round4/FrozenWindowLiquidationExit.t.sol -vv   # non-IR only to shorten the compile
cd ../.. && git apply "$GIST"/R4-2_fix.diff && cd packages/perps
FOUNDRY_VIA_IR=false forge test --match-path test/perps/poc/round4/FrozenWindowLiquidationExit.t.sol -vv   # with the fix: 2 passed, 2 failed (trader saving -420.50)
```

Output (excerpt; A = voluntary frozen close, C = neutral-keeper liquidation, B = same by the trader's second EOA, T_c = first liquidatable second):

```
  trader delta            A / C / B (B = T + own EOA): -2082.05 / -502.05 / -302.05
  pool.totalAssets delta  A / C / B                 : 1922.05 / 302.05 / 302.05
  treasury delta          A / C / B                 : 160.00 / 0.00 / 0.00
  trader C - A  (neutral-keeper liquidation saving)  : 1579.99
  pool A - C    (LP compensation not received)       : 1619.99
  conservation C (trader+alt+keeper+poolRaw+treasury deltas, units): 0
  executeLiquidation by neutral keeper at T_c - 60 s: reverted CfdEngine__PositionIsSolvent
  top-up 0.01 USDG at T_c - 60 s: healthy at T_c + 60 s = true; new T_c = old T_c + 420 s (7 min)
Suite result: ok. 4 passed; 0 failed; 0 skipped
```

Key test (FrozenWindowLiquidationExit.t.sol lines 354-378):

```solidity
    function test_FrozenWindow_NeutralKeeperLiquidationSkipsSpreadAndFee() public {
        console2.log("=== frozen window: neutral-keeper liquidation vs voluntary frozen close (400k LONG, 10M Junior) ===");
        _setupScenario();
        _calibrate(0, KEEPER);

        Out memory base = _measure();
        _logOut("SNAPSHOT (before any exit)", base);
        uint256 snap = vm.snapshotState();

        Out memory a = _branchVoluntaryClose(KEEPER);
        _logOut("BRANCH A voluntary frozen full close at T_c - 60 s (executed by the neutral keeper)", a);
        assertTrue(vm.revertToState(snap), "revert A");

        (Out memory c, ICfdEngineTypes.LiquidationPreview memory lpC) = _branchLiquidation(KEEPER);
        _logOut("BRANCH C liquidation by a neutral third-party keeper at T_c + 60 s", c);
        _logLiqPreview("C", lpC);
        assertTrue(vm.revertToState(snap), "revert C");

        (Out memory b, ICfdEngineTypes.LiquidationPreview memory lpB) = _branchLiquidation(TRADER_ALT);
        _logOut("BRANCH B (secondary) liquidation by the trader's own second EOA at T_c + 60 s", b);
        _logLiqPreview("B", lpB);

        _assertBranchShape(base, a, b, c);
        _assertEconomics(base, a, b, c, lpB, lpC);
    }
```

Headline asserts in `_assertEconomics` (lines 748-755):

```solidity
        // Lead result (neutral keeper), then the secondary own-EOA row.
        assertGe(_tw(c) - _tw(a), 1_500e6, "neutral-keeper liquidation leaves the trader >= 1,500 better off than the voluntary close");
        assertGe(int256(a.pool) - int256(c.pool), 1_500e6, "LP receives >= 1,500 less on the neutral-keeper liquidation exit");
        assertGe(int256(a.juniorPending) - int256(c.juniorPending), 1_500e6, "Junior pending principal >= 1,500 lower on C");
        assertGe(_tw(b) - _tw(a), 1_500e6, "own-second-EOA liquidation leaves the trader >= 1,500 better off");
        assertGe(int256(a.pool) - int256(b.pool), 1_500e6, "LP receives >= 1,500 less on the own-EOA liquidation exit");
        assertLt(_tw(base) - _tw(c), int256(cal.x0 + cal.xMid + cal.x1), "C: top-ups come back in the residual");
        assertLe(uint256(_tw(base) - _tw(c)), (notional * 13) / 10_000, "C: trader exit cost <= 13 bp of notional");
```
