# 56346 sc insight redundant calculation of feeamount in repay function

**Submitted on Oct 14th 2025 at 19:04:52 UTC by @magtentic for** [**Audit Comp | Alchemix V3**](https://immunefi.com/audit-competition/alchemix-v3-audit-competition)

* **Report ID:** #56346
* **Report Type:** Smart Contract
* **Report severity:** Insight
* **Target:** <https://github.com/alchemix-finance/v3-poc/blob/immunefi\\_audit/src/AlchemistV3.sol>
* **Impacts:**

## Description

## Brief/Intro

In the `repay` function of the contract, the protocol fee is calculated and stored in the `feeAmount` variable.\
However, the same calculation is repeated multiple times later in the function instead of reusing the variable.

## Vulnerability Details

Within the `repay` function, there are multiple places where the same calculation is been done creating redundant code where as the calculation has already been done in `feeAmount`, the values used for the calculation do not change so it is safe to use the variable `feeAmount`.

```
    function repay(uint256 amount, uint256 recipientTokenId) public returns (uint256) {
        ...
        // Debt is subject to protocol fee similar to redemptions
        uint256 feeAmount = creditToYield * protocolFee / BPS;
        ...
        } else {
            account.collateralBalance -= creditToYield * protocolFee / BPS;
        }
        ...
        TokenUtils.safeTransfer(myt, protocolFeeReceiver, creditToYield * protocolFee / BPS);
        _mytSharesDeposited -= creditToYield * protocolFee / BPS;
        ...
    } 
```

`creditToYield * protocolFee / BPS` can be replaced by `feeAmount`

## Impact Details

This is primarily a code optimization issue.\
Reusing feeAmount improves:

* Gas efficiency
* Readability and maintainability
* Reduces the risk of inconsistencies if the formula changes

## References

Alchemist V3 contract - <https://github.com/alchemix-finance/v3-poc/blob/immunefi\\_audit/src/AlchemistV3.sol>

## Proof of Concept

Insight showing code optimization within the `repay` function.
