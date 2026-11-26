# \#99962 [SC-Insight] CfdEngineLens.previewClose reports the execution fee net of the VPI rebate, contradicting its documented field and overstating close proceeds

**Submitted on Nov 15th 2026 at 22:28:47 UTC by @v4rvl for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #99962
- **Report Type:** Smart Contract
- **Report severity:** Insight
- **Target:** https://github.com/immunefi-team/audit-comp-plether/tree/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src
- **Impacts:**
  - Smart contract unable to operate due to lack of token funds
  - Contract fails to deliver promised returns, but doesn't lose value

## Description

## Summary

`CfdEngineLens.previewClose` returns a `ClosePreview` whose `executionFeeAssets` field is documented in `ICfdEngineTypes` as the "Protocol execution fee assessed on the close notional". When the close earns a VPI rebate (any skew-reducing close), the lens instead returns the fee after the planner has netted the rebate against it. The same struct also reports that rebate separately in `vpiDeltaAssets`. A client that computes the trader's net close result as `realizedPnlAssets - vpiDeltaAssets - executionFeeAssets` counts the rebate twice. This is the identity the engine's own close kernel uses with the nominal fee. The open preview's field has the same wording and returns the nominal fee.

Measured on the repository's own test fixture with the deployment fee (4 bps) and VPI factor (0.01), at a constant price:

- 100-lot close (10,000 USDG notional), rebate smaller than the fee: nominal fee 4,000,000 atoms, rebate 998,003. The field returns 3,001,997, and the documented formula gives -2,003,994. The trader's balance actually changes by -3,001,997, so the preview overstates the proceeds by 998,003 atoms (the rebate):
  -2,003,994 - (-3,001,997) = 998,003
- 1,000-lot full close (100,000 USDG notional), rebate larger than the fee: nominal fee 40,000,000, rebate 49,897,714. The field returns 0, and the documented formula gives +49,897,714. The trader actually receives +9,897,694, so the preview overstates the proceeds by 40,000,020 atoms (the whole fee plus 20 atoms of carry):
  49,897,714 - 9,897,694 = 40,000,020, and 40,000,020 - 40,000,000 = 20
- Control, a close that pays VPI instead of earning a rebate: the field equals the nominal fee.

No funds move because of the view. The executed close is correct under the planner's netting rule. The defect is the lens output, which does not match its documented definition.

## Vulnerability Detail

1. The public field definition, `packages/perps/src/interfaces/ICfdEngineTypes.sol:265-268`:

   ```solidity
   /// @param realizedPnlAssets Signed price PnL realized by the requested size before separate costs.
   /// @param vpiDeltaAssets Signed close VPI: positive is a charge and negative is a rebate.
   /// @param vpiAssets Positive VPI charge exposed for legacy clients; zero for rebates.
   /// @param executionFeeAssets Protocol execution fee assessed on the close notional.
   ```

   The open-preview sibling uses the same wording at `ICfdEngineTypes.sol:332` ("Protocol execution fee assessed on trade notional"), and there the lens returns the nominal fee (`CfdEnginePlanLib.sol:399`, `delta.executionFeeAssets = openState.executionFeeAssets`).

2. The fee on the close notional is computed in `packages/perps/src/libraries/CloseAccountingLib.sol:120-126`. In the same place the kernel defines the trader's net with that nominal fee:

   ```solidity
   state.executionFeeAssets = (notionalAssets * inputs.executionFeeBps) / 10_000;
   ...
   state.netSettlementAssets = state.realizedPnlAssets - state.vpiDeltaAssets
       - int256(state.executionFeeAssets + state.frozenSpreadAssets);
   ```

3. The close planner nets the VPI rebate against the fee and then overwrites `delta.executionFeeAssets` with the capped amount. `packages/perps/src/libraries/CfdEnginePlanLib.sol:1041-1068`:

   ```solidity
   int256 actionNetAssets = cs.vpiDeltaAssets
       + int256(cs.executionFeeAssets + cs.frozenSpreadAssets + delta.pendingCarryAssets - delta.realizedCarryAssets);
   ...
   uint256 executionFeeEligibleAssets = nonClawbackWithheldAssets + nonVpiReserveCollectedAssets;
   delta.executionFeeAssets =
       cs.executionFeeAssets < executionFeeEligibleAssets ? cs.executionFeeAssets : executionFeeEligibleAssets;
   ```

   With a rebate R, the collected action charge is `fee - R`, or 0 when R exceeds the fee, so `delta.executionFeeAssets` becomes `fee - R` or 0.

