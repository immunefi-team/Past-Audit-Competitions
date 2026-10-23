# 57816 sc insight critical incentive failure in calculateliquidation leads to protocol insolvency risk during global bad debt

**Submitted on Oct 29th 2025 at 02:03:47 UTC by @fullstop for** [**Audit Comp | Alchemix V3**](https://immunefi.com/audit-competition/alchemix-v3-audit-competition)

* **Report ID:** #57816
* **Report Type:** Smart Contract
* **Report severity:** Insight
* **Target:** <https://github.com/alchemix-finance/v3-poc/blob/immunefi\\_audit/src/AlchemistV3.sol>
* **Impacts:**
  * Protocol insolvency

## Description

## Brief/Intro

A critical logic flaw exists in the AlchemistV3.sol contract's calculateLiquidation function. When the protocol becomes globally undercollateralized (a high-stress scenario), the function incorrectly sets the primary liquidator incentive (the base fee) to zero. This design fully relies on a secondary, exhaustible fee vault (alchemistFeeVault) to pay liquidators. If this secondary vault is empty—a predictable state during a market crash due to high liquidation volume—all liquidator incentives become zero. This halts all liquidation activity at the most critical time, preventing the protocol from clearing bad debt and leading to a "death spiral" that could result in total insolvency.

## Vulnerability Details

The protocol's liquidation mechanism relies on two distinct fee sources to incentivize external liquidators:

1. Base Fee (feeInYield): The primary and most reliable incentive. It is calculated based on the liquidated position's surplus and paid directly from the position's own collateral.
2. Outsourced Fee (feeInUnderlying): A secondary, supplemental incentive. It is paid from an external alchemistFeeVault contract and is not guaranteed; if the vault is empty, this fee is 0.

The vulnerability is triggered when the protocol's total collateralization (alchemistCurrentCollateralization) drops below the alchemistMinimumCollateralization threshold. This is the exact moment the protocol is in greatest danger and relies most heavily on liquidators.

In this specific "global bad debt" scenario, the calculateLiquidation function enters a flawed logical branch:

```
        if (alchemistCurrentCollateralization < alchemistMinimumCollateralization) {
            outsourcedFee = (debt * feeBps) / BPS;
            // fully liquidate debt in high ltv global environment
            return (debt, debt, 0, outsourcedFee);
        }
```

As shown, the function intentionally returns 0 as the third value, which corresponds to the Base Fee.

This logic incorrectly switches off the reliable, primary incentive (fee) and shifts 100% of the incentive burden onto the unreliable, secondary outsourcedFee.

This creates a "perfect storm" scenario:

1. A market crash causes alchemistCurrentCollateralization to fall, triggering the bug (`fee = 0`).
2. The same market crash causes a high volume of liquidations, which rapidly drains the alchemistFeeVault as it pays out the outsourcedFee for each one.
3. The alchemistFeeVault is an exhaustible buffer, not an infinite source. Once it is depleted (a predictable event in this scenario), the \_doLiquidation function's check `if (vaultBalance > 0)` will fail, and the outsourcedFee will also become 0.

At this point, both the primary and secondary incentives are 0, and rational liquidators will no longer participate.

## Impact Details

The severity of this vulnerability is Critical. It creates a "death spiral" scenario from which the protocol cannot recover.

The direct impact is the total loss of liquidator incentives during a protocol-wide crisis.

This leads to a cascading failure:

1. Market Crash: The protocol becomes globally undercollateralized.
2. Bug Triggered: The calculateLiquidation flaw sets the base fee to 0.
3. Vault Drain: High liquidation volume (from the crash) drains the alchemistFeeVault, setting the outsourced fee to 0.
4. Incentive Collapse: The total reward for liquidators becomes 0.
5. Liquidation Halt: Rational liquidator bots stop all activity, as they would lose money on gas fees.
6. Death Spiral: Bad debt (underwater positions) is no longer cleared from the system. The protocol's health (alchemistCurrentCollateralization) continues to fall, permanently locking the contract in the flawed logic branch.
7. Protocol Insolvency: Unchecked bad debt accumulates until the protocol's liabilities (debt tokens) exceed its assets, leading to a loss of funds for all users.

## References

Vulnerable Contract: AlchemistV3.sol

Flawed Function (calculateLiquidation): AlchemistV3.sol 1104)

Fee Payout Logic (\_doLiquidation): AlchemistV3.sol

Fee Vault (alchemistFeeVault): AlchemistV3.sol

## Proof of Concept

## Proof of Concept

The following Foundry test, when added to AlchemistV3.t.sol, reproduces the vulnerability. It simulates the "perfect storm" by (1) triggering the global bad debt state via price manipulation and (2) emptying the alchemistFeeVault. It then asserts that the liquidator receives zero fees.

