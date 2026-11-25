# \#98915 [SC-Medium] Close planner never tops up the execution fee netted into a VPI rebate, so the treasury misses that fee

**Submitted on Nov 10th 2026 at 22:11:55 UTC by @arcomus for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #98915
- **Report Type:** Smart Contract
- **Report severity:** Medium
- **Target:** https://github.com/immunefi-team/audit-comp-plether/tree/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src
- **Impacts:**
  - Contract fails to deliver promised returns, but doesn't lose value

## Description

## Brief/Intro

When a skew-reducing close earns a VPI rebate, the planner nets the 4 bp execution fee into it and never pays that part to the treasury. The trader still pays the fee; LP pool cash keeps it. A fee netted against a price gain, or on an open with the same rebate, is topped up to the treasury from free cash. At release parameters, the PoC's 400k close pays the treasury 0 instead of 160 USDG.

## Vulnerability Details

`_planCloseActionSettlement` adds VPI, fee, frozen spread and unpaid carry into one action amount (CfdEnginePlanLib.sol:1041-1044). When it's negative, only `actionRebateAssets` is set (:1059-1061), so the fee-eligible base (withheld gain plus collected cash, per the NatSpec at CfdEnginePlanTypes.sol:318) is 0 and `delta.executionFeeAssets` comes out 0 (:1064-1068; in-scope `CfdEngineLens.previewClose` shows the same). The top-up request at :984 is 0 before any free-cash cap, and `_settleProtocolFeeTopUp(host, 0, 0)` returns early (CfdEngineSettlementSidecar.sol:626-628, :913-916). The pool pays the trader |VPI| minus the fee and the fee stays in pool cash for good. A fee withheld from a price gain is eligible, so :984-987 tops it up from free cash, and opens request the full fee (CfdEnginePlanLib.sol:399, Sidecar :498).

```solidity
// CfdEnginePlanLib.sol, _planCloseActionSettlement (excerpt, comments ours)
} else if (actionNetAssets < 0) {                                   // :1059, VPI + fee + spread + carry < 0
    delta.actionRebateAssets = uint256(-actionNetAssets);            // the fee is already netted in here
}
// ... (:1063, netReleasedMarginAssets, unrelated)
uint256 nonClawbackWithheldAssets = delta.actionChargeWithheldAssets - vpiClawbackWithheldAssets;               // 0
uint256 nonVpiReserveCollectedAssets = delta.actionChargeCollectedAssets - delta.vpiRebateReserveConsumedAssets; // 0
uint256 executionFeeEligibleAssets = nonClawbackWithheldAssets + nonVpiReserveCollectedAssets;                  // 0
delta.executionFeeAssets =
    cs.executionFeeAssets < executionFeeEligibleAssets ? cs.executionFeeAssets : executionFeeEligibleAssets;    // 0

// CfdEnginePlanLib.sol:984, back in the caller
uint256 feeTopUpAssets = delta.executionFeeAssets - delta.actionProtocolFeeCreditedAssets;                      // 0 - 0
```

No audit or Known Limitation covers this. Cecuro LOW-26 and MED-6 cap a requested top-up by free cash; here the request is zero with free cash over 38,000 times the fee, and Plether's LOW-26 reply says an open's fee is "funded directly by trade cost plus a top-up capped by post-open settlement-free pool cash". The repo invariant says "A net VPI rebate can offset all or part of the assessed fee; nominal fee arithmetic alone is not treasury revenue" (PerpVpiFrozenAccountingInvariant.t.sol:118-119), yet books the mirror open at the full fee (:94).

## Impact Details

This fits Low, "Contract fails to deliver promised returns, but doesn't lose value". The execution fee is "Protocol service revenue on traded notional" for the "Treasury, only when cash credited" (WHITEPAPER.md:810; open planning at :412-413 says it "belongs to treasury rather than LP assets"), and free-cash-funded top-ups count as treasury margin (ACCOUNTING_SPEC.md:1118). Value is conserved (pool, trader and treasury deltas sum to 0 units on all 22 PoC actions).

Measured in the PoC at release parameters with a 10M Junior pool: B's 400k LONG close (skew 3.9M to 3.5M) pays the treasury 0, while C's 400k SHORT open from the same snapshot, with the same skew change and |VPI| (1,478.87), pays it 160.00 via a pool top-up. Both traders net +1,318.87. A VPI-neutral 400k close with a 2,000 gain moves cash the same way: the pool pays 1,999.99, split 1,840.79 to the trader and 159.20 to the treasury. B's close pays 1,318.87, all to B; run after that price move, it still credits the treasury 0. At |VPI| 119.99 the close credits 40.00 and the mirror open 160.00.