4. The lens copies the capped planner value into the documented field, and the rebate into `vpiDeltaAssets`. `packages/perps/src/CfdEngineLens.sol:395-399`:

   ```solidity
   preview.vpiDeltaAssets = delta.closeState.vpiDeltaAssets;
   if (delta.closeState.vpiDeltaAssets > 0) {
       preview.vpiAssets = uint256(delta.closeState.vpiDeltaAssets);
   }
   preview.executionFeeAssets = delta.executionFeeAssets;
   ```

   The rebate is therefore in the struct twice: once as `vpiDeltaAssets < 0`, and once as the reduction of `executionFeeAssets`.

No public document defines the field as the netted fee. The close preview guide (`packages/perps/CLOSE_PREVIEW.md:9`) describes the separate `CfdClosePreview` contract. Its only remark on this lens is that `CfdEngineLens.previewClose` "has a different interface". The engine lens close preview section of the accounting specification (`packages/perps/ACCOUNTING_SPEC.md:1074-1084`) defines only the payout destination fields.

## Impact

Who is affected: wallets, frontends and integrations that use the in-scope `CfdEngineLens.previewClose` (or `simulateClose`, which shares `_previewClose`) to show a trader the cost or proceeds of a close before signing. For every skew-reducing close they show a better result than the trader receives:

| case | nominal fee | rebate | `executionFeeAssets` returned | documented net (`pnl - vpiDelta - fee`) | actual trader change | overstatement |
|---|---|---|---|---|---|---|
| 100 of 200 lots, rebate < fee | 4,000,000 | 998,003 | 3,001,997 | -2,003,994 | -3,001,997 | 998,003 |
| 1,000 lots full close, rebate > fee | 40,000,000 | 49,897,714 | 0 | +49,897,714 | +9,897,694 | 40,000,020 |
| control: close pays VPI | 4,000,000 | none | 4,000,000 | matches | matches | 0 |

How the numbers fit together: the overstatement equals `min(R, fee)`. When R is below the fee, the field is `fee - R` and the formula adds R twice. When R is above the fee, the field is 0, so the whole fee disappears from the formula (the 20 extra atoms are carry accrued between commitment and execution). The overstatement scales linearly with close notional, at up to 4 bps of the notional with the deployment fee. The control shows that the field equals the fee on the close notional whenever no rebate is netted, so the rebate netting is the only cause.

Users who rely on the preview receive less than the lens told them they would. That is a promised return the contract does not deliver, with no loss of protocol value.

Mapping to the program row: "Contract fails to deliver promised returns, but doesn't lose value." The in-scope lens's documented return value (the fee on the close notional) is not what it returns. The difference is a measured, systematic misstatement of the trader's close result, and no value is lost.


## Limits

- No funds are lost or frozen, and no on-chain path reads `ClosePreview.executionFeeAssets`. The only on-chain lens reads are `isLiquidatableAt` (`packages/perps/src/router/OrderLiquidationHandler.sol:57`) and `previewOpenRevertCode` (`packages/perps/src/router/OrderLiquidationBatchLogic.sol:432`), and neither uses this field.
- The executed close is correct under the planner's netting rule. This report does not dispute the netting. It disputes only that the public field returns the netted value while its definition says the fee on the close notional.
- The internal planner struct describes its own field as the "Assessed execution fee capped by eligible withheld price gain and collected action cash" (`packages/perps/src/CfdEnginePlanTypes.sol:318`). That comment is on an internal type. The public `ClosePreview` definition that integrators read (`ICfdEngineTypes.sol:268`) says the fee on the close notional, and the open preview field with the same wording returns the nominal fee.

## References

- `packages/perps/src/interfaces/ICfdEngineTypes.sol:265-268` (close field definition), `:331-332` (open sibling with the same wording)
- `packages/perps/src/CfdEngineLens.sol:395-399` (lens assignment), `:72-78` (`previewClose`), `:182-189` (`simulateClose`)
- `packages/perps/src/libraries/CfdEnginePlanLib.sol:1036-1074` (close action netting and fee cap), `:399` (open fee is nominal)
- `packages/perps/src/libraries/CloseAccountingLib.sol:120-126` (nominal close fee and net settlement identity)

## Proof of Concept

