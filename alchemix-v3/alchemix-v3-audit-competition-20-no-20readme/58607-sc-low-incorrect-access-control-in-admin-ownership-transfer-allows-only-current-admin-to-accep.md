# 58607 sc low incorrect access control in admin ownership transfer allows only current admin to accept ownership instead of pending admin

**Submitted on Nov 3rd 2025 at 14:48:38 UTC by @dobrevaleri for** [**Audit Comp | Alchemix V3**](https://immunefi.com/audit-competition/alchemix-v3-audit-competition)

* **Report ID:** #58607
* **Report Type:** Smart Contract
* **Report severity:** Low
* **Target:** <https://github.com/alchemix-finance/v3-poc/blob/immunefi\\_audit/src/AlchemistCurator.sol>
* **Impacts:**
  * Contract fails to deliver promised returns, but doesn't lose value

## Description

## Brief/Intro

The `AlchemistCurator::acceptAdminOwnership()` function uses the `onlyAdmin` modifier, which restricts access to the current admin instead of the pending admin. This breaks the two-step admin transfer pattern and makes it impossible for a pending admin to accept ownership.

## Vulnerability Details

The `AlchemistCurator` contract implements a two-step admin ownership transfer mechanism where:

1. The current admin calls `transferAdminOwnerShip()` to set a pending admin
2. The pending admin should call `acceptAdminOwnership()` to finalize the transfer

However, the implementation is flawed:

```solidity
function transferAdminOwnerShip(address _newAdmin) external onlyAdmin {
    pendingAdmin = _newAdmin;
}

function acceptAdminOwnership() external onlyAdmin {
    admin = pendingAdmin;
    pendingAdmin = address(0);
    emit AdminChanged(admin);
}
```

The `acceptAdminOwnership()` function uses the `onlyAdmin` modifier, which requires `msg.sender == admin` (from `PermissionedProxy`). This means only the current admin can call this function, not the pending admin.

The correct implementation can be seen in other contracts like `Transmuter` and `AlchemistV3`:

```solidity
function acceptAdmin() external {
    _checkState(pendingAdmin != address(0));
    
    if (msg.sender != pendingAdmin) { 
        revert Unauthorized();
    }
    
    admin = pendingAdmin;
    pendingAdmin = address(0);
    // emit events
}
```

This vulnerability breaks the guarantee of two-step admin transfers, which is designed to prevent accidental admin changes by requiring explicit acceptance from the new admin.

## Impact Details

No pending admin can ever accept ownership since they lack the current admin privileges required by the `onlyAdmin` modifier.

## References

<https://github.com/alchemix-finance/v3-poc/blob/a192ab313c81ba3ab621d9ca1ee000110fbdd1e9/src/AlchemistCurator.sol#L31-L35>

<https://github.com/alchemix-finance/v3-poc/blob/a192ab313c81ba3ab621d9ca1ee000110fbdd1e9/src/Transmuter.sol#L94-L106>

## Proof of Concept

## Proof of Concept

```solidity
// SPDX-License-Identifier: MIT

pragma solidity 0.8.28;

import {Test, console} from "forge-std/Test.sol";
import {AlchemistCurator} from "../AlchemistCurator.sol";

contract PoC_AdminOwnershipVulnerability is Test {
    AlchemistCurator public curator;

    address public currentAdmin = makeAddr("currentAdmin");
    address public newAdmin = makeAddr("newAdmin");
    address public operator = makeAddr("operator");

    function setUp() public {
        // Deploy AlchemistCurator with current admin
        vm.prank(currentAdmin);
        curator = new AlchemistCurator(currentAdmin, operator);
    }

    function test_AlchemistCurator_PendingAdminCannotAccept() public {
        // Current admin initiates transfer to new admin
        vm.prank(currentAdmin);
        curator.transferAdminOwnerShip(newAdmin);

        // The pending admin should be able to accept, but can't due to onlyAdmin modifier
        vm.startPrank(newAdmin);
        vm.expectRevert(bytes("PD")); // PermissionedProxy "Permission Denied"
        curator.acceptAdminOwnership();
        vm.stopPrank();
    }
}
```
