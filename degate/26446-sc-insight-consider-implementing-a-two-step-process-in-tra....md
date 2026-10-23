# 26446 - \[SC - Insight] Consider implementing a two step process in tra...

Submitted on Dec 3rd 2023 at 04:50:11 UTC by @Bauchibred for [Boost | DeGate](https://immunefi.com/bounty/boosteddegatebugbounty/)

Report ID: #26446

Report type: Smart Contract

Report severity: Insight

Target: <https://etherscan.io/address/0x9C07A72177c5A05410cA338823e790876E79D73B#code>

Impacts:

* *Potentially* losing all access to the functionalities meant to be accessed by the owner.

## Description

## Proof of Concept

Take a look at [transferProxyOwnership()](https://www.contractreader.io/contract/mainnet/0x9C07A72177c5A05410cA338823e790876E79D73B)

```solidity
/**
  /**
   * @dev Allows the current owner to transfer control of the contract to a newOwner.
   * @param newOwner The address to transfer ownership to.
   */
  function transferProxyOwnership(address newOwner) public onlyProxyOwner {
    require(newOwner != address(0));
    emit ProxyOwnershipTransferred(proxyOwner(), newOwner);
    setUpgradabilityOwner(newOwner);
  }

   * @dev Sets the address of the owner
   */
  function setUpgradabilityOwner(address newProxyOwner) internal {
    bytes32 position = proxyOwnerPosition;
    assembly {
      sstore(position, newProxyOwner)
    }
  }

```

It's possible that the `onlyProxyOwner` role mistakenly transfers ownership to the wrong address, resulting in a loss of the `onlyProxyOwner` role. The current ownership transfer process involves the current owner calling `transferProxyOwnership()`. This function checks the new owner is not the zero address and proceeds to write the new owner's address into the owner's state variable via `setUpgradabilityOwner()`. If the nominated EOA account is not a valid account, it is entirely possible the owner may accidentally transfer ownership to an uncontrolled account, breaking all functions with the `onlyProxyOwner()` modifier.

## Impact

Lack of two-step procedure for critical operations leaves them error-prone if the address is incorrect, the new address will take on the functionality of the new role immediately, or in this case if the address is a wrong one then the access to `onlyProxyOwner()` is forever bricked, in this case the most impactful issue would be that there would be no `proxy owner` to upgrade the current version of the proxy.

## Recommended Mitigation Steps

Consider implementing a two step process where the owner nominates an account and the nominated account needs to call an acceptOwnership() function for the transfer of ownership to fully succeed. This ensures the nominated EOA account is a valid and active account.

## Proof of concept

```solidity
/**
  /**
   * @dev Allows the current owner to transfer control of the contract to a newOwner.
   * @param newOwner The address to transfer ownership to.
   */
  function transferProxyOwnership(address newOwner) public onlyProxyOwner {
    require(newOwner != address(0));
    emit ProxyOwnershipTransferred(proxyOwner(), newOwner);
    setUpgradabilityOwner(newOwner);
  }

   * @dev Sets the address of the owner
   */
  function setUpgradabilityOwner(address newProxyOwner) internal {
//@audit
    bytes32 position = proxyOwnerPosition;
    assembly {
      sstore(position, newProxyOwner)
    }
  }

```

> see `@audit` tag