```
    function testLiquidate_Poc_NoIncentive_Globally_Undercollateralized_And_EmptyFeeVault() external {
        // [POC] Replicate vulnerability: Global bad debt + Empty FeeVault = 0 Liquidation Incentive
        //
        // 1. Set up a position and apply maximum leverage
        vm.startPrank(someWhale);
        IMockYieldToken(mockStrategyYieldToken).mint(whaleSupply, someWhale);
        vm.stopPrank();

        vm.startPrank(address(0xbeef));
        SafeERC20.safeApprove(address(vault), address(alchemist), depositAmount + 100e18);
        alchemist.deposit(depositAmount, address(0xbeef), 0);
        uint256 tokenIdFor0xBeef = AlchemistNFTHelper.getFirstTokenId(address(0xbeef), address(alchemistNFT));
        alchemist.mint(tokenIdFor0xBeef, alchemist.totalValue(tokenIdFor0xBeef) * FIXED_POINT_SCALAR / minimumCollateralization, address(0xbeef));
        vm.stopPrank();

        (uint256 prevCollateral, uint256 prevDebt,) = alchemist.getCDP(tokenIdFor0xBeef);

        // 2. Manipulate price to make the position and the [entire system] undercollateralized
        // We need alchemistCurrentCollateralization < globalMinimumCollateralization
        // In setUp, globalMinimumCollateralization = 1.111e18
        // A price drop of 5.9% (590 bps) is enough to trigger global bad debt
        uint256 initialVaultSupply = IERC20(address(mockStrategyYieldToken)).totalSupply();
        IMockYieldToken(mockStrategyYieldToken).updateMockTokenSupply(initialVaultSupply);
        uint256 modifiedVaultSupply = (initialVaultSupply * 590 / 10_000) + initialVaultSupply;
        IMockYieldToken(mockStrategyYieldToken).updateMockTokenSupply(modifiedVaultSupply);

        // Verify the global bad debt condition is met
        uint256 alchemistCurrentCollateralization =
            alchemist.normalizeUnderlyingTokensToDebt(alchemist.getTotalUnderlyingValue()) * FIXED_POINT_SCALAR / alchemist.totalDebt();
        assertTrue(alchemistCurrentCollateralization < alchemist.globalMinimumCollateralization(), "System must be globally undercollateralized");

        // 3. **Key step to replicate vulnerability: Drain alchemistFeeVault**
        // The setUp function dealt 10,000 ether to the vault
        uint256 vaultBalance = alchemistFeeVault.totalDeposits();
        assertTrue(vaultBalance > 0, "Fee vault should have funds initially");
        
        // alOwner is the owner of the vault
        vm.startPrank(alOwner); 
        alchemistFeeVault.withdraw(alOwner, vaultBalance);
        vm.stopPrank();
        assertEq(alchemistFeeVault.totalDeposits(), 0, "Fee vault should be empty");

        // 4. Liquidator (externalUser) attempts to liquidate
        vm.startPrank(externalUser);
        uint256 liquidatorPrevTokenBalance = IERC20(address(vault)).balanceOf(address(externalUser));
        uint256 liquidatorPrevUnderlyingBalance = IERC20(vault.asset()).balanceOf(address(externalUser));

        // Call calculateLiquidation to see what the contract *thinks* it will pay
        (uint256 liquidationAmount, uint256 expectedDebtToBurn, uint256 expectedBaseFee, uint256 expectedOutsourcedFee) = alchemist.calculateLiquidation(
            alchemist.totalValue(tokenIdFor0xBeef),
            prevDebt,
            alchemist.minimumCollateralization(),
            alchemistCurrentCollateralization,
            alchemist.globalMinimumCollateralization(),
            liquidatorFeeBPS
        );

        // **Check the calculation results from the flawed logic**
        // expectedBaseFee (i.e., feeInYield) should be 0, as the bug branch was triggered
        assertEq(expectedBaseFee, 0, "calculateLiquidation should return base fee 0");
        // expectedOutsourcedFee should be > 0, as the contract *intends* to pay it
        assertTrue(expectedOutsourcedFee > 0, "calculateLiquidation should calculate an outsourced fee");

        // Execute liquidation
        (uint256 assets, uint256 feeInYield, uint256 feeInUnderlying) = alchemist.liquidate(tokenIdFor0xBeef);

        // 5. **Assert the vulnerability (the "death spiral" condition)**
        // (A) Liquidator received 0 base fee (feeInYield) because calculateLiquidation returned 0
        assertEq(feeInYield, 0, "Liquidator should receive 0 yield token fee (Bug)");
        // (B) Liquidator received 0 outsourced fee (feeInUnderlying) because alchemistFeeVault was drained
        assertEq(feeInUnderlying, 0, "Liquidator should receive 0 underlying token fee (Empty Vault)");

        // Check the liquidator's balances - they should not have changed, incentive was 0
        uint256 liquidatorPostTokenBalance = IERC20(address(vault)).balanceOf(address(externalUser));
        uint256 liquidatorPostUnderlyingBalance = IERC20(vault.asset()).balanceOf(address(externalUser));
        assertEq(liquidatorPostTokenBalance, liquidatorPrevTokenBalance, "Liquidator yield token balance should not change");
        assertEq(liquidatorPostUnderlyingBalance, liquidatorPrevUnderlyingBalance, "Liquidator underlying balance should not change");
        
        vm.stopPrank();

        // The position itself was still liquidated (even though the liquidator got no reward)
        (uint256 depositedCollateral, uint256 debt,) = alchemist.getCDP(tokenIdFor0xBeef);
        assertEq(debt, 0, "Debt should be 0 after full liquidation");
    }
```
