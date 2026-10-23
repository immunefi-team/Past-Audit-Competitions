# #37634 \[SC-Low] Incorrect Builtin ERC4626 Call Signature

**Submitted on Dec 11th 2024 at 08:45:58 UTC by @anatomist for** [**Attackathon | Ethereum Protocol**](https://immunefi.com/audit-competition/ethereum-protocol-attackathon)

* **Report ID:** #37634
* **Report Type:** Smart Contract
* **Report severity:** Low
* **Target:** <https://github.com/vyperlang/vyper>
* **Impacts:**
  * (Compiler) Unexpected behavior
  * (Compiler) Incorrect bytecode generation leading to incorrect behavior

## Description

## Brief/Intro

Vyper provides several common builtin interfaces to save developer the troubles of having to define those themselves. Unfortunately, the interface of ERC4626 may produce incorrect signatures, potentially leading to call failures or calling incorrect functions.

## Vulnerability Details

The ERC4626.vyi builtin interface provided by vyper contains several functions with optional arguments.

```
@external
def deposit(assets: uint256, receiver: address=msg.sender) -> uint256:
    ...

@external
def mint(shares: uint256, receiver: address=msg.sender) -> uint256:
    ...

@external
def withdraw(assets: uint256, receiver: address=msg.sender, owner: address=msg.sender) -> uint256:
    ...

@external
def redeem(shares: uint256, receiver: address=msg.sender, owner: address=msg.sender) -> uint256:
    ...
```

However, it is overlooked that for optional arguments in contract abi functions, the function signature is calculated WITHOUT the arguments using default values. This means if developers call the builtin interface function while using default values, they'll end up calling an incorrect function.

For example, the two calls in the contract below ends up resolving to different signatures, and only the first signature is correct.

```
from ethereum.ercs import IERC4626

@deploy
def __init__(x: IERC4626):
    extcall x.withdraw(0, self, self)   #0xb460af94 "withdraw(uint256,address,address)
    extcall x.withdraw(0)               #0x2e1a7d4d "withdraw(uint256)"
```

## Impact Details

In most cases, using an incorrect interface will likely result in a call failure, which prevents normal operation of the contract. In certain scenarios, the incorrect call might result in calling the fallback function or another function with matching interface. In such cases, unexpected actions may be performed on behalf of the contract, resulting in more severe consequences.

## References

* `https://github.com/vyperlang/vyper/blob/12ab4919cc4618fcac4f5d24d45a0e7fdbc4a48c/vyper/builtins/interfaces/IERC4626.vyi#L47`

## Proof of Concept

## Proof of Concept

Already shown in Vulnerability Details section.
