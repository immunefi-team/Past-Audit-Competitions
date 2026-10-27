# Boost \_ Folks Finance 34183 - \[Smart Contract - Insight] rebalanceUp could be used to lower the user

Submitted on Tue Aug 06 2024 07:27:50 GMT-0400 (Atlantic Standard Time) by @alix\_40 for [Boost | Folks Finance](https://immunefi.com/bounty/folksfinance-boost/)

Report ID: #34183

Report type: Smart Contract

Report severity: Insight

Target: <https://testnet.snowtrace.io/address/0x2cAa1315bd676FbecABFC3195000c642f503f1C9>

Impacts:

* Contract fails to deliver promised returns, but doesn't lose value

## Description

> This report is intended to be submitted under my team account "A2Security" but I reached the report submission rate limit on the last day. Please count this report as though it were from "A2Security".

## Impact

users could use rebalanceUp to rebalance down, if their loan.stableInterest

## Recomendation

To fix this we need to simply check that the userloan.stableInterestRate is not lower than the pool interest rate

```diff
    function executeRebalanceUp(
        mapping(bytes32 => LoanManagerState.UserLoan) storage userLoans,
        mapping(uint8 => IHubPool) storage pools,
        DataTypes.ExecuteRebalanceParams memory params
    ) external {
        LoanManagerState.UserLoan storage userLoan = userLoans[params.loanId];

        // user cannot rebalance borrow which they don't have or is not stable
        // borrow present iff loan type created and pool added so no need to check this
        if (!userLoan.hasStableBorrowIn(params.poolId)) {
            revert NoStableBorrowInLoanForPool(params.loanId, params.poolId);
        }

        // pool pre-checks and update interest indexes
        IHubPool pool = pools[params.poolId];
        // @note update index + checks threshold

        DataTypes.BorrowPoolParams memory borrowPoolParams = pool.preparePoolForRebalanceUp();
+       if (borrowPoolParams.stableInterestRate < userLoan.stableInterestRate) {
+       revert RateLowerThanPoolRate
+    }
        // rebalance the user loan borrow
        LoanManagerState.UserLoanBorrow storage loanBorrow = userLoan.borrows[params.poolId];
    }
```

## Proof of concept

## Proof Of Concept

```solidity
    function prepareForRebalanceUp(HubPoolState.PoolData storage pool) external returns (DataTypes.BorrowPoolParams memory borrowPoolParams) {
        // can rebalance even if pool is depreciated
        // update interest indexes before the interest rates change
        pool.updateInterestIndexes();
        uint256 utilizationRatio = MathUtils.calcUtilisationRatio(pool.variableBorrowData.totalAmount + pool.stableBorrowData.totalAmount, pool.depositData.totalAmount);
        // @note Calculates threshold as: (rebalanceUpRate(capsdata) * totalVariableRate) / 100
        // @note totalVariableRate = varaible.vr0 + variable.vr1 + variable.vr2
        uint256 rebalanceUpThreshold = MathUtils.calcRebalanceUpThreshold(
            pool.stableBorrowData.rebalanceUpDepositInterestRate, pool.variableBorrowData.vr0, pool.variableBorrowData.vr1, pool.variableBorrowData.vr2
        );
        // @note U can't rebalance UP if we are under a rebalance threshold
        // check conditions for rebalance
 @1       if (utilizationRatio < MathUtils.from4DPto18DP(pool.stableBorrowData.rebalanceUpUtilisationRatio)) {
            revert RebalanceUpUtilisationRatioNotReached();
        }
 @2       if (rebalanceUpThreshold < pool.depositData.interestRate) revert RebalanceUpThresholdNotReached();

        borrowPoolParams.variableInterestIndex = pool.variableBorrowData.interestIndex;
        borrowPoolParams.stableInterestRate = pool.stableBorrowData.interestRate;
    }
```

As we can see in the rates checks in `prepareForRebalanceUp()` we don't use the userLoan.stableInterestRate and don't do any validation on it, meaning if the userLoan.stableRate is bigger than pool.stableInterestRate, and the condition @1 and @2 are fullfilled a user can still call RebalanceUp to lower the stableRate of his loan
