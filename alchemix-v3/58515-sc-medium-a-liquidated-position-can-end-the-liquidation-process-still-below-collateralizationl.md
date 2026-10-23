# 58515 sc medium a liquidated position can end the liquidation process still below collateralizationlowerbound allowing for double liquidation of positions&#x20;

**Submitted on Nov 2nd 2025 at 23:36:41 UTC by @Tadev for** [**Audit Comp | Alchemix V3**](https://immunefi.com/audit-competition/alchemix-v3-audit-competition)

* **Report ID:** #58515
* **Report Type:** Smart Contract
* **Report severity:** Medium
* **Target:** <https://github.com/alchemix-finance/v3-poc/blob/immunefi\\_audit/src/AlchemistV3.sol>
* **Impacts:**
  * Contract fails to deliver promised returns, but doesn't lose value

## Description

## Brief/Intro

If a position's collateralization ratio is too low, below `collateralizationLowerBound`, it becomes eligible for liquidation.

The purpose of the `liquidate` function is to restore the collateralization ratio:

* above `collateralizationLowerBound` in the case of a force repay of the earmarked debt big enough to restore the ratio above `collateralizationLowerBound`
* above `minimumCollateralization` in the case of a real liquidation that triggers `_doLiquidation`

The problem arises because the current design allows for a position to be liquidated twice via `liquidate` function. Indeed, there is a possibility for the first liquidation not to restore a ratio above `collateralizationLowerBound` while still being successful.

## Vulnerability Details

The issue lies in the `_liquidate` function, at the end:

```
        collateralInUnderlying = totalValue(accountId);
        collateralizationRatio = collateralInUnderlying * FIXED_POINT_SCALAR / account.debt;

        if (collateralizationRatio <= collateralizationLowerBound) {
            return _doLiquidation(accountId, collateralInUnderlying, repaidAmountInYield);
        } else {
            // Since only a repayment happened, send repayment fee to caller
            feeInYield = _resolveRepaymentFee(accountId, repaidAmountInYield);
            TokenUtils.safeTransfer(myt, msg.sender, feeInYield);
            // @audit 3rd parameter returned should be feeInUnderlying
            return (repaidAmountInYield, feeInYield, 0);
        }
```

At this point of the function, if the position had earmarked debt, a `_forceRepay` happened, potentially restoring the ratio above `collateralizationLowerBound`.

The problem arises when the `forceRepay` call restores a ratio juste above `collateralizationLowerBound`. In that case, the `else` branch is executed and the fee is calculated and sent to the liquidator. This means we check the ratio first, and after that the liquidator fee reduces the `account.collateralBalance` in `_resolveRepaymentFee` function:

```
    function _resolveRepaymentFee(uint256 accountId, uint256 repaidAmountInYield) internal returns (uint256 fee) {
        Account storage account = _accounts[accountId];
        // calculate repayment fee and deduct from account
        fee = repaidAmountInYield * repaymentFee / BPS;
        account.collateralBalance -= fee > account.collateralBalance ? account.collateralBalance : fee;
        emit RepaymentFee(accountId, repaidAmountInYield, msg.sender, fee);
        return fee;
    }
```

Because we don't check again the collateralization ration, it is possible that the fee sent to the liquidator puts the position back below `collateralizationLowerBound`. In this case, the `liquidate` call is successful but the position is still eligible for liquidation.

## Impact Details

The impact of this issue can be considered medium as it result in a significant disruption of the liquidation process. Liquidations are expected to work properly and restore the ratios without issues.

## Proof of Concept

## Proof of Concept

Please copy paste the following test in *AlchemistV3.t.sol* file:

```
    function testCanLiquidateTwice() external {
        vm.prank(alOwner);
        alchemist.setProtocolFee(protocolFee);

        vm.startPrank(address(0xbeef));
        uint256 amount = 200_000e18;
        SafeERC20.safeApprove(address(vault), address(alchemist), amount);
        // deposit MYT tokens in the Alchemist
        alchemist.deposit(amount, address(0xbeef), 0);
        uint256 tokenIdFor0xBeef = AlchemistNFTHelper.getFirstTokenId(address(0xbeef), address(alchemistNFT));
        uint256 mintAmount = alchemist.totalValue(tokenIdFor0xBeef) * FIXED_POINT_SCALAR / minimumCollateralization;
        // mint debt token in the Alchemist
        alchemist.mint(tokenIdFor0xBeef, mintAmount, address(0xbeef));
        vm.stopPrank();

        // create a redemption to start earmarking debt
        vm.startPrank(anotherExternalUser);
        SafeERC20.safeApprove(address(alToken), address(transmuterLogic), mintAmount);
        transmuterLogic.createRedemption(mintAmount);
        vm.stopPrank();

        // skip to a future block - 13% of the way through the transmutation period (5_256_000 blocks)
        uint256 earmarkPercent = 1300;
        vm.roll(block.number + (5_256_000 * earmarkPercent / 10_000));

        // modify yield token price via modifying underlying token supply
        // increasing yield token supply by 10.6% while keeping the underlying supply unchanged
        uint256 initialVaultSupply = IERC20(address(mockStrategyYieldToken)).totalSupply();
        uint256 modifiedVaultSupply = (initialVaultSupply * 10_600 / 10_000);
        IMockYieldToken(mockStrategyYieldToken).updateMockTokenSupply(modifiedVaultSupply);

        uint256 lowerBound = 1_052_631_578_950_000_000;
        console.log("Collateralization Lower bound:", lowerBound);

        // Calculate the collateralization ratio
        (, uint256 debt,) = alchemist.getCDP(tokenIdFor0xBeef);
        uint256 collateralInDebt = alchemist.totalValue(tokenIdFor0xBeef);
        uint256 collateralizationRatio = collateralInDebt * FIXED_POINT_SCALAR / debt;
        console.log("Collateralization ratio: before first liquidation", collateralizationRatio);

        // poke to update state
        alchemist.poke(tokenIdFor0xBeef);

        // liquidate the position
        vm.startPrank(externalUser);
        alchemist.liquidate(tokenIdFor0xBeef);

        // Calculate the collateralization ratio
        (, debt,) = alchemist.getCDP(tokenIdFor0xBeef);
        collateralInDebt = alchemist.totalValue(tokenIdFor0xBeef);
        collateralizationRatio = collateralInDebt * FIXED_POINT_SCALAR / debt;
        console.log("Collateralization ratio after first liquidation:", collateralizationRatio);

        // liquidate the position a second time in a row
        alchemist.liquidate(tokenIdFor0xBeef);
        vm.stopPrank();

        // Calculate the collateralization ratio
        (, debt,) = alchemist.getCDP(tokenIdFor0xBeef);
        collateralInDebt = alchemist.totalValue(tokenIdFor0xBeef);
        collateralizationRatio = collateralInDebt * FIXED_POINT_SCALAR / debt;
        console.log("Collateralization ratio after second liquidation:", collateralizationRatio);
    }
```

This tests highlights a situation where a user position is liquidated twice in a row. The output of the test is :

```
Logs:
  Collateralization Lower bound: 1052631578950000000
  Collateralization ratio: before first liquidation 1048218029350104821
  Collateralization ratio after first liquidation: 1052434516494373357
  Collateralization ratio after second liquidation: 1111111111111111110
```

This output shows:

* a position collateralization ratio below `collateralizationLowerBound` before first liquidation
* a collateralization ratio still below `collateralizationLowerBound` after the first liquidation which is not supposed to happen
* a collateralization ratio at the target minimum collateralization ratio after the second liquidation
