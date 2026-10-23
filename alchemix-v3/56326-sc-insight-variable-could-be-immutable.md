# 56326 sc insight variable could be immutable

**Submitted on Oct 14th 2025 at 16:18:58 UTC by @PotEater for** [**Audit Comp | Alchemix V3**](https://immunefi.com/audit-competition/alchemix-v3-audit-competition)

* **Report ID:** #56326
* **Report Type:** Smart Contract
* **Report severity:** Insight
* **Target:** <https://github.com/alchemix-finance/v3-poc/blob/immunefi\\_audit/src/AlchemistV3Position.sol>
* **Impacts:**

## Description

## Brief/Intro

In the `AlchemistV3Position.sol` contract, the declared variable `alchemist` could be immutable. This would significantly save on gas costs during contract initialization and also during contract interactions.

## Vulnerability Details

The variable `alchemist` could be set as immutable. This would save on gas costs \~2,100 gas per call, this variable is used in the `onlyAlchemist` modifier. This modifier is used a lot in this contract.

```solidity
    address public alchemist;
```

Setting this variable to immutable would save gas costs because immutable variables cost less gas when interacted with, because they are not taken from the contract's storage.

## Impact Details

This is a code optimization..

This will save \~2,100 gas per interaction with this variable.

## References

<https://github.com/alchemix-finance/v3-poc/blob/b2e2aba046c36ff5e1db6f40f399e93cd2bdaad0/src/AlchemistV3Position.sol#L20>

## Proof of Concept

## Proof of Concept

Setting variable to immutable saves gas, because the contract will not have to read from the contract storage when interacting with this variable.

This saves \~2,100 gas per call.
