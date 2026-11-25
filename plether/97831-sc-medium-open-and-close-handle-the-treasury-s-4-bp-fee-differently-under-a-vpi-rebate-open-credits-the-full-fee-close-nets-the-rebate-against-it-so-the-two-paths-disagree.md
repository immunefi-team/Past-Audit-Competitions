# \#97831 [SC-Medium] Open and close handle the treasury's 4 bp fee differently under a VPI rebate: open credits the full fee, close nets the rebate against it, so the two paths disagree

**Submitted on Nov 3rd 2026 at 13:31:59 UTC by @Web3Vinay for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #97831
- **Report Type:** Smart Contract
- **Report severity:** Medium
- **Target:** https://github.com/immunefi-team/audit-comp-plether/tree/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src
- **Impacts:**
  - Contract fails to deliver promised returns, but doesn't lose value

## Description

### Brief/Intro

A voluntary close that reduces skew earns a negative VPI. `_planCloseActionSettlement` folds that negative VPI into one signed number together with the execution fee, spread and carry, and then caps the treasury fee at what the trader actually paid net. Nothing tops the fee back up afterwards. The open path does the opposite: it takes the full fee and funds the part the trader did not pay from pool free cash. So the same rebate costs the pool on open and costs the treasury on close.

The trigger is not limited to `rebate > fee`. Whenever a skew-reducing close earns any rebate, the treasury is credited `fee − rebate` (or 0 if `rebate ≥ fee`).

### Plain Summary (and where the uncertainty is)

1. Every trade pays a small fee (4 bp) that should go to the treasury.
2. Some trades also earn a discount (the VPI rebate) for making the book more balanced.
3. **On open:** the treasury gets the full fee. The discount is paid by the pool.
4. **On close:** the discount is taken out of the fee. In the PoC the fee is 120 USDG, the discount is about 90, and the treasury gets only about 30.
5. So the two paths do the same thing differently: on open the pool pays for the discount, on close the treasury does.

### Vulnerability Details

Close path nets the rebate against the fee:

```solidity
// CfdEnginePlanLib.sol:1041-1044
@>  int256 actionNetAssets = cs.vpiDeltaAssets
        + int256(
            cs.executionFeeAssets + cs.frozenSpreadAssets + delta.pendingCarryAssets - delta.realizedCarryAssets
        );
```

The fee that reaches the treasury is capped by the net charge actually withheld or collected:

```solidity
// CfdEnginePlanLib.sol:1062-1065
    uint256 executionFeeEligibleAssets = nonClawbackWithheldAssets + nonVpiReserveCollectedAssets;
@>  delta.executionFeeAssets =
        cs.executionFeeAssets < executionFeeEligibleAssets ? cs.executionFeeAssets : executionFeeEligibleAssets;
```

The close top-up only covers `executionFeeAssets − actionProtocolFeeCreditedAssets`, i.e. the already-capped number, so the netted part is never restored:

```solidity
// CfdEnginePlanLib.sol:984-987
@>  uint256 feeTopUpAssets = delta.executionFeeAssets - delta.actionProtocolFeeCreditedAssets;
    freePoolCashAssets =
        _freePoolCashForProtocolFee(snap, delta, poolCashAfterCollectionsAssets, claimsAfterPriceNettingAssets);
    delta.protocolFeeTopUpAssets = feeTopUpAssets < freePoolCashAssets ? feeTopUpAssets : freePoolCashAssets;
```

The open path does not net. `tradeCostAssets = vpiAssets + executionFeeAssets` (`OpenAccountingLib.sol:135`), but the shortfall against the full fee is topped up from pool free cash:

```solidity
// CfdEnginePlanLib.sol:719-748
    uint256 directlyCreditedAssets =
        delta.tradeCostAssets > 0 ? Math.min(uint256(delta.tradeCostAssets), delta.executionFeeAssets) : 0;
    ...
@>  uint256 feeShortfallAssets = delta.executionFeeAssets - directlyCreditedAssets;
    uint256 topUpAssets = feeShortfallAssets == 0 ? 0 : Math.min(feeShortfallAssets, _freePoolCashAfterLiabilityReserves(...));
```

The spec says closes pay the fee and only cash-collected fees and free-cash-funded top-ups become treasury margin:

> "voluntary closes pay the 4-basis-point treasury execution fee" (`WHITEPAPER.md:1455`)
>
> "only cash-collected execution or liquidation fees and free-cash-funded top-ups become treasury margin" (`ACCOUNTING_SPEC.md:1118`)

### Impact Details

