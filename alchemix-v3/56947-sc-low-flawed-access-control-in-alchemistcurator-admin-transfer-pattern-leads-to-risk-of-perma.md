# 56947 sc low flawed access control in alchemistcurator admin transfer pattern leads to risk of permanent loss of control

**Submitted on Oct 22nd 2025 at 03:41:12 UTC by @fullstop for** [**Audit Comp | Alchemix V3**](https://immunefi.com/audit-competition/alchemix-v3-audit-competition)

* **Report ID:** #56947
* **Report Type:** Smart Contract
* **Report severity:** Low
* **Target:** <https://github.com/alchemix-finance/v3-poc/blob/immunefi\\_audit/src/AlchemistCurator.sol>
* **Impacts:**
  * Permanent freezing of funds

## Description

## Brief/Intro

The AlchemistCurator.sol contract implements a two-step administrative transfer pattern, but the second step, acceptAdminOwnership(), is incorrectly protected by the onlyAdmin modifier. This modifier validates the current admin, not the pending admin. If the current admin nominates an incorrect address (e.g., due to a typo) and then finalizes the transfer, all administrative functions of the contract will be permanently locked, as the new "admin" will be an uncontrollable address.

## Vulnerability Details

The standard two-step ownership transfer pattern (transfer + accept) is designed to prevent a common and critical error: transferring ownership to an incorrect address. The pattern's security relies on the new owner (pendingAdmin) proving they control the address by calling the accept function.

The AlchemistCurator contract attempts to implement this pattern but fails in the acceptance step.

1. The Flawed Implementation (AlchemistCurator.sol)

The contract defines transferAdminOwnerShip and acceptAdminOwnership. Both are protected by the onlyAdmin modifier.

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

2. The Modifier (PermissionedProxy.sol)

The onlyAdmin modifier, inherited from PermissionedProxy.sol, checks if the msg.sender is the current admin.

```solidity
    modifier onlyAdmin() {
        require(msg.sender == admin, "PD");
        _;
    }
```

3. The Defect

Because acceptAdminOwnership uses onlyAdmin, it requires msg.sender == admin. This means the current admin—not the pending admin—is the only one who can call this function. This completely bypasses the security check of the two-step pattern.

4. Contrast with Correct Implementation (in the same project)

The AlchemistV3.sol contract demonstrates the correct implementation, proving this was likely an unintentional error in AlchemistCurator. AlchemistV3.sol correctly checks if the msg.sender is the pendingAdmin.

```solidity
    function acceptAdmin() external {
        _checkState(pendingAdmin != address(0));

        if (msg.sender != pendingAdmin) {
            revert Unauthorized();
        }

        admin = pendingAdmin;
        pendingAdmin = address(0);

        emit AdminUpdated(admin);
        emit PendingAdminUpdated(address(0));
    }
```

#### Impact Details

The impact of this vulnerability is the permanent loss of administrative control over the AlchemistCurator contract.

The admin of this contract is responsible for all critical management functions, including:

Adding new MYT strategies.

Removing old MYT strategies.

Managing absolute and relative deposit caps for all strategies.

#### Attack Scenario (Accidental):

The current, legitimate admin decides to transfer ownership.

They call transferAdminOwnerShip(address \_newAdmin) but make a typo, setting pendingAdmin to an incorrect address (e.g., a dead address or an address they do not control).

Not noticing the typo, the admin immediately calls acceptAdminOwnership() to "complete" the transfer.

The call succeeds because msg.sender is still the current admin.

The admin role is now permanently transferred to the incorrect address.

Consequence: The AlchemistCurator contract is now "bricked." No one can ever again manage strategy caps or update the list of strategies. This would freeze a core component of the protocol, preventing reactions to market conditions, managing risk, or adding new yield sources. This is an irreversible denial of service for the contract's management.

## Impact Details

Provide a detailed breakdown of possible losses from an exploit, especially if there are funds at risk. This illustrates the severity of the vulnerability, but it also provides the best possible case for you to be paid the correct amount. Make sure the selected impact is within the program’s list of in-scope impacts and matches the impact you selected.

## References

Vulnerable Contract: src/AlchemistCurator.sol Correct Implementation (for comparison): src/AlchemistV3.sol

## Proof of Concept

## Proof of Concept

This Foundry test can be added to src/test/AlchemistCurator.t.sol to reproduce the vulnerability. The test will pass, proving the vulnerability exists.

```solidity
    /// @notice This test reproduces the flaw in the admin transfer pattern.
    /// @dev The current admin (originalAdmin) can nominate a new admin
    /// and then *immediately accept* the transfer themselves,
    /// because `acceptAdminOwnership` is incorrectly protected by `onlyAdmin`.
    /// A correct implementation would require `msg.sender == pendingAdmin`.
    function testReproduce_FlawedAdminTransfer() public {
        address newAdmin = address(0x1234567890123456789012345678901234567890);
        address originalAdmin = admin; // As defined in setUp

        // 1. Current admin nominates a new admin
        vm.startPrank(originalAdmin);
        mytCuratorProxy.transferAdminOwnerShip(newAdmin);
        assertEq(mytCuratorProxy.pendingAdmin(), newAdmin, "pendingAdmin should be set to newAdmin");

        // 2. VULNERABILITY: The *current* admin calls `acceptAdminOwnership`.
        // This call succeeds because of the `onlyAdmin` modifier,
        // which checks if `msg.sender == admin` (which is true).
        mytCuratorProxy.acceptAdminOwnership();
        vm.stopPrank();

        // 3. Verify the transfer completed successfully
        // The pendingAdmin should be cleared
        assertEq(mytCuratorProxy.pendingAdmin(), address(0), "pendingAdmin should be cleared after acceptance");

        // 4. Verify that newAdmin is now the actual admin
        // We test this by having newAdmin successfully call an admin-only function.
        vm.startPrank(newAdmin);
        mytCuratorProxy.transferAdminOwnerShip(originalAdmin); // This should succeed
        vm.stopPrank();

        // 5. Verify that the originalAdmin is no longer the admin
        // This call should now fail with the "PD" (Permission Denied) error.
        vm.startPrank(originalAdmin);
        vm.expectRevert(bytes("PD"));
        mytCuratorProxy.transferAdminOwnerShip(newAdmin);
        vm.stopPrank();
    }
```
