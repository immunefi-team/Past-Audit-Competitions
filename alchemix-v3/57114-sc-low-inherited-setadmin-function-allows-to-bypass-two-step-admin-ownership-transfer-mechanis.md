# 57114 sc low inherited setadmin function allows to bypass two step admin ownership transfer mechanism

**Submitted on Oct 23rd 2025 at 15:53:46 UTC by @ByteKnight for** [**Audit Comp | Alchemix V3**](https://immunefi.com/audit-competition/alchemix-v3-audit-competition)

* **Report ID:** #57114
* **Report Type:** Smart Contract
* **Report severity:** Low
* **Target:** <https://github.com/alchemix-finance/v3-poc/blob/immunefi\\_audit/src/AlchemistCurator.sol>
* **Impacts:**
  * Contract fails to deliver promised returns, but doesn't lose value

## Description

## Brief/Intro

The **AlchemistCurator** contract implements a two-step admin ownership transfer mechanism via the `transferAdminOwnerShip()` and `acceptAdminOwnership()` functions. However, since **AlchemistCurator** inherits from **PermissionedProxy**, it also inherits the `setAdmin()` function at **PermissionedProxy**, which allows the current admin to directly change the admin address in a single transaction, completely bypassing the intended two-step transfer security mechanism.

## Vulnerability Details

The **AlchemistCurator** contract tries to implement a secure two-step ownership transfer mechanism:

```sol
function transferAdminOwnerShip(address _newAdmin) external onlyAdmin {
    pendingAdmin = _newAdmin;
}

function acceptAdminOwnership() external onlyAdmin {
    admin = pendingAdmin;
    pendingAdmin = address(0);
    emit AdminChanged(admin);
}
```

However, **AlchemistCurator** inherits from **PermissionedProxy**, which provides a direct single-step admin change function:

```sol
function setAdmin(address _admin) external onlyAdmin {
    require(_admin != address(0), "zero");
    admin = _admin;
    emit AdminUpdated(_admin);
}
```

This inheritance creates two parallel admin transfer mechanisms:

1. Two-step mechanism (desired): transferAdminOwnerShip() -> acceptAdminOwnership()
2. Single-step bypass (inherited): setAdmin()

The presence of the `setAdmin()` completely undermines the security benefits of the two-step mechanism, as this function can be used intentionally or accidentally to instantly transfer ownership to any address without requiring confirmation from the new admin.

**NOTE: This finding is related to but distinct from the&#x20;*****Report ID: 56911*****, which identifies that the two-step pattern itself is incorrectly implemented (using onlyAdmin instead of checking pendingAdmin). This finding demonstrates that even if that issue were fixed, the inherited setAdmin() function would still allow bypassing the two-step mechanism entirely.**

## Impact Details

Having both a two-step and single-step mechanism creates confusion about which pattern should be used and can lead to instant transfer of a control to an incorrect address (due to typo, copy-paste error, or sending to a contract that cannot manage the protocol), which could lead to the contract being inaccessible.

## References

* Two-step transfer mechanism: <https://github.com/alchemix-finance/v3-poc/blob/a192ab313c81ba3ab621d9ca1ee000110fbdd1e9/src/AlchemistCurator.sol#L27-L35>
* Single-step transfer mechanism: <https://github.com/alchemix-finance/v3-poc/blob/a192ab313c81ba3ab621d9ca1ee000110fbdd1e9/src/utils/PermissionedProxy.sol#L31-L34>
* Related finding: Report 56911

## Proof of Concept

## Proof of Concept

To verify the flaw, do the following steps:

1. In the **PermissionedProxy** contract change the visibility of the `admin` state variable to public. We only need this to be able to log admin address in the terminal with console2.log();

```sol
    address public admin;
```

2. Add the following test to existing test suite in the **AlchemistCurator.t.sol** file:

```sol
function testTwoStepAdminOwnershipCanBeBypassed() public {
        // First, we check who is the current admin in the Alchemist Curator contract
        address currentAdmin = mytCuratorProxy.admin();
        console2.log("Current admin: ", currentAdmin);

        // Next, admin can accidentally or intentionally call the SetAdmin function from PermissionedProxy
        // Thus, bypassing the two-step ownership transfer mechanism implemented in the Alchemist Curator contract with transferAdminOwnerShip and acceptAdminOwnership functions
        address newAdmin = makeAddr("newAdmin");
        console2.log("New admin address: ", newAdmin);

        vm.prank(admin);
        mytCuratorProxy.setAdmin(newAdmin);

        // Lastly, we check that admin was set to a new admin address without him explicitly accepting it. This may lead to a constant ownership loss
        currentAdmin = mytCuratorProxy.admin();
        console2.log("Updated admin: ", currentAdmin);
    }
```

3. To run the test, run the following script in terminal and paste your fork url:

```
FOUNDRY_PROFILE=default forge test --fork-url <YOUR_FORK_URL> --mt testTwoStepAdminOwnershipCanBeBypassed  -vvvv --evm-version cancun
```