## Proof of Concept

### Steps to reproduce

1. With the deployment risk parameters (execution fee 4 bps, VPI factor 0.01e18), a trader deposits 100,000 USDG and opens a LONG alone, so it creates the skew (`OrderRouter.commitOrder` then `executeOrder` at price 1.00).
2. The trader commits a close of part or all of the position (`commitOrder(..., isClose = true)`).
3. In the execution block, read `CfdEngineLens.previewClose(trader, size, price)`.
4. Execute the close and compare the trader's clearinghouse balance change and the treasury credit with the preview fields.

### Proof of concept

Save as `packages/perps/test/perps/PreviewCloseFeeField.t.sol` and run:

```sh
cd packages/perps
forge test --match-path test/perps/PreviewCloseFeeField.t.sol -vv
```

The test uses the repository's `OrderRouterPythTestBase` fixture. It sets the deployment risk parameters through the engine admin timelock, which the fixture owns. Trading uses only public entrypoints: `clearinghouse.deposit`, `router.updateMarkPrice`, `router.commitOrder` and `router.executeOrder`. No storage writes, etches or mocked calls are used.

```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

import {CfdTypes} from "@plether/perps/CfdTypes.sol";
import {OrderV3Types} from "@plether/perps/OrderV3Types.sol";
import {ICfdEngineAdminHost} from "@plether/perps/interfaces/ICfdEngineAdminHost.sol";
import {ICfdEngineTypes} from "@plether/perps/interfaces/ICfdEngineTypes.sol";
import {MockPyth} from "@plether/test-utils/MockPyth.sol";

import {console2} from "forge-std/console2.sol";

import {OrderRouterPythTestBase} from "./shared/OrderRouterTestBase.sol";

/// CfdEngineLens.previewClose returns `executionFeeAssets` net of the close VPI rebate, while
/// ICfdEngineTypes.ClosePreview documents it as the fee assessed on the close notional.
contract PreviewCloseFeeFieldTest is OrderRouterPythTestBase {
    address internal keeper = address(0x4EE9);
    address internal trader = address(0xA11CE);
    address internal other = address(0xDA7E);

    uint256 internal constant T0 = SETUP_TIMESTAMP + 10 hours;
    int64 internal constant PX = 100_000_000; // 1.00, constant: realized price PnL is 0

    /// Deployment risk parameters (executionFeeBps = 4, vpiFactor = 0.01e18).
    function _deploymentRiskConfig() internal {
        ICfdEngineAdminHost.EngineRiskConfig memory config;
        config.riskParams = CfdTypes.RiskParams({
            vpiFactor: 0.01e18,
            maxSkewRatio: 0.4e18,
            maintMarginBps: 10,
            initMarginBps: 20,
            fadMarginBps: 300,
            baseCarryBps: 500,
            minBountyAssets: 1e6,
            bountyBps: 10,
            keeperShareBps: 5_000,
            protocolShareBps: 0
        });
        config.executionFeeBps = 4;
        config.frozenCloseSpreadBps = 50;
        config.settlementBufferBps = 25;
        engineAdmin.proposeRiskConfig(config);
        vm.warp(vm.getBlockTimestamp() + 72 hours + 1);
        engineAdmin.finalizeRiskConfig();
    }

    function _blob(uint256 publishTime) internal pure returns (bytes[] memory data) {
        MockPyth.FeedUpdate[] memory u = new MockPyth.FeedUpdate[](2);
        u[0] = MockPyth.FeedUpdate(FEED_A, MockPyth.MockPrice(PX, 0, int32(-8), publishTime, publishTime - 1));
        u[1] = MockPyth.FeedUpdate(FEED_B, MockPyth.MockPrice(PX, 0, int32(-8), publishTime, publishTime - 1));
        data = new bytes[](1);
        data[0] = abi.encode(u);
    }

    function _at(uint256 t) internal {
        vm.warp(t);
        vm.roll(vm.getBlockNumber() + 1);
    }

    function _mark(uint256 t) internal {
        _at(t);
        vm.prank(keeper);
        router.updateMarkPrice(_blob(t - 1));
    }

    function _fund(address who, uint256 amt) internal {
        settlementToken.mint(who, amt);
        vm.startPrank(who);
        settlementToken.approve(address(clearinghouse), type(uint256).max);
        clearinghouse.deposit(who, amt);
        vm.stopPrank();
    }

    function _open(address who, CfdTypes.Side side, uint256 lots, uint256 t) internal {
        _at(t);
        vm.prank(who);
        uint64 id = router.commitOrder(side, lots * 100e18, 20_000e6, 0, false);
        _at(t + 2);
        vm.prank(who);
        OrderV3Types.ExecutionResult memory r = router.executeOrder(id, _blob(t + 1));
        require(r.status == OrderV3Types.LifecycleStatus.Executed, "open must execute");
    }

    function _wealth(address who) internal view returns (uint256) {
        return clearinghouse.balanceAssets(who) + engine.traderClaimBalanceAssets(who);
    }

    function _treasury() internal view returns (uint256) {
        address t = engine.protocolTreasury();
        return settlementToken.balanceOf(t) + clearinghouse.balanceAssets(t);
    }

    /// Opens `openLots` LONG alone (the trader creates the skew), then closes `closeLots`, which reduces skew
    /// and earns a VPI rebate. The preview is read in the execution block at the execution price.
    function _run(uint256 openLots, uint256 closeLots)
        internal
        returns (uint256 nominalFee, ICfdEngineTypes.ClosePreview memory pv, int256 traderDelta, uint256 treasuryDelta)
    {
        _deploymentRiskConfig();
        _mark(T0);
        _fund(trader, 100_000e6);
        _open(trader, CfdTypes.Side.LONG, openLots, T0 + 100);

        uint256 size = closeLots * 100e18;
        _mark(T0 + 200);
        _at(T0 + 210);
        vm.prank(trader);
        uint64 id = router.commitOrder(CfdTypes.Side.LONG, size, 0, 0, true);
        _at(T0 + 212);

        pv = engineLens.previewClose(trader, size, uint256(uint64(PX)));
        nominalFee = _engineExecutionFeeAssets(size, uint256(uint64(PX)));

        uint256 w0 = _wealth(trader);
        uint256 t0 = _treasury();
        vm.prank(trader);
        OrderV3Types.ExecutionResult memory r = router.executeOrder(id, _blob(T0 + 211));
        require(r.status == OrderV3Types.LifecycleStatus.Executed, "close must execute");
        traderDelta = int256(_wealth(trader)) - int256(w0);
        treasuryDelta = _treasury() - t0;

        int256 documentedNet = pv.realizedPnlAssets - pv.vpiDeltaAssets - int256(pv.executionFeeAssets);
        console2.log("nominal fee on close notional (4 bps):", nominalFee);
        console2.log("previewClose.executionFeeAssets:      ", pv.executionFeeAssets);
        console2.log("previewClose.vpiDeltaAssets:");
        console2.logInt(pv.vpiDeltaAssets);
        console2.log("pnl - vpiDelta - executionFeeAssets:");
        console2.logInt(documentedNet);
        console2.log("trader balance change at execution:");
        console2.logInt(traderDelta);
        console2.log("treasury credit at execution:", treasuryDelta);
    }

    /// Rebate smaller than the fee: field = fee - rebate.
    function test_rebateBelowFee_fieldIsNettedFee() public {
        (uint256 nominalFee, ICfdEngineTypes.ClosePreview memory pv, int256 traderDelta, uint256 treasuryDelta) =
            _run(200, 100);
        assertTrue(pv.valid);
        assertEq(pv.realizedPnlAssets, 0, "constant price");
        assertLt(pv.vpiDeltaAssets, 0, "close earns a VPI rebate");
        uint256 rebate = uint256(-pv.vpiDeltaAssets);
        assertEq(pv.executionFeeAssets, nominalFee - rebate, "field = nominal fee - rebate");
        assertEq(treasuryDelta, pv.executionFeeAssets);
        assertEq(traderDelta, -int256(nominalFee) + int256(rebate), "trader pays fee - rebate");
        int256 documentedNet = pv.realizedPnlAssets - pv.vpiDeltaAssets - int256(pv.executionFeeAssets);
        assertEq(documentedNet - traderDelta, int256(rebate), "documented reading overstates by the rebate");
    }

    /// Rebate larger than the fee: field = 0 although a fee of `nominalFee` was charged against the rebate.
    function test_rebateAboveFee_fieldIsZero() public {
        (uint256 nominalFee, ICfdEngineTypes.ClosePreview memory pv, int256 traderDelta, uint256 treasuryDelta) =
            _run(1000, 1000);
        assertTrue(pv.valid);
        assertLt(pv.vpiDeltaAssets, 0, "close earns a VPI rebate");
        uint256 rebate = uint256(-pv.vpiDeltaAssets);
        assertGt(rebate, nominalFee);
        assertEq(pv.executionFeeAssets, 0, "field reports zero fee");
        assertEq(treasuryDelta, 0);
        assertEq(pv.payoutToFreeSettlementAssets, rebate - nominalFee, "rebate paid = rebate - nominal fee");
        int256 documentedNet = pv.realizedPnlAssets - pv.vpiDeltaAssets - int256(pv.executionFeeAssets);
        // Up to a few atoms of carry accrued between commitment and execution.
        assertApproxEqAbs(documentedNet - traderDelta, int256(nominalFee), 100, "overstated by the nominal fee");
    }

    /// Control: a close that pays VPI (no rebate) reports the nominal fee.
    function test_control_noRebate_fieldIsNominalFee() public {
        _deploymentRiskConfig();
        _mark(T0);
        _fund(other, 100_000e6);
        _open(other, CfdTypes.Side.SHORT, 2000, T0 + 50);
        _fund(trader, 100_000e6);
        _open(trader, CfdTypes.Side.LONG, 200, T0 + 100);
        _mark(T0 + 200);
        uint256 size = 100 * 100e18;
        ICfdEngineTypes.ClosePreview memory pv = engineLens.previewClose(trader, size, uint256(uint64(PX)));
        assertGt(pv.vpiDeltaAssets, 0, "close pays VPI");
        assertEq(pv.executionFeeAssets, _engineExecutionFeeAssets(size, uint256(uint64(PX))));
    }
}
```

