# #43729 \[SC-Low] Silent execution failure on \`Dispatcher::\_dispatch\` due to unchecked return value on \`Dispatcher:TRANSFER\_NATIVE\` operation

**Submitted on Apr 10th 2025 at 13:58:25 UTC by @blackgrease for** [**Audit Comp | Spectra Finance**](https://immunefi.com/audit-competition/audit-comp-spectra-finance)

* **Report ID:** #43729
* **Report Type:** Smart Contract
* **Report severity:** Low
* **Target:** <https://github.com/immunefi-team/Spectra-Audit-Competition/blob/main/src/router/Dispatcher.sol>
* **Impacts:**
  * Contract fails to deliver promised returns, but doesn't lose value

## Description

## Introduction

Within the `Dispatcher::_dispatch`, any `TRANSFER_NATIVE` commands that fail, do so silently disrupting functionality due to an unchecked return value.

## Description

The `Dispatcher.sol` is an abstract contract that is inherited by the `Router.sol`. The `Dispatcher.sol` has multiple functions but the most notable for this report is the `_dispatch()`. This function is called by `Router::execute` which "executes encoded commands along with provided inputs" and is the entry point into Dispatcher.sol functions.

There are many operations that the `Dispatcher::_dispatch` function can chose to execute using `if` statements. The most interesting operations for this report are `TRANSFER, TRANSFER_FROM, TRANSFER_NATIVE.`

* TRANSFER\_FROM: Transfers tokens from `msg.sender` to the Router.
* TRANSFER: Transfers tokens from the Router to a recipient.
* TRANSFER\_NATIVE: Transfers native token to a recipient.

> The Code Issue

The `TRANSFER` and `TRANSFER_FROM` - all throw revert when the function fails due to different reasons; low allowance, insufficient balance etc which alerts the user that something went wrong.

However, when the `TRANSFER_NATIVE` commands is used, if it fails for any reason - insufficient balance, recipient rejects etc - the operation fails silently and the user is unaware. The problematic code is in `Dispatcher::_dispatch#L483`

```solidity

 else if (command == Commands.TRANSFER_NATIVE) {
            (address recipient, uint256 amount) = abi.decode(_inputs, (address, uint256));
            (bool success, ) = payable(recipient).call{value: amount}("");
        } 
```

The code contains a `bool success` but this is not checked resulting in any failures to occur silently without informing the contract/user.

In comparison, operations such as `KYBER_SWAP` define a `bool success` which is correctly checked and reverts on failure correctly informing the contract/user that the transaction failed to successfully execute.

> Additional Info

The `Router::execute` itself does not return any value after execution. It leaves the logic of checking success to the functions implemented in `Dispatcher`. Therefore, this creates room for potential errors going unchecked if its forgotten in the `Dispatcher` contract.

## Impact

As the code does not cause user funds to be lost and the protocol still holds value, the severity is low as the contract fails to deliver promised returns. The contracts promises to transfer tokens but when it does not, it does not alert the user. As no value is lost, but the expected outcomes are not guaranteed the severity is classified as low.

This may lead to

1. Users being unaware transfers have failed.
2. User frustration and inconvenience due to hard to difficulty in debug the issue.
3. Extra work for users as they would have to implement their own logic into their infrastructure; this can potentially imply loss of trust in the Spectras Protocols capabilities to successfully deliver.

## Mitigation

The mitigation to this is to simply check that the function executed successfully; using either a `require` or `if` statement

```diff
//Within Error Declarations
    error KyberRouterNotSet();
+    error NativeTransferFailed();

//Implementation
 else if (command == Commands.TRANSFER_NATIVE) {
            (address recipient, uint256 amount) = abi.decode(_inputs, (address, uint256));
            (bool success, ) = payable(recipient).call{value: amount}("");
+           if(!success){
+				revert NativeTransferFailed();
				}
        } 

```

Another recommendation as a safety net, is to have the `Router::execute` implement a return bool value which the caller can also check.

## Link to Proof of Concept

<https://gist.github.com/blackgrease/f58dcf54cfcd58efc040c7c5e6f8a200>

## Proof of Concept

## Proof-of-Concept

To prove the above scenario, I have created a test suite that carries out 3 tests

1. `TRANSFER_NATIVE` to recipient fails silently (due to insufficient balance)

* Run with `forge test --mt testRouterSilentlyFailsOnNativeTransferToUser -vvv`
  * User transfers ETH into Router
  * User calls `TRANSFER_NATIVE` to send more Ether than is in the Router to recipient but `_dispatch` fails silently due to insufficient balance.

2. `TRANSFER_NATIVE` to contract fails silently (due to no `receive()` on recipient)

* Run with `forge test --mt testRouterSilentlyFailsOnNativeTransferToContract -vvv`
  * User transfers ETH into Router
  * User calls `TRANSFER_NATIVE` to send correct Ether than is in the Router to contract but transaction fails silently due to contract not having `receive()` balance.

3. Comparison of how TRANSFER\_FROM fails loudly.

* Run with `forge test --mt testTransferFromFailsLoudly -vvv`
  * User transfer ERC20 token into Router
  * User calls `TRANSFER` to send ERC20 token to recipient but function fails loudly due to insufficient balance.

The test suite is available in the private gist link: "<https://gist.github.com/blackgrease/f58dcf54cfcd58efc040c7c5e6f8a200>"
