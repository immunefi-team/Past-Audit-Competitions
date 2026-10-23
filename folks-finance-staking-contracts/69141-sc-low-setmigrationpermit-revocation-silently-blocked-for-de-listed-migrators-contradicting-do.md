# 69141 sc low setmigrationpermit revocation silently blocked for de listed migrators contradicting documented guarantee

**Submitted on Mar 13th 2026 at 02:52:01 UTC by @VinayVig for** [**Audit Comp | Folks Finance: Staking Contracts**](https://immunefi.com/audit-competition/audit-comp-folks-finance-staking-contracts)

* **Report ID:** #69141
* **Report Type:** Smart Contract
* **Report severity:** Low
* **Target:** <https://github.com/Folks-Finance/folks-staking-contracts/blob/main/src/Staking.sol>

## Description

## Brief/Intro

The README explicitly guarantees that migration permission *"can be revoked at any time",* but the implementation of `setMigrationPermit` unconditionally checks that the target address still holds `MIGRATOR_ROLE` before allowing any update — including revocations. If a migrator's role is revoked by the admin, users holding a permit for that address lose their ability to revoke it, directly violating the protocol's own documented invariant. When combined with the fact that permits are never cleared after migration completes, a revoked-then-re-granted migrator can silently re-inherit stale permits from users who had no mechanism to clean up after the original role removal.

## Vulnerability Details

The README states under the Migration User Flow section:

*"The permission can be revoked at any time by calling setMigrationPermit(migratorAddress, false)."*

The implementation does not fulfill this guarantee:

```solidity
// Staking.sol:77-82
function setMigrationPermit(address _migrator, bool _isMigrationPermitted) external {
    if (!hasRole(MIGRATOR_ROLE, _migrator)) revert MigratorNotFound(_migrator);
    migrationPermits[_migrator][msg.sender] = _isMigrationPermitted;
    emit MigrationPermitUpdated(_migrator, msg.sender, _isMigrationPermitted);
}
```

The role check runs unconditionally regardless of whether `_isMigrationPermitted` is `true` or `false`. When a migrator's `MIGRATOR_ROLE` is revoked — for example, due to compromise, decommission, or a protocol upgrade — any user who previously granted that migrator a permit cannot call `setMigrationPermit(migrator, false)`. The call reverts with `MigratorNotFound`.

<details>

<summary>Compound scenario that exposes the real risk</summary>

User calls setMigrationPermit(migrator, true) — permit grantedMigrator calls migratePositionsFrom(user) — positions migrated, permit remains true (never cleared)Admin revokes MIGRATOR\_ROLE from migrator — migrator decommissionedUser tries setMigrationPermit(migrator, false) — reverts, permit stuck at trueAdmin re-grants MIGRATOR\_ROLE to same address — permit silently reactivates with no user action

At step 5, the migrator can immediately call `migratePositionsFrom(user)` on any new stakes the user has made since the original migration, without requiring fresh consent.

</details>

## Impact Details

No direct fund loss occurs — migrated tokens go to V2, not an attacker. However, the impact is a violation of the protocol's core user consent guarantee:

* Users cannot exercise the revocation right that the documentation explicitly promises them
* Stale permits from decommissioned migrators accumulate permanently with no in-protocol cleanup path
* A migrator that is revoked and later re-granted the role inherits all historical permits without user re-consent, silently bypassing the opt-in mechanism the protocol is designed around

The README's Key Properties section states: *"No migration can happen without the user's active approval."* The combination of missing permit cleanup and blocked revocation means this property can be violated in a realistic admin key rotation or migrator upgrade scenario.

## Recommended fix

Allow revocation regardless of current role status by gating the role check only on grants:

```diff
function setMigrationPermit(address _migrator, bool _isMigrationPermitted) external {
+  if (_isMigrationPermitted && !hasRole(MIGRATOR_ROLE, _migrator)) {
        revert MigratorNotFound(_migrator);
    }
    migrationPermits[_migrator][msg.sender] = _isMigrationPermitted;
    emit MigrationPermitUpdated(_migrator, msg.sender, _isMigrationPermitted);
}
```

## References

<https://github.com/Folks-Finance/folks-staking-contracts/blob/main/src/Staking.sol#L77-L82>

## Proof of Concept

Add this test in `test/Staking.t.sol`

```solidity
// =========================================================================
    // POC: Permit Revocation Blocked After Migrator Role Removal
    //
    // Root cause: setMigrationPermit() checks hasRole(MIGRATOR_ROLE, _migrator)
    //             unconditionally — even when _isMigrationPermitted = false.
    //
    // Contradiction: README states "The permission can be revoked at any time
    //                by calling setMigrationPermit(migratorAddress, false)"
    //
    // Impact: After admin revokes a migrator's role (e.g. decommission/upgrade),
    //         users with existing permits for that migrator cannot revoke them.
    //         If the role is later re-granted to the same address, stale permits
    //         silently reactivate with no user action required.
    //
    // Run with:
    //   forge test --match-test test_POC_PermitRevocationBlockedAfterRoleRemoval -vvv
    // =========================================================================
    function test_POC_PermitRevocationBlockedAfterRoleRemoval() public {
        // ── Step 1: Alice grants permit to migrator ───────────────────────────
        vm.prank(alice);
        staking.setMigrationPermit(migrator, true);
        assertEq(staking.migrationPermits(migrator, alice), true);

        // ── Step 2: Admin revokes MIGRATOR_ROLE (e.g. migrator decommissioned) ─
        // Note: read role constant before vm.prank — contract calls consume the prank
        bytes32 MIGRATOR_ROLE = staking.MIGRATOR_ROLE();
        vm.prank(admin);
        staking.revokeRole(MIGRATOR_ROLE, migrator);
        assertEq(staking.hasRole(MIGRATOR_ROLE, migrator), false);

        // ── Step 3: Alice tries to revoke her permit — README says "at any time"
        //           but the call reverts with MigratorNotFound ──────────────────
        vm.prank(alice);
        vm.expectRevert(abi.encodeWithSelector(IStakingV1.MigratorNotFound.selector, migrator));
        staking.setMigrationPermit(migrator, false);

        // Permit is permanently stuck at true — Alice cannot clean it up
        assertEq(staking.migrationPermits(migrator, alice), true);

        // ── Step 4: Admin re-grants role to same migrator address ─────────────
        vm.prank(admin);
        staking.grantRole(MIGRATOR_ROLE, migrator);

        // Stale permit immediately reactivates — migrator can migrate Alice's
        // positions without any new consent from her
        assertEq(staking.migrationPermits(migrator, alice), true);
        assertEq(staking.hasRole(MIGRATOR_ROLE, migrator), true);
    }
```

### Results

```bash
Ran 1 test for test/Staking.t.sol:StakingTest
[PASS] test_POC_PermitRevocationBlockedAfterRoleRemoval() (gas: 75875)
Suite result: ok. 1 passed; 0 failed; 0 skipped; finished in 6.19ms (511.42µs CPU time)

Ran 1 test suite in 41.08ms (6.19ms CPU time): 1 tests passed, 0 failed, 0 skipped (1 total tests)
```