Generalizing the measured cases from the code, a non-frozen close loses the whole fee once its rebate exceeds fee plus unpaid carry, which at release parameters takes skew averaging about 4% of depth over the close. The rebate is capped at the VPI the position has paid, so this hits positions opened on the crowded side and closed while it's still crowded.

## Recommendation

Request the rebate-netted part of the fee as a top-up, as opens do; :985-987 still caps it by free cash. Patched, the PoC's main close credits the treasury 160.00 and draws 1,478.87 from the pool, like the mirror open; trader deltas, opens and controls are unchanged.

```diff
--- a/packages/perps/src/libraries/CfdEnginePlanLib.sol
+++ b/packages/perps/src/libraries/CfdEnginePlanLib.sol
@@ -981,7 +981,12 @@
         delta.freshPayoutIsImmediate = !delta.pricePayoutCreatesClaim;
         delta.freshPayoutCreatesClaim = delta.pricePayoutCreatesClaim;
 
-        uint256 feeTopUpAssets = delta.executionFeeAssets - delta.actionProtocolFeeCreditedAssets;
+        // The part of the fee netted against a VPI rebate stays in the pool as a smaller rebate payout, like a fee
+        // withheld from a price gain. Request it as a free-cash top-up too, as the open path does.
+        uint256 feeTopUpAssets = delta.executionFeeAssets - delta.actionProtocolFeeCreditedAssets
+            + (cs.executionFeeAssets > delta.actionChargeAssessedAssets
+                ? cs.executionFeeAssets - delta.actionChargeAssessedAssets
+                : 0);
         freePoolCashAssets =
             _freePoolCashForProtocolFee(snap, delta, poolCashAfterCollectionsAssets, claimsAfterPriceNettingAssets);
         delta.protocolFeeTopUpAssets = feeTopUpAssets < freePoolCashAssets ? feeTopUpAssets : freePoolCashAssets;
```

## References

- https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/libraries/CfdEnginePlanLib.sol#L1036-L1074
- https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/libraries/CfdEnginePlanLib.sol#L984-L987
- https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/libraries/CfdEnginePlanLib.sol#L399
- https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/CfdEngineSettlementSidecar.sol#L615-L628
- https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/CfdEngineSettlementSidecar.sol#L905-L926
- https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/CfdEngineSettlementSidecar.sol#L498
- https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/CfdEngineLens.sol#L399
- https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/CfdEnginePlanTypes.sol#L318
- https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/test/perps/invariant/properties/PerpVpiFrozenAccountingInvariant.t.sol#L94
- https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/test/perps/invariant/properties/PerpVpiFrozenAccountingInvariant.t.sol#L118-L124
- https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/ACCOUNTING_SPEC.md#L781-L782
- https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/ACCOUNTING_SPEC.md#L1118-L1119
- https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/WHITEPAPER.md#L810
- https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/WHITEPAPER.md#L411-L413
- https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/DEPLOYMENT.md#L142

## Link to Proof of Concept

https://gist.github.com/antonio-rosendo/43ce5fc28489eddc3f4f8ed0382d363d

## Proof of Concept

## Proof of Concept

Runs in the repo's Foundry suite at the frozen commit, no in-scope contract modified. `setUp` installs and asserts the DEPLOYMENT.md release values (the owner's 72 h timelocked config); then traders commit through Router V3 and execute their own orders. The router is the repo's `LegacyOrderRouterHarness` (production `OrderRouter` plus an unused overload); only USDG and Pyth are mocked. `FOUNDRY_VIA_IR=false` keeps the compile short; the repo default is via-IR. Secret gist: https://gist.github.com/antonio-rosendo/43ce5fc28489eddc3f4f8ed0382d363d

```
git clone https://github.com/immunefi-team/audit-comp-plether && cd audit-comp-plether
git checkout a82d4fefb02768e5b3bddcc97dee371eee24fea8 && git submodule update --init --recursive
export GIST=/absolute/path/to/this/gist
mkdir -p packages/perps/test/perps/poc/round3 && cp "$GIST"/CloseFeeNetting.t.sol packages/perps/test/perps/poc/round3/
cd packages/perps
FOUNDRY_VIA_IR=false forge test --match-path test/perps/poc/round3/CloseFeeNetting.t.sol -vv
git apply "$GIST/R3-3_fix.diff"   # then re-run the forge test: 1 passes (the charge/gain controls test), 5 fail because their treasury asserts now see the full fee
```

