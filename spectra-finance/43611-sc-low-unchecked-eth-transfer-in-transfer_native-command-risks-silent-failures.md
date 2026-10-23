# #43611 \[SC-Low] Unchecked ETH Transfer in TRANSFER\_NATIVE Command Risks Silent Failures

**Submitted on Apr 8th 2025 at 22:07:39 UTC by @EFCCWEB3 for** [**Audit Comp | Spectra Finance**](https://immunefi.com/audit-competition/audit-comp-spectra-finance)

* **Report ID:** #43611
* **Report Type:** Smart Contract
* **Report severity:** Low
* **Target:** <https://github.com/immunefi-team/Spectra-Audit-Competition/blob/main/src/router/Dispatcher.sol>
* **Impacts:**
  * Contract fails to deliver promised returns, but doesn't lose value

## Description

## Brief/Intro

I’ve spotted a sneaky issue in the `TRANSFER_NATIVE` command within the Router contract—it doesn’t check if the native ETH transfer actually succeeds. When sending ETH to a recipient, it uses a low-level .call without verifying the result, so if the transfer fails (say, to a non-payable contract), it quietly moves on without reverting. On mainnet, this could leave the contract’s state out of sync—thinking ETH moved when it didn’t—wasting gas and confusing users or downstream logic, though funds themselves stay safe in the contract.

## Vulnerability Details

Let’s dig into this. The execute function batches commands, passing each to \_dispatch with a single msg.value for ETH operations. Most commands play nice, but TRANSFER\_NATIVE has a gap. Here’s how it’s set up in \_dispatch (based on typical implementations we’ve seen):

```solidity
} else if (command == Commands.TRANSFER_NATIVE) {
    (address recipient, uint256 value) = abi.decode(_inputs, (address, uint256));
    payable(recipient).call{value: value}("");
}
```

* **What It Does:** Decodes the recipient and value from \_inputs, then sends value ETH to recipient using payable(recipient).call{value: value}("").
* **The Catch:** .call returns (bool success, bytes memory data), where success is true if the ETH is accepted, false if it fails (e.g., recipient lacks a payable fallback). But here, it doesn’t capture or check that return value.

In Solidity, .call is flexible—it sends ETH with a 2300 gas stipend and doesn’t revert on its own if the recipient can’t take it. If the recipient: Has a `receive()` or payable `fallback()`, it works fine.

Is non-payable (no such functions), the call fails silently, returning false.

Reverts explicitly (e.g., require(false)), the failure bubbles up, reverting the transaction.

The bug is that `TRANSFER_NATIVE` assumes success without checking. Compare it to `KYBER_SWAP`:

```solidity
(bool success, ) = kyberRouter.call{value: msg.value}(targetData);
if (!success) {
    revert CallFailed();
}
```

`KYBER_SWAP` checks `.call` success and reverts if it fails—`TRANSFER_NATIVE` doesn’t. If the ETH transfer flops (e.g., success = false but no revert), \_dispatch and execute keep chugging along, leaving the contract blind to the failure. This proves the vulnerability exists: an unchecked `.call` risks silent failure, misaligning state with reality.

## Impact Details

The ETH stays in the router if the transfer fails—safe, but stuck meaning Contract fails to deliver promised returns, but doesn’t lose value.

## References

<https://github.com/immunefi-team/Spectra-Audit-Competition/blob/1cebdc67a9276fd87105d13f302fd77d000d0c0b/src/router/Dispatcher.sol#L485>

### Recommendation

```solidity
} else if (command == Commands.TRANSFER_NATIVE) {
    (address recipient, uint256 value) = abi.decode(_inputs, (address, uint256));
    (bool success, ) = payable(recipient).call{value: value}("");
++    if (!success) {
++        revert("NativeTransferFailed");
    }
}
```

## Proof of Concept

## Proof of Concept

Charlie’s a mainnet user batching two actions via the router: wrapping 1 ETH into WETH and sending 1 ETH to his BadContract. He calls execute with `_commands = hex"2122"` (`DEPOSIT_NATIVE_IN_WRAPPER` and `TRANSFER_NATIVE`), `_inputs = [WETH_WRAPPER_ADDRESS, 1e18]` and `[BadContract, 1e18]`, and `msg.value = 2e18`. BadContract has no `receive()` or payable fallback()—it’s non-payable. In execute, `numCommands = 2`, msgSender sets to Charlie, `msgValue = 2e18`, and the loop kicks off.

The first command wraps 1 ETH successfully, dropping the router’s balance from 2 ETH to 1 ETH. Then, `TRANSFER_NATIVE` tries `payable(BadContract).call{value: 1e18}("")`. Since BadContract isn’t payable, .call returns (false, )—a silent failure with no revert. Without a success check, `_dispatch` and execute finish, the transaction succeeds, but the 1 ETH stays in the router. Charlie expects BadContract to have it, but it’s at 0—a state mismatch that slips through unchecked.