Output:

```text
Ran 3 tests for test/perps/PreviewCloseFeeField.t.sol:PreviewCloseFeeFieldTest
[PASS] test_control_noRebate_fieldIsNominalFee() (gas: 8173151)
[PASS] test_rebateAboveFee_fieldIsZero() (gas: 7458737)
Logs:
  nominal fee on close notional (4 bps): 40000000
  previewClose.executionFeeAssets:       0
  previewClose.vpiDeltaAssets:
  -49897714
  pnl - vpiDelta - executionFeeAssets:
  49897714
  trader balance change at execution:
  9897694
  treasury credit at execution: 0

[PASS] test_rebateBelowFee_fieldIsNettedFee() (gas: 7352619)
Logs:
  nominal fee on close notional (4 bps): 4000000
  previewClose.executionFeeAssets:       3001997
  previewClose.vpiDeltaAssets:
  -998003
  pnl - vpiDelta - executionFeeAssets:
  -2003994
  trader balance change at execution:
  -3001997
  treasury credit at execution: 3001997

Suite result: ok. 3 passed; 0 failed; 0 skipped; finished in 13.75ms (21.88ms CPU time)

Ran 1 test suite in 14.54ms (13.75ms CPU time): 3 tests passed, 0 failed, 0 skipped (3 total tests)
```

