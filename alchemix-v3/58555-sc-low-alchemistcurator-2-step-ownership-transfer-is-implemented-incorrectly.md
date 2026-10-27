# 58555 sc low alchemistcurator 2 step ownership transfer is implemented incorrectly

**Submitted on Nov 3rd 2025 at 07:54:24 UTC by @Oxdeadmanwalking for** [**Audit Comp | Alchemix V3**](https://immunefi.com/audit-competition/alchemix-v3-audit-competition)

* **Report ID:** #58555
* **Report Type:** Smart Contract
* **Report severity:** Low
* **Target:** <https://github.com/alchemix-finance/v3-poc/blob/immunefi\\_audit/src/AlchemistCurator.sol>
* **Impacts:**
  * Contract fails to deliver promised returns, but doesn't lose value
  * Ownership transfer design error

## Description

## Brief/Intro

`AlchemistCurator` implements a 2 step ownership transfer system that allows the current admin to propose a `pendingAdmin` which can later accept the ownership of the contract by calling `acceptAdminOwnership()`. This function however is only callable by the current admin so the pending admin will never be able to accept the ownership, leaving room for errors.

## Vulnerability Details

`AlchemistCurator` 2-step ownership transfer system is similar to `Ownable2Step` by Openzeppelin which aims to minimize the room for error when performing ownership transfering operations.

While other contracts in the system implement the mechanism correctly, in `AlchemistCurator` , `acceptAdminOwnership` is only callable by the current admin and not the pending admin breaking the intended design and making the process as error prone as a single step ownership transfer.

```
    // ===== Admin Management =====
    function transferAdminOwnerShip(address _newAdmin) external onlyAdmin {
        pendingAdmin = _newAdmin;
    }

    // @audit this can never be executed by the pending admin because it is currently not the admin
    function acceptAdminOwnership() external onlyAdmin {
        admin = pendingAdmin;
        pendingAdmin = address(0);
        emit AdminChanged(admin);
    }
```

`acceptAdminOwnership` is gated by an `onlyAdmin` modifier. The only address that can call `acceptAdminOwnership` is thus, the current admin

## Impact Details

The issue essentially breaks the 2-step ownership transfer system leaving the process vulnerable to errors. If the `pendingAdmin` is set to a wrong address and `acceptAdminOwnership` is executed then the whole MYT management can break.

## References

<https://github.com/alchemix-finance/v3-poc/blob/a192ab313c81ba3ab621d9ca1ee000110fbdd1e9/src/AlchemistCurator.sol#L31>

## Proof of Concept

## Proof of Concept

1. Add this test to `AlchemistCurator.t.sol`. Import console at the top of the file

```
import "forge-std/console.sol";
```

```
    function test_poc_wrong_2_step_ownership_transfer() public {
        address newAdmin = makeAddr("newAdmin");

        vm.startPrank(admin);
        console.log("Transferring ownership to new admin: ", newAdmin);
        mytCuratorProxy.transferAdminOwnerShip(newAdmin);
        vm.stopPrank();

        console.log("Accepting ownership by new admin: ", newAdmin, "expecting revert...");
        vm.startPrank(newAdmin);
        vm.expectRevert(abi.encode("PD"));
        mytCuratorProxy.acceptAdminOwnership();
        vm.stopPrank();
    }
```

2. Observe the logs

```
[PASS] test_poc_wrong_2_step_ownership_transfer() (gas: 52202)
Logs:
  Transferring ownership to new admin:  0x67ED0Ac45b537A79406E01656d90659325455585
  Accepting ownership by new admin:  0x67ED0Ac45b537A79406E01656d90659325455585 expecting revert...
```
