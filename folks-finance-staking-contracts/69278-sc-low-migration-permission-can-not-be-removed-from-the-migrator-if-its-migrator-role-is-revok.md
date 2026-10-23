# 69278 sc low migration permission can not be removed from the migrator if its migrator role is revoked in advance

**Submitted on Mar 13th 2026 at 22:41:09 UTC by @piken for** [**Audit Comp | Folks Finance: Staking Contracts**](https://immunefi.com/audit-competition/audit-comp-folks-finance-staking-contracts)

* **Report ID:** #69278
* **Report Type:** Smart Contract
* **Report severity:** Low
* **Target:** <https://github.com/Folks-Finance/folks-staking-contracts/blob/main/src/Staking.sol>
* **Impacts:**
  * Contract fails to deliver promised returns, but doesn't lose value

## Description

### Brief/Intro

Once `MIGRATOR_ROLE` is revoked from `migrator`, any previous granted migration permssion can not be removed by position holder.

### Vulnerability Details

When a `migrator` is granted `MIGRATOR_ROLE`, any position holder can call `setMigrationPermit()` to grant migration permission to it.

`README.MD` stated that **The permission can be revoked at any time**.\
<https://github.com/Folks-Finance/folks-staking-contracts/tree/main?tab=readme-ov-file#user-flow>:

> The migrator must hold the `MIGRATOR_ROLE` in the staking contract. The permission can be revoked at any time by calling setMigrationPermit(migratorAddress, false).

However, if `MIGRATOR_ROLE` is revoked from a migrator, position holders are unable to remove migration permissions previously granted to that address.

### Impact Details

Poition holder might change their idea about position migrating but their can only withdraw their request only when the designated `migrator` is granted `MIGRATOR_ROLE` again. And the migration could happen immediately once the migrator is granted `MIGRATOR_ROLE` again. Most of position holders might not be noticed in time.

### References

<https://github.com/Folks-Finance/folks-staking-contracts/blob/main/src/Staking.sol#L77-L82>

### Mitigation

Do not check if `migrator` is granted `MIGRATOR_ROLE` when removing permission:

```diff
function setMigrationPermit(address _migrator, bool _isMigrationPermitted) external {
-   if (!hasRole(MIGRATOR_ROLE, _migrator)) revert MigratorNotFound(_migrator);
+   if (!hasRole(MIGRATOR_ROLE, _migrator) && _isMigrationPermitted) revert MigratorNotFound(_migrator);

    migrationPermits[_migrator][msg.sender] = _isMigrationPermitted;
    emit MigrationPermitUpdated(_migrator, msg.sender, _isMigrationPermitted);
}
```

## Proof of Concept

Copy below codes into <https://github.com/Folks-Finance/folks-staking-contracts/blob/main/test/Staking.t.sol> and run `forge test --match-test test_Migration_SetMigrationPermitRevert`

```solidity
function test_Migration_SetMigrationPermitRevert() public {
    assertEq(staking.migrationPermits(migrator, alice), false);

    vm.prank(alice);
    staking.setMigrationPermit(migrator, true);
    assertEq(staking.migrationPermits(migrator, alice), true);
    
    vm.prank(admin);
    staking.revokeRole(keccak256("MIGRATOR"), migrator);

    vm.expectRevert(abi.encodeWithSelector(IStakingV1.MigratorNotFound.selector, migrator));
    vm.prank(alice);
    staking.setMigrationPermit(migrator, false);
}
```

As demonstrated in above codes, alice can not revoke migration permssion.