Main test output (excerpt):

```
--- BRANCH 1  B closes LONG 400k (skew 3.9M -> 3.5M) ---
  price 100000000  notional fee 160.00  depth 10007606.09  free cash 6097856.09
  delta: treasury 0.00  pool cash -1318.87  pool.totalAssets -1318.87
  delta: trader 1318.87  junior preview -1318.87
  ActionRebateSettled assessed 1318.87 paid 1318.87 waived 0.00
  CfdEngineLens.previewClose before execution: valid true  executionFeeAssets 0  vpiDeltaAssets -1478875154
--- BRANCH 2  C opens SHORT 400k (skew 3.9M -> 3.5M), mirror ---
  delta: treasury 160.00  pool cash -1478.87  pool.totalAssets -1478.87
  delta: trader 1318.87  junior preview -1478.87
Suite result: ok. 6 passed; 0 failed; 0 skipped
```

Key test (CloseFeeNetting.t.sol lines 363-371, 396-403, 418-428):

```solidity
    function test_R3_3_RebateClose_vs_MirrorOpen() public {
        Act memory b0 = _mainState(3_500_000e18, "MAIN: Junior 10M, A LONG 3.5M, B LONG 400k (skew 3.9M LONG)");

        uint256 snap = vm.snapshotState();
        Act memory cl = _act(T_B, CfdTypes.Side.LONG, SIZE_B, 0, true);
        _log("BRANCH 1  B closes LONG 400k (skew 3.9M -> 3.5M)", cl);
        vm.revertToState(snap);
        Act memory op = _act(T_C, CfdTypes.Side.SHORT, SIZE_B, 20_000e6, false);
        _log("BRANCH 2  C opens SHORT 400k (skew 3.9M -> 3.5M), mirror", op);
        // ...
        // (a) rebate close: fee netted, treasury gets nothing, although free cash is ample
        assertGe(cl.freeCashBefore, 100 * cl.fee, "free settlement cash >= 100 x fee before the close (not LOW-26)");
        // the planner itself marks no part of the fee as treasury fee (in-scope CfdEngineLens, read before execution)
        assertTrue(cl.previewValid, "lens previewClose valid");
        assertEq(cl.previewExecFee, 0, "LENS: previewClose.executionFeeAssets == 0 on the rebate close");
        assertTrue(cl.rebateEvent, "ActionRebateSettled emitted");
        assertFalse(cl.chargeEvent, "no action charge on the close");
        assertEq(cl.dTreasury, 0, "CLOSE: treasury delta == 0");
        // ...
        // (b) mirror open: full fee credited via a pool-funded top-up
        assertGe(op.freeCashBefore, 100 * op.fee, "same ample free cash before the open");
        assertEq(op.dTreasury, int256(op.fee), "OPEN: treasury delta == fee");
        assertEq(-op.dPoolTokens, int256(absVpi), "OPEN: pool outflow == |VPI| (rebate |VPI| - fee + fee top-up)");
        assertEq(op.dTrader, int256(absVpi) - int256(op.fee), "OPEN: trader receives |VPI| - fee");
        assertEq(op.sum, 0, "conservation on the open");

        // Asymmetry: same trader result, the fee moves from treasury to pool cash
        assertApproxEqAbs(cl.dTrader, op.dTrader, 1e4, "trader is unaffected by the difference");
        assertApproxEqAbs(cl.dPoolTokens - op.dPoolTokens, int256(op.fee), 1e4, "pool keeps exactly the fee");
        assertEq(op.dTreasury - cl.dTreasury, int256(cl.fee), "treasury misses exactly the fee");
```

Gain-close control, `test_R3_3_Controls_ChargeCloseAndGainClose` lines 605-608:

```solidity
        // same lens, same planner: the fee netted against the gain is the treasury fee in full
        assertTrue(gn.previewValid, "(ii) lens previewClose valid");
        assertEq(gn.previewExecFee, gn.fee, "(ii) LENS: previewClose.executionFeeAssets == fee on the gain close");
        assertEq(gn.dTreasury, int256(gn.fee), "(ii) GAIN CLOSE: treasury == fee (pool-funded top-up)");
```
