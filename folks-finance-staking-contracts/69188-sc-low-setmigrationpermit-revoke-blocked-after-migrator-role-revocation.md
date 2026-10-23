# 69188 sc low setmigrationpermit revoke blocked after migrator role revocation

**Submitted on Mar 13th 2026 at 12:12:08 UTC by @kujen for** [**Audit Comp | Folks Finance: Staking Contracts**](https://immunefi.com/audit-competition/audit-comp-folks-finance-staking-contracts)

* **Report ID:** #69188
* **Report Type:** Smart Contract
* **Report severity:** Low
* **Target:** <https://github.com/Folks-Finance/folks-staking-contracts/blob/main/src/interfaces/IMigratorV1.sol>
* **Impacts:**
  * Griefing (e.g. no profit motive for an attacker, but damage to the users or the protocol)

## Description

## Brief/Intro

Users should always be able to revoke their migration permissions, but the current implementation blocks revocation when a migrator loses the `MIGRATOR_ROLE`. This can result in stale migration permits that reactivate automatically if the role is restored, violating user autonomy.

## Vulnerability Details

`src/Staking.sol`:

```solidity
function setMigrationPermit(address _migrator, bool _isMigrationPermitted) external {
        if (!hasRole(MIGRATOR_ROLE, _migrator)) revert MigratorNotFound(_migrator);

        migrationPermits[_migrator][msg.sender] = _isMigrationPermitted;
        emit MigrationPermitUpdated(_migrator, msg.sender, _isMigrationPermitted);
    }
```

## Impact Details

Users should always be able to revoke migration permissions. The `hasRole` guard currently blocks revocation once a migrator loses their role, leaving users permanently unable to clean up stale permits.

1. Admin revokes `MIGRATOR_ROLE` from migratorV1.
2. Users who previously granted `migrationPermits[migratorV1][user] = true` cannot clear it.
3. If the role is later re-granted, all stale permits reactivate without user consent.

Users lose control over their migration permits for deprecated migrators.

## Proof of Concept

```solidity
function test_exploit_revokePermitBlockedAfterMigratorRoleRevoked() public {
    vm.prank(alice);
    staking.setMigrationPermit(migrator, true);

    vm.prank(admin);
    staking.revokeRole(keccak256("MIGRATOR"), migrator);

    vm.prank(alice);
    vm.expectRevert(
        abi.encodeWithSelector(IStakingV1.MigratorNotFound.selector, migrator)
    );
    staking.setMigrationPermit(migrator, false);  // REVERTS — stuck at true

    assertTrue(staking.migrationPermits(migrator, alice));
}
```

Output:

```bash
$ forge test --mt test_exploit_revokePermitBlockedAfterMigratorRoleRevoked -vvv
[⠊] Compiling...
No files changed, compilation skipped

Ran 1 test for test/Staking.t.sol:StakingTest
[PASS] test_exploit_revokePermitBlockedAfterMigratorRoleRevoked() (gas: 56786)
Suite result: ok. 1 passed; 0 failed; 0 skipped; finished in 5.55ms (1.36ms CPU time)

Ran 1 test suite in 15.94ms (5.55ms CPU time): 1 tests passed, 0 failed, 0 skipped (1 total tests)
```
