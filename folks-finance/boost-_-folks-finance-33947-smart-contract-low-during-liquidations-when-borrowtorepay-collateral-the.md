# Boost \_ Folks Finance 33947 - \[Smart Contract - Low] During liquidations when borrowToRepay collater

Submitted on Fri Aug 02 2024 09:38:03 GMT-0400 (Atlantic Standard Time) by @iamandreiski for [Boost | Folks Finance](https://immunefi.com/bounty/folksfinance-boost/)

Report ID: #33947

Report type: Smart Contract

Report severity: Low

Target: <https://testnet.snowtrace.io/address/0xc1FBF54B25816B60ADF322d8A1eaCA37D9A50317>

Impacts:

* Protocol insolvency
* Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield

## Description

## Brief/Intro

When a user is undercollateralized and eligible for liquidation, a liquidator can initiate a liquidation process in which the borrowed amount(either partial or whole) of the token-in-question + interest will be transferred to the liquidator for later repayment as well as the collateral in the amount of the borrowed token + 10% liquidation bonus. The problem arises when the borrow amount that should be repaid is greater than the collateral amount to-be-received by the liquidator. In those cases, due to invalid calculations, the liquidator will actually receive 10% more borrow amount to repay (or 10% less collateral than the borrow amount), effectively damaging the liquidator.

## Vulnerability Details

There are more than one outcome in which this can result:

* Violator will pay 10% less collateral for the borrowed amount, effectively "incentivizing" users with underwater loans.
* This will disincentivize liquidators to liquidate these kind of loans effectively leading to the accumulation of more bad debt and essentially undercollateralized loans / protocol/pool insolvency.

When a user initiates a liquidation, eventually `calcLiquidationAmounts()` will be called in order to calculate the collateral and borrow amounts based on amounts owed, user input, etc.

```
function calcLiquidationAmounts(
        DataTypes.LiquidationLoansParams memory loansParams,
        mapping(bytes32 => LoanManagerState.UserLoan) storage userLoans,
        mapping(uint16 => LoanManagerState.LoanType) storage loanTypes,
        IHubPool collPool,
        IOracleManager oracleManager,
        uint256 maxRepayBorrowValue,
        uint256 maxAmountToRepay
    ) external view returns (DataTypes.LiquidationAmountParams memory liquidationAmounts) {
        LoanManagerState.UserLoan storage violatorLoan = userLoans[loansParams.violatorLoanId];

        uint8 collPoolId = loansParams.collateralPoolId;
        uint8 borrPoolId = loansParams.borrowPoolId;

        LoanManagerState.UserLoanCollateral storage violatorLoanCollateral = violatorLoan.collaterals[collPoolId];
        LoanManagerState.LoanPool storage borrowLoanPool = loanTypes[violatorLoan.loanTypeId].pools[borrPoolId];
        LoanManagerState.UserLoanBorrow storage violatorLoanBorrow = violatorLoan.borrows[borrPoolId];

        DataTypes.PriceFeed memory borrPriceFeed = oracleManager.processPriceFeed(borrPoolId);
        DataTypes.PriceFeed memory collPriceFeed = oracleManager.processPriceFeed(collPoolId);
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
                repayBorrowAmount = seizeUnderlyingCollateralAmount.convToRepayBorrowAmount(
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

First the `repayBorrowAmount` will be calculated based on the following "formula":

```
repayBorrowAmount = Math.min(maxAmountToRepay, Math.min(maxRepayBorrowAmount, violatorLoanBorrow.balance));
```

It will take the smaller amount between a user-input value of the amount that they'd want to liquidate OR either the balance of the loan OR the maxRepayBorrowAmount (which is a calculation based upon the amount that needs to be liquidated to make the loan healthy).

For the sake of this situation let's say that the `violatorLoanBorrow.balance` was picked as the "smallest" to be liquidated, which is the total balance of the loan of that pool.

After the above-mentioned amount was determined, it will be converted to its equivalent collateral value:

```
 uint256 seizeUnderlyingCollateralAmount = repayBorrowAmount.convToSeizedCollateralAmount(
                collPriceFeed.price,
                collPriceFeed.decimals,
                borrPriceFeed.price,
                borrPriceFeed.decimals,
                borrowLoanPool.liquidationBonus
            );
```

For the sake of this example, let's say that the borrowed amount to be repaid is 1 WETH, and the collateral is USDC, with the WETH/USDC price at 3000 USDC.

In underlying collateral amount, this would be 3300 USDC (Taking in consideration the liquidation bonus(if it's 10%)).

The problem arises if the collateral that the user has in the pool is less than this, let's say 2500 USDC.

The following will occur:

```
 uint256 violatorUnderlingCollateralBalance = violatorLoanCollateral.balance.toUnderlingAmount(
                collDepositInterestIndex
            );
            if (seizeUnderlyingCollateralAmount > violatorUnderlingCollateralBalance) {
                seizeUnderlyingCollateralAmount = violatorUnderlingCollateralBalance;
                repayBorrowAmount = seizeUnderlyingCollateralAmount.convToRepayBorrowAmount(
                    collPriceFeed.price,
                    collPriceFeed.decimals,
                    borrPriceFeed.price,
                    borrPriceFeed.decimals,
                    borrowLoanPool.liquidationBonus
                );
```

The `seizeUnderlyingCollateralAmount` will be the `violatorUnderlingCollateralBalance` which is the underlying amount OR 2500 USDC.

But when the `repayBorrowAmount` is calculated in order to determine how much of the borrow amount should the liquidator repay:

```
    function convToRepayBorrowAmount(
        uint256 collAmount,
        uint256 collPrice,
        uint8 collDecimals,
        uint256 borrPrice,
        uint8 borrDecimals,
        uint256 liquidationBonus
    ) internal pure returns (uint256) {
        return Math.mulDiv(
            convertAssetAmount(collAmount, collPrice, collDecimals, borrPrice, borrDecimals),
            (MathUtils.ONE_4_DP + liquidationBonus),
            MathUtils.ONE_4_DP
        );
    }

```

This would result in converting 2500 USDC to WETH, or 0.83 WETH, the problem is that the liquidation bonus would be added to this value:

```
0.833 * (1e4 + 0.1e4) / 1e4 = 0.916
```

This would cause the liquidator to receive only 2500 USDC collateral, but pay the equivalent of 2750 USDC in borrow amount, due to the incorrect calculation.

* `It should be : assetAmount * 1e4 / (1e4 + liquidationBonus)`

Even if liquidators intentionally input lower-than-collateral `maxAmountToRepay` values, after some time this will also result in bad debt accumulation as the collateral would be transferred to liquidators, but there would be "residual" borrow amounts.

## Impact Details

Loans in which the collateral is less than the amount-to-be-repayed will result in the liquidator paying 10% more borrowAmount, while receiving 10% less collateral amount. This is effectively damaging the liquidator in the benefit of the borrower, and could potentially lead to a pool insolvency if enough bad debt is accumulated.

## References

Below PoC is in Foundry, the only thing needed to run it is the importation of MathUtils in the test suite.

## Proof of concept

## Proof of Concept

```
function testLiquidationCalcs() public {
        
        //WETH/USDC Price: 3000 USDC per WETH
        //Collateral in USDC pool: 2500 USDC
        //Outstanding borrow amount: 1 WETH

        uint256 repayBorrowAmount = 1e18;
        uint256 collateralInPool = 2500e6;

        //First we turn the the borrow token into collateral amount: 

        uint256 seizeUnderlyingCollateralAmount = repayBorrowAmount.convToSeizedCollateralAmount(
            1e18,
            6,
            3000e18,
            18,
            0.1e4
        );
       
       console.log("Seized Collateral (USDC)", seizeUnderlyingCollateralAmount);
       assertEq(seizeUnderlyingCollateralAmount, 3300e6);
       
       if (seizeUnderlyingCollateralAmount > collateralInPool) {
           seizeUnderlyingCollateralAmount = collateralInPool;
           repayBorrowAmount = seizeUnderlyingCollateralAmount.convToRepayBorrowAmount(
            1e18,
            6,
            3000e18,
            18,
            0.1e4
           );


       }

       console.log("Repay Borrow Amount: (WETH)", repayBorrowAmount);
       assertEq(repayBorrowAmount, 916666666666666666); // OR 0.916e18

       //We'll also convert the new repayBorrowAmount to its collateral counterpart just to compare it:

       uint256 newRepayCollateralCounterpart = repayBorrowAmount.convToSeizedCollateralAmount(
         1e18,
            6,
            3000e18,
            18,
            0 // Here we're assigning the bonus as 0 not to mess up calculations

       );

       console.log("New Repay Borrow Amount in Collateral (USDC):", newRepayCollateralCounterpart);
       assertEq(newRepayCollateralCounterpart, 2749999999); // OR 2749e6

       assert(newRepayCollateralCounterpart > collateralInPool);


    }
```
