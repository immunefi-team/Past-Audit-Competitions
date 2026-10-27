# 69936 sc low users cannot revoke migration permits once the migrator s role has been revoked

**Submitted on Mar 17th 2026 at 13:23:17 UTC by @Happy\_Hunter for** [**Audit Comp | Folks Finance: Staking Contracts**](https://immunefi.com/audit-competition/audit-comp-folks-finance-staking-contracts)

* **Report ID:** #69936
* **Report Type:** Smart Contract
* **Report severity:** Low
* **Target:** <https://github.com/Folks-Finance/folks-staking-contracts/blob/main/src/Staking.sol>
* **Impacts:**
  * Contract fails to deliver promised returns, but doesn't lose value

## Description

### Brief/Intro

The function only allows changing a migration permit when `_migrator` currently has `MIGRATOR_ROLE`. Once an admin revokes that role from a migrator, users who had approved them can no longer revoke the approval: `setMigrationPermit(_migrator, false)` reverts with `MigratorNotFound`.

### Vulnerability Details

```solidity
function setMigrationPermit(address _migrator, bool _isMigrationPermitted) external {
    if (!hasRole(MIGRATOR_ROLE, _migrator)) revert MigratorNotFound(_migrator);
    migrationPermits[_migrator][msg.sender] = _isMigrationPermitted;
    emit MigrationPermitUpdated(_migrator, msg.sender, _isMigrationPermitted);
}
```

When a user had approved a migrator that held `MIGRATOR_ROLE` and the admin later revokes the role, the user's attempt to revoke their own permit (`setMigrationPermit(_migrator, false)`) reverts with `MigratorNotFound`. The storage entry `migrationPermits[_migrator][user] = true` is never cleared.

No other function modifies `migrationPermits`; the user has no alternative way to revoke the permit once the migrator has lost the role.

**Affected code (root cause):**

```solidity
if (!hasRole(MIGRATOR_ROLE, _migrator)) revert MigratorNotFound(_migrator);
```

### Impact Details

Contract fails to deliver promised returns, but doesn't lose value.

### References

* `src/Staking.sol`

### Recommendation

Require `_migrator` to have `MIGRATOR_ROLE` only when the caller is **setting** a permit to `true`; when clearing a permit (`false`), skip the role check so users can revoke for any address.

```solidity
if (_isMigrationPermitted) {
    if (!hasRole(MIGRATOR_ROLE, _migrator)) revert MigratorNotFound(_migrator);
}
```

## Proof of Concept

### PoC Code

Add the following tests to `folks-staking-contracts/test/Staking.t.sol` (same setup as existing tests: `admin`, `alice`, `migrator`, `token`, `staking` with `MIGRATOR_ROLE` granted to `migrator` in `setUp()`):

```solidity
function test_PoC_UserCannotRevokePermitAfterMigratorRoleRevoked() public {
    vm.prank(alice);
    staking.setMigrationPermit(migrator, true);
    assertTrue(staking.migrationPermits(migrator, alice), "permit should be true");

    vm.startPrank(admin);
    staking.revokeRole(staking.MIGRATOR_ROLE(), migrator);
    vm.stopPrank();
    assertFalse(staking.hasRole(staking.MIGRATOR_ROLE(), migrator), "migrator should have no role");

    vm.expectRevert(abi.encodeWithSelector(IStakingV1.MigratorNotFound.selector, migrator));
    vm.prank(alice);
    staking.setMigrationPermit(migrator, false);

    assertTrue(staking.migrationPermits(migrator, alice), "stale permit still true, user cannot revoke");
}
```

### Test Result

```
Compiling 1 files with Solc 0.8.30
Solc 0.8.30 finished in 6.32s
Compiler run successful!

Ran 1 test for test/Staking.t.sol:StakingTest
[PASS] test_PoC_UserCannotRevokePermitAfterMigratorRoleRevoked() (gas: 82972)
Suite result: ok. 1 passed; 0 failed; 0 skipped; finished in 6.54ms (2.23ms CPU time)

Ran 1 test suite in 14.54ms (6.54ms CPU time): 1 test passed, 0 failed, 0 skipped (1 total tests)
```
