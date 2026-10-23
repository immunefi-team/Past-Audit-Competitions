# 58333 sc low incorrect onlyadmin modifier in acceptadminownership

**Submitted on Nov 1st 2025 at 10:40:22 UTC by @silverologist for** [**Audit Comp | Alchemix V3**](https://immunefi.com/audit-competition/alchemix-v3-audit-competition)

* **Report ID:** #58333
* **Report Type:** Smart Contract
* **Report severity:** Low
* **Target:** <https://github.com/alchemix-finance/v3-poc/blob/immunefi\\_audit/src/AlchemistCurator.sol>
* **Impacts:**

## Description

## Brief/Intro

The admin ownership transfer logic in the AlchemistCurator contract is implemented incorrectly. Instead of using a proper two step handover process where the new admin confirms ownership, the contract requires two separate actions from the current admin, effectively making it a single party transfer.

## Vulnerability Details

The intended admin transfer process should occur in two distinct steps:

Step 1: The current admin initiates the transfer by designating a new pending admin. This part is implemented correctly in `transferAdminOwnerShip`:

```solidity
    function transferAdminOwnerShip(address _newAdmin) external onlyAdmin {
        pendingAdmin = _newAdmin;
    }
```

Step 2: The pending admin should then confirm and accept ownership. However, this step is incorrectly implemented in `acceptAdminOwnership`:

```solidity
    function acceptAdminOwnership() external onlyAdmin {
        admin = pendingAdmin;
        pendingAdmin = address(0);
        emit AdminChanged(admin);
    }
```

The issue lies in the use of the onlyAdmin modifier, which restricts this function to the current admin instead of allowing the pending admin to call it.

Expected flow:

* The current admin initiates the transfer to a new pending admin.
* The pending admin accepts and becomes the new admin.

Actual flow:

* The current admin initiates the transfer to a new pending admin.
* The current admin must also execute the acceptance step to finalize the transfer.

## Proof of Concept

## Proof of Concept

Add the following test to `./src/test/AlchemistCurator.t.sol`:

```solidity
    function testAdminOwnershipPoC() public {
        address newAdmin = address(0x1234567890123456789012345678901234567890);

        // the current admin requests a new admin
        vm.prank(admin);
        mytCuratorProxy.transferAdminOwnerShip(newAdmin);

        // the new admin should be able to accept admin ownership, but currently this reverts
        vm.prank(newAdmin);
        vm.expectRevert(abi.encode("PD"));
        mytCuratorProxy.acceptAdminOwnership();

        // instead, the current admin is able to accept the ownership of the new admin
        vm.prank(admin);
        mytCuratorProxy.acceptAdminOwnership();
    }
```
