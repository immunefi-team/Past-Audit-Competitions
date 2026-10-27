# Boost \_ Folks Finance 33746 - \[Smart Contract - Insight] Rounding down to zero leads to liquidate fu

Submitted on Sun Jul 28 2024 06:45:59 GMT-0400 (Atlantic Standard Time) by @ICP for [Boost | Folks Finance](https://immunefi.com/bounty/folksfinance-boost/)

Report ID: #33746

Report type: Smart Contract

Report severity: Insight

Target: <https://testnet.snowtrace.io/address/0xc1FBF54B25816B60ADF322d8A1eaCA37D9A50317>

Impacts:

* Permanent freezing of funds

## Description

## Brief/Intro

Rounding down to zero leads to panic error in Virutal Machine and halts the liquidate functionality there is no check to whether the collateralAdjustFactor is higher than borrowAdjustFactor before operation in `calcMaxRepayBorrowValue()`

## Vulnerability Details

Function `executeLiquidate()` will be called to initiate the liquidation process and calls the `getMaxRepayBorrowValue()`.In order to get the maximum Repay borrow value from the user loans and types.

```solidity
        // check violator loan is under-collateralized and calc max repay value
        uint256 maxRepayBorrowValue;
        {
            DataTypes.LoanLiquidityParams memory violatorLiquidity = loansParams.prepareLiquidation(
                userLoans,
                loanTypes,
                pools,
                params.oracleManager
            );
            maxRepayBorrowValue = loansParams.getMaxRepayBorrowValue(userLoans, loanTypes, violatorLiquidity);
        }
```

In `getMaxRepayBorrowValue()` calls `calcMaxRepayBorrowValue()` function to determine max repay value by factor of Adjusted collateral Factor and Adjusted Borrow Factor in below we can see the code :-

```solidity
    function calcMaxRepayBorrowValue(
        DataTypes.LoanLiquidityParams memory violatorLiquidity,
        LoanManagerState.LoanPool storage borrowLoanPool,
        uint256 collateralFactor,
        uint32 loanTargetHealth
    ) internal view returns (uint256 maxRepayBorrowValue) {

        uint256 effectiveBorrowValueTarget = violatorLiquidity.effectiveBorrowValue.calcBorrowValueTarget(
            loanTargetHealth
        );

        uint256 borrowAdjustFactor = (loanTargetHealth * borrowLoanPool.borrowFactor) / MathUtils.ONE_4_DP;

        uint256 collateralAdjustFactor = (collateralFactor * (MathUtils.ONE_4_DP + borrowLoanPool.liquidationBonus)) / MathUtils.ONE_4_DP;

        maxRepayBorrowValue =((effectiveBorrowValueTarget - violatorLiquidity.effectiveCollateralValue) * MathUtils.ONE_4_DP) /
         (borrowAdjustFactor - collateralAdjustFactor); // @audit  check here

    }

```

In above code we can confirm how the adjusted factor determined and leads to division by zero value.

Scenario :- If loan pool is created by the following valid values Scenario

1. liquidationBonus = 5000 (50% 0.5e4) 2.BorrowFactor = 10000 (100% 1e4) 3.CollateralFactor = 8000 (80 % 0.8 ) 4.LoanTargetHealth = 10000 (100 % 1e4)

The pool which is created as liquidationBonus value will 50% to incentive the liquidators to acquire the default loans and others values(BF, CF and LoanTargetHealth) are default. (Note : LiquidationBonus can be <1e4).

Evalution :-

1. (borrowAdjustFactor - collateralAdjustFactor) = 0.66666 (Rounds to Zero)

2 . maxRepayBorrowValue =((effectiveBorrowValueTarget - violatorLiquidity.effectiveCollateralValue) \* MathUtils.ONE\_4\_DP) / (borrowAdjustFactor - collateralAdjustFactor)// Division by zero halted

## Impact Details

The pool will be created in the above values will leads to prevent the liquidators to acquire the default loans and funds will permanently stuck in the on-chain only , the above values will make Adjusted collateral value is greater than Adjusted Borrow value and

## Recommendation

Add The check whether the collateralAdjustFactor is higher than borrowAdjustFactorbefore division in `calcMaxRepayBorrowValue()` function and add default value to division instead of Zero.

```solidity
   if(collateralAdjustFactor > borrowAdjustFactor ){
     maxRepayBorrowValue =((effectiveBorrowValueTarget - violatorLiquidity.effectiveCollateralValue) * MathUtils.ONE_4_DP) /
        DefaultFactor; // @audit check here.
  }
```

We hope this error will cause loss of funds if any query please ping me.

## Code snippet

<https://github.com/Folks-Finance/folks-finance-xchain-contracts/blob/main/contracts/hub/logic/LoanManagerLogic.sol#L457> <https://github.com/Folks-Finance/folks-finance-xchain-contracts/blob/main/contracts/hub/logic/LiquidationLogic.sol#L168C9-L168C28> <https://github.com/Folks-Finance/folks-finance-xchain-contracts/blob/main/contracts/hub/logic/LiquidationLogic.sol#L284C1-L290C59>

## Proof of concept

## Proof of Concept

We would recommend That paste the below code in test/hub/foo.test.ts

And run this command `npx hardhat test test/hub/foo.test.ts`.

```solidity
// SPDX-License-Identifier: BUSL-1.1
pragma solidity 0.8.23;

import "../libraries/DataTypes.sol";
import "../libraries/MathUtils.sol";
import "../LoanManagerState.sol";
import "../logic/LoanPoolLogic.sol";
import "../logic/UserLoanLogic.sol";
import "../logic/RewardLogic.sol";

library LiquidationLogic {
    using MathUtils for uint256;
    using LoanPoolLogic for LoanManagerState.LoanPool;
    using UserLoanLogic for LoanManagerState.UserLoan;

    error LoanTypeMismatch(uint16 violatorLoanTypeId, uint16 liquidatorLoanTypeId);
    error BorrowTypeMismatch(bytes32 violatorLoanId, bytes32 liquidatorLoanId, uint8 poolId);
    error UnderCollateralizedLoan(bytes32 loanId);
    error OverCollateralizedLoan(bytes32 loanId);
    error InsufficientSeized();
    error NoCollateralInLoanForPool(bytes32 loanId, uint8 poolId);
    error NoBorrowInLoanForPool(bytes32 loanId, uint8 poolId);

    /// @notice Updates violator and liquidator loans moving borrow from violator to liquidator in order to keep the loan healthy.
    /// @param loansParams LiquidationLoansParams struct including the violator and liquidator loan ids and the pool ids.
    /// @param borrowPoolParams BorrowPoolParams struct including the borrow pool's variable interest index and stable interest rate.
    /// @param repayBorrowAmount The amount to repay.
    /// @param userLoans The mapping of the loan ID to user loan including loan type, collateral and borrow details.
    /// @return liquidationBorrowTransfer LiquidationBorrowTransfer struct including the amount paid, interest paid and excess paid.
    function updateLiquidationBorrows(
        DataTypes.LiquidationLoansParams memory loansParams,
        DataTypes.BorrowPoolParams memory borrowPoolParams,
        uint256 repayBorrowAmount,
        mapping(bytes32 => LoanManagerState.UserLoan) storage userLoans
    ) external returns (DataTypes.LiquidationBorrowTransfer memory liquidationBorrowTransfer) {
        LoanManagerState.UserLoan storage violatorLoan = userLoans[loansParams.violatorLoanId];
        LoanManagerState.UserLoan storage liquidatorLoan = userLoans[loansParams.liquidatorLoanId];

        LoanManagerState.UserLoanBorrow storage violatorLoanBorrow = violatorLoan.borrows[loansParams.borrowPoolId];

        DataTypes.UpdateUserLoanBorrowParams memory updateLoanBorrowParams = DataTypes.UpdateUserLoanBorrowParams({
            poolId: loansParams.borrowPoolId,
            amount: 0,
            poolVariableInterestIndex: borrowPoolParams.variableInterestIndex,
            poolStableInterestRate: borrowPoolParams.stableInterestRate,
            isStableInterestRateToUpdate: false
        });

        (uint256 repaidBorrowAmount, uint256 repaidBorrowBalance, uint256 violatorStableRate) = UserLoanLogic
            .transferBorrowFromViolator(violatorLoan, loansParams.borrowPoolId, repayBorrowAmount);
        UserLoanLogic.transferBorrowToLiquidator(
            liquidatorLoan,
            updateLoanBorrowParams,
            repaidBorrowAmount,
            repaidBorrowBalance,
            violatorStableRate
        );

        liquidationBorrowTransfer.amountRepaid = repaidBorrowAmount;
        liquidationBorrowTransfer.balanceRepaid = repaidBorrowBalance;
        liquidationBorrowTransfer.isStable = violatorLoanBorrow.stableInterestRate > 0;
    }

    /// @notice Updates violator and liquidator loans moving the collateral seized.
    /// @param loansParams LiquidationLoansParams struct including the violator and liquidator loan ids and the pool ids.
    /// @param seizeCollateralFAmount The amount of collateral to seize.
    /// @param minSeized The minimum amount to seize acceptable for the liquidator.
    /// @param userLoans The mapping of the loan ID to user loan including loan type, collateral and borrow details.
    /// @param loanTypes The mapping of the type ID to loan types data including a mapping with the pools' details.
    /// @return collateralSeized CollateralSeizedParams struct including the total amount, liquidator amount and reserve amount.
    function updateLiquidationCollaterals(
        DataTypes.LiquidationLoansParams memory loansParams,
        uint256 seizeCollateralFAmount,
        uint256 repayBorrowToCollateralFAmount,
        uint256 minSeized,
        mapping(bytes32 => LoanManagerState.UserLoan) storage userLoans,
        mapping(uint16 => LoanManagerState.LoanType) storage loanTypes
    ) external returns (DataTypes.CollateralSeizedParams memory collateralSeized) {
        LoanManagerState.UserLoan storage violatorLoan = userLoans[loansParams.violatorLoanId];

        LoanManagerState.UserLoan storage liquidatorLoan = userLoans[loansParams.liquidatorLoanId];

        uint8 colPoolId = loansParams.collateralPoolId;
        uint16 liquidationFee = loanTypes[violatorLoan.loanTypeId].pools[colPoolId].liquidationFee;
        // CollateralSeized - liquidationfe.
        collateralSeized = calcCollateralSeized(seizeCollateralFAmount, repayBorrowToCollateralFAmount, liquidationFee);

        if (collateralSeized.liquidatorAmount < minSeized) revert InsufficientSeized(); // @audit check here.

        violatorLoan.decreaseCollateral(colPoolId, collateralSeized.totalAmount);
        liquidatorLoan.increaseCollateral(colPoolId, collateralSeized.liquidatorAmount);
    }

    /// @notice Checks if the liquidation is possible and returns the violator's liquidity details.
    /// @param loansParams LiquidationLoansParams struct including the violator and liquidator loan ids, the pool ids and borrow type.
    /// @param userLoans The mapping of the loan ID to user loan including loan type, collaterals and borrows details.
    /// @param loanTypes The mapping of the type ID to loan types data including a mapping with the pools' details.
    /// @param pools The mapping of the pool ID to the pool contract.
    /// @param oracleManager The OracleManager contract.
    /// @return violatorLiquidity LoanLiquidityParams struct including the violator's liquidity: effective collateral value and effective borrow value.
    function prepareLiquidation(
        DataTypes.LiquidationLoansParams memory loansParams,
        mapping(bytes32 => LoanManagerState.UserLoan) storage userLoans,
        mapping(uint16 => LoanManagerState.LoanType) storage loanTypes,
        mapping(uint8 => IHubPool) storage pools,
        IOracleManager oracleManager
    ) external returns (DataTypes.LoanLiquidityParams memory violatorLiquidity) {
        LoanManagerState.UserLoan storage violatorLoan = userLoans[loansParams.violatorLoanId];

        LoanManagerState.UserLoan storage liquidatorLoan = userLoans[loansParams.liquidatorLoanId];

        LoanManagerState.LoanType storage loanType = loanTypes[violatorLoan.loanTypeId];

        uint8 collPoolId = loansParams.collateralPoolId;

        uint8 borrPoolId = loansParams.borrowPoolId;

        // user cannot repay borrow and seize collateral which they don't have
        // borrow/collateral present iff loan type created and pool added so no need to check this
        if (!violatorLoan.hasBorrowIn(borrPoolId)) revert NoBorrowInLoanForPool(loansParams.violatorLoanId, borrPoolId);

        if (!violatorLoan.hasCollateralIn(collPoolId))
            revert NoCollateralInLoanForPool(loansParams.violatorLoanId, collPoolId);

        // check loans are compatible
        if (violatorLoan.loanTypeId != liquidatorLoan.loanTypeId)
            revert LoanTypeMismatch(violatorLoan.loanTypeId, liquidatorLoan.loanTypeId);

        // if applicable, check borrows are compatible
        LoanManagerState.UserLoanBorrow storage violatorLoanBorrow = violatorLoan.borrows[borrPoolId];

        LoanManagerState.UserLoanBorrow storage liquidatorLoanBorrow = liquidatorLoan.borrows[borrPoolId];

        bool isViolatorStableBorrow = violatorLoanBorrow.stableInterestRate > 0;

        bool isLiquidatorStableBorrow = liquidatorLoanBorrow.stableInterestRate > 0;

        if (liquidatorLoanBorrow.amount > 0 && isViolatorStableBorrow != isLiquidatorStableBorrow)
            revert BorrowTypeMismatch(loansParams.violatorLoanId, loansParams.liquidatorLoanId, borrPoolId);

        // check loan is under-collateralized
        violatorLiquidity = violatorLoan.getLoanLiquidity(pools, loanType.pools, oracleManager);

        if (violatorLiquidity.effectiveCollateralValue >= violatorLiquidity.effectiveBorrowValue)
            revert OverCollateralizedLoan(loansParams.violatorLoanId);

        // update the violator borrow balance in anticipation of calc liquidation amounts
        UserLoanLogic.updateLoanBorrowInterests(
            violatorLoanBorrow,
            DataTypes.UpdateUserLoanBorrowParams({
                poolId: borrPoolId,
                amount: 0,
                poolVariableInterestIndex: pools[borrPoolId].getUpdatedVariableBorrowInterestIndex(),
                poolStableInterestRate: 0,
                isStableInterestRateToUpdate: false
            })
        );
    }

    /// @notice Checks if the liquidation is possible and returns the violator's liquidity details.
    /// @param loansParams LiquidationLoansParams struct including the violator and liquidator loan ids, the pool ids and borrow type.
    /// @param userLoans The mapping of the loan ID to user loan including loan type, collaterals and borrows details.
    /// @param loanTypes The loan type data including a mapping with the pools' details.
    /// @return maxRepayBorrowValue LiquidationAmountParams struct including max repay borrow amount and max seize collateral amount.
    function getMaxRepayBorrowValue(
        DataTypes.LiquidationLoansParams memory loansParams,
        mapping(bytes32 => LoanManagerState.UserLoan) storage userLoans,
        mapping(uint16 => LoanManagerState.LoanType) storage loanTypes,
        DataTypes.LoanLiquidityParams memory violatorLiquidity
    ) external view returns (uint256 maxRepayBorrowValue) {

        LoanManagerState.UserLoan storage violatorLoan = userLoans[loansParams.violatorLoanId];

        LoanManagerState.LoanType storage loanType = loanTypes[violatorLoan.loanTypeId];

        uint8 collPoolId = loansParams.collateralPoolId;

        uint8 borrPoolId = loansParams.borrowPoolId;

        LoanManagerState.LoanPool storage borrowLoanPool = loanType.pools[borrPoolId];

        uint256 collateralFactor = loanType.pools[collPoolId].collateralFactor;

        maxRepayBorrowValue = calcMaxRepayBorrowValue(
            violatorLiquidity,
            borrowLoanPool,
            collateralFactor,
            loanType.loanTargetHealth
        );
    }

    /// @notice Calculates the borrow amount to repay and the collateral amount to seize based on the violator's liquidity.
    /// @param loansParams LiquidationLoansParams struct including the violator and liquidator loan ids, the pool ids and borrow type.
    /// @param userLoans The mapping of the loan ID to user loan including loan type, collaterals and borrows details.
    /// @param loanTypes The loan type data including a mapping with the pools' details.
    /// @param collPool The pool contract of the collateral.
    /// @param oracleManager The OracleManager contract.
    /// @param maxRepayBorrowValue The maximum borrow value to repay.
    /// @param maxAmountToRepay The amount to repay set by the liquidator.
    /// @return liquidationAmounts LiquidationAmountParams struct including the repay borrow amount and seize collateral amount.
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

                repayBorrowAmount = seizeUnderlyingCollateralAmount.convToRepayBorrowAmount( // @audit check here
                    collPriceFeed.price,
                    collPriceFeed.decimals,
                    borrPriceFeed.price,
                    borrPriceFeed.decimals,
                    borrowLoanPool.liquidationBonus
                );
            }

            liquidationAmounts.repayBorrowAmount = repayBorrowAmount;
            liquidationAmounts.repayBorrowToCollateralFAmount = repayBorrowAmount.convToCollateralFAmount( //@audit check here
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
    /// @dev Checks if the liquidator loan is over collateralized and the violator loan health is under the target.
    /// @param loansParams LiquidationLoansParams struct including the violator and liquidator loan ids and the pool ids.
    /// @param userLoans The mapping of the loan ID to user loan including loan type, collateral and borrow details.
    /// @param loanTypes The mapping of the type ID to loan types data including a mapping with the pools' details.
    /// @param pools The mapping of the pool ID to the pool contract.
    /// @param oracleManager The OracleManager contract.
    function checkLiquidatorLoan(
        DataTypes.LiquidationLoansParams memory loansParams,
        mapping(bytes32 => LoanManagerState.UserLoan) storage userLoans,
        mapping(uint16 => LoanManagerState.LoanType) storage loanTypes,
        mapping(uint8 => IHubPool) storage pools,
        IOracleManager oracleManager
    ) external view {
        LoanManagerState.UserLoan storage liquidatorLoan = userLoans[loansParams.liquidatorLoanId];
        if (!liquidatorLoan.isLoanOverCollateralized(pools, loanTypes[liquidatorLoan.loanTypeId].pools, oracleManager))
            revert UnderCollateralizedLoan(loansParams.liquidatorLoanId);
    }

    /// @dev Calculates the maximum borrow value to repay based on the violator's liquidity.
    /// @param violatorLiquidity LoanLiquidityParams struct including the violator's liquidity: effective collateral value and effective borrow value.
    /// @param borrowLoanPool The loan pool of the borrow.
    /// @param collateralFactor 4dp - The collateral factor of the collateral.
    /// @param loanTargetHealth 4dp - The target health of the loan.
    /// @return maxRepayBorrowValue The maximum borrow value to repay.
    function calcMaxRepayBorrowValue(
        DataTypes.LoanLiquidityParams memory violatorLiquidity,
        LoanManagerState.LoanPool storage borrowLoanPool,
        uint256 collateralFactor,
        uint32 loanTargetHealth
    ) internal view returns (uint256 maxRepayBorrowValue) {

        uint256 effectiveBorrowValueTarget = violatorLiquidity.effectiveBorrowValue.calcBorrowValueTarget(
            loanTargetHealth
        );

        uint256 borrowAdjustFactor = (loanTargetHealth * borrowLoanPool.borrowFactor) / MathUtils.ONE_4_DP;

        uint256 collateralAdjustFactor = (collateralFactor * (MathUtils.ONE_4_DP + borrowLoanPool.liquidationBonus)) / MathUtils.ONE_4_DP;

        maxRepayBorrowValue =((effectiveBorrowValueTarget - violatorLiquidity.effectiveCollateralValue) * MathUtils.ONE_4_DP) /
         (borrowAdjustFactor - collateralAdjustFactor); // @audit  check here

    }

    /// @dev Calculates the collateral fAmounts to seize: total, reserve and liquidator amounts.
    /// @param liquidationFee 4dp - The liquidation fee of the collateral.
    /// @param seizeCollateralFAmount The amount of collateral to seize.
    /// @return collateralSeized CollateralSeizedParams struct including the total amount, liquidator amount and reserve amount.
    function calcCollateralSeized(
        uint256 seizeCollateralFAmount,
        uint256 repayBorrowToCollateralFAmount,
        uint16 liquidationFee
    ) internal pure returns (DataTypes.CollateralSeizedParams memory collateralSeized) {
        collateralSeized.totalAmount = seizeCollateralFAmount;
        collateralSeized.reserveAmount = collateralSeized.totalAmount.calcReserveCol(
            repayBorrowToCollateralFAmount,
            liquidationFee
        );
        collateralSeized.liquidatorAmount = collateralSeized.totalAmount - collateralSeized.reserveAmount;
    }

    function updateLiquidationRewards(
        DataTypes.LiquidationLoansParams memory params,
        mapping(bytes32 => LoanManagerState.UserLoan) storage userLoans,
        mapping(uint16 loanTypeId => LoanManagerState.LoanType) storage loanTypes,
        mapping(bytes32 accountId => mapping(uint8 poolId => LoanManagerState.UserPoolRewards)) storage userPoolRewards
    ) internal {
        LoanManagerState.UserLoan storage liquidatorLoan = userLoans[params.liquidatorLoanId];
        LoanManagerState.UserLoan storage violatorLoan = userLoans[params.violatorLoanId];
        LoanManagerState.LoanType storage loanType = loanTypes[liquidatorLoan.loanTypeId];
        LoanManagerState.LoanPool storage collateralLoanPool = loanType.pools[params.collateralPoolId];
        LoanManagerState.LoanPool storage borrowLoanPool = loanType.pools[params.borrowPoolId];

        RewardLogic.updateRewardIndexes(collateralLoanPool, params.collateralPoolId);
        RewardLogic.updateRewardIndexes(borrowLoanPool, params.borrowPoolId);

        RewardLogic.updateUserCollateralReward(
            userPoolRewards,
            liquidatorLoan,
            collateralLoanPool,
            params.collateralPoolId
        );
        RewardLogic.updateUserBorrowReward(userPoolRewards, liquidatorLoan, borrowLoanPool, params.borrowPoolId);
        RewardLogic.updateUserCollateralReward(
            userPoolRewards,
            violatorLoan,
            collateralLoanPool,
            params.collateralPoolId
        );
        RewardLogic.updateUserBorrowReward(userPoolRewards, violatorLoan, borrowLoanPool, params.borrowPoolId);
    }
}

```

OutPut :-

```
  1) Liquidate Fails if the Collateral Factor is greater than Borrow Factor
       Should successfully liquidate stable borrow when seizing new borrow and collateral:
     Error: VM Exception while processing transaction: reverted with panic code 0x12 (Division or modulo division by zero)
    at LiquidationLogic.calcMaxRepayBorrowValue (contracts/hub/logic/LiquidationLogic.sol:313)
    at LiquidationLogic.getMaxRepayBorrowValue (contracts/hub/logic/LiquidationLogic.sol:184)
    at LoanManagerLogic.executeLiquidate (contracts/hub/logic/LoanManagerLogic.sol:457)
    at LoanManager.liquidate (contracts/hub/LoanManager.sol:248)
    at EdrProviderWrapper.request (node_modules/hardhat/src/internal/hardhat-network/provider/provider.ts:430:41)
    at async HardhatEthersSigner.sendTransaction (node_modules/@nomicfoundation/hardhat-ethers/src/signers.ts:125:18)
    at async send (node_modules/ethers/src.ts/contract/contract.ts:313:20)
    at async Proxy.liquidate (node_modules/ethers/src.ts/contract/contract.ts:352:16)
    at async Context.<anonymous> (test/hub/Liquidation.test.ts:781:23)

```

In Above output we can confirm that Liquidation functionality will be halted because of the Division by Zero because of calculation of the `maxRepayBorrowValue` will rounded down to Zero. Both StableBorrow and VariableBorrow will cause the panic Error.

We recommend to search the key `@audit` in poc to see the changes.
