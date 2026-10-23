# 56350 sc insight implementation contract alchemistv3 not locked disableinitializers missing&#x20;

**Submitted on Oct 14th 2025 at 21:15:54 UTC by @wylis for** [**Audit Comp | Alchemix V3**](https://immunefi.com/audit-competition/alchemix-v3-audit-competition)

* **Report ID:** #56350
* **Report Type:** Smart Contract
* **Report severity:** Insight
* **Target:** <https://github.com/alchemix-finance/v3-poc/blob/immunefi\\_audit/src/AlchemistV3.sol>
* **Impacts:**

## Description

## Brief/Intro

`AlchemistV3` is designed for use behind a proxy (tests deploy a `TransparentUpgradeableProxy` and call `initialize(...)` via the proxy). The implementation contract uses `constructor() initializer {}` instead of calling `_disableInitializers()` in its constructor. While not exploitable today, this is a known upgrade security issue and a deviation from current OpenZeppelin best practice.

## Vulnerability Details

OpenZeppelin’s initialization system uses a version number. Each initializer/reinitializer consumes a version so it can’t be run again. The docs say: “The initialization functions use a version number. Once a version number is used, it is consumed and cannot be reused.”

Conventionally, the `initializer` modifier is the first step and is treated as version 1. Therefore, `initializer` is effectively equivalent to `reinitializer(1)`. Subsequent upgrades use `reinitializer(2), reinitializer(3)`, etc. Function call `_disableInitializers()` sets the internal “initialized version” to a terminal state so no `initializer` or any future `reinitializer(n)` can ever run on that specific contract instance (the implementation). OpenZeppelin recommends calling it in the implementation’s constructor for proxy-based systems.

## Impact Details

This is an Insight for Security best practice (implementation not hard-locked; increased risk of accidental initialization/reinitialization on the implementation address in future versions).

## References

<https://docs.openzeppelin.com/contracts/5.x/api/proxy#initializable> "Avoid leaving a contract uninitialized. An uninitialized contract can be taken over by an attacker. This applies to both a proxy and its implementation contract, which may impact the proxy. To prevent the implementation contract from being used, you should invoke the Initializable.\_disableInitializers function in the constructor to automatically lock it."

<https://docs.openzeppelin.com/upgrades-plugins/writing-upgradeable#initializing-the-implementation-contract> "To prevent the implementation contract from being used, you should invoke the \_disableInitializers function in the constructor to automatically lock it"

## Proof of Concept

## Proof of Concept

Not required for Insights.
