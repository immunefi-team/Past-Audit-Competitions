# Mitigation Audit \_ Folks Finance 34929 - \[Smart Contract - Critical] Accounting Discrepancy in Fee R

Submitted on Sun Sep 01 2024 10:46:32 GMT-0400 (Atlantic Standard Time) by @A2Security for [Mitigation Audit | Folks Finance](https://immunefi.com/bounty/mitigation-audit-folksfinance/)

Report ID: #34929

Report type: Smart Contract

Report severity: Critical

Target: <https://github.com/Folks-Finance/folks-finance-xchain-contracts/pull/75>

Impacts:

* Permanent freezing of funds
* Protocol insolvency

## Description

## Brief/Intro

The current implementation of fee retention and withdrawal in the lending pool system has a critical accounting issue that can lead to insolvency and permanent loss of funds.

## Vulnerability Details

The bug arises form the inacurate calculation of available liquidity in the pool. The liquidity check has been added to prevent a criticial bug that we already reported in the first boost.

To check for available liquidity in the pool, the function `calcAvailableLiquidity()` has been added. It calculates the available liquidity by subtracting the total debt from the total deposit data

```js

    function prepareForBorrow(
        HubPoolState.PoolData storage pool,
        uint256 amount,
        DataTypes.PriceFeed memory priceFeed,
        uint256 maxStableRate
    ) external returns (DataTypes.BorrowPoolParams memory borrowPoolParams) {
        if (pool.isDeprecated()) revert DeprecatedPool();

        bool isStable = maxStableRate > 0;
        uint256 stableBorrowInterestRate = pool.stableBorrowData.interestRate;
        uint256 totalDebt = pool.variableBorrowData.totalAmount + pool.stableBorrowData.totalAmount;
@>>        if (amount > MathUtils.calcAvailableLiquidity(totalDebt, pool.depositData.totalAmount))
            revert InsufficientLiquidity();
```

This check is done on each operations that will results in funds out flow from the protocol (e.g borrow, withdraw) and is done in the `prepareForBorrow` and `prepareForWithdraw` functions.

The problem arises from the fact that the protocol misshandles the retainedFee claculations, leading to an incorrect `depositData.totalAmount` which inflates the available liquidity calculation in the pool. The available liquidity is calculated as `totalDeposit.totalAmount - totalDebt`, totalDeposit.totalAmount however includes the retained fees,

```js
    function calcAvailableLiquidity(uint256 totalDebt, uint256 totalDeposits) internal pure returns (uint256) {
@>>        return totalDeposits - totalDebt;
    }
```

The retained fees are updated in the `updateInterestIndexes()` function, which increase it by taking a protocol fee from the accrued debt interest. The problem however arises in the repayment, when repaying all the accrued interest assossiate with a borrow, will be added to the depositData.totalAmount. This accrued Interst however also **contains the retained fees**.

```js

    function updateWithRepay(
        HubPoolState.PoolData storage pool,
        uint256 principalPaid,
        uint256 interestPaid,
        uint256 loanStableRate,
        uint256 excessAmount
    ) external {
        if (loanStableRate > 0) {
            pool.stableBorrowData.averageInterestRate = MathUtils.calcDecreasingAverageStableBorrowInterestRate(
                principalPaid,
                loanStableRate,
                pool.stableBorrowData.totalAmount,
                pool.stableBorrowData.averageInterestRate
            );
            pool.stableBorrowData.totalAmount -= principalPaid;
        } else pool.variableBorrowData.totalAmount -= principalPaid;

        pool.feeData.totalRetainedAmount += excessAmount;
@>>       pool.depositData.totalAmount += interestPaid;
        pool.updateInterestRates();
    }
```

**So the pool.depositData.totalAmount will be inflated by the retained fees, leading to a false available liquidity calculation. This effect will compound with each fee withdrawal by the protocol. As the totalAmount will still include the retained fees, even though the protocol have withdrawn the fees.**

```js
    function clearTokenFees() external override onlyRole(HUB_ROLE) nonReentrant returns (uint256) {
        uint256 amount = _poolData.feeData.totalRetainedAmount;
        _poolData.feeData.totalRetainedAmount = 0;

        emit ClearTokenFees(amount);
        return amount;
    }
```

As we can see, the depositData.totalAmount is not adjusted when the protcol withdraws the retained fees.

## Impact Details

the impacts of this issue :

1. **protocol insolvency** : This insolvency occurs because the protocol's will recorded liabilities exceed its actual assets (available funds). As retained fees are withdrawn without adjusting the total pool amount, which grow by time. This leads to a situation where the protocol cannot fulfill its obligations to all depositors, even if all loans were repaid, meeting the definition of insolvency .
2. **Permanent freezing of funds** : Permanent freezing of funds occurs when a user's withdrawal or borrow request slightly exceeds the spoke contract's balance. The entire transaction reverts, but the hub records it as successful. This leads to the user losing access to their full requested amount, not just the excess, potentially locking up significant funds indefinitely.
3. **incorrect intrest rate calculation** : this issue is basically the same [here](https://github.com/Folks-Finance/folks-finance-xchain-contracts/issues/44) , but here however it doesn't lead to Utilisation ration exceeding 100%. but it still effects the intrest rate calculation though (calculate interest rate based on inflated totalDeposit)

## References

* <https://github.com/Folks-Finance/folks-finance-xchain-contracts/blob/ee9b4a85e6b1ef11032f6cf90fadf87d065036ec/contracts/hub/Hub.sol#L55-L79>
* <https://github.com/Folks-Finance/folks-finance-xchain-contracts/blob/ee9b4a85e6b1ef11032f6cf90fadf87d065036ec/contracts/hub/logic/HubPoolLogic.sol#L159>
* <https://github.com/Folks-Finance/folks-finance-xchain-contracts/blob/ee9b4a85e6b1ef11032f6cf90fadf87d065036ec/contracts/hub/HubPool.sol#L58-L64>

### Recommendation :

To address this issue, we recommend two changes:

1. Subtract the retained amount from `pool.depositData.totalAmount` when clearing fees.
2. Add the `excessAmount` to `pool.depositData.totalAmount` when repaying to accurately calculate the available balance.

```diff
// In HubPoolLogic library:
function updateWithRepay(/*params*/) external {
    // Previous code...
    pool.feeData.totalRetainedAmount += excessAmount;
-   pool.depositData.totalAmount += interestPaid;
+   pool.depositData.totalAmount += interestPaid + excessAmount;
    pool.updateInterestRates();
}

// In HubPool contract:
function clearTokenFees() external override onlyRole(HUB_ROLE) nonReentrant returns (uint256) {
    uint256 amount = _poolData.feeData.totalRetainedAmount;
    _poolData.feeData.totalRetainedAmount = 0;
+   _poolData.depositData.totalAmount -= amount;
    emit ClearTokenFees(amount);
    return amount;
}
```

## Proof of concept

### proof of concept :

#### Example Scenario :

1. **Initial state:**

Assume we have a pool (non-bridged token pool) with the following state:

* `pool.depositData.totalAmount` = 1000 USDC
* `totalBorrow` = 500 USDC (stable + variable)

2. **Repayment occurs:**

* Now `200 USDC` of borrowed funds are repaid by the user, and since the interest is paid before the principal, assume that (100 principal, 100 interest):
* `Interest paid` = 100 USDC
* `Retained amount (50% of interest)` = 50 USDC

The pool state will be:

* `pool.depositData.totalAmount` = 1100 USDC (we add interest repaid)
* `totalBorrow` = 400 USDC (stable + variable)
* `totalRetainedAmount` = 50 USDC

3. **Fee withdrawal:**

* `clearTokenFees()` is called, withdrawing 50 USDC
* `pool.depositData.totalAmount` remains at 1100 USDC (not decreased)
* Actual pool balance is now: `1100 - 400 - 50 = 650 USDC`

4. **New user attempt to borrow:**

* User tries to borrow 700 USDC and sends a transaction from the spoke chain to do so.
* System check for available liquidity will pass since: 1100 (`pool.depositData.totalAmount`) - 400 (`totalBorrow`) = 700 USDC available
* Borrow appears valid and is processed on the Hub chain, and a cross-chain message is sent back to the spoke chain to release the funds for the borrower.

5. **Spoke execution:**

* On the spoke chain, the message will be received, and the spokeToken contract will attempt to transfer the 700 USDC to the borrower.
* Since actual available funds are `650 USDC` (as explained above), the transaction will revert due to insufficient funds, and the borrower will not be able to receive any funds.

> Please note that This issue compounds with each fee clearance operation. Every time fees are cleared, the discrepancy between the recorded pool balance and the actual available funds increases. The pool.depositData.totalAmount remains unchanged while the actual balance decreases, widening the gap.
