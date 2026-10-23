# 69650 sc low setmigrationpermit blocks revocation after role revoke enabling stale consent reuse

**Submitted on Mar 16th 2026 at 05:37:24 UTC by @oxKrodhan for** [**Audit Comp | Folks Finance: Staking Contracts**](https://immunefi.com/audit-competition/audit-comp-folks-finance-staking-contracts)

* **Report ID:** #69650
* **Report Type:** Smart Contract
* **Report severity:** Low
* **Target:** <https://github.com/Folks-Finance/folks-staking-contracts/blob/main/src/Staking.sol>
* **Impacts:**
  * Griefing (e.g. no profit motive for an attacker, but damage to the users or the protocol)

## Description

## Brief/Intro

The contract binds permit updates to current role membership in a way that blocks user revocation during certain admin state transitions. This creates a consent lifecycle flaw, stale migration approvals can revive after role re grant.

## Vulnerability Details

The issue is caused by how `setMigrationPermit` enforces the migrator role check before allowing any permit update. In Staking contract, a user can only write `migrationPermits[migrator][user]` if that migrator currently has `MIGRATOR_ROLE`. This creates a state lock condition, a user may successfully set permission to true while the migrator is authorized, but if governance later revokes that role, the same user can no longer call `setMigrationPermit(migrator, false)` because the call reverts with MigratorNotFound. As a result, the stored approval remains true in state with no user side revocation path during the revoked period. If the same migrator address is granted `MIGRATOR_ROLE` again in the future, the old approval becomes effective again, and `migratePositionsFrom` can proceed because it checks role authorization and the existing boolean permit. In short, revocation capability is coupled to current role status, which allows stale migration consent to persist and later reactivate without fresh user intent.

## Impact Details

* User cannot revoke prior migration consent after role revocation event.
* Governance/operator can later re enable the same migrator address and stale approvals become effective without a fresh opt in.
* This weakens user control guarantees and can cause unexpected migration execution against user expectations.

## References

<https://github.com/Folks-Finance/folks-staking-contracts/blob/3131a2d46b5afa76f606bf08adfd85452a47e2d8/src/Staking.sol#L77>

## Steps to reproduce scenario

1. Deploy staking and grant MIGRATOR\_ROLE to address M.
2. User Alice opens a stake position.
3. Alice calls setMigrationPermit(M, true).
4. Admin revokes MIGRATOR\_ROLE from M.
5. Alice attempts setMigrationPermit(M, false).
6. Transaction reverts with MigratorNotFound, so revocation fails.
7. Admin grants MIGRATOR\_ROLE to M again.
8. M calls migratePositionsFrom(Alice) successfully.
9. Result, migration uses old approval even though Alice attempted to revoke.

## Proof of Concept

```solidity
function test_PoC_MigrationPermitReactivatesAfterRoleRegrant() public {
    deal(address(token), address(staking), 1000 ether);
    deal(address(token), alice, 100 ether);
    bytes32 migratorRole = staking.MIGRATOR_ROLE();

    uint8 periodIndex = addStakingPeriodByManager(50 ether, 20, 10, 5000, true);
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
    assertEq(staking.getUserStakes(alice).length, 0);
}
```

Add above test to the `Staking.t.sol` and run the test.
