# \#99081 [SC-Medium] Close execution fee lost to LPs when VPI rebate dominates, depriving treasury of promised protocol revenue

**Submitted on Nov 11th 2026 at 19:15:07 UTC by @papikoo for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #99081
- **Report Type:** Smart Contract
- **Report severity:** Medium
- **Target:** https://github.com/immunefi-team/audit-comp-plether/tree/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src
- **Impacts:**
  - Theft of unclaimed yield

## Description

## Brief/Intro

When a trader closes a position that reduces market skew, the VPI (Virtual Price Impact) rebate can exceed the execution fee. In this case, `_planCloseActionSettlement` nets the execution fee against the VPI rebate and sets `delta.executionFeeAssets = 0`, causing the protocol treasury to receive nothing. On opens, the treasury is **always** topped up with the full execution fee — even when VPI rebate dominates — via `_settleProtocolFeeTopUp` from pool free cash. This asymmetry causes systematic loss of treasury revenue on every skew-reducing close where VPI rebate exceeds the fee.

## Vulnerability Details

The root cause is an asymmetry between the open and close fee settlement paths.

**Open path** (`CfdEngineSettlementSidecar.executeOpen`, line 498):
```solidity
_settleProtocolFeeTopUp(host, delta.executionFeeAssets, protocolFeeCreditedAssets);
```
Here `delta.executionFeeAssets` is always the **full** execution fee (`notional × executionFeeBps / 10_000`), regardless of VPI direction. Even when `tradeCostAssets < 0` (VPI rebate > fee), the treasury gets topped up from pool free cash for the unfunded portion. The `_settleProtocolFeeTopUp` function (line 905-926) computes `poolFundedAssets = amountAssets - clearinghouseCreditedAssets` and transfers that from `pool.getSettlementFreeAssets()`.

**Close path** (`_planCloseActionSettlement`, CfdEnginePlanLib lines 1036-1074):
```solidity
int256 actionNetAssets = cs.vpiDeltaAssets
    + int256(cs.executionFeeAssets + cs.frozenSpreadAssets + delta.pendingCarryAssets - delta.realizedCarryAssets);
```
When `actionNetAssets <= 0` (VPI rebate dominates), the code takes the `else if` branch (line 1059-1061) which only sets `actionRebateAssets`. All fee-related fields remain at their default zero values:
- `delta.actionChargeWithheldAssets = 0`
- `delta.actionChargeCollectedAssets = 0`
- `delta.executionFeeAssets = 0` (line 1067-1068: `min(cs.executionFeeAssets, 0) = 0`)
- `delta.actionProtocolFeeCreditedAssets = 0`

Then `protocolFeeTopUpAssets` is computed (line 984):
```solidity
uint256 feeTopUpAssets = delta.executionFeeAssets - delta.actionProtocolFeeCreditedAssets; // 0 - 0 = 0
```

And the close settlement (line 626-628):
```solidity
_settleProtocolFeeTopUp(host, 0 + 0, 0); // amountAssets = 0 → no top-up
```

The treasury receives nothing.

**The economic consequence:** The execution fee that the whitepaper promises the treasury on closes is instead absorbed into the trader's VPI rebate. The trader receives a larger net rebate than they should, and the protocol treasury loses the fee. The pool (LPs) bears the same VPI rebate outflow either way — the difference is that the fee portion should be routed to treasury, not to the trader.

This is NOT by-design. The whitepaper explicitly states:
- Line 1455: *"voluntary closes pay the 4-basis-point treasury execution fee"* — no VPI qualifier
- Line 412: *"the protocol execution fee belongs to treasury rather than LP assets"*
- Line 810: *"Execution fee | Protocol service revenue on traded notional | Treasury, only when cash credited"* — on opens, cash IS always credited via pool free cash top-up

The `ACCOUNTING_SPEC.md` says (line 1118): *"only cash-collected execution or liquidation fees and free-cash-funded top-ups become treasury margin"*. On opens, the free-cash-funded top-up ensures the treasury always receives the fee. On closes, the top-up mechanism exists but is passed 0 as the amount.

## Impact Details

**Theft of unclaimed yield** — the protocol treasury is systematically deprived of execution fee revenue on every skew-reducing close where VPI rebate exceeds the fee.

