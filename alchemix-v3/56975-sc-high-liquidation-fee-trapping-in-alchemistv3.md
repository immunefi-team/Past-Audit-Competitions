# 56975 sc high liquidation fee trapping in alchemistv3

**Submitted on Oct 22nd 2025 at 11:01:12 UTC by @Paludo0x for** [**Audit Comp | Alchemix V3**](https://immunefi.com/audit-competition/alchemix-v3-audit-competition)

* **Report ID:** #56975
* **Report Type:** Smart Contract
* **Report severity:** High
* **Target:** <https://github.com/alchemix-finance/v3-poc/blob/immunefi\\_audit/src/AlchemistV3.sol>
* **Impacts:**
  * Permanent freezing of unclaimed royalties

## Description

## Brief/Intro

In `AlchemistV3::liquidate()` the liquidator’s `feeInYield` is carved out of the seized collateral `amountLiquidated` but its payout is conditionally gated by `account.collateralBalance >= feeInYield` **after** the seizure.

If the gate fails, the fee is neither sent to the liquidator nor forwarded to the Transmuter, leaving these tokens **locked** in the AlchemistV3 contract.

With no recovery function, this results in **permanent freezing of unclaimed royalties (liquidator fees)**.

## Vulnerability Details

This is the vulnerability flow in **AlchemixV3::\_doLiquidation()**:

1. Seize the borrower's collateral

```
account.collateralBalance = account.collateralBalance > amountLiquidated ? account.collateralBalance - amountLiquidated : 0;

```

2. Send NET to the Transmuter: (gross - fee)

```
TokenUtils.safeTransfer(myt, address(transmuter), amountLiquidated - feeInYield);
```

3. Pay the liquidator FEE ONLY IF enough residual collateral remains

```
if (feeInYield > 0 && account.collateralBalance >= feeInYield) {
    TokenUtils.safeTransfer(myt, msg.sender, feeInYield);
```

These are the keypoints that make implementation wrong:

* Step (2) reserves `feeInYield` by sending only `(amountLiquidated - feeInYield)` to the Transmuter.
* Step (3) may skip sending the fee if the post-seizure accounting residual is `< feeInYield`.
* There is **no fallback path** to re-route the fee anywhere. The reserved amount remains in the AlchemistV3 contract balance, **permanently frozen**.

The branch is real, but its reachability depends on parameters. In debt units, fee trapping requires:

```
collateral < debtToBurn + 2*fee
```

With `m = minimumCollateralization` and fee fraction `phi = liquidatorFee / 10_000`, this reduces to the threshold:

```
phi > m / (2m − 1)
```

For the defaults in this repo (`m ≈ 1.111…`), the threshold is ≈ 90.91% (≈ 9091 bps). With the default `liquidatorFee = 300 bps (3%)`, the inequality does not hold, so the problematic branch is not reachable under normal settings. It becomes reachable only with extreme fee configurations.

## Impact Details

The impact is **High**: Permanent freezing of unclaimed royalties.

* **Permanent freezing:** The unpaid liquidator fee is never sent and cannot be reclaimed on-chain (no skim/sweep/recover path), leaving the fee **permanently frozen** in AlchemistV3.
* **Incentive erosion:** Liquidators may be underpaid exactly when positions are near the threshold, reducing liquidation participation and increasing systemic risk.

## (Recommended Fix)

1. When the gate fails, send all seized to the Transmuter.
2. Implement a governance function `sweepTrappedFees`

## Proof of Concept

## Proof of Concept

What PoC tests show:

* `test_FeeTrapping_Condition_Unreachable_With_Default_Fee`: Uses `calculateLiquidation` to sample undercollateralized states and shows that, with default fee, `collateral ≥ debtToBurn + 2*fee` always holds; the branch cannot trigger in practice.
* `test_FeeTrapping_Condition_Becomes_Possible_When_Fee_Above_Threshold`: Temporarily raises the fee above the threshold and demonstrates `collateral < debtToBurn + 2*fee` can hold, proving the branch exists but is extreme-only.

```
// SPDX-License-Identifier: UNLICENSED
pragma solidity ^0.8.20;

import {AlchemistV3Test} from "./AlchemistV3.t.sol";


contract FeeTrapInheritedTest is AlchemistV3Test {
    // Proof-style test: with the default liquidator fee the trapping condition is unreachable.
    // It checks calculateLiquidation outcomes over undercollateralized states and asserts
    // collateral >= debtToBurn + 2*fee, which implies no fee can be trapped.
    function test_FeeTrapping_Condition_Unreachable_With_Default_Fee() public {
        // Protocol parameters
        uint256 m = alchemist.minimumCollateralization();
        uint256 gmin = alchemist.globalMinimumCollateralization();
        uint256 phiBps = alchemist.liquidatorFee(); // default 300 (3%)

        // Theoretical threshold: phi > m/(2m-1) is required for trapping to be possible
        // For m ≈ 1.111 the threshold ≈ 9091 bps (90.91%)
        uint256 thresholdPhi1e18 = (m * 1e18) / (2 * m - 1e18);
        uint256 phi1e18 = (phiBps * 1e18) / 10_000;
        assertLe(phi1e18, thresholdPhi1e18, "default phi exceeds threshold; trapping may be possible");

        // Sweep a range with fixed D and C between 1.01x and 1.10x D (below m)
        uint256 debt = 1e18;
        uint256 alchemistCurrent = gmin + 1; // > gmin to avoid outsourcedFee

        for (uint256 permil = 1010; permil <= 1100; permil += 5) { // 1.010 .. 1.100
            uint256 collateral = (debt * permil) / 1000;
            (uint256 gross, uint256 dBurn, uint256 fee, uint256 outsourced) = alchemist.calculateLiquidation(
                collateral,
                debt,
                m,
                alchemistCurrent,
                gmin,
                phiBps
            );

            // Skip points where no liquidation occurs
            if (gross == 0) continue;

            // Ensure we are not in the "global bad debt" branch
            assertEq(outsourced, 0, "unexpected outsourcedFee in local liquidation scenario");

            // Equivalent condition to C' >= F, in debt units:
            // C - (dBurn + fee) >= fee   =>   C >= dBurn + 2*fee
            assertGe(collateral, dBurn + (2 * fee), "C' < F would imply possible trapping; should not occur with defaults");
        }
    }

    // Proof-style test: raising the fee above the threshold makes trapping theoretically possible
    // (no on-chain liquidation path needed here, purely logic-based through calculateLiquidation).
    function test_FeeTrapping_Condition_Becomes_Possible_When_Fee_Above_Threshold() public {
        uint256 m = alchemist.minimumCollateralization();
        uint256 gmin = alchemist.globalMinimumCollateralization();

        // Compute threshold in bps and set phi = threshold + 1 (if possible)
        uint256 thresholdPhi1e18 = (m * 1e18) / (2 * m - 1e18);
        uint256 thresholdBps = (thresholdPhi1e18 * 10_000) / 1e18;
        if (thresholdBps >= 10_000) return; // degenerate case

        uint256 targetPhi = thresholdBps + 1; // slightly above threshold

        // Temporarily raise fee for the demonstration
        vm.prank(alOwner);
        alchemist.setLiquidatorFee(targetPhi);

        uint256 debt = 1e18;
        uint256 collateral = (debt * 1050) / 1000; // 1.05x D, below m
        uint256 alchemistCurrent = gmin + 1;

        (uint256 gross, uint256 dBurn, uint256 fee, uint256 outsourced) = alchemist.calculateLiquidation(
            collateral,
            debt,
            m,
            alchemistCurrent,
            gmin,
            alchemist.liquidatorFee()
        );

        // Local liquidation should occur (no outsourced fee)
        assertGt(gross, 0, "expected positive liquidation amount");
        assertEq(outsourced, 0, "unexpected outsourced fee");

        // Now the trapping precondition can hold: C < dBurn + 2*fee
        assertLt(collateral, dBurn + (2 * fee), "trapping precondition not met above threshold fee");
    }
}
```
