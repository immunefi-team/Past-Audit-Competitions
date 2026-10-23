# 69747 sc low broken migration permit revocation allows stale user consent to reactivate after migrator role is re granted

**Submitted on Mar 16th 2026 at 16:25:08 UTC by @SamuilKosev778857 for** [**Audit Comp | Folks Finance: Staking Contracts**](https://immunefi.com/audit-competition/audit-comp-folks-finance-staking-contracts)

* **Report ID:** #69747
* **Report Type:** Smart Contract
* **Report severity:** Low
* **Target:** <https://github.com/Folks-Finance/folks-staking-contracts/blob/main/src/Staking.sol>
* **Impacts:**
  * Contract fails to deliver promised returns, but doesn't lose value

## Description

### Brief/Intro

The migration authorization flow contains a revocation flaw in `setMigrationPermit()`. A user can approve a migrator while it has `MIGRATOR_ROLE`, but if that role is later removed, the user can no longer revoke the previously granted approval because the function still requires the target address to currently hold `MIGRATOR_ROLE` even when the user is setting the permit to false.

### Vulnerability Details

The issue is in the permit-setting logic:

```solidity
function setMigrationPermit(address _migrator, bool _isMigrationPermitted) external {
    if (!hasRole(MIGRATOR_ROLE, _migrator)) revert MigratorNotFound(_migrator);

    migrationPermits[_migrator][msg.sender] = _isMigrationPermitted;
    emit MigrationPermitUpdated(_migrator, msg.sender, _isMigrationPermitted);
}
```

The problem is that the role check is unconditional. It is applied both when the user is granting permission and when the user is trying to revoke permission.

### Impact Details

Users are given a per-migrator approval mechanism via `setMigrationPermit()`, which implies they can allow or deny migration authority for their own positions.

The practical consequences are:

* users may believe they have revoked migration permission when they have not,
* stale approvals can persist indefinitely,
* migration authority can return later without fresh user consent.

### References

<https://github.com/Folks-Finance/folks-staking-contracts/blob/3131a2d46b5afa76f606bf08adfd85452a47e2d8/src/Staking.sol#L77>

## Proof of Concept

```solidity
function test_Migration_StalePermitCannotBeRevokedWhenRoleRemoved() public { 
    deal(address(token), address(staking), 1000 ether);
    deal(address(token), bob, 100 ether);

    uint8 periodIndex = addStakingPeriodByManager(50 ether, 20, 10, 5000, true);
    approveAndStake(bob, periodIndex, 10 ether, 20, 10, 5000, address(0));
    assertEq(staking.getUserStakes(bob).length, 1);

    // Bob opts in to this migrator
    vm.prank(bob);
    staking.setMigrationPermit(migrator, true);
    assertEq(staking.migrationPermits(migrator, bob), true);

    bytes32 migratorRole = staking.MIGRATOR_ROLE();

    // Migrator loses the role
    vm.prank(admin);
    staking.revokeRole(migratorRole, migrator);
    assertEq(staking.hasRole(migratorRole, migrator), false);

    // Bob now tries to revoke the permit, but cannot because the target no longer has MIGRATOR_ROLE
    vm.expectRevert(abi.encodeWithSelector(IStakingV1.MigratorNotFound.selector, migrator));
    vm.prank(bob);
    staking.setMigrationPermit(migrator, false);

    // approval remains stored
    assertEq(staking.migrationPermits(migrator, bob), true);

    // the same address regains MIGRATOR_ROLE
    vm.prank(admin);
    staking.grantRole(migratorRole, migrator);
    assertEq(staking.hasRole(migratorRole, migrator), true);

    // old approval becomes usable again
    assertEq(staking.migrationPermits(migrator, bob), true);

    vm.prank(migrator);
    IStakingV1.UserStake[] memory migratedStakes = staking.migratePositionsFrom(bob);

    assertEq(migratedStakes.length, 1);
    assertEq(migratedStakes[0].amount, 10 ether);
    assertEq(staking.getUserStakes(bob).length, 0);

    // Migrator directly received Bob's remaining principal + reward
    assertEq(token.balanceOf(migrator), migratedStakes[0].amount + migratedStakes[0].reward);
}
```
