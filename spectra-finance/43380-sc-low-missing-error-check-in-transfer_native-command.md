# #43380 \[SC-Low] Missing Error Check in TRANSFER\_NATIVE Command

**Submitted on Apr 5th 2025 at 06:56:40 UTC by @Paludo0x for** [**Audit Comp | Spectra Finance**](https://immunefi.com/audit-competition/audit-comp-spectra-finance)

* **Report ID:** #43380
* **Report Type:** Smart Contract
* **Report severity:** Low
* **Target:** <https://github.com/immunefi-team/Spectra-Audit-Competition/blob/main/src/router/Dispatcher.sol>
* **Impacts:**
  * Contract fails to deliver promised returns, but doesn't lose value

## Description

## Vulnerability Details

In the TRANSFER\_NATIVE command branch, the contract decodes the recipient and amount, then performs a native Ether transfer using the low-level call without checking the returned success flag. This can lead to silent failures in transferring funds.

The low-level call to transfer native Ether (call{value: amount}) does not verify the return value (success). In Solidity, the call function returns a boolean indicating whether the call succeeded. Not checking this value can allow a situation in which the call fails (for example, if the recipient is a contract that reverts on receiving Ether) while the execution of the function continues normally.

## Impact Details

If the transfer fails, the contract does not revert the transaction, leaving the system in an inconsistent state or causing funds not to be transferred as expected.

An attacker might deliberately force transfers to fail by using a contract with a fallback function that always reverts.

## Proof of Concept

The code snippet is as follows:

```
} else if (command == Commands.TRANSFER_NATIVE) {
    (address recipient, uint256 amount) = abi.decode(_inputs, (address, uint256));
    (bool success, ) = payable(recipient).call{value: amount}("");
}
```

To mitigate this issue, it is important to check the return value of the call and revert the transaction if the transfer fails. The code should be modified as follows:

```
} else if (command == Commands.TRANSFER_NATIVE) {
    (address recipient, uint256 amount) = abi.decode(_inputs, (address, uint256));
    (bool success, ) = payable(recipient).call{value: amount}("");
    require(success, "Native transfer failed");
}
```
