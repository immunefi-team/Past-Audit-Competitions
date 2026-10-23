# 57637 sc low acceptadminownership doesn t allow expected user approval

**Submitted on Oct 27th 2025 at 19:46:04 UTC by @magtentic for** [**Audit Comp | Alchemix V3**](https://immunefi.com/audit-competition/alchemix-v3-audit-competition)

* **Report ID:** #57637
* **Report Type:** Smart Contract
* **Report severity:** Low
* **Target:** <https://github.com/alchemix-finance/v3-poc/blob/immunefi\\_audit/src/AlchemistCurator.sol>
* **Impacts:**
  * Contract fails to deliver promised returns, but doesn't lose value

## Description

## Brief/Intro

`acceptAdminOwnership` is a function to transfer a `pendingAdmin` request as part of a 2 step process.\
As such, there is an expectancy that the `pendingAdmin` should be able to call the function without the function reverting.

## Vulnerability Details

When `pendingAdmin` calls `acceptAdminOwnership`, the transaction reverts.\
This is unexpected, as the function is meant to be called by the `pendingAdmin`.\
The reason for this mainly resides with the function having the `onlyAdmin` modifier attached to it which then forces the `admin` to only call this function.

```
    function acceptAdminOwnership() external onlyAdmin {
...
```

`onlyAdmin` modifier :

```
    modifier onlyAdmin() {
        require(msg.sender == admin, "PD");
        _;
    }
```

This process just nullifies the 2 step process of calling `transferAdminOwnerShip` then the `pendingAdmin` calling `acceptAdminOwnership` as the current admin could've then just called `setAdmin` with the new admin details.

## Impact Details

`pendingAdmin` cannot accept admin ownership, in the context of the naming, acceptance of a `pendingAdmin` to `admin` is done by the `pendingAdmin` and not the current `admin`.

## References

* AlchemistCurator : <https://github.com/alchemix-finance/v3-poc/blob/immunefi\\_audit/src/AlchemistCurator.sol?utm\\_source=immunefi>
* PermissionedProxy : <https://github.com/alchemix-finance/v3-poc/blob/immunefi\\_audit/src/utils/PermissionedProxy.sol?utm\\_source=immunefi>

As a sidenote, the function `transferAdminOwnerShip` has an uppercase S in the name which should be lowercase for readability.

## Proof of Concept

## Proof of Concept

Steps below shows how a pendingAdmin cannot accept the admin ownership to themselves and needs to be done by the existing admin on their behalf.

* Current admin configures a new admin by calling `transferAdminOwnerShip`
* New admin tries to accept admin ownership by calling `acceptAdminOwnership` which then fails.
* Current admin calls `acceptAdminOwnership` which now passes due to the `onlyAdmin` modifier.
* Additional flows calling `setAdmin` to transfer ownership back to previous admin showing only a single step was required from the current admin

```
// SPDX-License-Identifier: MIT

pragma solidity 0.8.28;

import {Test} from "forge-std/Test.sol";
import {VaultV2} from "../../lib/vault-v2/src/VaultV2.sol";
import {ERC20Mock} from "../../lib/vault-v2/test/mocks/ERC20Mock.sol";
import {IVaultV2} from "../../lib/vault-v2/src/interfaces/IVaultV2.sol";
import {TestYieldToken} from "./mocks/TestYieldToken.sol";
import {TestERC20} from "./mocks/TestERC20.sol";
import {TokenUtils} from "../libraries/TokenUtils.sol";
import {MockYieldToken} from "./mocks/MockYieldToken.sol";
import {IMockYieldToken} from "./mocks/MockYieldToken.sol";
import {MockMYTStrategy} from "./mocks/MockMYTStrategy.sol";
import {MockAlchemistCurator} from "./mocks/MockAlchemistCurator.sol";
import {MYTTestHelper} from "./libraries/MYTTestHelper.sol";
import {IMYTStrategy} from "../interfaces/IMYTStrategy.sol";

contract AlchemistCuratorTest is Test {
    using MYTTestHelper for *;

    MockAlchemistCurator public mytCuratorProxy;
    VaultV2 public vault;
    address public operator = address(0x2222222222222222222222222222222222222222); // default operator
    address public admin = address(0x4444444444444444444444444444444444444444); // DAO OSX
    address public newAdmin = address(0x12345); 
    address public mockVaultCollateral = address(new TestERC20(100e18, uint8(18)));
    address public mockStrategyYieldToken = address(new MockYieldToken(mockVaultCollateral));
    uint256 public defaultStrategyAbsoluteCap = 200 ether;
    uint256 public defaultStrategyRelativeCap = 1e18; // 100%
    MockMYTStrategy public mytStrategy;

    function setUp() public {
        vm.startPrank(admin);
        mytCuratorProxy = new MockAlchemistCurator(admin, operator);
        vault = MYTTestHelper._setupVault(mockVaultCollateral, admin, address(mytCuratorProxy));
        mytStrategy = MYTTestHelper._setupStrategy(address(vault), mockStrategyYieldToken, admin, "MockToken", "MockTokenProtocol", IMYTStrategy.RiskClass.LOW);
        vm.stopPrank();
    }

    function testAcceptPendingAdminFailure() public {
        vm.prank(admin);
        mytCuratorProxy.transferAdminOwnerShip(newAdmin);

        //New Admin tries to accept and fails miserably
        vm.startPrank(newAdmin);
        vm.expectRevert();
        mytCuratorProxy.acceptAdminOwnership();
        vm.stopPrank();

        //Current admin then has to acceptAdminOwnership for new admins
        vm.startPrank(admin);
        mytCuratorProxy.acceptAdminOwnership();
        vm.stopPrank();

        //2 step process for no reason as admin could've just called setAdmin to transfer ownership
        vm.prank(newAdmin);
        mytCuratorProxy.setAdmin(admin);

        //Test with new admin to transfer ownership to verify that it doesn't revert
        vm.prank(admin);
        mytCuratorProxy.transferAdminOwnerShip(newAdmin);
    }
}

```
