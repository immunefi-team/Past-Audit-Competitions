# \#98281 [SC-Medium] A close whose VPI refund exceeds its execution fee charges the trader the 4 bp fee but never credits it to the treasury; the pool keeps it

**Submitted on Nov 6th 2026 at 17:23:54 UTC by @Naresh_Kandula for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #98281
- **Report Type:** Smart Contract
- **Report severity:** Medium
- **Target:** https://github.com/immunefi-team/audit-comp-plether/tree/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src
- **Impacts:**
  - Contract fails to deliver promised returns, but doesn't lose value

## Description

## Severity / impact

- **Impact (program row):** Low, *Contract fails to deliver promised returns, but doesn't lose value*. The protocol treasury does not receive the execution fee it is owed on every rebate close. The trader still pays that fee, and LP pool assets keep it.
- **No user loses funds.** This is a fee-routing defect: on this branch the treasury's revenue goes to LPs instead.

## Asset

`packages/perps/src/libraries/CfdEnginePlanLib.sol` and `packages/perps/src/CfdEngineSettlementSidecar.sol` at `a82d4fefb02768e5b3bddcc97dee371eee24fea8`.

## Root cause

In `_planCloseAction`, the close execution fee is netted into the action amount together with the fresh VPI delta:

```solidity
// CfdEnginePlanLib.sol:1041-1044
int256 actionNetAssets = cs.vpiDeltaAssets
    + int256(cs.executionFeeAssets + cs.frozenSpreadAssets + delta.pendingCarryAssets - delta.realizedCarryAssets);
```

When the refund exceeds the fee, `actionNetAssets < 0` and the close becomes an action rebate (`:1059-1061`). Only `actionRebateAssets` is set. Both `actionChargeWithheldAssets` and `actionChargeCollectedAssets` stay 0, so the fee the treasury receives is capped at 0:

```solidity
// CfdEnginePlanLib.sol:1063-1068
uint256 nonClawbackWithheldAssets = delta.actionChargeWithheldAssets - vpiClawbackWithheldAssets;      // 0
uint256 nonVpiReserveCollectedAssets = delta.actionChargeCollectedAssets - delta.vpiRebateReserveConsumedAssets; // 0
uint256 executionFeeEligibleAssets = nonClawbackWithheldAssets + nonVpiReserveCollectedAssets;          // 0
delta.executionFeeAssets =
    cs.executionFeeAssets < executionFeeEligibleAssets ? cs.executionFeeAssets : executionFeeEligibleAssets; // 0
```

This leaves `feeTopUpAssets = 0`, so `protocolFeeTopUpAssets = 0` (`:984-987`). The sidecar's `_settleProtocolFeeTopUp(host, 0 + 0, 0)` (`CfdEngineSettlementSidecar.sol:626-628`) returns early at `:913-915`. Meanwhile the pool pays the trader only `|VPI| − fee` (`:621-623`). The fee was charged to the trader, but it stays in pool cash.

The open path handles the same situation correctly. A skew-healing open whose rebate exceeds the fee calls `_settleProtocolFeeTopUp(host, delta.executionFeeAssets, …)` (`CfdEngineSettlementSidecar.sol:498`), which pays the full fee to the treasury from free pool cash (`:905-926`, the post-MED-6 behaviour). The close path never requests that top-up, even when ample free cash is available.

The documentation says the fee belongs to the treasury:
- WHITEPAPER.md:411-413: *"Positive VPI increases physical pool assets; a VPI rebate reduces them; the protocol execution fee belongs to treasury rather than LP assets."*
- WHITEPAPER.md:1455, in the stateful replay assumptions: *"voluntary closes pay the 4-basis-point treasury execution fee"*.
- The function's own natspec (`CfdEnginePlanLib.sol:1035`) says it *"allocates the collected execution fee"*. On this branch the fee is collected from the trader but never allocated.

## Relationship to known issues

- Cecuro **MED-6** (*"Protocol treasury can spend LP withdrawal reserves on fee top-ups"*, Acknowledged/addressed) and **LOW-26** (*"Trader can be blocked from opening by underfunded protocol fee top-ups"*) concern how the open-path top-up is funded and when it is available. This report concerns the close path never requesting a top-up at all.
- None of the SECURITY.md Known Limitations, Router and keeper limitations, or VPI limitations sections mention fee routing on rebate closes.

## Recommended fix

On the close rebate branch the fee has already been taken from the trader, because it is netted out of the rebate. Count it as eligible: when `actionNetAssets < 0`, set `executionFeeEligibleAssets = cs.executionFeeAssets`. Then route it exactly like the open path, through `protocolFeeTopUpAssets` / `_settleProtocolFeeTopUp`, paid from the pool cash that currently keeps it. Add test A above as a regression test.


## Proof of Concept

## Proof of concept

