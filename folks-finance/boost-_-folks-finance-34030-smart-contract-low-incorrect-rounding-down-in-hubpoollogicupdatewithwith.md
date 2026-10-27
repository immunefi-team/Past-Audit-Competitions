# Boost \_ Folks Finance 34030 - \[Smart Contract - Low] Incorrect rounding down in HubPoolLogicupdateWi

Submitted on Sun Aug 04 2024 10:17:19 GMT-0400 (Atlantic Standard Time) by @nnez for [Boost | Folks Finance](https://immunefi.com/bounty/folksfinance-boost/)

Report ID: #34030

Report type: Smart Contract

Report severity: Low

Target: <https://testnet.snowtrace.io/address/0x96e957bF63B5361C5A2F45C97C46B8090f2745C2>

Impacts:

* Contract fails to deliver promised returns, but doesn't lose value
* Smart contract unable to operate due to lack of token funds

## Description

## Description

Users' deposit balance is stored in *fToken*\* amount. **fToken** is a type of interest-bearing token so its exchange rate back to underlying amount grows as interest index grows. Users who want to withdraw can specify whether to withdraw an exact underlying amount or withdraw with their **fToken** amount.

When users withdraw by specifying an underlying amount, the `toFAmount` function converts this amount back to fToken. The conversion formula utilized in `toFAmount` is as follows:

```solidity
File: /contracts/hub/logic/HubPoolLogic.sol
function updateWithWithdraw(
    HubPoolState.PoolData storage pool,
    uint256 amount,
    bool isFAmount
) external returns (DataTypes.WithdrawPoolParams memory withdrawPoolParams) {
    // can withdraw even if pool is depreciated
    // update interest indexes before the interest rates change
    pool.updateInterestIndexes();

    if (isFAmount) {
        withdrawPoolParams.fAmount = amount;
        withdrawPoolParams.underlingAmount = amount.toUnderlingAmount(pool.depositData.interestIndex);
    } else {
        withdrawPoolParams.underlingAmount = amount;
        withdrawPoolParams.fAmount = amount.toFAmount(pool.depositData.interestIndex); @<-- incorrect rounding down
    }

    pool.depositData.totalAmount -= withdrawPoolParams.underlingAmount;
    pool.updateInterestRates();
}

File: /contracts/hub/libraries/MathUtils.sol
function toFAmount(uint256 underlyingAmount, uint256 depositInterestIndexAtT) internal pure returns (uint256) {
    return underlyingAmount.mulDiv(ONE_18_DP, depositInterestIndexAtT);
}
```

We can clearly see that `toFAmount` rounds down the division result. This causes an issue when users withdraw small amounts, like 1 wei. The `fToken` amount may round down to zero, allowing users to withdraw 1 wei of underlying token without correctly reducing their `fToken` balance.

## Impact

Adversaries can continously withdraw 1 wei of underlying amount of token without losing `fToken` balance. Although the gas cost from transction would offset attacker's profit, it can potentially cause an accounting error in HubPool's `totalAmount`.

## Proof of concept

## Proof-of-Concept

A test in the secret gist demonstrates the following situation:

1. BOB deposits 1e6 USDC and in his loan.
2. BOB withdraws all in fUSDC amount, but left 1 wei of USDC in his loan.
3. BOB withdraws again in underling amount, specifying 1 wei.
4. BOB gets 1 wei of USDC but still keep his 1 wei of fUSDC in the loan.

### Steps

1. Run `forge init --no-commit --no-git --vscode`.
2. Create a new test file, `FolksWithdraw.t.sol` in `test` directory.
3. Put the test from secret gist in the file: <https://gist.github.com/nnez/ee5f1791aa5083286932bb3fd9c78e71>
4. Run `forge t --match-contract FolksWithdrawRoundingTest -vv`
5. Observe that BOB's usdc balance is up by 20 wei but he loses 0 amount of fUSDC in his loan.
