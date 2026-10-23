# Boost \_ Folks Finance 33807 - \[Smart Contract - Low] updateInterestRate uses incorrect reference of

Submitted on Mon Jul 29 2024 17:44:26 GMT-0400 (Atlantic Standard Time) by @nnez for [Boost | Folks Finance](https://immunefi.com/bounty/folksfinance-boost/)

Report ID: #33807

Report type: Smart Contract

Report severity: Low

Target: <https://testnet.snowtrace.io/address/0x96e957bF63B5361C5A2F45C97C46B8090f2745C2>

Impacts:

* Permanent freezing of unclaimed yield

## Description

## Summary

`updateInterestRate` function uses an old borrow interest rate to calculate the deposit interest rate. If a block concludes with an operation that raises the borrow interest rate, the deposit interest rate and deposit interest index will not be updated accordingly. This discrepancy can ultimately result in lenders losing their lending yield.

## Description

For any operation affecting the utilization ratio, `updateInterestRates` function is invoked to update the borrow and deposit interest rates.\
In its implementation, the function first loads the current data, including the interest rates, from `poolData`. It then calculates the new variable and stable borrow interest rates using the updated utilization ratio. However, when calculating the deposit interest rate, it uses the old borrow interest rates from the cached `poolData`. This results in a discrepancy between the new deposit interest rate and the old borrow interest rates.

```solidity
File: /contracts/hub/logic/HubPoolLogic.sol
function updateInterestRates(HubPoolState.PoolData storage poolData) internal {
        HubPoolState.PoolAmountDataCache memory poolAmountDataCache = getPoolAmountDataCache(poolData);
        uint256 totalDebt = poolAmountDataCache.variableBorrowTotalAmount + poolAmountDataCache.stableBorrowTotalAmount;
        uint256 utilisationRatio = MathUtils.calcUtilisationRatio(totalDebt, poolData.depositData.totalAmount);
        uint32 vr1 = poolData.variableBorrowData.vr1;

        // calculate new interest rates
        uint256 variableBorrowInterestRate = MathUtils.calcVariableBorrowInterestRate(
            poolData.variableBorrowData.vr0,
            vr1,
            poolData.variableBorrowData.vr2,
            utilisationRatio,
            poolData.depositData.optimalUtilisationRatio
        );
        uint256 stableBorrowInterestRate = MathUtils.calcStableBorrowInterestRate(
            vr1,
            poolData.stableBorrowData.sr0,
            poolData.stableBorrowData.sr1,
            poolData.stableBorrowData.sr2,
            poolData.stableBorrowData.sr3,
            utilisationRatio,
            poolData.depositData.optimalUtilisationRatio,
            MathUtils.calcStableDebtToTotalDebtRatio(poolAmountDataCache.stableBorrowTotalAmount, totalDebt),
            poolData.stableBorrowData.optimalStableToTotalDebtRatio
        );
        uint256 depositInterestRate = MathUtils.calcDepositInterestRate(
            utilisationRatio,
            MathUtils.calcOverallBorrowInterestRate(
                poolAmountDataCache.variableBorrowTotalAmount,
                poolAmountDataCache.stableBorrowTotalAmount,
                -->@ poolData.variableBorrowData.interestRate, @<-- use an old one
                -->@ poolData.stableBorrowData.averageInterestRate @<-- use an old one
            ),
            poolData.feeData.retentionRate
        );

        // update interest rates
        poolData.variableBorrowData.interestRate = variableBorrowInterestRate;
        poolData.stableBorrowData.interestRate = stableBorrowInterestRate;
        poolData.depositData.interestRate = depositInterestRate;

        emit InterestRatesUpdated(variableBorrowInterestRate, stableBorrowInterestRate, depositInterestRate);

File: /contracts/hub/libraries/MathUtils.sol
    function calcOverallBorrowInterestRate(
        uint256 totalVarDebt,
        uint256 totalStblDebt,
        uint256 variableBorrowInterestRateAtT,
        uint256 avgStableBorrowInterestRateAtT
    ) internal pure returns (uint256) {
        uint256 totalDebt = totalVarDebt + totalStblDebt;
        return
            totalDebt > 0
                ? (totalVarDebt.mulDiv(variableBorrowInterestRateAtT, ONE_18_DP) +
                    totalStblDebt.mulDiv(avgStableBorrowInterestRateAtT, ONE_18_DP)).mulDiv(ONE_18_DP, totalDebt)
                : 0;
    }

    function calcDepositInterestRate(
        uint256 utilisationRatioAtT,
        uint256 overallBorrowInterestRateAtT,
        uint32 retentionRate
    ) internal pure returns (uint256) {
        return
            utilisationRatioAtT.mulDiv(overallBorrowInterestRateAtT, ONE_18_DP).mulDiv(
                ONE_6_DP - retentionRate,
                ONE_6_DP
            );
    }
```

To illustrate more on the issue, let's consider the fUSDC pool with the following initial state:\
fUSDC pool's setting:

* Vr0 = 0.0175
* Vr1 = 0.0500
* Vr2 = 1
* optimalUtilsation = 0.85
* RR (Retention Rate) = 0.1

Formula for interest rate calculation: <https://docs.google.com/document/d/1UU-zhy-Ik6h-EhKS2TvcIsd0Q377H7HKF6MGP5WdwAk/>

| Pool's state     |                                                      |
| ---------------- | ---------------------------------------------------- |
| totalAmount      | 100e6                                                |
| totalDebt        | 20e6                                                 |
| utilisationRatio | 0.2                                                  |
| borrowInterest   | 0.0175 + (0.2/0.85)\*0.0500=0.02926470588=**2.926%** |

Now, supposed that **ALICE** borrows another 20e6 at the end of the block.\
The final state would be the following:

| Pool's state     |                                                      |
| ---------------- | ---------------------------------------------------- |
| totalAmount      | 100e6                                                |
| totalDebt        | 40e6                                                 |
| utilisationRatio | 0.4                                                  |
| borrowInterest   | 0.0175 + (0.4/0.85)\*0.0500=0.04102941176=**4.103%** |

And according to the implementation, we can calculate deposit interest from the following:

`Ut * overallBorrowInterest * (1-RR)`\
`0.4 * (40e6*0.02926470588/40e6) * (1-0.1) = 0.01053529411`

Supposed that this block ends with this borrow operation (raising borrow amount), and the next block comes in next 2 seconds The interest accrue from the last block to this block would be:

deposit interest: `100e6 * 0.01053529411 * 2 / (365*86400) = 0.06681439694`\
borrow interest: `40e6 * 0.04102941176 * 2 / (365*86400) = 0.10408272897`

We can see that accrued deposit interest is over 50% less than expected (even if we include the retention rate) compare to borrow interest.\
This discrepancy results in a loss of unclaimed yield for depositors/lenders.

The loss amplifies when the pool and utilisation rate grows larger.

## Impact

* Lenders lose their unclaimed yield if the block ends with operation that raise borrow interest rate.

## Proof of concept

## Proof-of-Concept

A test included in the secret gist simulates the above situation. It shows that the interest rate and accured deposit interest are different when the block ends with borrow operation compared to the block that ends with deposit (with 0 amount).

**Steps**

1. Run `forge init --no-commit --no-git --vscode`.
2. Create a new test file, `FolksDepositInterest.t.sol` in `test` directory.
3. Put the test from secret gist in the file: <https://gist.github.com/nnez/10a59d5a40ad57ca50beb6ae68ee41a9>
4. Run `forge t --match-contract FolksLoreTest -vv`
5. Observe that in the first case, accured deposit interest is less than the second case.
