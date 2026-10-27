# 58742 sc high liquidators will not earn fees in some cases

**Submitted on Nov 4th 2025 at 11:10:34 UTC by @Lion47624 for** [**Audit Comp | Alchemix V3**](https://immunefi.com/audit-competition/alchemix-v3-audit-competition)

* **Report ID:** #58742
* **Report Type:** Smart Contract
* **Report severity:** High
* **Target:** <https://github.com/alchemix-finance/v3-poc/blob/immunefi\\_audit/src/AlchemistV3.sol>
* **Impacts:**
  * Permanent freezing of unclaimed yield

## Description

## Brief/Intro

A logical flaw exists within the \_doLiquidation function in the AlchemistV3.sol contract. The check to determine if a liquidator's fee can be paid is incorrectly performed against the liquidated user's remaining collateral after the seized collateral has already been deducted. In scenarios where a liquidation leaves the user with little or no collateral, this check fails, preventing the liquidator from receiving their rightful fee and potentially disincentivizing the liquidation process.

## Vulnerability Details

When a position is liquidated, the `_doLiquidation` function is called. The sequence of operations is as follows:

1. `calculateLiquidation` is called to determine `amountLiquidated` (the total collateral to seize, including the fee) and `feeInYield` (the portion of the seized collateral that serves as the liquidator's fee).
2. The user's collateral balance is reduced by the full `amountLiquidated`:

   ```solidity
   account.collateralBalance = account.collateralBalance > amountLiquidated ? account.collateralBalance - amountLiquidated : 0;
   ```
3. The contract then attempts to pay the liquidator's fee with the following check:

   ```solidity
   if (feeInYield > 0 && account.collateralBalance >= feeInYield)
   ```

The vulnerability lies in this check. It compares `feeInYield` to the user's `collateralBalance` *after* it has been debited. If the liquidation consumes most or all of the user's collateral, the remaining balance will be less than `feeInYield`, causing the check to fail and the fee transfer to be skipped. The fee is part of the `amountLiquidated` that was already seized by the contract and should be paid from that amount, not from the user's remaining balance.

## Impact Details

This bug leads to a loss of funds for liquidators. By failing to pay the fee, the protocol disincentivizes liquidators from maintaining the health of the system. A lack of liquidators could lead to an accumulation of undercollateralized debt, increasing the overall risk to the protocol. The unpaid fees remain locked within the Alchemist contract.

## References

<https://github.com/alchemix-finance/v3-poc/blob/a192ab313c81ba3ab621d9ca1ee000110fbdd1e9/src/AlchemistV3.sol#L867-L880>

## Proof of Concept

## Proof of Concept

This test demonstrates the liquidation fee vulnerability by creating a position that is almost entirely wiped out, and then asserts that the liquidator is not paid their fee despite one being calculated. Copy and add the following test in the AlchemixV3.t.sol file and run forge test --mt testLiquidate\_POC\_Check -vvv to view the logs.

```solidity
function testLiquidate_POC_Check() external {
        // --- 1. Setup ---
        // Set a high liquidator fee to make the effect more pronounced.
        vm.startPrank(alOwner);
        alchemist.setLiquidatorFee(9500);
        vm.stopPrank();

        // Provide liquidity to the system via a whale.
        vm.startPrank(someWhale);
        IMockYieldToken(mockStrategyYieldToken).mint(whaleSupply, someWhale);
        vm.stopPrank();
        // Create a healthy position to ensure the system's global collateralization remains stable.
        vm.startPrank(yetAnotherExternalUser);
        SafeERC20.safeApprove(address(vault), address(alchemist), depositAmount * 2);
        alchemist.deposit(depositAmount, yetAnotherExternalUser, 0);
        vm.stopPrank();

        // --- 2. Create Vulnerable Position ---
        // A user (0xbeef) deposits collateral and borrows the maximum amount possible.
        // This creates a position with a collateralization ratio exactly at the minimum requirement,
        // making it highly sensitive to price drops.
        vm.startPrank(address(0xbeef));
        SafeERC20.safeApprove(address(vault), address(alchemist), depositAmount + 100e18);
        alchemist.deposit(depositAmount, address(0xbeef), 0);
        uint256 tokenIdFor0xBeef = AlchemistNFTHelper.getFirstTokenId(address(0xbeef), address(alchemistNFT));
        alchemist.mint(tokenIdFor0xBeef, alchemist.totalValue(tokenIdFor0xBeef) * FIXED_POINT_SCALAR / minimumCollateralization, address(0xbeef));
        vm.stopPrank();

        // --- 3. Manipulate Price ---
        // Devalue the collateral asset significantly to trigger a large partial liquidation.
        // An 11.1% price drop (1110 bps increase in yield token supply) is chosen to leave the
        // position barely solvent, ensuring the liquidation seizes almost all collateral.
        (uint256 prevCollateral, uint256 prevDebt,) = alchemist.getCDP(tokenIdFor0xBeef);
        uint256 initialVaultSupply = IERC20(address(mockStrategyYieldToken)).totalSupply();
        IMockYieldToken(mockStrategyYieldToken).updateMockTokenSupply(initialVaultSupply);
        uint256 modifiedVaultSupply = (initialVaultSupply * 1110 / 10_000) + initialVaultSupply;
        IMockYieldToken(mockStrategyYieldToken).updateMockTokenSupply(modifiedVaultSupply);

    
        // --- 4. Liquidate the Position ---
        vm.startPrank(externalUser);
        uint256 liquidatorPrevTokenBalance = IERC20(address(vault)).balanceOf(address(externalUser));
        uint256 liquidatorPrevUnderlyingBalance = IERC20(vault.asset()).balanceOf(address(externalUser));

        // Pre-calculate what the liquidation fee *should* be. This is non-zero.
        uint256 alchemistCurrentCollateralization =
            alchemist.normalizeUnderlyingTokensToDebt(alchemist.getTotalUnderlyingValue()) * FIXED_POINT_SCALAR / alchemist.totalDebt();
        (uint256 liquidationAmount, , uint256 expectedBaseFee,) = alchemist.calculateLiquidation(
            alchemist.totalValue(tokenIdFor0xBeef),
            prevDebt,
            alchemist.minimumCollateralization(),
            alchemistCurrentCollateralization,
            alchemist.globalMinimumCollateralization(),
            liquidatorFeeBPS
        );
        uint256 expectedBaseFeeInYield = alchemist.convertDebtTokensToYield(expectedBaseFee);
       
        // Execute the liquidation.
        (uint256 assets, uint256 feeInYield, uint256 feeInUnderlying) = alchemist.liquidate(tokenIdFor0xBeef);
        vm.stopPrank();

        // --- 5. Verify the Bug ---
        // Check the liquidator's balance after the transaction. It should be unchanged.
        uint256 liquidatorTokenBalanceAfter = IERC20(address(vault)).balanceOf(address(externalUser));
        uint256 liquidatorUnderlyingAfter = IERC20(vault.asset()).balanceOf(address(externalUser));
        assertEq(liquidatorPrevUnderlyingBalance, liquidatorUnderlyingAfter, "Liquidator underlying balance should not change");
        assertEq(liquidatorPrevTokenBalance, liquidatorTokenBalanceAfter, "Liquidator yield token balance should not change");
       
        // The core of the POC:
        // `expectedBaseFeeInYield` is the fee that was calculated and *should* have been paid. It is > 0.
        // This proves the liquidator was not paid their fee.
        console.log("Fee that SHOULD have been paid but now stucked:", expectedBaseFeeInYield);
        assertTrue(expectedBaseFeeInYield > 0, "A non-zero fee should have been calculated.");
    }
```
