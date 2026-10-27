# #43803 \[SC-Low] Boolean success returned from address.call{value: amount}() not checked

**Submitted on Apr 11th 2025 at 17:43:30 UTC by @kaysoft for** [**Audit Comp | Spectra Finance**](https://immunefi.com/audit-competition/audit-comp-spectra-finance)

* **Report ID:** #43803
* **Report Type:** Smart Contract
* **Report severity:** Low
* **Target:** <https://github.com/immunefi-team/Spectra-Audit-Competition/blob/main/src/router/Dispatcher.sol>
* **Impacts:**
  * Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield

## Description

## Brief/Intro

The `_dispatch(...)` function in Dispatch.sol use a low level call to transfer ETH for the `Commands.TRANSFER_NATIVE` command but does not check the returned boolean value.

When Eth transfer through the low level call fails, the ETH is lost to the Router.sol contract. This can cause malicious users to setup bots to monitor the balance of the Router.sol contract for this kind of bug and immediately steal the ETH with the same execute function.

The following reasons can cause low level call to revert:

1. When the recipient is a contract but without receive() or fallback() function.
2. The recipient explicitly reverts the transaction maybe due to some conditions(require)

## Vulnerability Details

The Router.sol inherits the Dispatch.sol contract.

The execute(...) function in the Router.sol can be used to transfer ETH to a `recipient` address. This is done by the internal \_dispatch() function.

However the issue is that after the low level `call{value: amount}` to transfer ETH, the returned boolean `success` is not checked to ensure it is `true` and if not revert.

```solidity
File: Dispatch.sol
function _dispatch(bytes1 _commandType, bytes calldata _inputs) internal {
...
} else if (command == Commands.TRANSFER_NATIVE) {
            (address recipient, uint256 amount) = abi.decode(_inputs, (address, uint256));
@>            (bool success, ) = payable(recipient).call{value: amount}("");
        } else {
            revert InvalidCommandType(command);
        }
    }
```

When sending ETH to the `recipient` address fails, the `bool success` returned is `false`. In a situation where the ETH transfer fails, the ETH will be lost to the Router.sol contract.

## Impact Details

Loss of ETher to the Router.sol contract which can be stolen by snipper bots that monitor the balance of the Router.sol contract.

## Recommendation

Consider checking the returned bool `success` this way:

```
File: Dispatch.sol
function _dispatch(bytes1 _commandType, bytes calldata _inputs) internal {
...
} else if (command == Commands.TRANSFER_NATIVE) {
            (address recipient, uint256 amount) = abi.decode(_inputs, (address, uint256));
            (bool success, ) = payable(recipient).call{value: amount}("");
++        require(success, "ETH transfer failed");
        } else {
            revert InvalidCommandType(command);
        }
    }
```

## Proof of Concept

## Proof of Concept

1. Alice calls the execute(...) function of Router.sol with 5 ETH and `receipient` which is a smart contract without `receive()` function. The command is `Commands.TRANSFER_NATIVE`
2. The 5 ETH is lost to the Router.sol contract
3. Bob already setup bots to monitor Router.sol balance
4. Immediately the balance of Router.sol is non zero Bob's bot steals the 5 ETH.
