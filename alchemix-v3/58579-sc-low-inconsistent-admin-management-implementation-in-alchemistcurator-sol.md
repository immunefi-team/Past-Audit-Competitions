# 58579 sc low inconsistent admin management implementation in alchemistcurator sol

**Submitted on Nov 3rd 2025 at 10:35:51 UTC by @Ratt13snak3 for** [**Audit Comp | Alchemix V3**](https://immunefi.com/audit-competition/alchemix-v3-audit-competition)

* **Report ID:** #58579
* **Report Type:** Smart Contract
* **Report severity:** Low
* **Target:** <https://github.com/alchemix-finance/v3-poc/blob/immunefi\\_audit/src/AlchemistCurator.sol>
* **Impacts:**
  * Permanent freezing of funds
  * Contract fails to deliver promised returns, but doesn't lose value

## Description

## **Description**

The [`AlchemistCurator`](https://github.com/alchemix-finance/v3-poc/blob/immunefi_audit/src/AlchemistCurator.sol) contract inherits from [`PermissionedProxy`](https://github.com/alchemix-finance/v3-poc/blob/immunefi_audit/src/utils/PermissionedProxy.sol), which already defines a one-step admin update function:

```solidity
function setAdmin(address _admin) external onlyAdmin {
    require(_admin != address(0), "zero");
    admin = _admin;
    emit AdminUpdated(_admin);
}
```

However, `AlchemistCurator` adds an entirely new two-step admin transfer mechanism using:

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

This introduces **two competing methods** for admin management: a one-step (`setAdmin`) and a two-step (`transferAdminOwnerShip` + `acceptAdminOwnership`). Both modify the same `admin` variable and are callable by the DAO (the admin), which can cause **confusion**, governance errors, or unintended privilege transfers if both flows are used inconsistently.

***

## **Impact** : Administrative confusion and potential misconfiguration

* DAO (admin) may mistakenly assume a two-step admin handover is enforced, when in fact the inherited `setAdmin()` can instantly replace the admin.

## **Recommendation**

If the intent is to use a **two-step transfer model**, remove or disable the `setAdmin()` function from `PermissionedProxy`. Alternatively, standardize on the single-step `setAdmin()` and delete `transferAdminOwnerShip()` and `acceptAdminOwnership()` from the curator.

## Proof of Concept

## Proof of Concept

insert the test function below into [`AlchemistCurator.t.sol`](https://github.com/alchemix-finance/v3-poc/blob/immunefi_audit/src/test/AlchemistCurator.t.sol)

```solidity
function testInconsistentAdminManagement() public {
    // Confirm current admin is the DAO
    assertEq(mytCuratorProxy.admin(), admin);

    // Admin sets a pendingAdmin using the curator's two-step system
    vm.prank(admin);
    mytCuratorProxy.transferAdminOwnerShip(address(0x5555555555555555555555555555555555555555));

    // At this point, pendingAdmin should be set but not yet effective
    assertEq(mytCuratorProxy.pendingAdmin(), address(0x5555555555555555555555555555555555555555));
    assertEq(mytCuratorProxy.admin(), admin);

    // However, the same admin can instantly override via inherited PermissionedProxy.setAdmin()
    vm.prank(admin);
    mytCuratorProxy.setAdmin(address(0x9999999999999999999999999999999999999999));

    // The one-step function bypasses the two-step safety pattern
    assertEq(mytCuratorProxy.admin(), address(0x9999999999999999999999999999999999999999));
    emit log("Admin changed instantly via inherited setAdmin(), bypassing pendingAdmin flow.");
}
```

then run the test:

```bash
forge test --mt testInconsistentAdminManagement
```
