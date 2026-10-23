# 29170 - \[SC - Medium] DoS by front-runnable externall call

Submitted on Mar 9th 2024 at 12:31:48 UTC by @Lastc0de for [Boost | ZeroLend](https://immunefi.com/bounty/zerolend-boost/)

Report ID: #29170

Report type: Smart Contract

Report severity: Medium

Target: <https://explorer.zksync.io/address/0x4d9429246EA989C9CeE203B43F6d1C7D83e3B8F8>

Impacts:

* Griefing (e.g. no profit motive for an attacker, but damage to the users or the protocol)

## Description

## Brief/Intro

EIP-2612 defined the highly popular ERC20 extension, Permit. It transfers the burden of holding native (gas) tokens away from users, by allowing them to sign an approval off-chain and send it to a trusted service, which could use the funds as if the user called approve(). It operates by a crypto signature of the following fields:

```
`owner` (user)

`spender` (trusted address)

`value` (amount)

`nonce` (an int that increases after every permit executes)

`deadline`
```

When permit() executes, the key things it checks are:

```
The sig is valid (compares it to a sig constructed using parameters + user's nonce)

Deadline hasn't passed.
```

Note that by design, the token ignores the msg.sender of the permit() call. Combined with the fact TXs can be observed in the mempool (by anyone, or at least by the sequencer in some L2s), it means that a permit() can be easily frontran (simply duplicate the TX arguments)

This will be problematic when `permit()` function is part of the body of an important function such as:

* deposit
* withdraw
* borrow
* repay borrow

Like yours `Pool.sol` contract functions :

* supplyWithPermit() - L178
* repayWithPermit() - L288

## Vulnerability Details

Vulnerable contract is `Pool.sol` : <https://explorer.zksync.io/address/0x4d9429246EA989C9CeE203B43F6d1C7D83e3B8F8?utm\\_source=immunefi>

<https://pacific-explorer.manta.network/address/0x2f9bB73a8e98793e26Cb2F6C4ad037BDf1C6B269?utm\\_source=immunefi>

Vulnerable functions is `supplyWithPermit()` & `repayWithPermit()`:

* supplyWithPermit() - L178 <https://github.com/zerolend/core-contracts/blob/2d518faa63833595979adb1786a63575a94264d4/contracts/protocol/pool/Pool.sol#L163C2-L194C1>
* repayWithPermit() - L288 <https://github.com/zerolend/core-contracts/blob/2d518faa63833595979adb1786a63575a94264d4/contracts/protocol/pool/Pool.sol#L271C3-L303C1>

NOTE: The type of vulnerability of both functions are similar to each other. In order not to confuse the analysis, I will only examine one function.

* Deep Dive to `supplyWithPermit()`:

```
  function supplyWithPermit(
    address asset,
    uint256 amount,
    address onBehalfOf,
    uint16 referralCode,
    uint256 deadline,
    uint8 permitV,
    bytes32 permitR,
    bytes32 permitS
  ) public virtual override {
    IERC20WithPermit(asset).permit(
      msg.sender,
      address(this),
      amount,
      deadline,
      permitV,
      permitR,
      permitS
    );
    SupplyLogic.executeSupply(
      _reserves,
      _reservesList,
      _usersConfig[onBehalfOf],
      DataTypes.ExecuteSupplyParams({
        asset: asset,
        amount: amount,
        onBehalfOf: onBehalfOf,
        referralCode: referralCode
      })
    );
  }
```

This function `supplyWithPermit()` in a `Pool.sol` contract on callable by user for supply collateralls. But what if an attacker extracts the `permitV - permitR - permitS` parameters from the `supplyWithPermit()` call and frontruns it with a direct permit() in `asset` token? In this case, the end result is `harmful`, since the user loses the functionality that follows the `permit()`.

In fact, any function call that unconditionally performs `permit()` can be forced to revert this way. In case there is a fallback code path (using direct user approval), the `DOS` is short-term, as eventually the user / dApp would switch to using an alternative. Otherwise, the `DOS` is long-term.

## Impact Details

Griefing is when the attacker calls certain functions of the smart contract that would put it in a suboptimal state, thus blocking normal function execution for any user. This would cause the user to lose money for sending the transaction, but when the smart contract is back to normal, the user would be able to call the function once again to complete it. In this instance, the attacker damaged the user by requiring them to send another transaction. The attacker does not profit, but they do damage the users or the protocol.

## References

<https://eips.ethereum.org/EIPS/eip-2612>

## Proof of Concept

```
// SPDX-License-Identifier: MIT
pragma solidity  ^0.8.13;

interface IERC20 {
    function permit(
        address owner,
        address spender,
        uint256 value,
        uint256 deadline,
        uint8 v,
        bytes32 r,
        bytes32 s
    ) external;
}


contract Griefing {
    address ZeroLendPoolZK = 0x4d9429246EA989C9CeE203B43F6d1C7D83e3B8F8;
    
    /* NOTE :  It should include the informations extracted by frontrunner from user transaction */
    address owner = address(user);
    address _asset = ;
    uint256 _amount = ;
    address _onBehalfOf ; 
    uint16 _referralCode = ;
    uint256 _deadline = ;
    uint8 _permitV = ;
    bytes32 _permitR = ;
    bytes32 _permitS = ;

    function TestGriefing() public {
        IERC20(_asset).permit(
            owner,
            ZeroLendPoolZK,
            _amount,
            _deadline,
            _permitV,
            _permitR,
            _permitS
        );
    }
}
```
