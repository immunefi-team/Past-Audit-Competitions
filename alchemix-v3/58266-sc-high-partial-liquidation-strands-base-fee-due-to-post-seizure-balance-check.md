# 58266 sc high partial liquidation strands base fee due to post seizure balance check

**Submitted on Oct 31st 2025 at 20:32:37 UTC by @zcai for** [**Audit Comp | Alchemix V3**](https://immunefi.com/audit-competition/alchemix-v3-audit-competition)

* **Report ID:** #58266
* **Report Type:** Smart Contract
* **Report severity:** High
* **Target:** <https://github.com/alchemix-finance/v3-poc/blob/immunefi\\_audit/src/AlchemistV3.sol>
* **Impacts:**
  * Permanent freezing of funds

## Description

## Brief/Intro

The liquidation flow contains a critical sequencing issue where collateral seizure occurs before fee validation, causing base liquidation fees to be stranded in the contract when the borrower's remaining collateral is insufficient.

## Vulnerability Details

The `_doLiquidation()` function implements a problematic sequence of operations that creates an atomicity failure in fee handling. The function first seizes the gross collateral amount (including the base fee) from the borrower's account and transfers the net amount to the transmuter. Only after these irreversible state changes does it attempt to validate and pay the base fee to the liquidator.

The vulnerability manifests in the conditional fee transfer logic:

```solidity
// Line 871: Collateral is seized first
account.collateralBalance = account.collateralBalance > amountLiquidated ? account.collateralBalance - amountLiquidated : 0;

// Line 875: Net amount sent to transmuter  
TokenUtils.safeTransfer(myt, transmuter, amountLiquidated - feeInYield);

// Line 878-880: Fee transfer is conditional on post-seizure balance
if (feeInYield > 0 && account.collateralBalance >= feeInYield) {
    TokenUtils.safeTransfer(myt, msg.sender, feeInYield);
}
```

When the borrower's remaining collateral after seizure is less than the calculated fee amount, the conditional check fails and the fee transfer is silently skipped. Since the gross collateral (including fee) was already debited from the borrower's account, the fee amount becomes stranded in the contract with no mechanism for recovery or reconciliation.

This occurs frequently in partial liquidations where the gross seizure amount consumes most of the user's available collateral, leaving insufficient balance for the subsequent fee validation check. The `calculateLiquidation()` function properly computes the fee as part of the gross seizure amount, but the implementation fails to ensure atomic execution of the entire liquidation flow.

## Impact Details

The vulnerability causes direct financial loss to borrowers whose collateral is reduced by the full gross liquidation amount while the corresponding base fee is neither paid to the liquidator nor returned to their account. The stranded funds remain locked in the contract without any recovery mechanism, representing a permanent loss for the affected borrowers. Additionally, liquidators receive reduced compensation when fees are skipped, potentially undermining the economic incentives that ensure timely liquidation of undercollateralized positions and threatening the overall stability of the liquidation system.

## References

<https://github.com/alchemix-finance/v3-poc/blob/immunefi\\_audit/src/AlchemistV3.sol?utm\\_source=immunefi#L852-L894>

## Link to Proof of Concept

<https://gist.github.com/i-am-zcai/d9c1db2875cc80255e19341a03a2d227>

## Proof of Concept

## Proof of Concept

// PoC: Demonstrate stranded liquidation fee when post-seizure borrower collateral is insufficient to pay base fee contract AlchemistV3\_LiquidationStrandedFee\_PoC is AlchemistV3Test { function test\_PoC\_LiquidationStrandsFeeWhenPostSeizureCollateralInsufficient() external { // Configure liquidator fee to 100% of surplus to deterministically force gross seize == full collateral vm.startPrank(alOwner); alchemist.setLiquidatorFee(10\_000); // 100% of surplus vm.stopPrank();

```
    // Ensure some extra collateral exists in the system to keep global collateralization above global minimum
    vm.startPrank(yetAnotherExternalUser);
    SafeERC20.safeApprove(address(vault), address(alchemist), depositAmount);
    alchemist.deposit(depositAmount, yetAnotherExternalUser, 0);
    vm.stopPrank();

    // Borrower deposits and mints at max capacity
    vm.startPrank(address(0xbeef));
    SafeERC20.safeApprove(address(vault), address(alchemist), depositAmount);
    alchemist.deposit(depositAmount, address(0xbeef), 0);
    uint256 tokenId = AlchemistNFTHelper.getFirstTokenId(address(0xbeef), address(alchemistNFT));
    // Mint up to max based on current minimumCollateralization
    uint256 maxBorrow = alchemist.totalValue(tokenId) * FIXED_POINT_SCALAR / alchemist.minimumCollateralization();
    alchemist.mint(tokenId, maxBorrow, address(0xbeef));
    vm.stopPrank();

    // Move share price down to push position below collateralizationLowerBound (simulate yield token depeg)
    vm.startPrank(someWhale);
    IMockYieldToken(mockStrategyYieldToken).mint(whaleSupply, someWhale);
    vm.stopPrank();
    // Nudge supply to create price change
    uint256 initialSupply = IERC20(address(mockStrategyYieldToken)).totalSupply();
    IMockYieldToken(mockStrategyYieldToken).updateMockTokenSupply(initialSupply);
    uint256 modifiedSupply = initialSupply + ((initialSupply * 590) / 10_000); // +5.9%
    IMockYieldToken(mockStrategyYieldToken).updateMockTokenSupply(modifiedSupply);

    // Preconditions: account is now liquidatable
    (uint256 collBeforeDebtUnits, uint256 debtBefore,) = alchemist.getCDP(tokenId);
    // Convert the returned collateral (which is in yield tokens) to debt units for calculation parity
    uint256 collBefore = alchemist.convertYieldTokensToDebt(collBeforeDebtUnits);

    // Compute expected liquidation terms at this moment
    uint256 alchemistCurrentCollat = alchemist.normalizeUnderlyingTokensToDebt(alchemist.getTotalUnderlyingValue())
        * FIXED_POINT_SCALAR / alchemist.totalDebt();
    (uint256 liquidationAmountDebt, uint256 debtToBurn, uint256 baseFeeDebt,) = alchemist.calculateLiquidation(
        alchemist.totalValue(tokenId),
        debtBefore,
        alchemist.minimumCollateralization(),
        alchemistCurrentCollat,
        alchemist.globalMinimumCollateralization(),
        alchemist.liquidatorFee()
    );

    // With 100% fee, we expect gross seize to equal entire collateral and debtToBurn == full debt
    assertGt(baseFeeDebt, 0, "base fee must be > 0");
    // liquidator gets fee only if leftover collateral >= fee; we will show it doesn't

    // Snapshot pre-balances
    uint256 liquidatorBalBefore = IERC20(address(vault)).balanceOf(externalUser);
    uint256 contractBalBefore = IERC20(address(vault)).balanceOf(address(alchemist));
    uint256 transmuterBalBefore = IERC20(address(vault)).balanceOf(address(transmuterLogic));

    // Execute liquidation by an external liquidator
    vm.startPrank(externalUser);
    (uint256 totalAmountLiquidatedYield, uint256 feeInYieldReturned, uint256 feeInUnderlying) = alchemist.liquidate(tokenId);
    vm.stopPrank();

    // Post-state
    (uint256 collAfterYield, uint256 debtAfter,) = alchemist.getCDP(tokenId);
    uint256 liquidatorBalAfter = IERC20(address(vault)).balanceOf(externalUser);
    uint256 contractBalAfter = IERC20(address(vault)).balanceOf(address(alchemist));
    uint256 transmuterBalAfter = IERC20(address(vault)).balanceOf(address(transmuterLogic));

    // 1) Liquidator fee was reported as positive, but not actually paid due to insufficient post-seizure collateral
    assertGt(feeInYieldReturned, 0, "expected a positive base fee");
    assertEq(feeInUnderlying, 0, "no outsourced underlying fee expected in this scenario");
    assertEq(liquidatorBalAfter - liquidatorBalBefore, 0, "liquidator did not receive base fee (stranded)");

    // 2) Entire borrower collateral was seized (collateral dropped to ~0), but only net (without fee) left the contract
    // With 100% fee, liquidationAmount in debt units equals full collateral in debt units
    uint256 collDeltaYield = collBeforeDebtUnits - collAfterYield; // seized from borrower internal accounting
    // Contract actual balance delta equals amount sent out (net to transmuter, zero to liquidator)
    uint256 contractDelta = contractBalBefore - contractBalAfter;
    // Net sent to transmuter equals converted debtToBurn; with 100% fee, debtToBurn == full debtBefore
    uint256 expectedNetToTransmuter = alchemist.convertDebtTokensToYield(debtToBurn);
    assertApproxEqAbs(transmuterBalAfter - transmuterBalBefore, expectedNetToTransmuter, 1e6, "net to transmuter mismatch");

    // The difference between what was seized from the borrower and what actually left the contract equals the stranded fee
    uint256 stranded = collDeltaYield - contractDelta;
    assertApproxEqAbs(stranded, feeInYieldReturned, 1e6, "stranded amount equals unpaid base fee");

    // 3) Position essentially fully repaid but lost the fee from collateral; allow minor rounding remainder
    assertLt(debtAfter, 1e4, "debt should be fully repaid within rounding dust");
}
```

}

```

**Command:**

```

forge test --match-path src/test/poc/AlchemistV3\_LiquidationStrandedFee.t.sol --match-test test\_PoC\_LiquidationStrandsFeeWhenPostSeizureCollateralInsufficient -vvv

```
```
