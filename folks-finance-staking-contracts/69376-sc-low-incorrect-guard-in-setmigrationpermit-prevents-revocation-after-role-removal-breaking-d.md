# 69376 sc low incorrect guard in setmigrationpermit prevents revocation after role removal breaking documented user control

**Submitted on Mar 14th 2026 at 14:33:33 UTC by @Orhuk1 for** [**Audit Comp | Folks Finance: Staking Contracts**](https://immunefi.com/audit-competition/audit-comp-folks-finance-staking-contracts)

* **Report ID:** #69376
* **Report Type:** Smart Contract
* **Report severity:** Low
* **Target:** <https://github.com/Folks-Finance/folks-staking-contracts/blob/main/src/Staking.sol>
* **Impacts:**
  * Contract fails to deliver promised returns, but doesn't lose value

## Description

The `setMigrationPermit` function applies the `MIGRATOR_ROLE` check unconditionally regardless of whether the caller is granting or revoking a permit:

```solidity
function setMigrationPermit(address _migrator, bool _isMigrationPermitted) external {
    if (!hasRole(MIGRATOR_ROLE, _migrator)) revert MigratorNotFound(_migrator);
    migrationPermits[_migrator][msg.sender] = _isMigrationPermitted;
    emit MigrationPermitUpdated(_migrator, msg.sender, _isMigrationPermitted);
}
```

When a migrator's role is revoked by the admin, any user who had previously granted that migrator a permit can no longer revoke it. The call reverts with `MigratorNotFound` even though the user is only attempting to modify their own state.

This directly contradicts the documented behavior in the README:

> *"The permission can be revoked at any time by calling setMigrationPermit(migratorAddress, false)."*

The fix is to apply the role guard only when granting, not when revoking:

```solidity
function setMigrationPermit(address _migrator, bool _isMigrationPermitted) external {
    if (_isMigrationPermitted && !hasRole(MIGRATOR_ROLE, _migrator)) {
        revert MigratorNotFound(_migrator);
    }
    migrationPermits[_migrator][msg.sender] = _isMigrationPermitted;
    emit MigrationPermitUpdated(_migrator, msg.sender, _isMigrationPermitted);
}
```

This preserves the behaviour that users cannot grant permits to non-migrators, while allowing unconditional revocation regardless of the migrator's current role status.

## Proof of Concept

**Steps:**

1. Alice grants a migration permit to `migrator` while the role is active
2. Admin revokes `MIGRATOR_ROLE` from `migrator`
3. Alice calls `setMigrationPermit(migrator, false)` to revoke her permit
4. The call reverts with `MigratorNotFound` — Alice has no on-chain remedy

```solidity
function test_revocationBlockedAfterRoleRemoved() public {
    deal(address(token), address(staking), 1000 ether);
    deal(address(token), alice, 100 ether);

    uint8 periodIndex = addStakingPeriodByManager(50 ether, 20, 10, 5000, true);
    approveAndStake(alice, periodIndex, 10 ether, 20, 10, 5000, address(0));

    // Step 1: Alice grants permit to migrator
    vm.prank(alice);
    staking.setMigrationPermit(migrator, true);
    assertTrue(staking.migrationPermits(migrator, alice));

    // Cache role before pranking to prevent prank being consumed by argument evaluation
    bytes32 migratorRole = staking.MIGRATOR_ROLE();

    // Step 2: Admin revokes migrator role
    vm.prank(admin);
    staking.revokeRole(migratorRole, migrator);
    assertFalse(staking.hasRole(migratorRole, migrator));

    // Step 3 & 4: Alice attempts to revoke her permit — reverts
    vm.expectRevert(
        abi.encodeWithSelector(IStakingV1.MigratorNotFound.selector, migrator)
    );
    vm.prank(alice);
    staking.setMigrationPermit(migrator, false);

    // Permit remains true with no on-chain remedy for Alice
    assertTrue(staking.migrationPermits(migrator, alice));
}
```

```bash
forge test --match-test test_revocationBlockedAfterRoleRemoved -vvv
```
