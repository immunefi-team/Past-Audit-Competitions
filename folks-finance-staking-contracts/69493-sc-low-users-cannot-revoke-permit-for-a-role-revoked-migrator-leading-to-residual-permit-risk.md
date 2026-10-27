# 69493 sc low users cannot revoke permit for a role revoked migrator leading to residual permit risk if such migrator s role is ever reinstated

**Submitted on Mar 15th 2026 at 07:46:16 UTC by @Another for** [**Audit Comp | Folks Finance: Staking Contracts**](https://immunefi.com/audit-competition/audit-comp-folks-finance-staking-contracts)

* **Report ID:** #69493
* **Report Type:** Smart Contract
* **Report severity:** Low
* **Target:** <https://github.com/Folks-Finance/folks-staking-contracts/blob/main/src/Staking.sol>
* **Impacts:**
  * Contract fails to deliver promised returns, but doesn't lose value

## Description

### Brief/Intro

The `setMigrationPermit` function requires the migrator to currently hold the `MIGRATOR_ROLE` before allowing any change to the permit mapping. If an admin revokes this role from a migrator contract, users are unable to revoke their personal approvals to that migrator. This leaves stale permits in place, which become active again if the role is ever reinstated, violating the principle that users should always have full control over their own permissions.

### Vulnerability Details

In `Staking.sol`, the function `setMigrationPermit` is implemented as follows:

```solidity
function setMigrationPermit(address _migrator, bool _isMigrationPermitted) external {
    if (!hasRole(MIGRATOR_ROLE, _migrator)) revert MigratorNotFound(_migrator);
    migrationPermits[_migrator][msg.sender] = _isMigrationPermitted;
    emit MigrationPermitUpdated(_migrator, msg.sender, _isMigrationPermitted);
}
```

The check `hasRole(MIGRATOR_ROLE, _migrator)` is performed unconditionally, even when the user attempts to set `_isMigrationPermitted = false` (i.e., to revoke a previously granted permit). If the migrator’s role has been revoked, the function will always revert with `MigratorNotFound`. Consequently:

* Users cannot revoke their approval, even though the migrator is no longer authorized by the protocol.
* The stored permit remains `true` in the `migrationPermits` mapping.
* If the migrator’s role is reinstated (for any reason), the old permit becomes active again potentially without the user’s consent.

This gives users no way to remove their approval once the migrator loses its role, effectively locking their permission until the role is restored — a situation that undermines user sovereignty over their own positions.

## Impact Details

Users lose the ability to withdraw their migration approval when a migrator is revoked. If the migrator is ever reactivated, previously approved users may have their positions migrated without a fresh confirmation. While the migrator itself cannot call `migratePositionsFrom` while its role is revoked, the lingering permit creates a future risk.

## References

* `Staking.sol` – `setMigrationPermit` function - <https://github.com/Folks-Finance/folks-staking-contracts/blob/3131a2d46b5afa76f606bf08adfd85452a47e2d8/src/Staking.sol#L78>

## Proof of Concept

Add the following test to `Staking.t.sol` inside the `StakingTest` contract. It demonstrates that after the migrator’s role is revoked, Alice cannot revoke her permit, and the permit remains `true`.

```solidity
function test_Migration_UserCannotRevokePermitAfterMigratorRoleRevoked() public {
    // Alice permits migrator
    vm.prank(alice);
    staking.setMigrationPermit(migrator, true);
    assertEq(staking.migrationPermits(migrator, alice), true);

    // Admin revokes migrator role
    vm.prank(admin);
    staking.revokeRole(keccak256("MIGRATOR"), migrator);

    // Now Alice tries to revoke the permit – this should revert
    vm.expectRevert(abi.encodeWithSelector(IStakingV1.MigratorNotFound.selector, migrator));
    vm.prank(alice);
    staking.setMigrationPermit(migrator, false);

    // Permit remains true, Alice cannot revoke it
    assertEq(staking.migrationPermits(migrator, alice), true);
}
```
