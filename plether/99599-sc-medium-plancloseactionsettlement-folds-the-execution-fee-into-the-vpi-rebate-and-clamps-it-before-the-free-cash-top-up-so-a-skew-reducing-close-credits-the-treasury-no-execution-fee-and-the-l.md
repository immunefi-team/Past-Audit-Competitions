# \#99599 [SC-Medium]  _planCloseActionSettlement folds the execution fee into the VPI rebate and clamps it before the free-cash top-up, so a skew-reducing close credits the treasury no execution fee and the LP pool keeps it

**Submitted on Nov 14th 2026 at 12:33:38 UTC by @maigadoh for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #99599
- **Report Type:** Smart Contract
- **Report severity:** Medium
- **Target:** https://github.com/immunefi-team/audit-comp-plether/tree/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src
- **Impacts:**
  - Contract fails to deliver promised returns, but doesn't lose value

## Description

## Summary
The whitepaper states that on a trade "the protocol execution fee belongs to treasury rather than LP assets" ([WHITEPAPER.md#L412-L413](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/WHITEPAPER.md#L412-L413)) and "voluntary closes pay the 4-basis-point treasury execution fee" ([WHITEPAPER.md#L1455](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/WHITEPAPER.md#L1455)); however, on a skew-reducing close the close planner folds the fee into one signed net with the VPI rebate and then clamps it to collected value, so the fee is never credited and never topped up from free pool cash, and the LP pool keeps it instead of the treasury.

## Root Cause
`_planCloseActionSettlement` sums the VPI delta and the fee into one `actionNetAssets`. When the close reduces skew, `cs.vpiDeltaAssets` is a rebate large enough to make the net negative, so the whole action becomes a rebate and nothing is withheld or collected. The clamp below caps the fee the planner carries forward at the net action charge, so the treasury loses `min(VPI rebate, fee)` on any skew-reducing close: when the rebate only partly offsets the fee the net stays a positive charge of `fee - rebate`, the eligible amount is bounded by that charge, so the fee credits to at most `fee - rebate` and the top-up is defeated too; the full-offset case below, which the PoC proves, is the extreme where the net turns negative and the whole fee is lost.

Link: [CfdEnginePlanLib.sol#L1041-L1061](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/libraries/CfdEnginePlanLib.sol#L1041-L1061)
```solidity
int256 actionNetAssets = cs.vpiDeltaAssets
    + int256(cs.executionFeeAssets + cs.frozenSpreadAssets + delta.pendingCarryAssets - delta.realizedCarryAssets);
if (actionNetAssets > 0) {
    ...
} else if (actionNetAssets < 0) {
    delta.actionRebateAssets = uint256(-actionNetAssets); // the fee is netted inside; nothing records it was borne
}
```

The fee is then clamped to withheld + collected value. With a full offset both are zero, so the fee the planner carries forward is zero.

Link: [CfdEnginePlanLib.sol#L1063-L1068](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/libraries/CfdEnginePlanLib.sol#L1063-L1068)
```solidity
uint256 nonClawbackWithheldAssets = delta.actionChargeWithheldAssets - vpiClawbackWithheldAssets;        // 0
uint256 nonVpiReserveCollectedAssets = delta.actionChargeCollectedAssets - delta.vpiRebateReserveConsumedAssets; // 0
uint256 executionFeeEligibleAssets = nonClawbackWithheldAssets + nonVpiReserveCollectedAssets;            // 0
delta.executionFeeAssets =
    cs.executionFeeAssets < executionFeeEligibleAssets ? cs.executionFeeAssets : executionFeeEligibleAssets; // 0
```

Because the clamped fee is zero, the free-cash top-up it feeds is zero, so the settlement credits the treasury nothing.

Link: [CfdEnginePlanLib.sol#L984-L987](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/libraries/CfdEnginePlanLib.sol#L984-L987) and the close settlement [CfdEngineSettlementSidecar.sol#L626-L628](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/CfdEngineSettlementSidecar.sol#L626-L628)
```solidity
uint256 feeTopUpAssets = delta.executionFeeAssets - delta.actionProtocolFeeCreditedAssets; // 0 - 0
...
_settleProtocolFeeTopUp(host, protocolFeeCreditedAssets + delta.protocolFeeTopUpAssets, protocolFeeCreditedAssets); // 0
```

The trader still bears the fee: the close kernel computes `netSettlementAssets = realizedPnl - vpiDelta - (executionFee + frozenSpread)` ([CloseAccountingLib.sol#L125-L126](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/libraries/CloseAccountingLib.sol#L125-L126)), so the fee is subtracted from the rebate the pool pays out. The pool keeps that value; "the protocol execution fee belongs to treasury rather than LP assets" is violated.

This is not a documented waiver. The sibling open path passes the full fee and tops the identical offset up from free pool cash, so the same offset reaches the treasury on open ([CfdEngineSettlementSidecar.sol#L498](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/CfdEngineSettlementSidecar.sol#L498)), and the whitepaper lists "free-cash-funded top-ups" as a treasury-margin source ([ACCOUNTING_SPEC.md#L1118](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/ACCOUNTING_SPEC.md#L1118)). The common top-up helper only ever pays from cash above every pool and engine reserve, so a close top-up cannot consume trader claims or LP withdrawal reserves ([CfdEngineSettlementSidecar.sol#L905-L926](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/CfdEngineSettlementSidecar.sol#L905-L926)).

## Attack Path
Preconditions (release parameters, [DEPLOYMENT.md#L131](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/DEPLOYMENT.md#L131), [#L142](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/DEPLOYMENT.md#L142)):
- `vpiFactor = 0.01e18`, `executionFeeBps = 4`, a pool with healthy free cash, and a solvent position closing in the skew-reducing direction so `cs.vpiDeltaAssets` is a rebate. The rebate fully offsets the fee once `notional >= 2 * feeBps * depth / vpiFactor` (~8% of pool depth closed into a flat book). That threshold is the worst case: skew cost is quadratic, `vpiFactor * skew^2 / (2 * depth)` ([CfdMath.sol#L118-L140](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/CfdMath.sol#L118-L140)), so when the book already carries skew the rebate per unit notional is larger and ordinary-sized closes suffice; near the `maxSkewRatio = 0.4e18` wall the rebate exceeds the fee for essentially any close size, so the loss lands on routine closes rather than only on one large trade. No attacker, privileged actor, special state or timing is required.

1. A trader opens a position that increases skew. VPI is a positive charge, the trade cost funds the full fee, and the treasury is credited the full 4 bps.
2. The trader closes at the same price. The close reduces skew, so `cs.vpiDeltaAssets` is negative; the lifetime clamp allows it up to the VPI accrued on open ([CloseAccountingLib.sol#L115-L117](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/libraries/CloseAccountingLib.sol#L115-L117)).
3. `actionNetAssets = vpiDelta + fee < 0`, so the action is a rebate, `executionFeeEligibleAssets` and the fee clamp to zero, and the free-cash top-up is never attempted. The treasury is credited nothing; the trader's rebate is reduced by exactly the fee, which the LP pool retains.

That the divergence is caused by the VPI offset, not by any terminal or collateral rule, is shown by the control: with `vpiFactor = 0` the identical close has no rebate, `actionNetAssets = fee > 0`, and the treasury is credited the full 40.000000.

## Impact
The treasury is denied `min(VPI rebate, fee)` on every solvent skew-reducing close that carries a VPI rebate — the entire fee once the rebate reaches it, the rebate portion otherwise — permanently and with no admin recovery (uncredited fee is "not withdrawable protocol inventory", [ACCOUNTING_SPEC.md#L1119](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/ACCOUNTING_SPEC.md#L1119)). The PoC forgoes the full 40.000000 USDG on a single $100k close while $901,799 of free pool cash could have funded it; because the offset grows with existing skew, the leak lands on ordinary-sized closes against a skewed book, and the loss scales with closed notional and trading volume. No user principal is lost and no actor profits: the fee is retained by the LP pool instead of reaching the treasury.

## Proof of Concept

## Proof of Concept
Add the two methods below to the existing `test/perps/accounting/fees/EngineProtocolFees.t.sol`. That file extends the protocol's own `CfdEngineTestBase` harness (real `CfdEngine`, `MarginClearinghouse`, `HousePool`, planner and settlement sidecar); only the risk params are set to the `DEPLOYMENT.md` release values, through the engine's ordinary governance path. Then run:

```bash
forge test --root packages/perps --match-path "test/perps/accounting/fees/EngineProtocolFees.t.sol" --match-test "test_PoC_CloseVpiRebateOffsetDeniesTreasuryFee|test_PoC_Control_NoVpiOffsetCreditsFullCloseFee" -vvvv
```

```solidity
    /// Release config (DEPLOYMENT.md): vpiFactor 0.01e18, maint 10bps, init 20bps, executionFee 4bps.
    /// A LONG opened into a flat book accrues positive VPI and pays treasury the full open fee.
    /// Closing it reduces skew, so close VPI is a rebate that nets the fee away: the planner zeroes
    /// the close execution fee and never attempts the free-cash top-up, so treasury is credited 0
    /// while free pool cash could have funded it. The identical fee is credited in full on open.
    function test_PoC_CloseVpiRebateOffsetDeniesTreasuryFee() public {
        CfdTypes.RiskParams memory params = _riskParams();
        params.vpiFactor = 0.01e18;
        params.maxSkewRatio = 0.4e18;
        params.maintMarginBps = 10;
        params.initMarginBps = 20;
        _setRiskParams(params);

        address trader = address(0xBEEF);
        _fundTrader(trader, 11_000e6);

        uint256 size = 100_000e18; // $100k notional at $1.00, ~10% of the ~1.002M pool depth
        uint256 price = 1e8; // $1.00, unchanged on entry and exit: no price PnL
        uint256 expectedFee = _engineExecutionFeeAssets(size, price); // 4bps of $100k = $40

        address treasury = engine.protocolTreasury();

        uint256 treasuryBeforeOpen = clearinghouse.balanceAssets(treasury);
        _open(trader, CfdTypes.Side.LONG, size, 5_000e6, price);
        uint256 openFeeCredited = clearinghouse.balanceAssets(treasury) - treasuryBeforeOpen;

        CfdEngine.ClosePreview memory preview = engineLens.previewClose(trader, size, price);
        uint256 freeCashBeforeClose = pool.getSettlementFreeAssets();

        uint256 treasuryBeforeClose = clearinghouse.balanceAssets(treasury);
        _close(trader, CfdTypes.Side.LONG, size, price);
        uint256 closeFeeCredited = clearinghouse.balanceAssets(treasury) - treasuryBeforeClose;

        emit log_named_decimal_uint("whitepaper 4bps close fee owed to treasury", expectedFee, 6);
        emit log_named_decimal_uint("OPEN : fee credited to treasury          ", openFeeCredited, 6);
        emit log_named_int("CLOSE: previewed vpiDelta (<0 = rebate)   ", preview.vpiDeltaAssets);
        emit log_named_decimal_uint("CLOSE: planner executionFee (clamped)     ", preview.executionFeeAssets, 6);
        emit log_named_decimal_uint("CLOSE: free pool cash for top-up          ", freeCashBeforeClose, 6);
        emit log_named_decimal_uint("CLOSE: fee credited to treasury           ", closeFeeCredited, 6);

        assertEq(openFeeCredited, expectedFee, "open credits the full 4bps to treasury");
        assertLt(preview.vpiDeltaAssets, 0, "close VPI is a rebate (skew-reducing)");
        assertGt(freeCashBeforeClose, expectedFee, "free pool cash could fund the close fee top-up");
        assertEq(preview.executionFeeAssets, 0, "planner zeroes the close execution fee under full offset");
        assertEq(closeFeeCredited, 0, "treasury is credited nothing on the close despite an affordable top-up");
    }

    /// Control: with vpiFactor = 0 the identical close has no VPI rebate to net the fee against, so the
    /// treasury is credited the full 4bps on the close. This isolates the VPI-rebate offset as the cause.
    function test_PoC_Control_NoVpiOffsetCreditsFullCloseFee() public {
        CfdTypes.RiskParams memory params = _riskParams();
        params.vpiFactor = 0;
        params.maxSkewRatio = 0.4e18;
        params.maintMarginBps = 10;
        params.initMarginBps = 20;
        _setRiskParams(params);

        address trader = address(0xBEEF);
        _fundTrader(trader, 11_000e6);

        uint256 size = 100_000e18;
        uint256 price = 1e8;
        uint256 expectedFee = _engineExecutionFeeAssets(size, price);
        address treasury = engine.protocolTreasury();

        _open(trader, CfdTypes.Side.LONG, size, 5_000e6, price);

        CfdEngine.ClosePreview memory preview = engineLens.previewClose(trader, size, price);
        uint256 treasuryBeforeClose = clearinghouse.balanceAssets(treasury);
        _close(trader, CfdTypes.Side.LONG, size, price);
        uint256 closeFeeCredited = clearinghouse.balanceAssets(treasury) - treasuryBeforeClose;

        emit log_named_int("CONTROL: close vpiDelta (0 = no offset)", preview.vpiDeltaAssets);
        emit log_named_decimal_uint("CONTROL: fee credited to treasury on close", closeFeeCredited, 6);

        assertEq(preview.vpiDeltaAssets, 0, "no VPI with vpiFactor 0");
        assertEq(closeFeeCredited, expectedFee, "with no VPI offset the close credits the full fee to treasury");
    }
```

Output:

```bash
[PASS] test_PoC_CloseVpiRebateOffsetDeniesTreasuryFee()
  whitepaper 4bps close fee owed to treasury: 40.000000
  OPEN : fee credited to treasury          : 40.000000
  CLOSE: previewed vpiDelta (<0 = rebate)   : -49897714
  CLOSE: planner executionFee (clamped)     : 0.000000
  CLOSE: free pool cash for top-up          : 901799.900199
  CLOSE: fee credited to treasury           : 0.000000
[PASS] test_PoC_Control_NoVpiOffsetCreditsFullCloseFee()
  CONTROL: close vpiDelta (0 = no offset): 0
  CONTROL: fee credited to treasury on close: 40.000000
```

The `-vvvv` trace shows the settled `CloseState` carrying the real fee `executionFeeAssets: 40000000` and a rebate `vpiDeltaAssets: -49897714`; `actionNetAssets = -49.897714 + 40 < 0` routes to the rebate branch, so `actionRebateAssets: 9897714` — the trader's VPI reclaim is cut from 49.897714 to 9.897714, i.e. they bear exactly the 40.000000 fee. In the same `CloseDelta` the planner has set `executionFeeAssets: 0`, `actionProtocolFeeCreditedAssets: 0` and `protocolFeeTopUpAssets: 0`, and the only `HousePool::payOut` is the 9897714 rebate to the trader — there is no payout to the treasury, whose balance is unchanged at the 40000000 open fee, although `getSettlementFreeAssets()` reports $901,799 of free cash the open path would have drawn on. The control run, with `vpiDeltaAssets: 0`, collects the charge (`actionChargeCollectedAssets: 40000000`) and credits `actionProtocolFeeCreditedAssets: 40000000`, so the same close pays the treasury the full 40.000000.

