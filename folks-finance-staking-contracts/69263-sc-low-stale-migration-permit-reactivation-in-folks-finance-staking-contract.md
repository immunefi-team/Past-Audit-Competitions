# 69263 sc low stale migration permit reactivation in folks finance staking contract

**Submitted on Mar 13th 2026 at 20:54:07 UTC by @jo13 for** [**Audit Comp | Folks Finance: Staking Contracts**](https://immunefi.com/audit-competition/audit-comp-folks-finance-staking-contracts)

* **Report ID:** #69263
* **Report Type:** Smart Contract
* **Report severity:** Low
* **Target:** <https://github.com/Folks-Finance/folks-staking-contracts/blob/main/src/Staking.sol>
* **Impacts:**
  * Smart contract unable to operate due to lack of token funds
  * Griefing (e.g. no profit motive for an attacker, but damage to the users or the protocol)

## Description

## Brief/Intro

The `Staking` contract allows users to grant migration permission only to addresses that currently hold `MIGRATOR_ROLE`, but it applies the same role check when users try to revoke an existing permission. This means that once a migrator loses its role, users can no longer explicitly clear the stored permit for that address. If the same address later receives `MIGRATOR_ROLE` again, the previously granted permission becomes active again automatically. In production, this can unexpectedly reactivate stale migration approvals and weaken user control over who can migrate their staking positions.

## Vulnerability Details

The issue is located in `setMigrationPermit()`:

```solidity
function setMigrationPermit(address _migrator, bool _isMigrationPermitted) external {
    if (!hasRole(MIGRATOR_ROLE, _migrator)) revert MigratorNotFound(_migrator);

    migrationPermits[_migrator][msg.sender] = _isMigrationPermitted;
    emit MigrationPermitUpdated(_migrator, msg.sender, _isMigrationPermitted);
}
```

This function uses a current-role existence check before allowing any update to the stored permission mapping. That is reasonable when the user is trying to newly authorize a migrator, but it becomes problematic when the user wants to revoke an already stored permission.

The `migrationPermits` mapping is keyed by migrator address and user address:

```solidity
mapping(address migrator => mapping(address user => bool isAuthorized)) public migrationPermits;
```

The stored value is not deleted automatically when admin revokes `MIGRATOR_ROLE`, and no cleanup is performed on role changes. As a result, the following sequence is possible:

1. A migrator `M` is granted `MIGRATOR_ROLE`.
2. A user calls `setMigrationPermit(M, true)`.
3. Admin revokes `MIGRATOR_ROLE` from `M`.
4. The user later tries to revoke the stale permit with `setMigrationPermit(M, false)`.
5. The call reverts with `MigratorNotFound(M)` because `M` no longer has the role.
6. At some later point, admin grants `MIGRATOR_ROLE` back to the same address `M`.
7. The old `migrationPermits[M][user] == true` value is still present and becomes usable again.

The migration entry point relies on both role possession and stored user permit:

```solidity
function migratePositionsFrom(address user)
    external
    nonReentrant
    onlyRole(MIGRATOR_ROLE)
    returns (UserStake[] memory)
{
    if (!migrationPermits[msg.sender][user]) revert MigratorNotPermitted(msg.sender, user);
    ...
}
```

Because the permit is persistent, re-granting the same role to the same address reactivates the old user authorization without any new user action.

This is a real permission-lifecycle flaw. The user is prevented from revoking a permission entry that already exists simply because the role status changed after the permission was created.

## Impact Details

* Users can lose practical control over old migration approvals once the migrator role is revoked.
* An address that regains `MIGRATOR_ROLE` later can reuse stale approvals that users were unable to clear.
* Users may reasonably assume that revoking a role or losing role status fully neutralizes an old migrator relationship, but the stored authorization remains in state.
* If the reauthorized migrator is malicious or compromised, it can again call `migratePositionsFrom(user)` for users who had previously approved it.

## References

* **Affected code**: `src/Staking.sol::setMigrationPermit()`
* **Role-gated revoke problem**: `src/Staking.sol:77-81`
* **Stored permit mapping**: `src/Staking.sol:38`
* **Migration entry point using stale permit**: `src/Staking.sol:166-172`
* **Related errors**: `src/interfaces/IStakingV1.sol:79-80`

## Proof of Concept

Add this function to `test/Staking.t.sol` and run `forge test --match-test test_Migration_StalePermitReactivatesWhenMigratorRoleGrantedAgain -vv`

```solidity
function test_Migration_StalePermitReactivatesWhenMigratorRoleGrantedAgain() public {
    deal(address(token), address(staking), 1000 ether);
    deal(address(token), alice, 100 ether);

    uint8 periodIndex = addStakingPeriodByManager(50 ether, 20, 10, 5000, true);
    bytes32 migratorRole = keccak256("MIGRATOR");
    approveAndStake(alice, periodIndex, 10 ether, 20, 10, 5000, address(0));

    vm.prank(alice);
    staking.setMigrationPermit(migrator, true);
    assertEq(staking.migrationPermits(migrator, alice), true);

    vm.prank(admin);
    staking.revokeRole(migratorRole, migrator);

    vm.expectRevert(abi.encodeWithSelector(IStakingV1.MigratorNotFound.selector, migrator));
    vm.prank(alice);
    staking.setMigrationPermit(migrator, false);

    assertEq(staking.migrationPermits(migrator, alice), true);

    vm.prank(admin);
    staking.grantRole(migratorRole, migrator);

    vm.prank(migrator);
    IStakingV1.UserStake[] memory migratedStakes = staking.migratePositionsFrom(alice);

    assertEq(migratedStakes.length, 1);
    assertEq(migratedStakes[0].amount, 10 ether);
    assertEq(staking.getUserStakes(alice).length, 0);
}
```
