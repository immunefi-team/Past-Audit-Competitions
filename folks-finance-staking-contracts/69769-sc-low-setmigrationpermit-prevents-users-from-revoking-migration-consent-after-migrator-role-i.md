# 69769 sc low setmigrationpermit prevents users from revoking migration consent after migrator role is revoked

**Submitted on Mar 16th 2026 at 18:12:11 UTC by @PotEater for** [**Audit Comp | Folks Finance: Staking Contracts**](https://immunefi.com/audit-competition/audit-comp-folks-finance-staking-contracts)

* **Report ID:** #69769
* **Report Type:** Smart Contract
* **Report severity:** Low
* **Target:** <https://github.com/Folks-Finance/folks-staking-contracts/blob/main/src/Staking.sol>
* **Impacts:**
  * Contract fails to deliver promised returns, but doesn't lose value

## Description

### Brief/Intro

The function `setMigrationPermit` checks that the migrator address holds the `MIGRATOR_ROLE` every time. This is problematic, when a migrator's role is revoked, any user who previously granted that migrator permission becomes permanently unable to revoke it.

### Vulnerability Details

In the `Staking.sol` contract, the function `setMigrationPermit` allows users to grant a migrator permit to `migrator`:

```solidity
function setMigrationPermit(address _migrator, bool _isMigrationPermitted) external {
    if (!hasRole(MIGRATOR_ROLE, _migrator)) revert MigratorNotFound(_migrator);

    migrationPermits[_migrator][msg.sender] = _isMigrationPermitted;
    emit MigrationPermitUpdated(_migrator, msg.sender, _isMigrationPermitted);
}
```

However this function applies the `hasRole` check regardless of whether the user is granting or revoking.

When the migrator role has been revoked from the migrator, the call reverts with `MigratorNotFound` error, leaving the permit stuck at `true` until the migrator gets the `MIGRATOR_ROLE` again.

The contract promises users to be able to manage their own migration permits via this function. However, this promise is broken in the following scenario:

1. User calls `setMigrationPermit(migrator, true)`, granting a permit.
2. Admin revokes migrator role.
3. User calls `setMigrationPermit(migrator, false)`, but tx reverts with `MigratorNotFound`.
4. User permit is stuck at `true` until the migrator acquires the `MIGRATOR_ROLE` again.

**NOTE:** This isnt a known issue, as the issue in the known issues section acknowledges the fact that the migrator role permits in storage:

**State “migrationPermits” may contain migrator which had its MIGRATOR\_ROLE later revoked**

This report is about users not being able to revoke the permit from the migrator and the migrator later reacquiring the `MIGRATOR_ROLE` and migrate users position without their current consent.\
This issue differs because fixing the known issue will not fix this one.

### Impact Details

Smart contract fails to deliver promised returns - The contract promises users to be able to revoke the migrator permit at any time. However, this promise is broken when the `MIGRATOR_ROLE` is revoked from the migrator. The user has no way to revoke the migrator permit and the migrator can later regain his role and migrate user position without his consent.

This can also severely disrupt users financial plans.

## References

Root cause: <https://github.com/Folks-Finance/folks-staking-contracts/blob/3131a2d46b5afa76f606bf08adfd85452a47e2d8/src/Staking.sol#L78>

## Proof of Concept

Add this function to the `Staking.t.sol` file:

For this test to run you may need to import:

```solidity
import {console} from "forge-std/console.sol";
```

Run with `forge test --match-test test_PoC -vvvv`

```solidity
function test_PoC() public {
    deal(address(token), address(staking), 1000 ether);
    deal(address(token), alice, 100 ether);

    uint8 periodIndex = addStakingPeriodByManager(50 ether, 20, 10, 5000, true);
    approveAndStake(alice, periodIndex, 10 ether, 20, 10, 5000, address(0));

    // Step 1: Alice grants migration permit to migrator
    vm.prank(alice);
    staking.setMigrationPermit(migrator, true);
    assertEq(staking.migrationPermits(migrator, alice), true);

    // Step 2: Admin revokes migrator role (routine operation, e.g. key rotation)
    vm.prank(admin);
    staking.revokeRole(keccak256("MIGRATOR"), migrator);
    assertEq(staking.hasRole(keccak256("MIGRATOR"), migrator), false);

    // Step 3: Alice tries to revoke her permit but CANNOT
    // setMigrationPermit checks hasRole and reverts with MigratorNotFound
    vm.expectRevert(abi.encodeWithSelector(IStakingV1.MigratorNotFound.selector, migrator));
    vm.prank(alice);
    staking.setMigrationPermit(migrator, false);

    // Permit is still true — Alice is stuck
    assertEq(staking.migrationPermits(migrator, alice), true);

    // Step 4: Admin re-grants migrator role (e.g. upgrade, new contract, or malicious re-grant)
    vm.prank(admin);
    staking.grantRole(keccak256("MIGRATOR"), migrator);
    assertEq(staking.hasRole(keccak256("MIGRATOR"), migrator), true);

    // Step 5: Migrator can now migrate Alice's position without her current consent
    // Alice never intended to allow this — she tried to revoke but couldn't
    vm.prank(migrator);
    IStakingV1.UserStake[] memory migratedStakes = staking.migratePositionsFrom(alice);

    // Alice's position was migrated against her wishes
    assertEq(migratedStakes.length, 1);
    assertEq(migratedStakes[0].amount, 10 ether);
    assertEq(staking.getUserStakes(alice).length, 0);

    console.log("Alice's permit stuck at: ", staking.migrationPermits(migrator, alice));
    console.log("Alice's stakes after unwanted migration: ", staking.getUserStakes(alice).length);
}
```
