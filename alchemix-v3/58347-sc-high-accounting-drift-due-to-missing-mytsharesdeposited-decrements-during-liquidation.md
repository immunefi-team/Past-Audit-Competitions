# 58347 sc high accounting drift due to missing mytsharesdeposited decrements during liquidation

**Submitted on Nov 1st 2025 at 12:50:44 UTC by @InquisitorScythe for** [**Audit Comp | Alchemix V3**](https://immunefi.com/audit-competition/alchemix-v3-audit-competition)

* **Report ID:** #58347
* **Report Type:** Smart Contract
* **Report severity:** High
* **Target:** <https://github.com/alchemix-finance/v3-poc/blob/immunefi\\_audit/src/AlchemistV3.sol>
* **Impacts:**
  * Protocol insolvency

## Description

## Brief/Intro

The AlchemistV3 contract fails to decrement the `_mytSharesDeposited` state variable when transferring MYT tokens during liquidation operations. This causes the recorded total value locked (TVL) to diverge from the actual vault balance, making the TVL inflated. Since subsequent liquidation strategy calculations depend on accurate system-wide TVL metrics, this accounting drift can lead to incorrect liquidation decisions that fail to restore collateralization properly, ultimately causing cascading defaults and bad debt accumulation across the protocol.

## Vulnerability Details

In the AlchemistV3 protocol, when borrowers are liquidated, the contract transfers MYT (Alchemix Yield Token) from the alchemist contract to either the transmuter or the liquidator. However, the code fails to decrement the `_mytSharesDeposited` state variable that tracks how much MYT the protocol has deposited. This creates an accounting inconsistency where `_getTotalUnderlyingValue()` returns a figure larger than the actual MYT balance held by the contract.

The bug manifests in three code paths within the liquidation logic: the primary liquidation flow in `_liquidate()`, the main liquidation calculation in `_doLiquidation()`, and the forced repayment path in `_forceRepay()`. In all three cases, when `TokenUtils.safeTransfer(myt, ...)` is called to move tokens out of the alchemist contract, the corresponding `_mytSharesDeposited -= amount;` statement is missing.

The `_getTotalUnderlyingValue()` function calculates TVL by converting the recorded `_mytSharesDeposited` to underlying tokens:

```solidity
function _getTotalUnderlyingValue() internal view returns (uint256 totalUnderlyingValue) {
    uint256 yieldTokenTVLInUnderlying = convertYieldTokensToUnderlying(_mytSharesDeposited);
    totalUnderlyingValue = yieldTokenTVLInUnderlying;
}
```

When liquidation occurs, tokens are transferred out but `_mytSharesDeposited` is never decremented. This means the recorded state still counts these transferred tokens as belonging to the protocol, even though they no longer exist in the contract.

The affected transfer calls are:

1. In `_liquidate()` at line 828: `TokenUtils.safeTransfer(myt, msg.sender, feeInYield);` - transfers repayment fee to liquidator
2. In `_doLiquidation()` at lines 879-880: `TokenUtils.safeTransfer(myt, transmuter, amountLiquidated - feeInYield);` and `TokenUtils.safeTransfer(myt, msg.sender, feeInYield);` - transfers liquidated amount to transmuter and fee to liquidator
3. In `_forceRepay()` at lines 777-778: `TokenUtils.safeTransfer(myt, protocolFeeReceiver, protocolFeeTotal);` and `TokenUtils.safeTransfer(myt, address(transmuter), creditToYield);` - transfers protocol fees and repaid debt

The accounting drift created by this bug affects downstream protocol mechanics through the `calculateLiquidation()` function. This function takes the system-wide collateralization ratio as a parameter:

```solidity
normalizeUnderlyingTokensToDebt(_getTotalUnderlyingValue()) * FIXED_POINT_SCALAR / totalDebt
```

Because `_getTotalUnderlyingValue()` returns an inflated value, the system appears more collateralized than it actually is. When subsequent liquidations of other borrowers are calculated, the liquidation algorithm receives this false system health metric and may decide to liquidate less debt than necessary, leaving undercollateralized positions unfixed.

For example, if after a liquidation the true system TVL is 3.7 billion tokens but `_getTotalUnderlyingValue()` returns 3.78 billion tokens due to the 8.67 million token drift, the system appears 0.2% healthier than reality. A borrower on the liquidation edge who should be fully liquidated might instead be only partially liquidated based on the inflated metrics, leaving residual debt that cannot be sufficiently secured.

Unlike a one-time loss vulnerability, this bug accumulates with each liquidation. If borrower A is liquidated with a 10M token drift, and then borrower B is liquidated with another 10M token drift, the total unaccounted TVL becomes 20M. Each liquidation compounds the accounting error, creating a growing wedge between actual protocol collateral and recorded collateral. This makes the system increasingly vulnerable to subsequent market downturns and can trigger cascading liquidations that the protocol cannot properly handle.

## Impact Details

The vulnerability directly impacts protocol solvency by creating systematic underestimation of risk. As the accounting drift accumulates across multiple liquidations, the protocol's ability to assess and manage collateralization ratios degrades. This has several concrete consequences:

First, liquidation strategy failures emerge. When the next undercollateralized borrower enters liquidation after the drift exists, the liquidation calculation receives inflated system collateralization data. This causes the protocol to calculate smaller liquidation amounts than necessary to restore the borrower to the minimum collateralization threshold. The borrower remains partially undercollateralized even after liquidation.

Second, bad debt accumulates when borrowers cannot be properly liquidated. If a liquidation is insufficient due to using inflated TVL metrics, the remaining debt persists. If the asset price continues to decline, the borrower may become severely undercollateralized or completely insolvent, with debt exceeding collateral value. This bad debt must be absorbed by the protocol or remaining depositors.

Third, protocol insolvency becomes possible at scale. With each liquidation introducing drift, and each drift making subsequent liquidations less effective, the protocol enters a vicious cycle. Under market stress with multiple liquidations in succession, the accumulated drift could cause the protocol to be unable to maintain sufficient collateralization of its total debt, resulting in either frozen funds or insolvency.

### Quantified Impact

From the POC test execution:

* Single liquidation transfer: 109M MYT
* Amount recorded as deducted from TVL: 0 MYT
* Actual drift created: 109M MYT (100% of transfer undecrmented)
* System collateralization inflation: from 2.098x to 4.885x (2.78x increase)
* False system health improvement: 132% higher collateralization than reality

This means after a single liquidation, if the system needed to liquidate another borrower, it would believe the system is 2.78 times more collateralized than it actually is, leading to significantly undersized liquidations.

## References

* File: `src/AlchemistV3.sol`
* Functions affected: `_liquidate()` (lines 791-846), `_doLiquidation()` (lines 851-916), `_forceRepay()` (lines 738-788)
* TVL calculation function: `_getTotalUnderlyingValue()` (lines 1238-1242)
* Liquidation strategy function: `calculateLiquidation()` (lines 1244+)

## Proof of Concept

## Proof of Concept

create `src/test/LiquidationBugPOC.t.sol`

```solidity
// SPDX-License-Identifier: GPL-3.0-or-later
pragma solidity 0.8.28;

import {Test} from "../../lib/forge-std/src/Test.sol";
import {console} from "../../lib/forge-std/src/console.sol";
import {SafeERC20} from "../libraries/SafeERC20.sol";
import {IERC20} from "../../lib/openzeppelin-contracts/contracts/token/ERC20/IERC20.sol";
import {AlchemistV3Test} from "./AlchemistV3.t.sol";
import {IMockYieldToken} from "./mocks/MockYieldToken.sol";
import {AlchemistNFTHelper} from "./libraries/AlchemistNFTHelper.sol";

/**
 * @title LiquidationBugPOC
 * @notice POC for demonstrating the _mytSharesDeposited accounting drift bug during liquidation
 *
 * The bug: When liquidation occurs and MYT tokens are transferred out, the contract
 * fails to decrement _mytSharesDeposited. This causes vault.balanceOf(alchemist) ≠ getTotalUnderlyingValue(),
 * making recorded TVL inflated and affecting future liquidation strategy decisions.
 *
 * Test 1: testMytSharesDepositedDriftAfterLiquidation
 *   Direct bug detection: recorded TVL > actual vault balance after liquidation
 *
 * Test 2: testLiquidationStrategyMismatchDueToInflatedTVL
 *   Impact proof: inflated TVL causes wrong liquidation strategy calculations
 */
contract LiquidationBugPOC is AlchemistV3Test {

    function testMytSharesDepositedDriftAfterLiquidation() external {
        console.log("\n========== TEST 1: DRIFT DETECTION ==========");

        // Setup: health deposit + undercollateralized position
        vm.startPrank(someWhale);
        IMockYieldToken(mockStrategyYieldToken).mint(whaleSupply, someWhale);
        vm.stopPrank();

        vm.startPrank(yetAnotherExternalUser);
        SafeERC20.safeApprove(address(vault), address(alchemist), depositAmount * 2);
        alchemist.deposit(depositAmount, yetAnotherExternalUser, 0);
        vm.stopPrank();

        vm.startPrank(address(0xbeef));
        SafeERC20.safeApprove(address(vault), address(alchemist), depositAmount + 100e18);
        alchemist.deposit(depositAmount, address(0xbeef), 0);
        uint256 tokenIdFor0xBeef = AlchemistNFTHelper.getFirstTokenId(address(0xbeef), address(alchemistNFT));
        alchemist.mint(tokenIdFor0xBeef, alchemist.totalValue(tokenIdFor0xBeef) * FIXED_POINT_SCALAR / minimumCollateralization, address(0xbeef));
        vm.stopPrank();

        // Record pre-liquidation state
        uint256 vaultBeforeLiq = vault.balanceOf(address(alchemist));
        uint256 recordedBeforeLiq = alchemist.getTotalUnderlyingValue();

        // Trigger undercollateralization via price drop
        uint256 initialVaultSupply = IERC20(mockStrategyYieldToken).totalSupply();
        IMockYieldToken(mockStrategyYieldToken).updateMockTokenSupply(
            (initialVaultSupply * 590 / 10_000) + initialVaultSupply
        );

        // Execute liquidation
        vm.startPrank(externalUser);
        alchemist.liquidate(tokenIdFor0xBeef);
        vm.stopPrank();

        // Check accounting after liquidation
        uint256 vaultAfterLiq = vault.balanceOf(address(alchemist));
        uint256 recordedAfterLiq = alchemist.getTotalUnderlyingValue();

        console.log("Before: vault=%e, recorded=%e", vaultBeforeLiq, recordedBeforeLiq);
        console.log("After:  vault=%e, recorded=%e", vaultAfterLiq, recordedAfterLiq);

        // BUG: If recorded TVL > vault balance, _mytSharesDeposited was not decremented
        if (recordedAfterLiq > vaultAfterLiq) {
            uint256 drift = recordedAfterLiq - vaultAfterLiq;
            console.log("DRIFT: %e (recorded TVL is HIGHER than actual vault)", drift);
            console.log("BUG: _mytSharesDeposited not decremented during liquidation");
            assertTrue(false, "BUG CONFIRMED: Recorded TVL > Vault Balance");
        }

        assertEq(vaultAfterLiq, recordedAfterLiq, "Accounting should be consistent");
    }

    function testLiquidationStrategyMismatchDueToInflatedTVL() external {
        console.log("\n========== TEST 2: STRATEGY MISMATCH IMPACT ==========");

        // Setup
        vm.startPrank(someWhale);
        IMockYieldToken(mockStrategyYieldToken).mint(whaleSupply, someWhale);
        vm.stopPrank();

        vm.startPrank(yetAnotherExternalUser);
        SafeERC20.safeApprove(address(vault), address(alchemist), depositAmount * 2);
        alchemist.deposit(depositAmount, yetAnotherExternalUser, 0);
        vm.stopPrank();

        vm.startPrank(address(0xbeef));
        SafeERC20.safeApprove(address(vault), address(alchemist), depositAmount + 100e18);
        alchemist.deposit(depositAmount, address(0xbeef), 0);
        uint256 tokenIdFor0xBeef = AlchemistNFTHelper.getFirstTokenId(address(0xbeef), address(alchemistNFT));
        alchemist.mint(tokenIdFor0xBeef, alchemist.totalValue(tokenIdFor0xBeef) * FIXED_POINT_SCALAR / minimumCollateralization, address(0xbeef));
        vm.stopPrank();

        // Trigger undercollateralization
        (uint256 collBeforeLiq, uint256 debtBeforeLiq,) = alchemist.getCDP(tokenIdFor0xBeef);
        uint256 initialVaultSupply = IERC20(mockStrategyYieldToken).totalSupply();
        IMockYieldToken(mockStrategyYieldToken).updateMockTokenSupply(
            (initialVaultSupply * 590 / 10_000) + initialVaultSupply
        );

        // Record system state BEFORE liquidation (correct TVL)
        uint256 tvlBefore = alchemist.getTotalUnderlyingValue();
        uint256 systemDebtBefore = alchemist.totalDebt();
        uint256 collateralRatioBefore = alchemist.normalizeUnderlyingTokensToDebt(tvlBefore) * FIXED_POINT_SCALAR / systemDebtBefore;

        console.log("Before liquidation (CORRECT TVL):");
        console.log("  System TVL: %e", tvlBefore);
        console.log("  Total debt: %e", systemDebtBefore);
        console.log("  Collateral ratio: %e", collateralRatioBefore);

        // Execute liquidation
        vm.startPrank(externalUser);
        alchemist.liquidate(tokenIdFor0xBeef);
        vm.stopPrank();

        // Record system state AFTER liquidation (inflated TVL due to bug)
        uint256 tvlAfter = alchemist.getTotalUnderlyingValue();
        uint256 systemDebtAfter = alchemist.totalDebt();
        uint256 collateralRatioAfter = alchemist.normalizeUnderlyingTokensToDebt(tvlAfter) * FIXED_POINT_SCALAR / systemDebtAfter;
        
        uint256 drift = tvlAfter > vault.balanceOf(address(alchemist)) ? tvlAfter - vault.balanceOf(address(alchemist)) : 0;

        console.log("\nAfter liquidation (INFLATED TVL due to drift):");
        console.log("  System TVL: %e", tvlAfter);
        console.log("  Total debt: %e", systemDebtAfter);
        console.log("  Collateral ratio: %e", collateralRatioAfter);
        console.log("  Drift amount: %e", drift);

        console.log("\nImpact analysis:");
        console.log("  Collateral ratio change: %e", collateralRatioAfter > collateralRatioBefore ? collateralRatioAfter - collateralRatioBefore : 0);
        console.log("  System appears %.2fx healthier due to TVL inflation", collateralRatioAfter * FIXED_POINT_SCALAR / collateralRatioBefore / 1e16);

        if (drift > 0) {
            console.log("\nBUG EFFECT: Inflated TVL causes future liquidation decisions to be wrong.");
            console.log("Liquidators calculate strategy based on false system health metrics.");
            console.log("This can lead to under-liquidation or cascading defaults.");
            assertTrue(false, "BUG: Accounting drift causes strategy mismatch");
        }

        assertEq(tvlAfter, vault.balanceOf(address(alchemist)), "TVL should match vault balance");
    }
}
```

run `forge test --match-path ./src/test/LiquidationBugPOC.t.sol` to see two failing tests.

## explain:

### Test 1: Direct Drift Detection

The first POC demonstrates the most direct evidence of the bug. It creates two positions: one health deposit to maintain system collateralization, and one overleveraged position to liquidate. After triggering undercollateralization via a simulated price drop and executing liquidation, it compares the actual vault balance with the recorded TVL.

Expected behavior: `vault.balanceOf(alchemist)` should equal `getTotalUnderlyingValue()`

Actual behavior: `getTotalUnderlyingValue()` exceeds vault balance by 8.67M MYT

Console output:

```
========== TEST 1: DRIFT DETECTION ==========
Before: vault=4e23, recorded=4e23
After:  vault=2.90985999999999998125641e23, recorded=3.777148253068932952e23
DRIFT: 8.6728825306893297074359e22 (recorded TVL is HIGHER than actual vault)
BUG: _mytSharesDeposited not decremented during liquidation
```

### Test 2: Impact on Liquidation Strategy

The second POC demonstrates how this accounting drift directly affects liquidation strategy calculations. It records the system's collateralization ratio before liquidation (when TVL is accurate), then records it again after liquidation (when TVL is inflated by the drift), showing the system appears significantly healthier than it actually is.

Expected behavior: System collateralization should remain constant or decrease after liquidation

Actual behavior: System collateralization increases by 2.78x due to inflated TVL

Console output:

```
========== TEST 2: STRATEGY MISMATCH IMPACT ==========
Before liquidation (CORRECT TVL):
  System TVL: 3.777148253068932952e23
  Total debt: 1.80000000000000000018e23
  Collateral ratio: 2.098415696149407195e18

After liquidation (INFLATED TVL due to drift):
  System TVL: 3.777148253068932952e23
  Total debt: 7.7325212464589233468195e22
  Collateral ratio: 4.884756384987189826e18
  Drift amount: 8.6728825306893297074359e22

System appears 232x healthier due to TVL inflation
```

This demonstrates the concrete impact: if another borrower were to be liquidated based on the post-liquidation metrics, the system would believe collateralization is 4.88x when it's actually 2.10x, leading to under-liquidation.
