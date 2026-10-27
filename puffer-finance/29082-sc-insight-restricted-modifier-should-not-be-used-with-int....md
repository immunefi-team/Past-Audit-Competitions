# 29082 - \[SC - Insight] Restricted modifier should not be used with int...

Submitted on Mar 6th 2024 at 19:19:09 UTC by @SAAJ for [Boost | Puffer Finance](https://immunefi.com/bounty/pufferfinance-boost/)

Report ID: #29082

Report type: Smart Contract

Report severity: Insight

Target: <https://etherscan.io/address/0x7276925e42f9c4054afa2fad80fa79520c453d6a>

Impacts:

* wrong usage of OpenZeppelin library in contract

## Description

## Vulnerability Details

`_authorizeUpgrade` is an internal function in `PufferDepositor` and `PufferVault` that uses the `restricted` modifier from the contract `AccessManagedUpgradeable`.

## Impact

The `restricted` modifier is designed to enforce access control based on the caller's identity and the function being called. It checks if the caller is authorized to perform the operation.

Internal functions are not accessible from outside the contract, the access control mechanism provided by the `restricted` modifier is effectively bypassed. This means that any internal function marked with restricted can be called without any checks, potentially allowing unauthorized access or operations.

## Code Reference

<https://github.com/PufferFinance/pufETH/blob/2768d69196717e9f77a6837153b426e06e15c51f/src/PufferDepositor.sol#L202>

<https://github.com/PufferFinance/pufETH/blob/2768d69196717e9f77a6837153b426e06e15c51f/src/PufferVault.sol#294>

## Recommendations

The recommendation is made to use `ownable2StepUpgradeable` instead of `AccessManagedUpgradeable` for both the the `ownable2StepUpgradeable` have `onlyAdmin` modifier which can be used without any restrictions.

## Proof of concept

## POC

OZ clearly warns about using the `restricted` modifier with internal functions that can lead to serious security issues as stated in the [contract](https://github.com/OpenZeppelin/openzeppelin-contracts-upgradeable/blob/789ba4f167cc94088e305d78e4ae6f3c1ec2e6f1/contracts/access/manager/AccessManagedUpgradeable.sol#L55):

‘Unless you know what you're doing, it should never be used on `internal` functions. Failure to follow these rules can have critical security implications! This is because the permissions are determined by the function that entered the contract, i.e. the function at the bottom of the call stack, and not the function where the modifier is visible in the source code.’

```
/**
     * @dev Authorizes an upgrade to a new implementation
     * Restricted access
     * @param newImplementation The address of the new implementation
     */
    function _authorizeUpgrade(address newImplementation) internal virtual override restricted { }
```
