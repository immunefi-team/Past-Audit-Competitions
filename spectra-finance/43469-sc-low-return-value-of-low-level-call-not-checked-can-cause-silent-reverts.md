# #43469 \[SC-Low] Return value of low level call not checked can cause silent Reverts

**Submitted on Apr 6th 2025 at 21:06:47 UTC by @ZestfulHedgehog609 for** [**Audit Comp | Spectra Finance**](https://immunefi.com/audit-competition/audit-comp-spectra-finance)

* **Report ID:** #43469
* **Report Type:** Smart Contract
* **Report severity:** Low
* **Target:** <https://github.com/immunefi-team/Spectra-Audit-Competition/blob/main/src/router/Dispatcher.sol>
* **Impacts:**
  * Contract fails to deliver promised returns, but doesn't lose value

## Description

## Brief/Intro

In the `Dispatcher.sol` contract ,Return value of low level call not checked

## Vulnerability Details

In the `Dispatcher.sol` contract , in the `_dispatch(bytes1 _commandType, bytes calldata _inputs)` function , if the `command` is `Commands.TRANSFER_NATIVE`, the function does a low level call to the recipient address.The problem is however, after the call is done , its returned boolean value is not checked to see if the transfer went seccessfully or not.If the transfer was not successful, this will cause a silent revert and the transaction is marked a successful even though the recipient did not receive any ether.

## Impact Details

If the call fails (e.g., recipient is a contract that reverts or runs out of gas) or (if it is a smart contract wallet that doesn't accept native ether) ,success will be false, but the transaction continues without reverting. This can lead to:

1. Lost ETH: The transfer fails, but the rest of the function executes.
2. Inconsistent state: The contract assumes the transfer succeeded when it didn’t.

## References

<https://github.com/immunefi-team/Spectra-Audit-Competition/blob/1cebdc67a9276fd87105d13f302fd77d000d0c0b/src/router/Dispatcher.sol#L485>

## Proof of Concept

## Proof of Concept

because the competition is not requiring to submit a mandatory Poc rule, iam not submitting a runnable Poc but if the judges require a Poc after validation ,I can definitely submit it.
