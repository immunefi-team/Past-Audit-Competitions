# Boost \_ Folks Finance 34132 - \[Smart Contract - Low] Liquidation bonus incorrectly inflates repayBor

Submitted on Mon Aug 05 2024 20:33:25 GMT-0400 (Atlantic Standard Time) by @zarkk for [Boost | Folks Finance](https://immunefi.com/bounty/folksfinance-boost/)

Report ID: #34132

Report type: Smart Contract

Report severity: Low

Target: <https://testnet.snowtrace.io/address/0xc1FBF54B25816B60ADF322d8A1eaCA37D9A50317>

Impacts:

* Protocol insolvency

## Description

## Brief/Intro

Liquidation bonus, which is supposed to incentivise liquidators to repay violators debt, is inflating the `repayBorrowAmount` instead of the `seizeUnderlyingCollateralAmount` breaking the functionality of the liquidation.

## Vulnerability Details

The vulnerability occurs when a liquidator attempts to liquidate a borrower's debt by invoking the `executeLiquidation` function within the `LoanManagerLogic` contract. This function, in turn, calls the `calcLiquidationAmounts` function from `LiquidationLogic` to calculate the necessary transfers of collateral and borrowed amounts between the liquidator and the borrower. This function is supposed to take into consideration the liquidation bonus that the liquidator will get as an extra for liquidation and acts as the incentivise for him to perform it. We can see the implementation here of `calcLiquidationAmounts` here :

```solidity
function calcLiquidationAmounts(
        // ...
    ) external view returns (DataTypes.LiquidationAmountParams memory liquidationAmounts) {
        // ...
        uint256 repayBorrowAmount;
        {
            uint256 maxRepayBorrowAmount = MathUtils.calcAssetAmount(
                maxRepayBorrowValue * MathUtils.ONE_10_DP,
                borrPriceFeed.price,
                borrPriceFeed.decimals
            );
            repayBorrowAmount = Math.min(maxAmountToRepay, Math.min(maxRepayBorrowAmount, violatorLoanBorrow.balance));
        }
        {
            uint256 seizeUnderlyingCollateralAmount = repayBorrowAmount.convToSeizedCollateralAmount(
                collPriceFeed.price,
                collPriceFeed.decimals,
                borrPriceFeed.price,
                borrPriceFeed.decimals,
                borrowLoanPool.liquidationBonus
            );
            uint256 collDepositInterestIndex = collPool.getUpdatedDepositInterestIndex();
            uint256 violatorUnderlingCollateralBalance = violatorLoanCollateral.balance.toUnderlingAmount(
                collDepositInterestIndex
            );
            if (seizeUnderlyingCollateralAmount > violatorUnderlingCollateralBalance) {
                seizeUnderlyingCollateralAmount = violatorUnderlingCollateralBalance;
@>                repayBorrowAmount = seizeUnderlyingCollateralAmount.convToRepayBorrowAmount(
                    collPriceFeed.price,
                    collPriceFeed.decimals,
                    borrPriceFeed.price,
                    borrPriceFeed.decimals,
                    borrowLoanPool.liquidationBonus
                );
            }

            liquidationAmounts.repayBorrowAmount = repayBorrowAmount;
            liquidationAmounts.repayBorrowToCollateralFAmount = repayBorrowAmount.convToCollateralFAmount(
                collPriceFeed.price,
                collPriceFeed.decimals,
                borrPriceFeed.price,
                borrPriceFeed.decimals,
                collDepositInterestIndex
            );
            liquidationAmounts.seizeCollateralFAmount = seizeUnderlyingCollateralAmount.toFAmount(
                collDepositInterestIndex
            );
        }
    }
```

This function should correctly account for the liquidation bonus, which serves as an incentive for liquidators. However, in the case where `seizeUnderlyingCollateralAmount` > `violatorUnderlingCollateralBalance`, the `repayBorrowAmount` is recalculated and incorrectly inflated by the liquidation bonus. The correct behavior should be that the liquidation bonus inflates the `seizeUnderlyingCollateralAmount`, allowing the liquidator to seize additional collateral as a reward for performing the liquidation. Instead, the current implementation results in the liquidator being required to repay an inflated borrow amount, effectively causing the liquidator to pay more than the market value for the collateral. This misalignment of incentives not only discourages liquidators from executing liquidations but also breaks the fundamental logic of the liquidation process, leading to potential financial losses for the liquidator.

## Impact Details

This vulnerability can lead to significant financial losses and operational inefficiencies within the system. Since the liquidation process is disincentivized, borrowers who fall below the required collateral thresholds may not be liquidated promptly, increasing systemic risk. Furthermore, liquidators who do engage in the process may suffer losses due to the inflated repayment amounts, which exceed the value of the collateral they seize. Generally speaking, the liquidation process is not working as expected.

## References

<https://github.com/Folks-Finance/folks-finance-xchain-contracts/blob/fb92deccd27359ea4f0cf0bc41394c86448c7abb/contracts/hub/logic/LiquidationLogic.sol#L228-L234>

## Proof of concept

## Proof of Concept

To demonstrate the vulnerability, you can add the following test under the `"Liquidate"` section in `LoanManager.test.ts` and run `npm test`:

```javascript
it.only("Should inflate incorrectly the repayBorrowAmount with the liquidation bonus instead of inflating the seizeCollateral", async () => {
      const {
        hub,
        loanManager,
        oracleManager,
        pools,
        loanId: violatorLoanId,
        accountId: violatorAccountId,
        loanTypeId,
        borrowAmount,
        usdcVariableInterestIndex: oldVariableInterestIndex,
      } = await loadFixture(depositEtherAndVariableBorrowUSDCFixture);

      // Config the liquidator.
      const liquidatorLoanId = getRandomBytes(BYTES32_LENGTH);
      const liquidatorAccountId = getAccountIdBytes("LIQUIDATOR_ACCOUNT_ID");
      const liquidatorLoanName = getRandomBytes(BYTES32_LENGTH);
      await loanManager.connect(hub).createUserLoan(liquidatorLoanId, liquidatorAccountId, loanTypeId, liquidatorLoanName);

      const liquidatorDepositAmount = BigInt(10000e6); // 10,000 USDC
      const liquidatorDepositFAmount = liquidatorDepositAmount;
      const liquidatorDepositInterestIndex = BigInt(1e18);
      const usdcPrice = BigInt(1e18);
      await pools.USDC.pool.setDepositPoolParams({fAmount: liquidatorDepositFAmount,depositInterestIndex: liquidatorDepositInterestIndex,priceFeed: { price: usdcPrice, decimals: pools.USDC.tokenDecimals },});
      await loanManager.connect(hub).deposit(liquidatorLoanId, liquidatorAccountId, pools.USDC.poolId, liquidatorDepositAmount);

      // prepare liquidation
      const ethNodeOutputData = getNodeOutputData(BigInt(500e18));
      await oracleManager.setNodeOutput(pools.ETH.poolId, pools.ETH.tokenDecimals, ethNodeOutputData);

      // calculate interest
      const variableInterestIndex = BigInt(1.1e18);
      const stableInterestRate = BigInt(0.1e18);
      await pools.USDC.pool.setBorrowPoolParams({ variableInterestIndex, stableInterestRate });
      await pools.USDC.pool.setUpdatedVariableBorrowInterestIndex(variableInterestIndex);
      const borrowBalance = calcBorrowBalance(borrowAmount, variableInterestIndex, oldVariableInterestIndex);

      // Violator:
      // Collateral 1 ETH = $500
      // Borrow 1,000 USDC = $1,000

      // Liquidator:
      // Collateral 10,000 USDC = $10,000
      // Borrow $0
      const seizeCollateralAmount = BigInt(1e18); // 1 ETH
      const seizeCollateralFAmount = BigInt(1e18);
      // The USDC that the liquidator will repay is 1 ETH in USDC + 4% bonus. He will repay 1.04 ETH in USDC.
      const repayAmount = convToRepayBorrowAmount(
        seizeCollateralAmount,
        ethNodeOutputData.price,
        pools.ETH.tokenDecimals,
        usdcPrice,
        pools.USDC.tokenDecimals,
        BigInt(0.04e4) // liquidation bonus for usdc is 4%
      );
      const collateralFAmount = convToCollateralFAmount(
        repayAmount,
        ethNodeOutputData.price,
        pools.ETH.tokenDecimals,
        usdcPrice,
        pools.USDC.tokenDecimals,
        BigInt(1e18)
      );
      const reserveCollateralFAmount = calcReserveCol(
        seizeCollateralFAmount,
        collateralFAmount,
        pools.ETH.liquidationFee
      );
      const liquidatorCollateralFAmount = seizeCollateralFAmount - reserveCollateralFAmount;
      const attemptedRepayAmount = repayAmount + BigInt(10e6);

      // Liquidation is executed.
      const minSeizedAmount = BigInt(0);
      const liquidate = await loanManager
        .connect(hub)
        .liquidate(
          violatorLoanId,
          liquidatorLoanId,
          liquidatorAccountId,
          pools.ETH.poolId,
          pools.USDC.poolId,
          attemptedRepayAmount,
          minSeizedAmount
        );

      const borrowUSD = repayAmount * usdcPrice / BigInt(1e6); // USD value that the liquidator gained as debt
      const collateralUSD = toUnderlingAmount(liquidatorCollateralFAmount, BigInt(1e18)) * ethNodeOutputData.price / BigInt(1e18); // USD value that the liquidator gained as collateral
      
      console.log("borrowUSD", borrowUSD.toString());
      console.log("collateralUSD", collateralUSD.toString());
      
      // He got more debt than collateral, so clearly he is disentivised to do that.
      expect(borrowUSD > collateralUSD).to.be.true;
    });
```

This test will demonstrate that the liquidator ends up with more debt than the collateral they receive, highlighting the disincentive created by the vulnerability.
