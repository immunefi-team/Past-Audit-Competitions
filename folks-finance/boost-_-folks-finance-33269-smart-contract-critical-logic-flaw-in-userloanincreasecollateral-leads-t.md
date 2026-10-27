# Boost \_ Folks Finance 33269 - \[Smart Contract - Critical] Logic flaw in UserLoanincreaseCollateral l

Submitted on Tue Jul 16 2024 17:44:11 GMT-0400 (Atlantic Standard Time) by @nnez for [Boost | Folks Finance](https://immunefi.com/bounty/folksfinance-boost/)

Report ID: #33269

Report type: Smart Contract

Report severity: Critical

Target: <https://testnet.snowtrace.io/address/0xf8E94c5Da5f5F23b39399F6679b2eAb29FE3071e>

Impacts:

* Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield
* Protocol insolvency

## Description

## Summary

Malicious users can call a deposit function with amount=0 to make `increaseCollateral` pushes the same `poolId` into `colPools`, causing `getLoanLiquidity` to double-count the total collateral effective value of the loan.

## Description

Users' loan position is stored in a mapping of a struct `UserLoan`

```
struct UserLoan {
    bool isActive;
    bytes32 accountId;
    uint16 loanTypeId;
    uint8[] colPools;
    uint8[] borPools;
    mapping(uint8 poolId => UserLoanCollateral) collaterals;
    mapping(uint8 poolId => UserLoanBorrow) borrows;
}
mapping(bytes32 loanId => UserLoan) internal _userLoans;
```

A member named `colPools` stores a list of `poolId` that users are using in their position.\
This is analogous to a list of markets user enters in other lending platforms.

This list is used in a calculation of users' effective collateral value.\
See: <https://github.com/Folks-Finance/folks-finance-xchain-contracts/blob/main/contracts/hub/logic/UserLoanLogic.sol#L283-L291\\>
and <https://github.com/Folks-Finance/folks-finance-xchain-contracts/blob/main/contracts/hub/logic/UserLoanLogic.sol#L230-L245>

```
function isLoanOverCollateralized(
    LoanManagerState.UserLoan storage loan,
    mapping(uint8 poolId => IHubPool) storage pools,
    mapping(uint8 poolId => LoanManagerState.LoanPool) storage loanPools,
    IOracleManager oracleManager
) internal view returns (bool) {
    DataTypes.LoanLiquidityParams memory loanLiquidity = getLoanLiquidity(loan, pools, loanPools, oracleManager);
    return loanLiquidity.effectiveCollateralValue >= loanLiquidity.effectiveBorrowValue;
}
...
...
function getLoanLiquidity(
    LoanManagerState.UserLoan storage loan,
    mapping(uint8 => IHubPool) storage pools,
    mapping(uint8 => LoanManagerState.LoanPool) storage loanPools,
    IOracleManager oracleManager
) internal view returns (DataTypes.LoanLiquidityParams memory loanLiquidity) {
    // declare common variables
    uint256 effectiveValue;
    uint256 balance;
    uint8 poolId;
    uint256 poolsLength;
    DataTypes.PriceFeed memory priceFeed;

    // calc effective collateral value
    poolsLength = loan.colPools.length;
    for (uint8 i = 0; i < poolsLength; i++) {
        poolId = loan.colPools[i];

        balance = loan.collaterals[poolId].balance.toUnderlingAmount(
            pools[poolId].getUpdatedDepositInterestIndex()
        );
        priceFeed = oracleManager.processPriceFeed(poolId);
        effectiveValue += MathUtils.calcCollateralAssetLoanValue(
            balance,
            priceFeed.price,
            priceFeed.decimals,
            loanPools[poolId].collateralFactor
        );
    }
    loanLiquidity.effectiveCollateralValue = effectiveValue;
    ...snipped...
    ...snipped...
```

It iterates over `colPool` to get all `poolId` and uses them to retrieve current deposit balance of the loan for each pool

Below is the code snippet responsible for adding a new `poolId` to user's loan.\
See: <https://github.com/Folks-Finance/folks-finance-xchain-contracts/blob/main/contracts/hub/logic/UserLoanLogic.sol#L20-L25>

```
function increaseCollateral(LoanManagerState.UserLoan storage loan, uint8 poolId, uint256 fAmount) external {
    // if the balance was prev zero, add pool to list of user loan collaterals
    if (loan.collaterals[poolId].balance == 0) loan.colPools.push(poolId);

    loan.collaterals[poolId].balance += fAmount;
}
```

If the loan is currently holding 0 token, then it pushes a `poolId` into `colPools`.\
However, this logic is flawed because the deposit amount of `0` is allowed. (I presume, since there is no validation preventing it).\
See: <https://github.com/Folks-Finance/folks-finance-xchain-contracts/blob/main/contracts/hub/LoanManager.sol#L74-L110> and <https://github.com/Folks-Finance/folks-finance-xchain-contracts/blob/main/contracts/hub/logic/LoanManagerLogic.sol#L66-L151>

Calling deposit function with amount=0 twice or more will cause `increaseCollateral` to push the same `poolId` into `colPools`.\
As a result, `getLoanLiquidity` will retrieve a duplicate of `poolId` and double or triple count the effective collateral value.

## Impact

Adversaries can leverage double-counting (or more) to steal funds from the contract by borrowing more than their effective collateral value.

## Proof of concept

## Proof-of-Concept

I modify a test in `LoanManager.test.ts` to demonstrate that calling deposit function with amount=0 will push the same `poolId` into `colPools`.\
I use `getUserLoan` to retrieve the current status of user's loan to show that `colPools` stores duplicates of `poolId`.

**Note**: `getUserLoan` has the following interface:

```
function getUserLoan(
    bytes32 loanId
) external view
returns (
    bytes32 accountId,
    uint16 loanTypeId,
    uint8[] memory colPools,
    uint8[] memory borPools,
    UserLoanCollateral[] memory,
    UserLoanBorrow[] memory
)
```

**Steps to reproduce**

1. Set up a project as per protocol's **README**
2. Create a new file name `DoubleCounting.test.ts` in `/test/hub/DoubleCounting.test.ts` with the code in this secret gist: <https://gist.github.com/nnez/aa74df43103230004697438d4471c43f>
3. Run the test in the root of directory, `npx hardhat test test/hub/DoubleCounting.test.ts --grep "double-counting of collateral"`

Expected result:

```
  LoanManager (unit tests)
    Deposit F Token
First deposit:  Result(6) [
  '0x000000000000000000000000000000000000000000004143434f554e545f4944',
  1n,
  Result(1) [ 2n ],
  Result(0) [],
  Result(1) [ Result(2) [ 0n, 0n ] ],
  Result(0) []
]
-----
Second deposit:  Result(6) [
  '0x000000000000000000000000000000000000000000004143434f554e545f4944',
  1n,
  Result(2) [ 2n, 2n ],
  Result(0) [],
  Result(2) [ Result(2) [ 0n, 0n ], Result(2) [ 0n, 0n ] ],
  Result(0) []
]
-----
Last deposit:  Result(6) [
  '0x000000000000000000000000000000000000000000004143434f554e545f4944',
  1n,
  Result(3) [ 2n, 2n, 2n ],
  Result(0) [],
  Result(3) [
    Result(2) [ 2000000000000000000n, 0n ],
    Result(2) [ 2000000000000000000n, 0n ],
    Result(2) [ 2000000000000000000n, 0n ]
  ],
  Result(0) []
]
-----
      ✔ Should demonstrate double-counting of collateral (1532ms)


  1 passing (2s)
```
