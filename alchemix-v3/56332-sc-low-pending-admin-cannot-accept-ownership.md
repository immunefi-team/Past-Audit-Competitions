# 56332 sc low pending admin cannot accept ownership

**Submitted on Oct 14th 2025 at 16:51:59 UTC by @PotEater for** [**Audit Comp | Alchemix V3**](https://immunefi.com/audit-competition/alchemix-v3-audit-competition)

* **Report ID:** #56332
* **Report Type:** Smart Contract
* **Report severity:** Low
* **Target:** <https://github.com/alchemix-finance/v3-poc/blob/immunefi\\_audit/src/AlchemistCurator.sol>
* **Impacts:**

## Description

## Brief/Intro

The function `acceptAdminOwnership` is guarded with the `onlyAdmin` modifier.

This is a mistake, because the pending admin cannot accept ownership, because he is not yet an admin when claiming the ownership.

## Vulnerability Details

The function `transferAdminOwnerShip` sets the `pendingAdmin` address. Then, the `pendingAdmin` is expected to call `acceptAdminOwnership` and accept his ownership. However, this is not possible, because the pending admin is not yet an admin.

Code snippet:

```solidity
   // ===== Admin Management =====
    function transferAdminOwnerShip(address _newAdmin) external onlyAdmin {
        pendingAdmin = _newAdmin;
    }

    function acceptAdminOwnership() external onlyAdmin {
        admin = pendingAdmin;
        pendingAdmin = address(0);
        emit AdminChanged(admin);
    }
```

## Impact Details

The impact is that the pending admin cannot accept the ownership.

This is a Denial of Service.

## References

<https://github.com/alchemix-finance/v3-poc/blob/b2e2aba046c36ff5e1db6f40f399e93cd2bdaad0/src/AlchemistCurator.sol#L31>

## Proof of Concept

## Proof of Concept

This PoC demonstrates how a pending admin tries to call `acceptAdminOwnership` but fails due to the `onlyAdmin` modifier:

Add this function in the `AlchemistCurator.t.sol` test file:

```solidity
    function test_PoC() external {
        address pendingAdminn = makeAddr("pendingadminn");

        vm.prank(admin);
        mytCuratorProxy.transferAdminOwnerShip(pendingAdminn);

        vm.expectRevert();
        vm.prank(pendingAdminn);
        mytCuratorProxy.acceptAdminOwnership();
    }
```
