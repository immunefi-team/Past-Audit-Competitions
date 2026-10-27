# 58269 sc high liquidator fee not paid when fee equals surplus

**Submitted on Oct 31st 2025 at 21:05:23 UTC by @xanony for** [**Audit Comp | Alchemix V3**](https://immunefi.com/audit-competition/alchemix-v3-audit-competition)

* **Report ID:** #58269
* **Report Type:** Smart Contract
* **Report severity:** High
* **Target:** <https://github.com/alchemix-finance/v3-poc/blob/immunefi\\_audit/src/AlchemistV3.sol>
* **Impacts:**
  * Smart contract unable to operate due to lack of token funds

## Description

## Brief/Intro

When liquidating an undercollateralized position in AlchemistV3, the protocol calculates a base liquidator fee as a percentage of the position's surplus (`collateral - debt`). However, due to an incorrect execution order in the fee payout logic, **liquidators receive zero tokens** when the calculated fee equals or exceeds the remaining collateral balance after seizure.

This vulnerability breaks the economic incentive mechanism for liquidations, potentially causing:

* Delayed or missed liquidations during market stress
* Accumulation of bad debt in the protocol
* Loss of liquidator confidence in the system

## Vulnerability Details

## Vulnerability Details

### Affected Component

* **Contract**: `src/AlchemistV3.sol`
* **Function**: `_doLiquidation()` (lines 852-900)
* **Vulnerable Code**: Line 878

### Root Cause

The bug occurs due to incorrect sequencing in `_doLiquidation()`:

```solidity
function _doLiquidation(uint256 accountId, uint256 collateralInUnderlying, uint256 repaidAmountInYield)
    internal
    returns (uint256 amountLiquidated, uint256 feeInYield, uint256 feeInUnderlying)
{
    Account storage account = _accounts[accountId];

    // Calculate liquidation parameters
    (uint256 liquidationAmount, uint256 debtToBurn, uint256 baseFee, uint256 outsourcedFee) = calculateLiquidation(...);

    amountLiquidated = convertDebtTokensToYield(liquidationAmount);
    feeInYield = convertDebtTokensToYield(baseFee);

    // BUG: Deduct FULL liquidation amount (including fee) FIRST
    account.collateralBalance = account.collateralBalance > amountLiquidated 
        ? account.collateralBalance - amountLiquidated : 0;
    _subDebt(accountId, debtToBurn);

    // Send net collateral to transmuter
    TokenUtils.safeTransfer(myt, transmuter, amountLiquidated - feeInYield);

    // BUG: Check if remaining balance >= fee (FAILS when balance was depleted above)
    if (feeInYield > 0 && account.collateralBalance >= feeInYield) {
        TokenUtils.safeTransfer(myt, msg.sender, feeInYield);
    }
    // ...
}
```

**The Problem**:

1. `amountLiquidated = debtToBurn + fee` (gross seizure amount)
2. Line 870: Account balance reduced by **full** `amountLiquidated`
3. Line 878: Check `account.collateralBalance >= feeInYield` **fails** because balance was already depleted
4. Result: Liquidator receives 0 tokens despite protocol calculating a non-zero fee

## Impact Details

### Direct Impact

* **Broken Incentives**: Liquidators receive zero compensation despite protocol promises
* **Delayed Liquidations**: Unprofitable liquidations ignored, allowing bad debt accumulation
* **System Instability**: Lack of timely liquidations during market volatility can threaten solvency

### Funds at Risk

Per-liquidation impact:

* Typical scenario: Position with $100k collateral, $95k debt → $5k surplus
* At 1% fee: $50 lost per liquidation
* At 5% fee: $250 lost per liquidation
* At 100% fee: $5,000 lost per liquidation

Systemic risk: If liquidations are delayed due to broken incentives, bad debt could accumulate to millions during volatile markets.

## References

* Vulnerable contract: `https://github.com/alchemix-finance/v3-poc/blob/immunefi_audit/src/AlchemistV3.sol#L1-L1327`
* Bug location: Line 878 (`_doLiquidation`)
* Fee calculation: Lines 1244-1288 (`calculateLiquidation`)

## Link to Proof of Concept

<https://gist.github.com/leojay-net/8d034edf48bb186ad76e3159dff42eb5>

## Proof of Concept

## Proof of Concept

```solidity
// SPDX-License-Identifier: MIT
pragma solidity 0.8.28;

import {AlchemistV3Test} from "./AlchemistV3.t.sol";
import {SafeERC20} from "../libraries/SafeERC20.sol";
import {IERC20} from "../../lib/openzeppelin-contracts/contracts/token/ERC20/IERC20.sol";
import {IMockYieldToken} from "./mocks/MockYieldToken.sol";
import {AlchemistNFTHelper} from "./libraries/AlchemistNFTHelper.sol";

contract TestBug_LiquidatorFeeNotPaidWhenFeeEqualsSurplus_FullBurn is AlchemistV3Test {
    function testBug_LiquidatorFeeNotPaidWhenFeeEqualsSurplus() external {
        // Arrange: set liquidator fee to 100% of surplus
        vm.startPrank(alOwner);
        alchemist.setLiquidatorFee(10_000);
        vm.stopPrank();

        // Fund whale and deposit for yetAnotherExternalUser to keep global collateralization healthy
        uint256 amount = 200_000e18;
        vm.startPrank(someWhale);
        IMockYieldToken(mockStrategyYieldToken).mint(whaleSupply, someWhale);
        vm.stopPrank();

        vm.startPrank(yetAnotherExternalUser);
        SafeERC20.safeApprove(address(vault), address(alchemist), amount);
        alchemist.deposit(amount, yetAnotherExternalUser, 0);
        vm.stopPrank();

        // Create the victim position: deposit and mint close to max
        vm.startPrank(address(0xbeef));
        SafeERC20.safeApprove(address(vault), address(alchemist), amount + 100e18);
        alchemist.deposit(amount, address(0xbeef), 0);
        uint256 tokenIdFor0xBeef = AlchemistNFTHelper.getFirstTokenId(address(0xbeef), address(alchemistNFT));
        // Mint up to the limit (min collateralization), so any price drop pushes into liquidation
        alchemist.mint(
            tokenIdFor0xBeef,
            (alchemist.totalValue(tokenIdFor0xBeef) * FIXED_POINT_SCALAR) / minimumCollateralization,
            address(0xbeef)
        );
        vm.stopPrank();

        // Induce undercollateralization by reducing price (increase share supply in mock)
        uint256 initialVaultSupply = IERC20(address(mockStrategyYieldToken)).totalSupply();
        IMockYieldToken(mockStrategyYieldToken).updateMockTokenSupply(initialVaultSupply);
        uint256 modifiedVaultSupply = ((initialVaultSupply * 590) / 10_000) + initialVaultSupply; // ~+5.9%
        IMockYieldToken(mockStrategyYieldToken).updateMockTokenSupply(modifiedVaultSupply);

        // Compute expected liquidation values
        (, uint256 prevDebt, ) = alchemist.getCDP(tokenIdFor0xBeef);
        uint256 collateralInDebt = alchemist.totalValue(tokenIdFor0xBeef);
        uint256 alchemistCurrentCollat = (alchemist.normalizeUnderlyingTokensToDebt(
            alchemist.getTotalUnderlyingValue()
        ) * FIXED_POINT_SCALAR) / alchemist.totalDebt();

        (uint256 liquidationAmountDebt, uint256 debtToBurn, uint256 baseFeeDebt, ) = alchemist.calculateLiquidation(
            collateralInDebt,
            prevDebt,
            alchemist.minimumCollateralization(),
            alchemistCurrentCollat,
            alchemist.globalMinimumCollateralization(),
            10_000 // 100% fee on surplus
        );

        // Sanity: expect non-zero base fee; debtToBurn ~ prevDebt (allow rounding tolerance)
        assertGt(baseFeeDebt, 0, "base fee > 0");
        assertApproxEqAbs(debtToBurn, prevDebt, 1e9, "full debt burn");

        // Act: liquidate
        vm.startPrank(externalUser);
        uint256 liquidatorPrevTokenBalance = IERC20(address(vault)).balanceOf(address(externalUser));
        (uint256 assetsSent, uint256 feeInYield, ) = alchemist.liquidate(tokenIdFor0xBeef);
        uint256 liquidatorPostTokenBalance = IERC20(address(vault)).balanceOf(address(externalUser));
        vm.stopPrank();

        // Assert: feeInYield is reported non-zero, but NOT actually paid to the liquidator due to post-deduction balance check
        assertGt(feeInYield, 0, "expected non-zero feeInYield");
        assertEq(liquidatorPostTokenBalance, liquidatorPrevTokenBalance, "liquidator did not receive base fee tokens");

        // And all seized tokens went to transmuter only (assetsSent is the liquidation amount in yield)
        // i.e., fee got stranded inside the Alchemist instead of being paid out
        assertApproxEqAbs(
            assetsSent,
            alchemist.convertDebtTokensToYield(liquidationAmountDebt),
            1,
            "all seized yield forwarded (no base fee paid)"
        );
    }
}

```
