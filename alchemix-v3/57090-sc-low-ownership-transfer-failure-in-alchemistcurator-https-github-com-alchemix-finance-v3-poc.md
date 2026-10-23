# 57090 sc low ownership transfer failure in alchemistcurator https github com alchemix finance v3 poc blob immunefi audit src alchemistcurator sol prevents future dao governance or recovery

**Submitted on Oct 23rd 2025 at 11:45:54 UTC by @Ratt13snak3 for** [**Audit Comp | Alchemix V3**](https://immunefi.com/audit-competition/alchemix-v3-audit-competition)

* **Report ID:** #57090
* **Report Type:** Smart Contract
* **Report severity:** Low
* **Target:** <https://github.com/alchemix-finance/v3-poc/blob/immunefi\\_audit/src/AlchemistCurator.sol>
* **Impacts:**
  * Contract fails to deliver promised returns, but doesn't lose value

## Description

### **Brief / Intro**

The AlchemistCurator contract defines a two-step ownership transfer mechanism (`transferAdminOwnerShip` and [`acceptAdminOwnership`](https://github.com/alchemix-finance/v3-poc/blob/a192ab313c81ba3ab621d9ca1ee000110fbdd1e9/src/AlchemistCurator.sol#L31)), but the latter function incorrectly restricts access to the current admin instead of the `pendingAdmin`. Although the underlying `PermissionedProxy` base contract allows direct admin reassignment via [`setAdmin()`](https://github.com/alchemix-finance/v3-poc/blob/a192ab313c81ba3ab621d9ca1ee000110fbdd1e9/src/utils/PermissionedProxy.sol#L31), this inconsistency introduces a broken transfer path. If the DAO or multisig relies on the two-step process for ownership transfer, ownership transfer will silently fail, potentially leading to governance confusion or lockout.

***

### **Vulnerability Details**

In `AlchemistCurator`, ownership transfer is handled through `transferAdminOwnerShip()` and `acceptAdminOwnership()`. However, the `acceptAdminOwnership()` function incorrectly restricts execution to the current admin instead of allowing the `pendingAdmin` to call it.

```solidity
function acceptAdminOwnership() external onlyAdmin {
    admin = pendingAdmin;
    pendingAdmin = address(0);
    emit AdminChanged(admin);
}
```

This means only the *existing* admin can call `acceptAdminOwnership()`, not the `pendingAdmin` who is meant to assume ownership. As a result, a new admin cannot be set unless `acceptAdminOwnership()` is called by the current admin, which breaks the intended flow. As a result:

* Ownership can be set correctly at deployment.
* But any later ownership transfer attempt will fail, since the `pendingAdmin` cannot accept the role.
* Following the intended flow, the contract remains permanently locked under the current admin, blocking any DAO or governance role rotation. unless `setAdmin()` or `acceptAdminOwnership()` is called by the admin.

Because `AlchemistCurator` is a permissioned proxy controlling privileged actions (like managing operators or Permissioned Calls), losing the ability to change the admin can freeze critical system functions.

***

### Impact Details

**Severity:** Medium

* Governance control confusion or lockout\
  Although ownership can still technically be changed via setAdmin(), the inconsistency between mechanisms can lead to misconfigured or irrecoverable governance state in practice.

***

### **References**

* [`AlchemistCurator::acceptAdminOwnership()`](https://github.com/alchemix-finance/v3-poc/blob/a192ab313c81ba3ab621d9ca1ee000110fbdd1e9/src/AlchemistCurator.sol#L31)
* [`PermissionedProxy.sol`](https://github.com/alchemix-finance/v3-poc/blob/a192ab313c81ba3ab621d9ca1ee000110fbdd1e9/src/utils/PermissionedProxy.sol)
* <https://keenanlukeom.github.io/alchemix-v3-docs/governance/onchain/why-onchain-governance>

## Proof of Concept

## Proof of Concept

insert the test function below into [AlchemistCurator.t.sol](https://github.com/alchemix-finance/v3-poc/blob/immunefi_audit/src/test/AlchemistCurator.t.sol)

```solidity
function testAcceptAdminOwnershipAlwaysReverts() public {
    // admin tries to transfer ownership to a new admin after deployment
    vm.prank(admin);
    mytCuratorProxy.transferAdminOwnerShip(address(0x5555555555555555555555555555555555555555));
    // the new admin tries to accept admin ownership but fails
    vm.prank(address(0x5555555555555555555555555555555555555555));
    vm.expectRevert(abi.encode("PD"));
    mytCuratorProxy.acceptAdminOwnership();
}
```

then run:

```bash
forge test --mt testAcceptAdminOwnershipAlwaysReverts
```