File: `packages/perps/test/perps/FeeRouting_CloseRebate.t.sol` (full source in the appendix). It uses the repository's `CfdEngineTestBase` with the real `CfdEngine`, `HousePool`, `MarginClearinghouse` and settlement sidecar, at vpiFactor 0.01 and the 4 bp execution fee. The ledger is independent: raw pool USDG, plus the treasury's and trader's clearinghouse balances. The fee the test expects is computed as 4 bp × notional, not read from a preview.

```
cd packages/perps
forge test --root . --match-path test/perps/FeeRouting_CloseRebate.t.sol -vv
```

```
[PASS] test_A_RebateClose_TreasuryGetsNothing_PoolKeepsFee()
  close vpiDeltaAssets: -1664632731
  close executionFeeAssets (preview, credited): 0
  close execution fee owed (4 bp x notional): 400000000
  treasury credit on close: 0
  trader settlement gain on close: 1264632252
  pool raw outflow on close: 1264632252
[PASS] test_B_Control_NoVpi_TreasuryGetsCloseFee()
  control: close fee: 400000000
  control: treasury credit on close: 400000000
[PASS] test_C_HealingOpen_TreasuryGetsFee()
  healing open vpiAssets: -1664632731
  healing open fee: 400000000
  treasury credit on healing open: 400000000
Suite result: ok. 3 passed; 0 failed
```

- **A (bug).** A 1,000,000 SHORT pays VPI on open, then closes fully at the same price and depth. The close refunds 1,664.63 of VPI.
  - The trader receives 1,264.63, which is the refund minus the 400.00 fee (±1 s of carry).
  - The treasury receives 0.
  - The pool's raw outflow equals the trader's gain exactly, so the pool kept the 400.00.
  - The engine's own preview records an execution fee of 0 for this close.
- **B (A/B control).** The same round trip with vpiFactor = 0. The treasury receives the 400.00 close fee.
- **C (contrast).** A skew-healing open with the same rebate size (1,664.63 against a 400.00 fee) credits the full 400.00 to the treasury.

## Appendix: `FeeRouting_CloseRebate.t.sol`

