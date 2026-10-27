# #43987 \[SC-Low] Unchecked low-level ETH transfer in \`Dispatcher.sol\` may lead to undetected failures

**Submitted on Apr 15th 2025 at 11:22:17 UTC by @DSbeX for** [**Audit Comp | Spectra Finance**](https://immunefi.com/audit-competition/audit-comp-spectra-finance)

* **Report ID:** #43987
* **Report Type:** Smart Contract
* **Report severity:** Low
* **Target:** <https://github.com/immunefi-team/Spectra-Audit-Competition/blob/main/src/router/Dispatcher.sol>
* **Impacts:**

## Description

## Brief/Intro

The `TRANSFER_NATIVE` command in the `Dispatcher.sol` allows users to forward ETH to any arbitrary address via a low-level .call{value: amount}(""). However, if the recipient address is a contract that does not support receiving ETH(or any other reason of which the transaction will revert), the call fails silently and user funds are locked in the contract. There is no validation of the recipient beforehand, nor is the failure communicated to the caller.

## Vulnerability Details

In the `_dispatch()` function of the `Dispatcher.sol` , the following command is handled:

```javascript
} else if (command == Commands.TRANSFER_NATIVE) {
    (address recipient, uint256 amount) = abi.decode(_inputs, (address, uint256));
    (bool success, ) = payable(recipient).call{value: amount}("");
    // success is not checked
}
```

The failure of .call is not tracked or surfaced in any way. From the user’s perspective, ETH is sent into the Router.sol and expected to be delivered — but no indication is given if the transfer fails.

While this is considered a user input error, the contract does not enforce or warn about this scenario(so the user expects it to revert if the transaction doesn't execute) — leading to silent loss of funds, even if not truly frozen or maliciously mishandled.

## Impact Details

This applies to any user sending ETH through a TRANSFER\_NATIVE command to a recipient that rejects ETH or causes the .call to fail for some reason:

Unexpected behavior: ETH silently fails to deliver, but execution continues.

Poor user experience: users are not alerted and all their funds that were meant for this transaction are locked

No mechanism is provided for recovery or refund.

## References

`_dispatch()` in `Dispatcher.sol`:

<https://github.com/immunefi-team/Spectra-Audit-Competition/blob/1cebdc67a9276fd87105d13f302fd77d000d0c0b/src/router/Dispatcher.sol#L485>

## Proof of Concept

## Proof of Concept

1. User sends ETH via TRANSFER\_NATIVE to a non-payable contract.
2. ETH is accepted into the Router via msg.value.
3. .call to the recipient fails silently. (because of the non-payable contract)
4. Execution continues, user is unaware of failure.
5. ETH remains in the Router with no recovery method.
