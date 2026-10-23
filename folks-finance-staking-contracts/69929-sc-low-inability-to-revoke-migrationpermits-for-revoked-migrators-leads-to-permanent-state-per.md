# 69929 sc low inability to revoke migrationpermits for revoked migrators leads to permanent state persistence of user approvals

**Submitted on Mar 17th 2026 at 12:41:58 UTC by @BBHGuild for** [**Audit Comp | Folks Finance: Staking Contracts**](https://immunefi.com/audit-competition/audit-comp-folks-finance-staking-contracts)

* **Report ID:** #69929
* **Report Type:** Smart Contract
* **Report severity:** Low
* **Target:** <https://github.com/Folks-Finance/folks-staking-contracts/blob/main/src/Staking.sol>
* **Impacts:**
  * Contract fails to deliver promised returns, but doesn't lose value
  * Improper State Persistance / Loss of User Control over Approvals

## Description

### Brief / Intro

The `Staking.sol` contract implements a permission system via `setMigrationPermit` to allow users to opt into a migration process. A logic flaw in the function's access control validation prevents users from revoking these permissions once the administrator has revoked the `MIGRATOR_ROLE` from a migrator address. This forces users into a state of permanent approval for an entity that is no longer authorized by the protocol, violating the principle of user control over their own approvals and creating a potential security risk should the migrator be re-authorized in the future.

### Vulnerability Details

The `setMigrationPermit` function enforces that the `_migrator` address must currently possess the `MIGRATOR_ROLE` regardless of whether the user is granting or revoking permission:

```solidity
function setMigrationPermit(address _migrator, bool _isMigrationPermitted) external {
    // Both granting (true) and revoking (false) revert if the migrator is not found
    if (!hasRole(MIGRATOR_ROLE, _migrator)) revert MigratorNotFound(_migrator);

    migrationPermits[_migrator][msg.sender] = _isMigrationPermitted;
    emit MigrationPermitUpdated(_migrator, msg.sender, _isMigrationPermitted);
}
```

When the protocol administrator performs an emergency action to revoke the `MIGRATOR_ROLE` from a specific address (e.g., due to a security incident or deprecation), the `hasRole(MIGRATOR_ROLE, _migrator)` check will fail for all subsequent calls. Consequently, any user who had previously opted into that migrator is now unable to call `setMigrationPermit(migrator, false)` to clear their state. The user's approval status remains permanently `true` in the contract state.

### Impact Details

**Impact Classification:** Impact not on list (State Persistence/Loss of User Control)\
**Severity:** Low

The impact is an **inability to manage user permissions**. While the lack of the `MIGRATOR_ROLE` prevents the compromised migrator from executing `migratePositionsFrom` on the current contract, the user is left with a "dangling" approval in the contract's state.

This violates the user's right to self-custody over their permissions. Should the admin ever re-grant the `MIGRATOR_ROLE` to the previously revoked address (e.g., following a false alarm or a temporary suspension), all users who were unable to revoke their permits would be immediately and silently re-exposed to that migrator, potentially allowing them to migrate user funds to a malicious V2 destination without the user's active, post-incident consent.

## References

* Staking Contract: `src/Staking.sol`
* `setMigrationPermit` logic: lines 78-83

## Proof of Concept

The following Foundry test demonstrates that once a migrator's role is revoked, a user's attempt to revoke their own permit reverts, leaving the state as `true`.

```solidity
function test_Migration_RevertWhen_RevokingPermitForRevokedMigrator() public {
    // 1. Alice grants migration permission to the active migrator
    vm.prank(alice);
    staking.setMigrationPermit(migrator, true);
    
    // Confirm the permit is active
    assertEq(staking.migrationPermits(migrator, alice), true);

    // 2. Admin revokes the MIGRATOR_ROLE from the migrator 
    vm.prank(admin);
    staking.revokeRole(keccak256("MIGRATOR"), migrator);
    
    // Confirm the role is revoked
    assertEq(staking.hasRole(keccak256("MIGRATOR"), migrator), false);

    // 3. Alice tries to proactively secure her account by revoking her permission
    vm.prank(alice);
    
    // The transaction REVERTS, blocking Alice from revoking the permit
    vm.expectRevert(abi.encodeWithSelector(IStakingV1.MigratorNotFound.selector, migrator));
    staking.setMigrationPermit(migrator, false);

    // 4. Proof of Impact: Alice's permission is permanently stuck as TRUE
    assertEq(staking.migrationPermits(migrator, alice), true);
}
```

**Steps to verify:**

1. Copy the code above into your existing `StakingTest` contract.
2. Run `forge test --mt test_Migration_RevertWhen_RevokingPermitForRevokedMigrator`.
3. The test will pass, demonstrating that the `MigratorNotFound` revert prevents the permission change.