Per-trade loss: `notional × executionFeeBps / 10_000` = 4 basis points of closed notional.

This is not a rare edge case. Any time a trader closes a position that was opened into skew (the most common VPI-paying open pattern), the close reduces skew and generates a VPI rebate. If the position was large enough to pay VPI > 4 bps on open (which happens at moderate sizes per the deployment's vpiFactor of 0.01e18), the close fee is lost.

**Quantification at deployment parameters:**
- vpiFactor = 0.01e18, executionFeeBps = 4, pool depth = 1M USDG
- VPI exceeds execution fee when: `0.01 × notional / (2 × poolDepth) > 4 / 10_000`
- Threshold: `notional > 80,000 USDG`
- Any close above ~80k USDG notional in a one-sided market loses the full fee to treasury

In the PoC: a single 200k USDG round-trip loses 80 USDG of treasury revenue. At realistic daily volume, this compounds to significant protocol revenue loss.

## References

- `CfdEnginePlanLib._planCloseActionSettlement()`: `packages/perps/src/libraries/CfdEnginePlanLib.sol` lines 1036-1074
- `CfdEnginePlanLib` close top-up computation: `packages/perps/src/libraries/CfdEnginePlanLib.sol` line 984-987
- `CfdEngineSettlementSidecar.executeClose` close top-up call: `packages/perps/src/CfdEngineSettlementSidecar.sol` lines 626-628
- `CfdEngineSettlementSidecar.executeOpen` open top-up call (always full fee): `packages/perps/src/CfdEngineSettlementSidecar.sol` line 498
- `CfdEngineSettlementSidecar._settleProtocolFeeTopUp()`: `packages/perps/src/CfdEngineSettlementSidecar.sol` lines 905-926
- WHITEPAPER line 1455: "voluntary closes pay the 4-basis-point treasury execution fee"
- WHITEPAPER line 810: "Execution fee | Treasury, only when cash credited"
- ACCOUNTING_SPEC line 1118: "only cash-collected execution or liquidation fees and free-cash-funded top-ups become treasury margin"

## Link to Proof of Concept

https://gist.github.com/divolt-research/12b1f509a556ef45db93801bc3443d95

## Proof of Concept

## Proof of Concept

### Steps to reproduce

1. Place the PoC file at `packages/perps/test/perps/poc/CloseFeeLossPoC.t.sol`
2. Run: `forge test --match-test "test_PoC_closeFeeNotCreditedToTreasuryWhenVpiRebateDominates" -vvv`
3. Observe: test passes — treasury gains 80 USDG on open, 0 on close for identical notional

### Step-by-step explanation

1. **Override riskParams** to set `vpiFactor = 0.01e18` (deployment default) and `baseCarryBps = 0` (isolate fee effect from carry noise).
2. **Fund trader** with 50,000 USDG.
3. **Record treasury balance** before any trade.
4. **Open LONG 200k notional** at price 1.00. This creates one-sided skew. VPI is charged to the trader (`0.01 × (200k)² / (2 × 1M) = 200 USDG`). Execution fee = 200k × 4 bps = 80 USDG. Treasury gains exactly 80 USDG (topped up from pool free cash).
5. **Record treasury balance** before close.
6. **Close the full LONG** at the same price (1.00). VPI gives a rebate (skew goes from 200k to 0). The VPI rebate (200 USDG) exceeds the execution fee (80 USDG), so `actionNetAssets < 0`. The code enters the rebate branch, `delta.executionFeeAssets = 0`, and `_settleProtocolFeeTopUp` is called with amount 0.
7. **Assert treasury gained 0** from the close — the 80 USDG execution fee is lost.

### PoC code

```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

import {BasePerpTest} from "../BasePerpTest.sol";
import {CfdTypes} from "@plether/perps/CfdTypes.sol";
import {console} from "forge-std/console.sol";

/// @notice Demonstrates that when VPI rebate dominates on close, the execution fee
///         that should go to the protocol treasury is lost — zeroed out by the netting
///         in _planCloseActionSettlement. On opens, the treasury is always topped up
///         from pool free cash. On closes with VPI rebate > fee, treasury gets nothing.
contract CloseFeeLossPoC is BasePerpTest {
    address trader = address(0xB001);

    function _riskParams() internal pure override returns (CfdTypes.RiskParams memory) {
        return CfdTypes.RiskParams({
            vpiFactor: 0.01e18,       // deployment default
            maxSkewRatio: 0.4e18,
            maintMarginBps: 100,
            initMarginBps: 150,
            fadMarginBps: 300,
            baseCarryBps: 0,          // zero carry to isolate fee effect
            minBountyAssets: 1e6,
            bountyBps: 10,
            keeperShareBps: 5000,
            protocolShareBps: 0
        });
    }

    function test_PoC_closeFeeNotCreditedToTreasuryWhenVpiRebateDominates() public {
        _fundTrader(trader, 50_000e6);

        address treasury = engine.protocolTreasury();
        uint256 treasuryBefore = clearinghouse.balanceAssets(treasury);

        // Step 1: Open a LONG position — creates one-sided skew, VPI is charged
        // 200k notional at price 1.00 = 200k USDG notional
        // VPI = 0.01 * (200k)^2 / (2 * 1M) = 200 USDG >> execution fee of 80 USDG
        _open(trader, CfdTypes.Side.LONG, 200_000e18, 10_000e6, 1e8);

        uint256 treasuryAfterOpen = clearinghouse.balanceAssets(treasury);
        uint256 openFeeToTreasury = treasuryAfterOpen - treasuryBefore;

        // Execution fee = 200k * 4 bps = 80 USDG = 80_000_000 (6 decimals)
        uint256 expectedFee = (200_000e18 * 1e8 / 1e20) * 4 / 10_000;
        console.log("Expected execution fee (6 dec):", expectedFee);
        console.log("Treasury gained on OPEN:", openFeeToTreasury);

        // Treasury should have gained at least the execution fee on open
        assertGe(openFeeToTreasury, expectedFee, "open: treasury gets execution fee");

        // Step 2: Close the LONG at same price — VPI gives rebate (reducing skew)
        uint256 treasuryBeforeClose = clearinghouse.balanceAssets(treasury);

        _close(trader, CfdTypes.Side.LONG, 200_000e18, 1e8);

        uint256 treasuryAfterClose = clearinghouse.balanceAssets(treasury);
        uint256 closeFeeToTreasury = treasuryAfterClose - treasuryBeforeClose;

        console.log("Treasury gained on CLOSE:", closeFeeToTreasury);
        console.log("Expected close fee (same notional):", expectedFee);

        // BUG: Treasury gains 0 on close because VPI rebate absorbed the execution fee
        // The fee that should go to treasury is lost
        assertEq(closeFeeToTreasury, 0, "close: treasury gets ZERO when VPI rebate dominates");

        // The trader's close execution fee was: 200k * 4 bps = 80 USDG
        // This 80 USDG should have gone to treasury but didn't
        // Over many trades, this is a systematic loss of protocol revenue
        console.log("");
        console.log("=== SUMMARY ===");
        console.log("Same 200k notional, same 4 bps fee rate");
        console.log("OPEN  -> treasury receives:", openFeeToTreasury);
        console.log("CLOSE -> treasury receives:", closeFeeToTreasury);
        console.log("Lost treasury revenue per close:", expectedFee);
    }
}
```

### Test output (passes)

```
[PASS] test_PoC_closeFeeNotCreditedToTreasuryWhenVpiRebateDominates() (gas: 1144796)
Logs:
  Expected execution fee (6 dec): 80000000
  Treasury gained on OPEN:        80000000
  Treasury gained on CLOSE:       0
  Expected close fee (same notional): 80000000

  === SUMMARY ===
  Same 200k notional, same 4 bps fee rate
  OPEN  -> treasury receives: 80000000
  CLOSE -> treasury receives: 0
  Lost treasury revenue per close: 80000000

Suite result: ok. 1 passed; 0 failed; 0 skipped
```

Open: treasury receives 80 USDG (the full execution fee).
Close: treasury receives 0 (the execution fee is absorbed by VPI rebate netting).
Same notional, same fee rate — 80 USDG of protocol revenue lost per round-trip.
