# 69570 sc low users cannot revoke migration approvals for removed migrators contrary to what the docs says

**Submitted on Mar 15th 2026 at 17:35:00 UTC by @volleyking1940 for** [**Audit Comp | Folks Finance: Staking Contracts**](https://immunefi.com/audit-competition/audit-comp-folks-finance-staking-contracts)

* **Report ID:** #69570
* **Report Type:** Smart Contract
* **Report severity:** Low
* **Target:** <https://github.com/Folks-Finance/folks-staking-contracts/blob/main/src/Staking.sol>
* **Impacts:**
  * Contract fails to deliver promised returns, but doesn't lose value

## Description

The README states that users may revoke a migration approval at any time by calling:

```solidity
staking.setMigrationPermit(migratorAddress, false);
```

Source:

* <https://github.com/Folks-Finance/folks-staking-contracts/blob/main/README.md#L140>

However, `setMigrationPermit()` only accepts addresses that currently hold `MIGRATOR_ROLE`:

* <https://github.com/Folks-Finance/folks-staking-contracts/blob/main/src/Staking.sol#L77-L80>

```solidity
function setMigrationPermit(address _migrator, bool _isMigrationPermitted) external {
    if (!hasRole(MIGRATOR_ROLE, _migrator)) revert MigratorNotFound(_migrator);

    migrationPermits[_migrator][msg.sender] = _isMigrationPermitted;
    emit MigrationPermitUpdated(_migrator, msg.sender, _isMigrationPermitted);
}
```

This creates a mismatch between the documented and actual behavior.

If a user approves a migrator and that address later loses `MIGRATOR_ROLE`, the user can no longer clear the approval because `setMigrationPermit(migrator, false)` will revert with `MigratorNotFound`. The approval remains stored in `migrationPermits`, and if the same address is later granted `MIGRATOR_ROLE` again, that prior approval becomes effective again automatically.

In other words, the approval becomes dormant rather than revoked.

## Impact

low.

This issue does not directly lead to asset loss on its own, since reactivation depends on privileged role management. However, it weakens the expected user-control model around migration approvals:

* users cannot revoke a previously granted approval after the migrator role is removed, despite the README stating revocation is possible "at any time";
* dormant approvals remain stored and silently reactivate if the same address later regains `MIGRATOR_ROLE`.

## Recommended Mitigation

If the intended behavior is that approvals are revocable at any time, users should be allowed to clear an existing permit even when the address no longer holds `MIGRATOR_ROLE`.

One simple approach is to:

* require `MIGRATOR_ROLE` only when setting a permit to `true`; and
* allow setting a permit to `false` whenever a stored approval already exists.

If the current implementation is intended instead, then the README should be updated to reflect that approvals can only be modified while the target address is an active migrator.

## Proof of Concept

Add the following tests to `test/Staking.t.sol`:

```solidity
function test_Migration_CannotRevokePermitAfterRoleRevocation() public {
    vm.prank(alice);
    staking.setMigrationPermit(migrator, true);
    assertEq(staking.migrationPermits(migrator, alice), true);

    vm.prank(admin);
    staking.revokeRole(staking.MIGRATOR_ROLE(), migrator);

    vm.expectRevert(abi.encodeWithSelector(IStakingV1.MigratorNotFound.selector, migrator));
    vm.prank(alice);
    staking.setMigrationPermit(migrator, false);

    assertEq(staking.migrationPermits(migrator, alice), true);
}

function test_Migration_StalePermitReactivatesAfterRoleRegrant() public {
    vm.prank(alice);
    staking.setMigrationPermit(migrator, true);

    vm.prank(admin);
    staking.revokeRole(staking.MIGRATOR_ROLE(), migrator);

    vm.prank(admin);
    staking.grantRole(staking.MIGRATOR_ROLE(), migrator);

    assertEq(staking.migrationPermits(migrator, alice), true);
}
```

Run:

```sh
forge test --match-test "test_Migration_(CannotRevokePermitAfterRoleRevocation|StalePermitReactivatesAfterRoleRegrant)"
```

Expected result:

* `test_Migration_CannotRevokePermitAfterRoleRevocation` shows that once the migrator loses `MIGRATOR_ROLE`, the user can no longer revoke the stored approval;
* `test_Migration_StalePermitReactivatesAfterRoleRegrant` shows that the old approval remains in storage and becomes effective again if the same address is re-authorized as a migrator.
