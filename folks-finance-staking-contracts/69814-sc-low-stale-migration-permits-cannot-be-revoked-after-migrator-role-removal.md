# 69814 sc low stale migration permits cannot be revoked after migrator role removal

**Submitted on Mar 16th 2026 at 23:32:46 UTC by @incogknito for** [**Audit Comp | Folks Finance: Staking Contracts**](https://immunefi.com/audit-competition/audit-comp-folks-finance-staking-contracts)

* **Report ID:** #69814
* **Report Type:** Smart Contract
* **Report severity:** Low
* **Target:** <https://github.com/Folks-Finance/folks-staking-contracts/blob/main/src/Staking.sol>
* **Impacts:**
  * Contract fails to deliver promised returns, but doesn't lose value

## Description

### Finding

The README says migration permission can be revoked at any time, but that is not true in Staking.sol: <https://github.com/Folks-Finance/folks-staking-contracts/blob/3131a2d46b5afa76f606bf08adfd85452a47e2d8/src/Staking.sol#L77-L82>

`setMigrationPermit()` requires the target address to currently hold `MIGRATOR_ROLE`. So if a user approves a migrator and that role is later revoked, calling `setMigrationPermit(migrator, false)` reverts with `MigratorNotFound`, leaving the old permit stored in `migrationPermits`.

If the same address receives `MIGRATOR_ROLE` again later, the old approval becomes usable again and `migratePositionsFrom()` can be called without any fresh user action.

### Impact

Users cannot revoke a previously granted migration permit once the role is removed. The old permit remains onchain. Re-granting `MIGRATOR_ROLE` makes the stale approval usable again.

### Recommendation

Allow `setMigrationPermit(migrator, false)` even if migrator no longer has `MIGRATOR_ROLE`.

## Proof of Concept

```solidity
function test_PoC_Info_MigrationPermitReactivatesAfterRoleRegrant() public {
    vm.prank(bob);
    staking.setMigrationPermit(migrator, true);
    assertEq(staking.migrationPermits(migrator, bob), true);

    vm.prank(admin);
    staking.revokeRole(staking.MIGRATOR_ROLE(), migrator);

    vm.expectRevert(abi.encodeWithSelector(IStakingV1.MigratorNotFound.selector, migrator));
    vm.prank(bob);
    staking.setMigrationPermit(migrator, false);

    assertEq(staking.migrationPermits(migrator, bob), true);

    vm.prank(admin);
    staking.grantRole(staking.MIGRATOR_ROLE(), migrator);

    vm.expectEmit(true, true, true, true);
    emit IStakingV1.MigrateFrom(migrator, bob);

    vm.prank(migrator);
    IStakingV1.UserStake[] memory migratedStakes = staking.migratePositionsFrom(bob);

    assertEq(migratedStakes.length, 0);
}
```

`forge test --match-test test_PoC_Info_MigrationPermitReactivatesAfterRoleRegrant -vv`