No user or LP funds are lost and no attacker profits. The invariant that does not hold: **on a voluntary close the treasury receives the assessed 4 bp execution fee whenever cash or free pool cash can fund it.**

- PoC ($300k long, full close, price unchanged): fee 120.000000 USDG, rebate 89.962396 USDG, treasury credited 30.037604 USDG. 89.962396 USDG (75 %) is not routed. On open, the same PoC credits the full 40 USDG fee to the treasury.
- Bound per close: `min(rebate, 4 bp × notional)`. At the deployed release parameters (`vpiFactor 0.01e18`, `maxSkewRatio 0.4e18`) on a 5M USDG pool, a $2M skew-reducing close earns a rebate of about 4,000 USDG, above the 800 USDG fee, so the whole fee is not credited.
- Frequency: in a 3-actor, 40-step random open/close walk (512 runs), 824.95 of 2,120.93 USDG assessed close fees reached the treasury (61 % not routed). An earlier harness run showed 382.09 of 2,470.12 USDG (84.5 % not routed). This comes from my fuzz harness, not from the standalone PoC below.
- Conservation held in every run: Δtrader + Δtreasury + Δ(pool − claims) + Δkeeper = 0. The shortfall never exceeded `min(rebate, fee)`.

The effect is reduced treasury revenue on every skew-reducing close, not a loss of principal. Suggested label: **Contract fails to deliver promised returns, but doesn't lose value**

### Mitigation

Make open and close behave the same: either fix close to credit the full fee (as open does), or change open to net the rebate against the fee (as close does) and document it.

Fix for close: top up the full assessed fee from pool free cash, as the open path does, so the rebate stays an LP-funded cost:

```diff
-        uint256 feeTopUpAssets = delta.executionFeeAssets - delta.actionProtocolFeeCreditedAssets;
+        // top up against the assessed fee (cs.executionFeeAssets), not the net-capped fee
+        uint256 feeTopUpAssets = cs.executionFeeAssets - delta.actionProtocolFeeCreditedAssets;
```

### References

- `packages/perps/src/libraries/CfdEnginePlanLib.sol:1041-1044`: close `actionNetAssets` nets VPI against the fee
- `packages/perps/src/libraries/CfdEnginePlanLib.sol:1062-1065`: treasury fee capped at the net charge
- `packages/perps/src/libraries/CfdEnginePlanLib.sol:984-987`: close top-up only covers the capped fee
- `packages/perps/src/libraries/CfdEnginePlanLib.sol:719-748`: open path tops the fee up from pool free cash
- `packages/perps/src/libraries/OpenAccountingLib.sol:135`: `tradeCostAssets = vpiAssets + executionFeeAssets`
- `packages/perps/WHITEPAPER.md:1455`: "voluntary closes pay the 4-basis-point treasury execution fee"
- `packages/perps/WHITEPAPER.md:811, 1301`: "only when cash credited"; "recovered action value is attributed to execution fee" (supports the intended-design reading)
- `packages/perps/ACCOUNTING_SPEC.md:781-782, 1118-1119`: top-up priority and treasury-margin rule


## Proof of Concept

## Proof of Concept

**Conditions under which this is valuable**
- A position is closed on the skew-reducing side, so the close earns a negative VPI. No special role, price move or timing is needed.
- Run on the repo's `BasePerpTest` fixture at the deployed release parameters. A mainnet fork is not needed: the behaviour is deterministic engine accounting.

Save as `packages/perps/test/lead2/RebateCloseFeeRouting.t.sol` and run `forge test --root packages/perps --offline --match-path test/lead2/RebateCloseFeeRouting.t.sol -vv`.

