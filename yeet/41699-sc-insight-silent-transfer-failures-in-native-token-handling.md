# #41699 \[SC-Insight] Silent Transfer Failures in Native Token Handling

**Submitted on Mar 17th 2025 at 16:38:27 UTC by @ZeroXGondar for** [**Audit Comp | Yeet**](https://immunefi.com/audit-competition/audit-comp-yeet)

* **Report ID:** #41699
* **Report Type:** Smart Contract
* **Report severity:** Insight
* **Target:** <https://github.com/immunefi-team/audit-comp-yeet/blob/main/src/contracts/Zapper.sol>
* **Impacts:**

## Description

## Summary

In `Zapper::_sendNativeToken` when transferring native tokens, the contract uses unsafe `transfer` method instead of the recommended low level `call` method.

This is problematic since `transfer` method only passes 2300 gas, which can cause the transaction to be reverted if the interacting contract uses more gas in its receive/fallback function.

## Details

The `_sendNativeToken()` function uses the deprecated `transfer()` method to send native BERA tokens. This approach has multiple security flaws:

* No verification of transfer success
* Silent failures possible when transfers fail
* Limited to 2300 gas, insufficient for contracts with complex receive/fallback functions

Note that this function can be accessed in many flows, making the issue more serious. It can be accessed by directly calling `Zapper::zapOutNative` function, or, in more complex scenarios, it can be accessed via `StakeV2::claimRewardsInNative` function which in turn calls `Zapper::zapOutNative` that calls `_sendNativeToken`

## Impact

* Users may believe their funds were successfully transferred when they weren't
* Protocol accounting may become inconsistent with actual token balances
* Lack of error handling prevents recovery mechanisms from being triggered
* Potential permanent loss of funds due to silent failures

## Faulty block

```solidity
function _sendNativeToken(address receiver, uint256 amount) internal {
    if (amount > 0) {
        wbera.withdraw(amount);
        payable(receiver).transfer(amount);
    }
}

```

## Recommended Fix

Replace `transfer()` with `call()` and add success verification:

```solidity
function _sendNativeToken(address receiver, uint256 amount) internal {
    if (amount > 0) {
        wbera.withdraw(amount);
        (bool success, ) = payable(receiver).call{value: amount}("");
        require(success, "Native token transfer failed");
    }
}

```

## Proof of Concept

## POC

1. A contract with a gas-intensive receive function (consuming >2300 gas) is deployed on the network.
2. The vulnerable flow is triggered in one of two ways:
   * Direct call to `Zapper::zapOutNative` with the gas-intensive contract as the receiver
   * Indirect call through `StakeV2::claimRewardsInNative` which ultimately calls `_sendNativeToken`
3. When `_sendNativeToken` executes:
   * It successfully withdraws tokens from wbera with `wbera.withdraw(amount)`
   * It attempts to transfer native tokens using `payable(receiver).transfer(amount)`
   * The transfer silently fails because the receiver's receive function exceeds the 2300 gas limit
   * No error is thrown or checked since the contract doesn't verify the transfer's success
4. The transaction completes without reverting, making it appear successful to users and the protocol.
