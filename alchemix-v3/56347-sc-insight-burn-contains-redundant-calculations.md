# 56347 sc insight burn contains redundant calculations

**Submitted on Oct 14th 2025 at 19:13:07 UTC by @magtentic for** [**Audit Comp | Alchemix V3**](https://immunefi.com/audit-competition/alchemix-v3-audit-competition)

* **Report ID:** #56347
* **Report Type:** Smart Contract
* **Report severity:** Insight
* **Target:** <https://github.com/alchemix-finance/v3-poc/blob/immunefi\\_audit/src/AlchemistV3.sol>
* **Impacts:**

## Description

## Brief/Intro

The burn function performs the same calculation three times sequentially.\
This can be optimized by storing the result in a local variable and reusing it.

## Vulnerability Details

In the `burn` function, there is the below calculations :

```
    function burn(uint256 amount, uint256 recipientId) external returns (uint256) {
        ...
        _accounts[recipientId].collateralBalance -= convertDebtTokensToYield(credit) * protocolFee / BPS;
        TokenUtils.safeTransfer(myt, protocolFeeReceiver, convertDebtTokensToYield(credit) * protocolFee / BPS);
        _mytSharesDeposited -= convertDebtTokensToYield(credit) * protocolFee / BPS;
        ...
    }
```

This can be refactored to :

```
    function burn(uint256 amount, uint256 recipientId) external returns (uint256) {
        ...
        uint256 creditAmount = convertDebtTokensToYield(credit) * protocolFee / BPS;
        _accounts[recipientId].collateralBalance -= creditAmount;
        TokenUtils.safeTransfer(myt, protocolFeeReceiver, creditAmount);
        _mytSharesDeposited -= creditAmount;
        ...
    }
```

Which makes it avoid duplicate calculation and makes it more readable and maintainable.

## Impact Details

This is a code optimization issue. Benefits include:

* Reduced gas usage
* Improved readability and maintainability
* Lower risk of inconsistencies if the calculation changes in the future

## References

AlchemistV3 - <https://github.com/alchemix-finance/v3-poc/blob/immunefi\\_audit/src/AlchemistV3.sol>

## Proof of Concept

Insight submission, no PoC required as this is a code optimization.