```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

import {BasePerpTest} from "../perps/BasePerpTest.sol";
import {CfdTypes} from "@plether/perps/CfdTypes.sol";

/// Deployed-release risk params (script/DeployPerpsArbitrumSepolia.s.sol:435-446), executionFeeBps = 4.
contract RebateCloseFeeRoutingPoC is BasePerpTest {
    uint256 internal constant Q = 1e20; // 1 lot = 100 USD notional at price 1e8
    address internal constant A = address(0xA11CE);
    address internal constant B = address(0xB0B);

    function _riskParams() internal pure override returns (CfdTypes.RiskParams memory) {
        return CfdTypes.RiskParams({
            vpiFactor: 0.01e18, maxSkewRatio: 0.4e18, maintMarginBps: 10, initMarginBps: 20, fadMarginBps: 300,
            baseCarryBps: 500, minBountyAssets: 1e6, bountyBps: 10, keeperShareBps: 5000, protocolShareBps: 0
        });
    }

    function _initialJuniorDeposit() internal pure override returns (uint256) {
        return 5_000_000e6;
    }

    function _ord(address who, uint256 lots, uint256 margin, CfdTypes.Side side, bool isClose)
        internal
        view
        returns (CfdTypes.Order memory)
    {
        return CfdTypes.Order({
            account: who, sizeDelta: lots * Q, marginDelta: margin, targetPrice: 0,
            commitTime: uint64(vm.getBlockTimestamp()), commitBlock: uint64(vm.getBlockNumber()),
            orderId: 0, side: side, isClose: isClose
        });
    }

    function _exec(CfdTypes.Order memory o) internal {
        vm.warp(vm.getBlockTimestamp() + 1);
        vm.roll(vm.getBlockNumber() + 1);
        uint256 depth = pool.totalAssets();
        vm.prank(address(router));
        engine.processOrderTyped(o, 1e8, depth, uint64(vm.getBlockTimestamp()));
    }

    function test_CloseNetsVpiRebateAgainstTreasuryFee() public {
        _fundTrader(A, 4_000_000e6);
        _fundTrader(B, 4_000_000e6);
        address treasury = engine.protocolTreasury();

        // ---- Control: OPEN on the skew-reducing side. Rebate is paid, treasury still gets the FULL 4 bp fee.
        _exec(_ord(A, 3000, 300_000e6, CfdTypes.Side.LONG, false)); // builds long skew
        uint256 t0 = clearinghouse.balanceAssets(treasury);
        _exec(_ord(B, 1000, 100_000e6, CfdTypes.Side.SHORT, false)); // short into long skew => VPI rebate
        uint256 openFeeAssessed = (1000 * 100e6 * 4) / 10_000; // 4 bp of $100k = 40 USDG
        uint256 openFeeCredited = clearinghouse.balanceAssets(treasury) - t0;
        emit log_named_uint("OPEN  fee assessed (6dp)", openFeeAssessed);
        emit log_named_uint("OPEN  fee credited to treasury (6dp)", openFeeCredited);

        // ---- Close B, then close A. Closing A removes the long skew => VPI rebate.
        _exec(_ord(B, 1000, 0, CfdTypes.Side.SHORT, true));
        t0 = clearinghouse.balanceAssets(treasury);
        uint256 poolBefore = pool.totalAssets();
        _exec(_ord(A, 3000, 0, CfdTypes.Side.LONG, true)); // full close, same price 1e8
        uint256 closeFeeAssessed = (3000 * 100e6 * 4) / 10_000; // 4 bp of $300k = 120 USDG
        uint256 closeFeeCredited = clearinghouse.balanceAssets(treasury) - t0;
        emit log_named_uint("CLOSE fee assessed (6dp)", closeFeeAssessed);
        emit log_named_uint("CLOSE fee credited to treasury (6dp)", closeFeeCredited);
        emit log_named_uint("CLOSE fee not routed to treasury (6dp)", closeFeeAssessed - closeFeeCredited);
        emit log_named_int("pool totalAssets delta over close", int256(pool.totalAssets()) - int256(poolBefore));

        assertEq(openFeeCredited, openFeeAssessed, "control: open routes full fee to treasury");
        // Spec: "voluntary closes pay the 4-basis-point treasury execution fee" (WHITEPAPER.md:1455).
        // Observed: the close credits LESS than the assessed fee, while the open path credited it in full.
        assertLt(closeFeeCredited, closeFeeAssessed, "close credits less than the assessed fee");
        assertGt(closeFeeAssessed - closeFeeCredited, (closeFeeAssessed * 7) / 10, "> 70% of the fee not routed");
    }
}
```

Result (actually run, `forge test --root packages/perps --offline --match-path test/lead2/RebateCloseFeeRouting.t.sol -vv`):

```
Ran 1 test for test/lead2/RebateCloseFeeRouting.t.sol:RebateCloseFeeRoutingPoC
[PASS] test_CloseNetsVpiRebateAgainstTreasuryFee() (gas: 2195949)
Logs:
  OPEN  fee assessed (6dp): 40000000
  OPEN  fee credited to treasury (6dp): 40000000
  CLOSE fee assessed (6dp): 120000000
  CLOSE fee credited to treasury (6dp): 30037604
  CLOSE fee not routed to treasury (6dp): 89962396
  pool totalAssets delta over close: 0

Suite result: ok. 1 passed; 0 failed; 0 skipped
```

Reading the output: the open credits 40 of 40 USDG, the close credits 30.04 of 120 USDG. Asserting the spec reading (`closeFeeCredited == closeFeeAssessed`) fails with `30037604 != 120000000`.

