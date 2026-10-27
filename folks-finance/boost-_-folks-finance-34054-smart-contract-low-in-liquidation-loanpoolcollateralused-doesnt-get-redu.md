# Boost \_ Folks Finance 34054 - \[Smart Contract - Low] In liquidation loanPoolcollateralUsed doesnt ge

Submitted on Sun Aug 04 2024 19:33:56 GMT-0400 (Atlantic Standard Time) by @A2Security for [Boost | Folks Finance](https://immunefi.com/bounty/folksfinance-boost/)

Report ID: #34054

Report type: Smart Contract

Report severity: Low

Target: <https://testnet.snowtrace.io/address/0x2cAa1315bd676FbecABFC3195000c642f503f1C9>

Impacts:

* Contract fails to deliver promised returns, but doesn't lose value

## Description

## Description

In each LoanPool the amount of collateral used by protocol users is tracked in the state variable `CollateralUsed`. There is a bug in `LoanManagerLogic.executeLiquidate()` that leads to the overvaluation of the `CollateralUsed`. This is caused by the fact that the protocol resreves amount of the collateral seized of the violator and mint it the `FeeTokenReciepient` and the problem is that eventhough this amount leaves the loanPool the `collateralUsed` doesn't get reduced.

## Impact

This breaks the accounting for tracking the amount of collateralUsed and overvalue it in the long term

## Recomnedation

To mitigate this we simply need to reduce collateralUsed by the reservedAmount

```diff
        loansParams.checkLiquidatorLoan(userLoans, loanTypes, pools, params.oracleManager);
++        LoanManagerState.UserLoan storage violatorLoan = userLoans[params.violatorLoanId];
++        LoanManagerState.LoanPool storage loanPool = loanTypes[userLoan.loanTypeId].pools[params.colPoolId];
++        loanPool.decreaseCollateral(collateralSeized.reserveAmount);
        // mint f token for fee recipient
        pools[params.colPoolId].mintFTokenForFeeRecipient(collateralSeized.reserveAmount);
```

## Proof of concept

## Proof Of Concept

In `executeLiquidate()` the protocol mints the amount reserved from the seizedCollateral to fee reciepient but doesn't reduce collateralUsed from LoanPool

```solidity
    function executeLiquidate(
        mapping(bytes32 => LoanManagerState.UserLoan) storage userLoans,
        mapping(uint16 loanTypeId => LoanManagerState.LoanType) storage loanTypes,
        mapping(uint8 => IHubPool) storage pools,
        mapping(bytes32 accountId => mapping(uint8 poolId => LoanManagerState.UserPoolRewards)) storage userPoolRewards,
        DataTypes.ExecuteLiquidationParams memory params
    ) external {
        DataTypes.LiquidationLoansParams memory loansParams = DataTypes.LiquidationLoansParams({
            liquidatorLoanId: params.liquidatorLoanId,
            violatorLoanId: params.violatorLoanId,
            collateralPoolId: params.colPoolId,
            borrowPoolId: params.borPoolId
        });

        // check violator loan is under-collateralized and calc max repay value
        uint256 maxRepayBorrowValue;
        {
            DataTypes.LoanLiquidityParams memory violatorLiquidity = loansParams.prepareLiquidation(userLoans, loanTypes, pools, params.oracleManager);
            //max  value in 8dec can be repayed of all borrowed assets in violatorLoan id
            maxRepayBorrowValue = loansParams.getMaxRepayBorrowValue(userLoans, loanTypes, violatorLiquidity);
        }
        // @note  calculate the actual amount to seize of coll and the amount to reduce the debt by
        // calc actual repay and seize amounts considering the user loan and max specified
        // note : returns : repayBorrowAmount ,repayBorrowToCollateralFAmount , seizeCollateralFAmount
        DataTypes.LiquidationAmountParams memory liquidationAmounts =
            loansParams.calcLiquidationAmounts(userLoans, loanTypes, pools[loansParams.collateralPoolId], params.oracleManager, maxRepayBorrowValue, params.maxRepayAmount);

        loansParams.updateLiquidationRewards(userLoans, loanTypes, userPoolRewards);

        DataTypes.LiquidationBorrowTransfer memory liquidationBorrowTransfer;
        {
            //@note update index , get pool index and rates
            // pool pre-checks and update interest indexes
            DataTypes.BorrowPoolParams memory borrowPoolParams = pools[params.borPoolId].preparePoolForRepay();

            // transfer borrow from violator to liquidator
            liquidationBorrowTransfer = loansParams.updateLiquidationBorrows(borrowPoolParams, liquidationAmounts.repayBorrowAmount, userLoans);
        }
        //@note moves coll also checks sllippage param min seize
        // transfer collateral from violator to liquidator
        DataTypes.CollateralSeizedParams memory collateralSeized = loansParams.updateLiquidationCollaterals(
            liquidationAmounts.seizeCollateralFAmount, liquidationAmounts.repayBorrowToCollateralFAmount, params.minSeizedAmount, userLoans, loanTypes
        );
        // @audit-issue loanPool doesn't get updated
        // @note update intrest rates :
        pools[params.borPoolId].updatePoolWithLiquidation();
        // check liquidator loan in over-collateralized after taking over part of the violator loan
        // @note if !liquidatorLoan.isLoanOverCollateralized revert
  >>      loansParams.checkLiquidatorLoan(userLoans, loanTypes, pools, params.oracleManager);

        // mint f token for fee recipient
  >>      pools[params.colPoolId].mintFTokenForFeeRecipient(collateralSeized.reserveAmount);

        emit Liquidate(
            params.violatorLoanId,
            params.liquidatorLoanId,
            params.colPoolId,
            params.borPoolId,
            liquidationAmounts.repayBorrowAmount,
            collateralSeized.liquidatorAmount,
            collateralSeized.reserveAmount
        );
```