### What each test shows

- `test_rebateBelowFee_fieldIsNettedFee`: the field equals the nominal fee minus the rebate. The documented formula overstates the trader's result by exactly the rebate (998,003 atoms).
- `test_rebateAboveFee_fieldIsZero`: the field is 0 although the 40,000,000-atom fee was charged against the rebate. The rebate actually paid is `rebate - fee`, and the documented formula overstates the result by the whole fee.
- `test_control_noRebate_fieldIsNominalFee`: without a rebate the field equals the fee on the close notional, so the netting is the only cause.

## Recommendation

Make the returned value match the documented field, and keep the collected amount available under its own name. In `CfdEngineLens._previewClose`:

```solidity
// Fee on the close notional, as documented in ICfdEngineTypes.ClosePreview.
preview.executionFeeAssets = delta.closeState.executionFeeAssets;
// New field: fee actually credited to the treasury after rebate netting and eligibility caps.
preview.executionFeeCollectedAssets = delta.executionFeeAssets;
```

If the ABI must stay unchanged, correct the documentation of `ICfdEngineTypes.ClosePreview.executionFeeAssets` (line 268) instead. It should say that the value is the fee credited to the treasury after it is netted against any close VPI rebate and capped by eligible collateral (zero when waived or fully offset), and that the trader's net is `realizedPnlAssets - vpiDeltaAssets - notional * executionFeeBps / 10_000`.

