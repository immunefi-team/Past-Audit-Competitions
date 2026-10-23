# Boost \_ Folks Finance 34052 - \[Smart Contract - Low] withdraw doesnt round in favour of protocol for

Submitted on Sun Aug 04 2024 18:47:57 GMT-0400 (Atlantic Standard Time) by @A2Security for [Boost | Folks Finance](https://immunefi.com/bounty/folksfinance-boost/)

Report ID: #34052

Report type: Smart Contract

Report severity: Low

Target: <https://testnet.snowtrace.io/address/0x2cAa1315bd676FbecABFC3195000c642f503f1C9>

Impacts:

* Contract fails to deliver promised returns, but doesn't lose value

## Description

## **Description:**

The `updateWithWithdraw` function in the smart contract is rounding down the `fAmount` instead of rounding up when the withdrawal action is processed. When calculating the amount of ftoken to reduce the user collateral by the protocol should round up to get as much ftoken as possible and don't take the precision loss

## **Impact:**

When reducing balances using shares, the protocol should round in its favour, to prevent losing value due to precision loss. This could be critical in the case that tokens with really low decimals are used, but for most cases this will lead to the loss in dust amounts, and this is why we set the severity as low

## **Recommended Mitigation Steps:**

Update the `updateWithWithdraw` function to ensure that `withdrawPoolParams.fAmount` rounds up instead of rounding down when calculating the fAmount for non-fAmount withdrawals.

We would recommend simply adding a function in mathutils to round up

```solidity
    function toFAmountUp(uint256 underlyingAmount, uint256 depositInterestIndexAtT) internal pure returns (uint256) {
        return underlyingAmount.mulDiv(ONE_18_DP, depositInterestIndexAtT,Math.Rounding.Ceil);
    }
```

and simply round up when converting from amount to Famount

```diff
    function updateWithWithdraw(HubPoolState.PoolData storage pool, uint256 amount, bool isFAmount) external returns (DataTypes.WithdrawPoolParams memory withdrawPoolParams) {
        // can withdraw even if pool is depreciated
        // update interest indexes before the interest rates change
        pool.updateInterestIndexes();

        if (isFAmount) {
            withdrawPoolParams.fAmount = amount;
            withdrawPoolParams.underlingAmount = amount.toUnderlingAmount(pool.depositData.interestIndex);
        } else {
            withdrawPoolParams.underlingAmount = amount;
--            withdrawPoolParams.fAmount = amount.toFAmount(pool.depositData.interestIndex);
++            withdrawPoolParams.fAmount = amount.toFAmountUp(pool.depositData.interestIndex);
        }

        pool.depositData.totalAmount -= withdrawPoolParams.underlingAmount;
        pool.updateInterestRates();
    }
```

## Proof of concept

## **Proof of Concept:**

The relevant code snippet from the function highlights the rounding issue where it currently rounds down: `contracts/hub/logic/HubPoolLogic.sol`

```solidity
    function updateWithWithdraw(HubPoolState.PoolData storage pool, uint256 amount, bool isFAmount) external returns (DataTypes.WithdrawPoolParams memory withdrawPoolParams) {
        // can withdraw even if pool is depreciated
        // update interest indexes before the interest rates change
        pool.updateInterestIndexes();

        if (isFAmount) {
            withdrawPoolParams.fAmount = amount;
            withdrawPoolParams.underlingAmount = amount.toUnderlingAmount(pool.depositData.interestIndex);
        } else {
            withdrawPoolParams.underlingAmount = amount;
            // @audit-issue this should round up, but it rounds down 
>>>            withdrawPoolParams.fAmount = amount.toFAmount(pool.depositData.interestIndex);
        }

        pool.depositData.totalAmount -= withdrawPoolParams.underlingAmount;
        pool.updateInterestRates();
    }
```

This is how the `toFamount()` is implemented in `contracts/hub/libraries/MathUtils.sol`:

```solidity
    function toFAmount(uint256 underlyingAmount, uint256 depositInterestIndexAtT) internal pure returns (uint256) {
>>        return underlyingAmount.mulDiv(ONE_18_DP, depositInterestIndexAtT);
    }
```

As we can see `toFAmount()` rounds down by calling `.mulDiv()`, when calculating the amount of ftokens to reduce the user loan balance by, the protocol should always round in its favour. The famount will be deducted from the userLoan.collateral.balance and it should round up
