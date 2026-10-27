# 57479 sc low logical bug in alchemistcurator acceptadminownership asking to current admin to accept ownership&#x20;

**Submitted on Oct 26th 2025 at 15:04:50 UTC by @Eagle\_Eye for** [**Audit Comp | Alchemix V3**](https://immunefi.com/audit-competition/alchemix-v3-audit-competition)

* **Report ID:** #57479
* **Report Type:** Smart Contract
* **Report severity:** Low
* **Target:** <https://github.com/alchemix-finance/v3-poc/blob/immunefi\\_audit/src/AlchemistCurator.sol>
* **Impacts:**
  * Protocol insolvency

## Description

## Brief/Intro

There is a logical issue in `AlchemistCurator::acceptAdminOwnership`, where only admin can call the function but pending admin should as in `AlchemistStrategyClassifier::acceptOwnership`

## Vulnerability Details

There are two state variables, `admin` and `pendingAdmin`. When admin calls `AlchemistCurator::transferAdminOwnerShip`, the `pendingAdmin` is set. And then only admin can call `acceptOwnership`, but in actual, `pendingAdmin` should call this function. Means admin transfers and accepts ownership itself, and even if it is intentional , second function is useless, can be done in the first function alone.

## Impact Details

* Admin can transfer and accept ownership by itself, by transferring ownership to malicious user.

## References

<https://github.com/alchemix-finance/v3-poc/blob/a192ab313c81ba3ab621d9ca1ee000110fbdd1e9/src/AlchemistCurator.sol#L31-L35>.

## Proof of Concept

## Proof of Concept

// SPDX-License-Identifier: MIT pragma solidity 0.8.28;

import { Test } from "forge-std/Test.sol"; import { AlchemistCurator } from "./AlchemistCurator.sol";

contract TestAlchemistCurator is Test {

```
address admin = address(0x123);

address operator = address(0x234);

address pendingAdmin = address(0x456);

AlchemistCurator alchemistCurator;

function setUp() public {

    alchemistCurator = new AlchemistCurator(admin, operator);

}

function test_AdminTransferAndAcceptOwnership() public {
    vm.startPrank(admin);
    alchemistCurator.transferAdminOwnerShip(pendingAdmin);
    alchemistCurator.acceptAdminOwnership();
    vm.stopPrank();
}
```

}
