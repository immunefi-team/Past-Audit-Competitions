# #43408 \[SC-Low] Not checking call success in \`TRANSFER\_NATIVE\`

**Submitted on Apr 5th 2025 at 16:53:29 UTC by @PotEater for** [**Audit Comp | Spectra Finance**](https://immunefi.com/audit-competition/audit-comp-spectra-finance)

* **Report ID:** #43408
* **Report Type:** Smart Contract
* **Report severity:** Low
* **Target:** <https://github.com/immunefi-team/Spectra-Audit-Competition/blob/main/src/router/Dispatcher.sol>
* **Impacts:**

## Description

## Brief/Intro

In the function `_dispatch` you are making a call to another address with ether, but there is no check whether the call was successful or not. This could lead to user frustration and inefficiencies.

## Vulnerability Details

The call function is a low-level operation in Solidity that is used to send Ether to an address. It returns a boolean value, success, indicating whether the call was successful or not. However, in this case, the success value is not being checked or handled.

```solidity
(bool success, ) = payable(recipient).call{value: amount}("");
// no check if call succeeded or not.
```

## Impact Details

Without error handling for the call failure, users may assume their transaction was successful, leading to confusion and frustration when they don’t see the expected results. This could erode trust in the contract and result in users abandoning the platform.

This is a security best practice.

## References

<https://github.com/immunefi-team/Spectra-Audit-Competition/blob/1cebdc67a9276fd87105d13f302fd77d000d0c0b/src/router/Dispatcher.sol#L485>

## Proof of Concept

## Proof of Concept

```solidity
 else if (command == Commands.TRANSFER_NATIVE) {
            (address recipient, uint256 amount) = abi.decode(_inputs, (address, uint256));
            (bool success, ) = payable(recipient).call{value: amount}(""); 
// forgot to check if the call was successful or not
```