```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

import {CfdTypes} from "@plether/perps/CfdTypes.sol";
import {ICfdEngineTypes} from "@plether/perps/interfaces/ICfdEngineTypes.sol";

import {CfdEngineTestBase} from "./shared/CfdEngineTestBase.sol";

/// @notice Close-side VPI rebate drops the protocol execution fee (pin a82d4fef).
///         A close whose VPI refund exceeds its execution fee is an action rebate (CfdEnginePlanLib.sol:1059-1061);
///         executionFeeEligibleAssets is then 0 (:1063-1068), so no protocol fee and no top-up is credited
///         (:984-987, CfdEngineSettlementSidecar.sol:626-628 -> :913-915). The trader still pays the fee (it is netted
///         out of the rebate) and the pool keeps it. The open path tops the fee up to treasury in the same situation
///         (CfdEngineSettlementSidecar.sol:498, 905-926; asserted by test_C_HealingOpen_TreasuryGetsFee).
///         Ledger is independent: raw USDG of pool / clearinghouse and settlement balances; previews are only used
///         to read the fee and VPI inputs of the action being measured.
contract FeeRouting_CloseRebateTest is CfdEngineTestBase {

    address internal constant LP = address(0xA11CE);
    address internal constant SHORT_A = address(0xB0B1);
    address internal constant SHORT_B = address(0xB0B2);
    uint256 internal constant SIZE = 1_000_000e18;
    uint256 internal constant PX = 1e8;

    function _params(
        uint256 vpiFactor
    ) internal pure returns (CfdTypes.RiskParams memory p) {
        p = CfdTypes.RiskParams({
            vpiFactor: vpiFactor,
            maxSkewRatio: 1e18,
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

    struct Ledger {
        uint256 poolRaw;
        uint256 treasury;
        uint256 trader;
    }

    function _ledger(
        address trader
    ) internal view returns (Ledger memory l) {
        l.poolRaw = settlementToken.balanceOf(address(pool));
        l.treasury = clearinghouse.balanceAssets(engine.protocolTreasury());
        l.trader = clearinghouse.balanceAssets(trader);
    }

    function _setup(
        uint256 vpiFactor
    ) internal {
        _setRiskParams(_params(vpiFactor));
        _fundJunior(LP, 2_000_000e6);
        _fundTrader(SHORT_A, 200_000e6);
        _fundTrader(SHORT_B, 200_000e6);
    }

    /// Opens a skew-increasing SHORT (pays VPI), then closes it fully at the same price and depth. The close restores
    /// skew, so its fresh VPI delta refunds the open's charge (lifetime clamp caps it at the charge).
    function _roundTrip(
        address who
    ) internal returns (ICfdEngineTypes.ClosePreview memory pv, Ledger memory b, Ledger memory a) {
        _open(who, CfdTypes.Side.SHORT, SIZE, 50_000e6, PX);
        vm.warp(vm.getBlockTimestamp() + 1);
        pv = engineLens.previewClose(who, SIZE, PX);
        require(pv.valid, "close preview");
        b = _ledger(who);
        _close(who, CfdTypes.Side.SHORT, SIZE, PX);
        a = _ledger(who);
    }

    /// BUG: a close with a VPI refund larger than the fee. The trader is charged the fee (rebate = |VPI| - fee),
    /// the treasury receives nothing, and the pool keeps the fee.
    function test_A_RebateClose_TreasuryGetsNothing_PoolKeepsFee() public {
        _setup(0.01e18);
        (ICfdEngineTypes.ClosePreview memory pv, Ledger memory b, Ledger memory a) = _roundTrip(SHORT_A);
        // Independent fee: 4 bp of the closed notional (SIZE at PX, 18 -> 6 decimals). The preview reports the
        // CREDITED fee, which is 0 on this branch; that is the defect, so it cannot be the expectation.
        uint256 fee = ((SIZE * PX / 1e8) / 1e12) * engine.executionFeeBps() / 10_000;
        emit log_named_int("close vpiDeltaAssets", pv.vpiDeltaAssets);
        emit log_named_uint("close executionFeeAssets (preview, credited)", pv.executionFeeAssets);
        emit log_named_uint("close execution fee owed (4 bp x notional)", fee);
        emit log_named_uint("treasury credit on close", a.treasury - b.treasury);
        emit log_named_uint("trader settlement gain on close", a.trader - b.trader);
        emit log_named_uint("pool raw outflow on close", b.poolRaw - a.poolRaw);
        assertGt(fee, 0, "close carries a 4 bp execution fee");
        assertLt(pv.vpiDeltaAssets, 0, "close is VPI-healing (refund)");
        assertGt(uint256(-pv.vpiDeltaAssets), fee, "refund exceeds fee: action rebate branch");
        assertEq(pv.executionFeeAssets, 0, "the engine itself records no execution fee for this close");
        assertEq(a.treasury, b.treasury, "treasury receives no close execution fee");
        // One second of carry is also netted into the close (< 0.001 USDG), hence the 0.01 USDG tolerance.
        assertApproxEqAbs(a.trader - b.trader, uint256(-pv.vpiDeltaAssets) - fee, 1e4, "trader paid the fee out of the rebate");
        assertEq(b.poolRaw - a.poolRaw, a.trader - b.trader, "everything the pool paid went to the trader: it kept the fee");
    }

    /// CONTROL A/B: identical round trip with vpiFactor = 0. The close is a plain charge and treasury gets the fee.
    function test_B_Control_NoVpi_TreasuryGetsCloseFee() public {
        _setup(0);
        (ICfdEngineTypes.ClosePreview memory pv, Ledger memory b, Ledger memory a) = _roundTrip(SHORT_A);
        emit log_named_uint("control: close fee", pv.executionFeeAssets);
        emit log_named_uint("control: treasury credit on close", a.treasury - b.treasury);
        assertGt(pv.executionFeeAssets, 0);
        assertEq(a.treasury - b.treasury, pv.executionFeeAssets, "treasury receives the close fee");
    }

    /// CONTRAST (open path, same sign of VPI): a skew-healing OPEN whose rebate exceeds the fee. The post-MED-6
    /// top-up credits the full fee to treasury from free pool cash.
    function test_C_HealingOpen_TreasuryGetsFee() public {
        _setup(0.01e18);
        _open(SHORT_A, CfdTypes.Side.LONG, SIZE, 50_000e6, PX); // skew long
        vm.warp(vm.getBlockTimestamp() + 1);
        ICfdEngineTypes.OpenPreview memory pv = engineLens.previewOpen(SHORT_B, CfdTypes.Side.SHORT, SIZE, 50_000e6, PX);
        require(pv.valid, "open preview");
        Ledger memory b = _ledger(SHORT_B);
        _open(SHORT_B, CfdTypes.Side.SHORT, SIZE, 50_000e6, PX);
        Ledger memory a = _ledger(SHORT_B);
        emit log_named_int("healing open vpiAssets", pv.vpiAssets);
        emit log_named_uint("healing open fee", pv.executionFeeAssets);
        emit log_named_uint("treasury credit on healing open", a.treasury - b.treasury);
        assertLt(pv.vpiAssets, 0, "healing open rebate");
        assertGt(uint256(-pv.vpiAssets), pv.executionFeeAssets, "rebate exceeds fee");
        assertEq(a.treasury - b.treasury, pv.executionFeeAssets, "open path credits the fee to treasury");
    }

}
```

