# Boost \_ Folks Finance 33870 - \[Smart Contract - Low] convToRepayBorrowAmount calculation is incorrec

Submitted on Wed Jul 31 2024 14:50:42 GMT-0400 (Atlantic Standard Time) by @nnez for [Boost | Folks Finance](https://immunefi.com/bounty/folksfinance-boost/)

Report ID: #33870

Report type: Smart Contract

Report severity: Low

Target: <https://testnet.snowtrace.io/address/0xc1FBF54B25816B60ADF322d8A1eaCA37D9A50317>

Impacts:

* Protocol insolvency

## Description

## Description

Folks Finance uses a liquidation bonus to incentivize liquidators to quickly liquidate underwater positions, protecting the protocol from bad debt.

Liquidators are rewarded by seizing violators' collateral at a discounted price relative to the debt they repay.\
For example, with a 5% liquidation bonus, liquidators repaying $100 would seize $105 worth of the violator's collateral.

The total amount of collateral seized with liquidation bonus is calculated in **LiquidationLogic.sol#calcLiquidationAmounts**.

```solidity
File: /contracts/hub/logic/LiquidationLogic.sol
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
        highliight #1 -->@ uint256 seizeUnderlyingCollateralAmount = repayBorrowAmount.convToSeizedCollateralAmount(
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
        highliight #2 -->@ if (seizeUnderlyingCollateralAmount > violatorUnderlingCollateralBalance) {
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

File: /contracts/hub/libraries/MathUtils.sol

function convToSeizedCollateralAmount(
    uint256 borrowAmount,
    uint256 collPrice,
    uint8 collDecimals,
    uint256 borrPrice,
    uint8 borrDecimals,
    uint256 liquidationBonus
) internal pure returns (uint256) {
    return
        Math.mulDiv(
            convertAssetAmount(borrowAmount, borrPrice, borrDecimals, collPrice, collDecimals),
            (MathUtils.ONE_4_DP + liquidationBonus),
            MathUtils.ONE_4_DP
        );
}

function convToRepayBorrowAmount(
    uint256 collAmount,
    uint256 collPrice,
    uint8 collDecimals,
    uint256 borrPrice,
    uint8 borrDecimals,
    uint256 liquidationBonus
) internal pure returns (uint256) {
    return
        Math.mulDiv(
            convertAssetAmount(collAmount, collPrice, collDecimals, borrPrice, borrDecimals),
            (MathUtils.ONE_4_DP + liquidationBonus),
            MathUtils.ONE_4_DP
        );
}
```

The seized collateral amount is calculated using `convToSeizedCollateralAmount`. If this amount exceeds the violator's collateral position, the total collateral is assigned as the seized amount, and the repay amount is then recalculated using `convToRepayBorrowAmount`.

However, the calculation in `convToRepayBorrowAmount` is incorrect. The transformed equation is:

`collateralAmountInBorrowAsset * (1 + liquidationBonus)`

This will always result in a higher repay amount for the given collateral amount.\
For instance, with a 5% liquidation bonus, liquidators seizing $100 worth of the violator's collateral would have to repay $105.\
As a result, instead of liquidators seizing collateral at a discount, they end up paying an extra percentage for the same collateral amount.

## Impact

* Discourage liquidators to liquidate underwater position in some cases, which eventually will lead to bad debt.

## Rationale for Severity

If liquidation becomes ineffective or unprofitable, it could discourage liquidators from participating, eventually leading to bad debt for the protocol, as no one would be willing to take a loss. However, in this particular case, it would only happen in specific situations.

Therfore,\
**Damage: High**\
**Likilihood: Medium**\
Hence, **High** severity.

or it could be **Medium** if we think differently about the **Likelihood**.

## Proof of concept

## Proof-of-Concept

The test does the following:

1. Mock the price of USDC at 1$ and AVAX at 10$
2. ALICE deposits 100 USDC and borrow 7 AVAX
3. Mock the price of AVAX at 100$
4. ALICE is underwater, C=100$ D=700$
5. BOB deposits 1\_000 USDC in preparation of liquidating ALICE
6. BOB liquidates ALICE, trying to seize all collateral by repaying 1 AVAX (=100$)
7. BOB should only have to repay less than 1 AVAX to seize 100 USDC
8. BOB seizes 100 USDC but takes 1.07 AVAX (=107$) due to the wrong calculation

**Steps**

1. Run `forge init --no-commit --no-git --vscode`.
2. Create a new test file, `FolksLiquidation.t.sol` in `test` directory.
3. Put the test from secret gist in the file: <https://gist.github.com/nnez/3ad342ab2b5c8556afe854a863296e97>
4. Run `forge t --match-contract FolksLiquidationTest -vv`
5. Observe that BOB has taken in debt more than he should.

**Expected Result**

```
[PASS] test_convToRepayBorrowAmountIncorrectCalculation() (gas: 2194937)
Logs:
  @> Mocking price | USDC=1$ | AVAX=10$
  @> ALICE deposits USDC and borrows AVAX to create a loan position
  @> ALICE successfully borrowed 7e18 AVAX
  @> BOB deposits 1_000e6 USDC in preparation of liquidating ALICE's position
  @> BOB deposits 1_000e6 USDC
  @> BOB successfully deposited 1_000e6 USDC
  @> Mock AVAX price | AVAX=100$
  @> ALICE's position is now underwater
  @> BOB's USDC in collateral (before liquidation): 9.99999999e8
  @> BOB attempts to liquidate ALICE's position to seize all of her collateral
  @> BOB successfully liquidated ALICE's position
  @> BOBs' USDC in collateral (after liquidation): 1.099999998e9
  @> BOBs' seized USDC: 9.9999999e7 ($99)
  @> BOB's AVAX in debt: 1.0799999892e18 ($107)

Suite result: ok. 1 passed; 0 failed; 0 skipped; finished in 760.99ms (16.09ms CPU time)
```
