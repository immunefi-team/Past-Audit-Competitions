# Boost \_ Folks Finance 33596 - \[Smart Contract - Low] Incorrect rounding direction in HubPoolLogicupd

Submitted on Wed Jul 24 2024 08:13:31 GMT-0400 (Atlantic Standard Time) by @nnez for [Boost | Folks Finance](https://immunefi.com/bounty/folksfinance-boost/)

Report ID: #33596

Report type: Smart Contract

Report severity: Low

Target: <https://testnet.snowtrace.io/address/0x96e957bF63B5361C5A2F45C97C46B8090f2745C2>

Impacts:

* Contract fails to deliver promised returns, but doesn't lose value

## Description

## Description

Users can choose to repay their debt with the same type of collateral in their loan by calling `repayWithCollateral` function in `SpokeCommon` endpoint.\
At almost the end of the execution flow, the final accounting is done in the following function:

```solidity
File: /contracts/hub/logic/HubPoolLogic.sol
function updateWithRepayWithCollateral(
    HubPoolState.PoolData storage pool,
    uint256 principalPaid,
    uint256 interestPaid,
    uint256 loanStableRate
) external returns (DataTypes.RepayWithCollateralPoolParams memory repayWithCollateralPoolParams) {
    if (loanStableRate > 0) {
        pool.stableBorrowData.averageInterestRate = MathUtils.calcDecreasingAverageStableBorrowInterestRate(
            principalPaid,
            loanStableRate,
            pool.stableBorrowData.totalAmount,
            pool.stableBorrowData.averageInterestRate
        );
        pool.stableBorrowData.totalAmount -= principalPaid;
    } else pool.variableBorrowData.totalAmount -= principalPaid;

    pool.depositData.totalAmount -= principalPaid - interestPaid;
    repayWithCollateralPoolParams.fAmount = (principalPaid + interestPaid).toFAmount(
        pool.depositData.interestIndex
    );

    pool.updateInterestRates();
}

File: /contracts/hub/libraries/MathUtils.sol
function toFAmount(uint256 underlyingAmount, uint256 depositInterestIndexAtT) internal pure returns (uint256) {
    return underlyingAmount.mulDiv(ONE_18_DP, depositInterestIndexAtT);
}

File: /contracts/hub/logic/LoanManagerLogic.sol
function executeRepayWithCollateral(
    ...
    ... snipped
    ...
    // update the pool
    DataTypes.RepayWithCollateralPoolParams memory repayWithCollateralPoolParams = pool
        .updatePoolWithRepayWithCollateral(principalPaid, interestPaid, loanStableRate);

    // decrease the user loan collateral and global collateral used for loan type
    userLoan.decreaseCollateral(params.poolId, repayWithCollateralPoolParams.fAmount);
    loanPool.decreaseCollateral(repayWithCollateralPoolParams.fAmount);
    ...
    ... snipped
    ...
```

A rounding issue occurs in the `updateWithRepayWithCollateral` function. If `principalPaid + interestPaid` equals 1 wei, the `toFAmount` function rounds down to zero.

The problem arises because `repayWithCollateralPoolParams.fAmount` is used to reduce the user's collateral amount used in repayment. This leads to a situation where the debt is repaid, the token amount is deducted from the HubPool (from `depositData.totalAmount`) but the corresponding collateral is not deducted.

Although the accounting error is in a dust amount, I believe this is a technically valid bug and should be fixed to ensure the correctness of the accounting in the HubPool.

## Impact

* Accounting error on the totalAmount in HubPool

## Proof of concept

## Proof-of-Concept

A test file in the secret gist does the following:

* Deposit USDC, then borrow USDC.
* RepayWithCollateral with amount=1 wei.
* Show that debt is repaid for 1 wei, but collateral balance is not deducted.

**Steps**

1. Run `forge init --no-commit --no-git --vscode`.
2. Create a new test file, `FolksRounding.t.sol` in `test` directory.
3. Put the test from secret gist in the file: <https://gist.github.com/nnez/cfe1bc405bcf307a1fa36e24f0dc9736>
4. Run `forge t --match-contract FolksRoundingTest -vv`
5. Observe that debt is repaid for 1 wei, but collateral balance is not deducted.
