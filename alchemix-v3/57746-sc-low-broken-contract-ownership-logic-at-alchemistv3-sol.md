# 57746 sc low broken contract ownership logic at alchemistv3 sol

**Submitted on Oct 28th 2025 at 16:28:14 UTC by @Cyborg for** [**Audit Comp | Alchemix V3**](https://immunefi.com/audit-competition/alchemix-v3-audit-competition)

* **Report ID:** #57746
* **Report Type:** Smart Contract
* **Report severity:** Low
* **Target:** <https://github.com/alchemix-finance/v3-poc/blob/immunefi\\_audit/src/AlchemistCurator.sol>
* **Impacts:**
  * Broken contract ownership transfer logic

## Description

## Brief/Intro

Broken contract ownership logic at AlchemistV3.sol due to impossible accept ownership method.

## Vulnerability Details

Inside *src/AlchemistCurator.sol* there are 2 methods to handle the admin transfership - methods `transferAdminOwnerShip` and `acceptAdminOwnership`. The problem here is that both methods are created with modifier `onlyAdmin` meaning that the selected `pendingAdmin` at method `transferAdminOwnerShip` can't really request method `acceptAdminOwnership` in order to accept the admin role, because he is not yet the admin thus the `onlyAdmin` modifier will revert. There is no way for the `pendingAdmin` to accept the ownership and there is no way for the current `admin` to force transfer the ownership. The smart contract has very important methods for Morpho's Vault V2 that are protected with the `onlyAdmin` modifier.

## Impact Details

The AlchemistV3 contract being stuck with the initial admin.

## Recommendation

Remove the `onlyAdmin` modifier from method `acceptAdminOwnership` and add validation to check if the `msg.sender == pendingAdmin`:

```
function acceptAdminOwnership() external {
    require(msg.sender == pendingAdmin, InvalidPendingAdmin());
    admin = pendingAdmin;
    pendingAdmin = address(0);
    emit AdminChanged(admin);
}
```

## Proof of Concept

## Proof of Concept

Create file `src/test/AlchemistCurator.test_acceptAdminOwnership.t.sol` and run it with command `forge test src/test/AlchemistCurator.test_acceptAdminOwnership.t.sol -vv`:

```
// SPDX-License-Identifier: MIT
pragma solidity 0.8.28;

import "forge-std/Test.sol";
import {MockAlchemistCurator} from "./mocks/MockAlchemistCurator.sol";


contract AlchemistCuratorTest is Test {
    MockAlchemistCurator public mytCuratorProxy;
    address public admin = address(0x4444444444444444444444444444444444444444); // DAO OSX
    address public newAdmin = address(0x123);

    function setUp() public {
        vm.startPrank(admin);
        mytCuratorProxy = new MockAlchemistCurator(admin, admin);
        vm.stopPrank();
    }

    function test_acceptAdminOwnership() public {
        vm.startPrank(admin);
        assertEq(mytCuratorProxy.pendingAdmin(), address(0));

        mytCuratorProxy.transferAdminOwnerShip(newAdmin);
        assertEq(mytCuratorProxy.pendingAdmin(), newAdmin);

        vm.startPrank(newAdmin);
        vm.expectRevert(bytes("PD")); // error returned from https://github.com/alchemix-finance/v3-poc/blob/immunefi_audit/src/utils/PermissionedProxy.sol#L18
        mytCuratorProxy.acceptAdminOwnership();
    }
}
```
