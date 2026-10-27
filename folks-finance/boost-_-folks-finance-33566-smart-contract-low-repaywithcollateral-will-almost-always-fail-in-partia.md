# Boost \_ Folks Finance 33566 - \[Smart Contract - Low] RepayWithCollateral will almost always fail in

Submitted on Tue Jul 23 2024 15:06:10 GMT-0400 (Atlantic Standard Time) by @nnez for [Boost | Folks Finance](https://immunefi.com/bounty/folksfinance-boost/)

Report ID: #33566

Report type: Smart Contract

Report severity: Low

Target: <https://testnet.snowtrace.io/address/0x96e957bF63B5361C5A2F45C97C46B8090f2745C2>

Impacts:

* Contract fails to deliver promised returns, but doesn't lose value

## Description

## Description

Users can choose to pay their debt using the same type of collateral in their loan position by calling `repayWithCollateral` in `SpokeCommon` endpoint.\
`UserLoanLogic.sol#decreaseBorrow` is responsible for the deduction of borrow amount in the position.

```solidity
function decreaseBorrow(
    LoanManagerState.UserLoan storage loan,
    DataTypes.UpdateUserLoanBorrowParams memory params
) external returns (uint256 principalPaid, uint256 interestPaid, uint256 excessPaid, uint256 loanStableRate) {
    LoanManagerState.UserLoanBorrow storage loanBorrow = loan.borrows[params.poolId];

    loanStableRate = loanBorrow.stableInterestRate;

    updateLoanBorrowInterests(loanBorrow, params);

    uint256 balance = loanBorrow.balance; 
    uint256 interest = balance - loanBorrow.amount;
    excessPaid = params.amount > balance ? params.amount - balance : 0;
    interestPaid = Math.min(params.amount, interest); 

    principalPaid = params.amount - interestPaid - excessPaid;

    loanBorrow.amount -= principalPaid;
    balance -= principalPaid + interestPaid;

    if (balance == 0) clearBorrow(loan, params.poolId);
    loanBorrow.balance = balance;
}
```

According to the implementation of this function, interest payment precedes principal payment.\
To illustrate, let's say the current borrow `balance=120` and `amount=100`. If user decides to pay back their debt, `amount=20`\
The result would be `interestPaid=20` and `principalPaid=0`.

Almost at the end of the repayment process, the pool accounting information is updated in this function:

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

    pool.depositData.totalAmount -= principalPaid - interestPaid; @< highlighted
    repayWithCollateralPoolParams.fAmount = (principalPaid + interestPaid).toFAmount(
        pool.depositData.interestIndex
    );

    pool.updateInterestRates();
}
```

The highlighted line shows that `totalAmount` in the deposit pool is reduced by `principalPaid - interestPaid`. This assumes `principalPaid` is always greater than or equal to `interestPaid`.

However, `interestPaid` precedes `principalPaid`. If we pluck in the result from above, the execution will revert due to integer underflow (`interestPaid=20 > principalPaid=0`).

Thus, if users attempt to partially repay their debt with an amount less than twice the interest, the transaction will revert.

## Impact

* Contract fails to deliver promised returns, but doesn't lose value

## Recommend Mitigations

```solidity
if( principalPaid > interestPaid ) pool.depositData.totalAmount -= principalPaid - interestPaid;
```

## Proof of concept

## Proof-of-Concept

A test file in the secret gist shows that a user attempting to repay with collateral less than twice the owed interest will fail.

**Steps**

1. Run `forge init --no-commit --no-git --vscode`.
2. Create a new test file, `FolksLore.t.sol` in `test` directory.
3. Put the test from secret gist in the file: <https://gist.github.com/nnez/a050528180dd013ea6a7a632f6f50b7a>
4. Run `forge t --match-contract FolksLoreTest -vv`
5. Observe that the first partial repayment fails because the amount is less than 2x of owed interest.
