# 56561 sc insight fee amount is recomputed multiple times when the initial value has already been cached

**Submitted on Oct 17th 2025 at 16:49:54 UTC by @Josh4324 for** [**Audit Comp | Alchemix V3**](https://immunefi.com/audit-competition/alchemix-v3-audit-competition)

* **Report ID:** #56561
* **Report Type:** Smart Contract
* **Report severity:** Insight
* **Target:** <https://github.com/alchemix-finance/v3-poc/blob/immunefi\\_audit/src/AlchemistV3.sol>
* **Impacts:**

## Description

## Brief/Intro

The fee amount is being calculated in every place that it is used, despite it being cached earlier. This results in additional gas usage and is also prone to errors.

## Vulnerability Details

Check the `repay` function in the Alchemix V3.

1. The `feeAmount` is calculated and stored.
2. The `feeAmount` variable is used.
3. 4. and 5. The feeAmount is recomputed 3 times, thus wasting gas and increasing the error surface.

```solidity
1)->  uint256 feeAmount = creditToYield * protocolFee / BPS;
2)->        if (feeAmount > account.collateralBalance) {
            revert("Not enough collateral to pay for debt fee");
        } else {
3)->            account.collateralBalance -= creditToYield * protocolFee / BPS; 
        }

        _subDebt(recipientTokenId, credit);

        // Transfer the repaid tokens to the transmuter.
        TokenUtils.safeTransferFrom(myt, msg.sender, transmuter, creditToYield); 
4)->        TokenUtils.safeTransfer(myt, protocolFeeReceiver, creditToYield * protocolFee / BPS); 
5)->         _mytSharesDeposited -= creditToYield * protocolFee / BPS; 

        emit Repay(msg.sender, amount, recipientTokenId, creditToYield);

```

## Impact Details

1. More gas is consumed.
2. The error surface is increased, especially when updating the code.

## References

<https://github.com/alchemix-finance/v3-poc/blob/immunefi\\_audit/src/AlchemistV3.sol?utm\\_source=immunefi#L530C4-L541C66>

## Proof of Concept

## Proof of Concept

Please run this on Remix.

It has two functions `repeat` and `noRepeat`. The repeat function repeats the fee calculation three times, the same as the above function, while the no repeat uses the cached value instead.

**output** repeat cost: 9759 to execute. noRepeat cost: 8204 to execute

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;


contract Repeat {

    uint constant BPS = 10000;
    uint public protocolFee = 1000;
    uint _mytSharesDeposited = type(uint).max;
    
    function repeat() external  {
        uint collateralBalance = 1000 ether;
        uint creditToYield = 1000 ether;

        uint256 feeAmount = creditToYield * protocolFee / BPS;
        if (feeAmount > collateralBalance) {
            revert("Not enough collateral to pay for debt fee");
        } else {
            collateralBalance -= creditToYield * protocolFee / BPS; //@audit insight M feeAmount is computed multiple times
        }

        // Mock the transfer of tokens
        mock_transfer(creditToYield * protocolFee / BPS); //@audit M H-2 fee is paid from the Alchemist, not the user?
        _mytSharesDeposited -= creditToYield * protocolFee / BPS; //@audit M H-3 Credit without fee should be added here
        
    }


    function noRepeat() external  {
        uint collateralBalance = 1000 ether;
        uint creditToYield = 1000 ether;

        uint256 feeAmount = creditToYield * protocolFee / BPS;
        if (feeAmount > collateralBalance) {
            revert("Not enough collateral to pay for debt fee");
        } else {
            collateralBalance -= feeAmount; //@audit insight M feeAmount is computed multiple times
        }

        // Mock the transfer of tokens
        mock_transfer(feeAmount); //@audit M H-2 fee is paid from the Alchemist, not the user?
        _mytSharesDeposited -= feeAmount; //@audit M H-3 Credit without fee should be added here
        
    }



    function mock_transfer(uint amount) internal {

    }


}
```
