# #43912 \[SC-Low] Lack of ETH Success Transfer Checks in Dispatcher.sol

**Submitted on Apr 14th 2025 at 07:17:05 UTC by @Allen\_George08 for** [**Audit Comp | Spectra Finance**](https://immunefi.com/audit-competition/audit-comp-spectra-finance)

* **Report ID:** #43912
* **Report Type:** Smart Contract
* **Report severity:** Low
* **Target:** <https://github.com/immunefi-team/Spectra-Audit-Competition/blob/main/src/router/Dispatcher.sol>
* **Impacts:**
  * Contract fails to deliver promised returns, but doesn't lose value

## Description

## Brief/Intro

The `Dispatcher.sol` has a command `TRANSFER_NATIVE` which is transferring native eth using a low-level call which returns a value. After the call the contract doesn't have a succcess check which will lead to un-noticed failed transfer.

## Vulnerability Details

Here's a snippet from the `Dispatcher.sol` contract.

[Dispatcher.sol](https://github.com/immunefi-team/Spectra-Audit-Competition/blob/1cebdc67a9276fd87105d13f302fd77d000d0c0b/src/router/Dispatcher.sol#L483)

```solidity
        else if (command == Commands.TRANSFER_NATIVE) {
            (address recipient, uint256 amount) = abi.decode(_inputs, (address, uint256));

            (bool success,) = payable(recipient).call{value: amount}("");
            //@audit lacks success checks
        } else {
            revert InvalidCommandType(command);
        }
```

This low-level call can fail silently for a number of reasons:

* The recipient contract reverts in its fallback/receive function.
* Insufficient gas provided for execution.
* Transfers to smart contracts that do not accept ETH.
* Transfer limits or blocked addresses enforced by recipient logic.

Because the return value (success) is ignored, such failures would go unnoticed and create inconsistencies in user expectations vs. actual contract state.

## Impact Details

* Silent reverting of calls.
* The vulnerability doesn't cause a damage to users or protocol but fails to deliver promised returns i.e It falls under the `Low` severity category of Immunefi.

## Proof of Concept

## Proof of Concept

1. A transfer native eth via `Router::execute()` is executed and sends commands and funds in.
2. The recipient is a contract that rejects ETH transfers (e.g., has no receive() or fallback() and doesn't implement payable).
3. The call fails internally, but since success isn’t checked, the router logic continues as if the transfer succeeded.
4. The recipient receives nothing, yet the system may assume payment has occurred.
