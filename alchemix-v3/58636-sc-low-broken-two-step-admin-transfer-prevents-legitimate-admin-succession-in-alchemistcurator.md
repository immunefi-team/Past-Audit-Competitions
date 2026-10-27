# 58636 sc low broken two step admin transfer prevents legitimate admin succession in alchemistcurator

**Submitted on Nov 3rd 2025 at 18:14:33 UTC by @pyman for** [**Audit Comp | Alchemix V3**](https://immunefi.com/audit-competition/alchemix-v3-audit-competition)

* **Report ID:** #58636
* **Report Type:** Smart Contract
* **Report severity:** Low
* **Target:** <https://github.com/alchemix-finance/v3-poc/blob/immunefi\\_audit/src/AlchemistCurator.sol>
* **Impacts:**
  * Smart contract unable to operate due to lack of token funds

## Description

## Brief/Intro

The `acceptAdminOwnership()` function wrongly restricts calls to the current admin instead of the nominated `pendingAdmin`, preventing legitimate admin transfers and potentially locking critical contract control indefinitely.

## Vulnerability Details

The AlchemistCurator contract uses a two-step process to transfer the admin role via `transferAdminOwnerShip()` and `acceptAdminOwnership()`. However, the `acceptAdminOwnership()` function incorrectly applies the `onlyAdmin` modifier, restricting calls to the current admin instead of the intended `pendingAdmin` set by the current admin. As a result, legitimate admin transfers cannot be completed, potentially locking the contract under the control of the current admin. If the current admin loses access to their keys, becomes unresponsive, or acts maliciously, they could deliberately or accidentally prevent the administrative role from being transferred, leading to governance stagnation and a permanent loss of control over critical contract functions.

Problem summary: the `acceptAdminOwnership()` function uses the wrong access control: it requires the caller to be the current admin (via `onlyAdmin`) instead of requiring the caller to be the `pendingAdmin`. This makes completing the two-step transfer impossible.

The admin transfer mechanism in AlchemistCurator.sol consists of two functions:

```javascript
    function transferAdminOwnerShip(address _newAdmin) external onlyAdmin {
        pendingAdmin = _newAdmin;
    }

    function acceptAdminOwnership() external onlyAdmin {  // BUG: onlyAdmin modifier shouldn't be here.
        admin = pendingAdmin;
        pendingAdmin = address(0);
        emit AdminChanged(admin);
    }
```

After `transferAdminOwnerShip(newAdmin)` is called by the current admin, `pendingAdmin` becomes `newAdmin`. But `acceptAdminOwnership()` is `onlyAdmin`, so `newAdmin` (not admin at this point) cannot call it. The accept step can never be executed by the intended new admin.

* Only current admin can call `acceptAdminOwnership()` due to `onlyAdmin` modifier.
* The intended new admin (`pendingAdmin`) cannot call the function because they're not yet admin.
* The transfer can never be completed by the intended new admin.

Root Cause: The access control check is applied to the wrong party. The acceptance should be performed by the recipient, not the sender.

## Impact Details

Due to the incorrect access control in `acceptAdminOwnership()`, the contract’s administrative role can **never be transferred**, potentially freezing all privileged operations.

This means that if the current admin loses access, becomes unresponsive, or acts maliciously, the protocol loses the ability to update, pause, or recover funds — permanently locking its control layer.

### Step-by-Step Exploitation

**Scenario 1 – Normal Admin Transfer Attempt**

1. Current admin (`0xAlice`) calls `transferAdminOwnerShip(0xBob)` → `pendingAdmin = 0xBob`
2. `0xBob` attempts to call `acceptAdminOwnership()` but reverts with `"PD"` due to `onlyAdmin` modifier.
3. Only `0xAlice` (the current admin) can call it — meaning the transfer can never complete.
4. **Result:** Admin handover is impossible.

**Scenario 2 – Admin Key Loss**

1. `0xAlice` loses private keys.
2. No other entity can assume admin privileges because `acceptAdminOwnership()` requires the lost key to succeed.
3. **Result:** Contract permanently bricked from an operational standpoint.

**Scenario 3 – Malicious Admin**

1. `0xAlice` intentionally calls `transferAdminOwnerShip()` to mislead others into thinking control will change.
2. The new admin (`0xBob`) can never actually assume ownership.
3. **Result:** Governance remains centralized and deceptive.

## References

* Found in src/AlchemistCurator.sol at line 31, branch `immunefi_audit`: [Line: 31](https://github.com/alchemix-finance/v3-poc/blob/immunefi_audit/src/AlchemistCurator.sol#L31)

## Link to Proof of Concept

<https://gist.github.com/m-sabonkudi/3b535c8cdf83edc23f59afb00aa4842b>

## Proof of Concept

## Proof of Concept

The following Foundry test demonstrates that the `acceptAdminOwnership()` function can only be called by the current admin instead of the `pendingAdmin`.

```javascript
// SPDX-License-Identifier: SEE LICENSE IN LICENSE
pragma solidity 0.8.28;


import { Test, console } from "forge-std/Test.sol";
import { AlchemistCurator } from "../src/AlchemistCurator.sol";


contract AlchemistCuratorTest is Test {
    AlchemistCurator alchemistCurator;

    address admin = makeAddr("admin");
    address operator = makeAddr("operator");

    address newAdmin = makeAddr("newadmin");

    function setUp() public {
        alchemistCurator = new AlchemistCurator(admin, operator);
    }

    ///// PRELIMINARY TESTS /////

    function test_RevertIfNotAdminAttemptsTransferAdminOwnership() public {
        vm.prank(operator);
        vm.expectRevert(bytes("PD"));
        alchemistCurator.transferAdminOwnerShip(newAdmin);
    }

    function test_AdminCanSuccessfullyTransferAdminOwnership() public {
        vm.prank(admin);
        alchemistCurator.transferAdminOwnerShip(newAdmin);
    }



    ////// AUDIT TESTS TO IDENTIFY BUG /////

    function test_NewAdminCannotAcceptAdminOwnership() public {
        vm.prank(admin);
        alchemistCurator.transferAdminOwnerShip(newAdmin);

        vm.prank(newAdmin);
        vm.expectRevert(bytes("PD"));
        alchemistCurator.acceptAdminOwnership();
        // @AUDIT: This reverts because newAdmin is NOT current admin
        // Expected: newAdmin should be able to accept ownership
        // Actual: Only current admin can call acceptAdminOwnership()       
    }

    function test_AdminCanAcceptAdminOwnership_NotIntendedBehavior() public {
        vm.startPrank(admin);
        alchemistCurator.transferAdminOwnerShip(newAdmin);
        alchemistCurator.acceptAdminOwnership();
        vm.stopPrank();
    }

}
```

**Output:**

```bash
Ran 4 tests for test/AlchemistCuratorTest.t.sol:AlchemistCuratorTest
[PASS] test_AdminCanAcceptAdminOwnership_NotIntendedBehavior() (gas: 33701)
[PASS] test_AdminCanSuccessfullyTransferAdminOwnership() (gas: 39215)
[PASS] test_NewAdminCannotAcceptAdminOwnership() (gas: 43158)
[PASS] test_RevertIfNotAdminAttemptsTransferAdminOwnership() (gas: 18145)
Suite result: ok. 4 passed; 0 failed; 0 skipped; finished in 26.48ms (17.15ms CPU time)

Ran 1 test suite in 102.68ms (26.48ms CPU time): 4 tests passed, 0 failed, 0 skipped (4 total tests)
```

To interpret, in the `test_NewAdminCannotAcceptAdminOwnership()`, the admin successfully called the `AlchemistCurator::transferAdminOwnerShip(newAdmin)` which updates `AlchemistCurator::pendingAdmin` to the `newAdmin` passed. Subsequently, the `newAdmin` now calls `AlchemistCurator::acceptAdminOwnership()`, but it reverts with `PD` because it has the `onlyOwner` modifier from the inherited `PermissionedProxy` contract. The `PermissionedProxy::onlyOwner()` modifier is what reverted with `PD` as per `require(msg.sender == admin, "PD");`.

**Recommended Mitigation:** Simply remove the `onlyOwner` modifier and add a check to make sure the caller is the `pendingAdmin`.

```diff
-    function acceptAdminOwnership() external onlyAdmin {
+    function acceptAdminOwnership() external {
+       require(msg.sender == pendingAdmin, "Not pending admin.");    
        admin = pendingAdmin;
        pendingAdmin = address(0);
        emit AdminChanged(admin);
    }
```

It is also recommended to check that the `_newAdmin` passed by the current admin is a valid address. On top of that, it is recommended to correct the typo in the function name from `transferAdminOwnerShip` to `transferAdminOwnership`.

```javascript
@>    function transferAdminOwnership(address _newAdmin) external onlyAdmin {
@>      require(_newAdmin != address(0), "INVALID_ADDRESS");
        pendingAdmin = _newAdmin;
    }
```
