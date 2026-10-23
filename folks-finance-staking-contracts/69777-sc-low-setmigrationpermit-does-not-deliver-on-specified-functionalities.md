# 69777 sc low setmigrationpermit does not deliver on specified functionalities&#x20;

**Submitted on Mar 16th 2026 at 19:24:43 UTC by @max10afternoon for** [**Audit Comp | Folks Finance: Staking Contracts**](https://immunefi.com/audit-competition/audit-comp-folks-finance-staking-contracts)

* **Report ID:** #69777
* **Report Type:** Smart Contract
* **Report severity:** Low
* **Target:** <https://github.com/Folks-Finance/folks-staking-contracts/blob/main/src/Staking.sol>
* **Impacts:**
  * Contract fails to deliver promised returns, but doesn't lose value

## Description

### Brief/Intro

In the documentation for mitigation process it is stated that: "The permission can be revoked at any time by calling setMigrationPermit(migratorAddress, false)" (<https://github.com/Folks-Finance/folks-staking-contracts/tree/main?tab=readme-ov-file#migration>). This is not the case as it will not always be possible to do so.

### Vulnerability Details

The setMigrationPermit allows a user to permit or revoke permission to a migration contract to call migratePositionsFrom:

```solidity
function setMigrationPermit(address _migrator, bool _isMigrationPermitted) external {
    if (!hasRole(MIGRATOR_ROLE, _migrator)) revert MigratorNotFound(_migrator);

    migrationPermits[_migrator][msg.sender] = _isMigrationPermitted;
    emit MigrationPermitUpdated(_migrator, msg.sender, _isMigrationPermitted);
}
```

In order to interact with this functionality it is necessary for the targeted address to have the migrator role, even when trying to revoke permissions.

This means that if the role were to be revoked to a migrator address, the user will no longer be able to revoke any previously assigned permission for it. Hence if the migrator contract will be reintroduced at a later time the migration can be performed even as the user tried to prevent it (unsuccessfully due this issue), breaking the trust assumption specified in the docs, expecially considering that the migration MIGHT be a permissionless process beside the checks performed in the Staking contract, as per documentation.

**Mitigation**

This can easily be fixed by allowing to modify the state of any element in migrationPermits for as long as \_isMigrationPermitted is set to false. Mitigating the issue without introducing any additional risks.

## Impact Details

Low - Contract fails to deliver promised returns, but doesn't lose value:

This issue matches the description of this severity categorization as it is expected for a user to be able to revoke such permission at ANY time, but the function does not allows to do so ( "The permission can be revoked at any time by calling setMigrationPermit(migratorAddress, false)" <https://github.com/Folks-Finance/folks-staking-contracts/tree/main?tab=readme-ov-file#migration> ).

## Proof of Concept

Add the following test case in the test file `/test/Staking.t.sol`:

```solidity
function test_Migration_RevokeFailure() public {
    deal(address(token), address(staking), 1000 ether);
    deal(address(token), alice, 100 ether);

    uint8 periodIndex = addStakingPeriodByManager(50 ether, 20, 10, 5000, true);
    approveAndStake(alice, periodIndex, 10 ether, 20, 10, 5000, address(0));

    //Alices allows migrations
    vm.prank(alice);
    staking.setMigrationPermit(migrator, true);
    assertEq(staking.getUserStakes(alice).length, 1);

    //Admin revoke migrator
    vm.prank(admin);
    staking.revokeRole(keccak256("MIGRATOR"), migrator);

    //Alice can no longer revoke pemrit
    vm.expectRevert(abi.encodeWithSelector(IStakingV1.MigratorNotFound.selector, migrator));
    vm.prank(alice);
    staking.setMigrationPermit(migrator, false);
    assertEq(staking.getUserStakes(alice).length, 1);

    //If at a later time amidn reintroduces the same migrator
    //The migrations will be executed even as Alice tried to prevent it
    vm.prank(admin);
    staking.grantRole(keccak256("MIGRATOR"), migrator);

    vm.expectEmit(true, true, true, true);
    emit IStakingV1.MigrateFrom(migrator, alice);
    vm.prank(migrator);
    IStakingV1.UserStake[] memory migratedStakes = staking.migratePositionsFrom(alice);
    assertEq(migratedStakes.length, 1);
    assertEq(migratedStakes[0].amount, 10 ether);
    assertEq(staking.getUserStakes(alice).length, 0);
}
```
