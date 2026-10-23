# 69396 sc low users unable to remove migration permission from migrator who had role revoked

**Submitted on Mar 14th 2026 at 17:10:44 UTC by @gesha17 for** [**Audit Comp | Folks Finance: Staking Contracts**](https://immunefi.com/audit-competition/audit-comp-folks-finance-staking-contracts)

* **Report ID:** #69396
* **Report Type:** Smart Contract
* **Report severity:** Low
* **Target:** <https://github.com/Folks-Finance/folks-staking-contracts/blob/main/src/Staking.sol>
* **Impacts:**
  * Contract fails to deliver promised returns, but doesn't lose value

## Description

## Brief/Intro

`setMigrationPermit` let's a user to manage consent for migrating their staked positions to a new contract version. The function guards **both** granting and revoking permission with an identical role check, making it impossible for a user to revoke a previously granted permit once the migrator's `MIGRATOR_ROLE` has been revoked by an admin. The user's consent is permanently frozen at `true`. It is likely that users attempt to revoke the permission "just in case" if the migrator is being changed for a new one. The transactions will revert to the users suprise, creating unnecessary confusion and concern.

## Vulnerability Details

`setMigrationPermit` validates that the target address holds `MIGRATOR_ROLE` before writing to the `migrationPermits` mapping, regardless of whether the user is granting or revoking permission:

```js
// Staking.sol:77-82
function setMigrationPermit(address _migrator, bool _isMigrationPermitted) external {
>   if (!hasRole(MIGRATOR_ROLE, _migrator)) revert MigratorNotFound(_migrator);

    migrationPermits[_migrator][msg.sender] = _isMigrationPermitted;
    emit MigrationPermitUpdated(_migrator, msg.sender, _isMigrationPermitted);
}
```

The role check makes sense for the granting direction, however, the same check incorrectly gates the revoking direction, which is a **protective action** whose validity should be independent of the migrator's current role status.

The sequence that permanently locks a user's consent (shown in the PoC):

1. Admin grants `MIGRATOR_ROLE` to migrator `M`
2. User calls `setMigrationPermit(M, true)` — succeeds, `migrationPermits[M][user] = true`
3. Admin revokes `MIGRATOR_ROLE` from `M` (e.g. the migrator contract is found to be misconfigured, or upgraded)
4. User calls `setMigrationPermit(M, false)` to revoke as a precaution — **reverts with `MigratorNotFound(M)`**

If `MIGRATOR_ROLE` is ever re-granted to `M`, it can immediately call `migratePositionsFrom(user)` using the stale permit without the user ever re-consenting.

## Impact Details

A user who has granted migration permission to a migrator address cannot revoke that permission after the admin revokes the migrator's role, even if the user's intent is purely defensive.

## References

* [`Staking.sol:77-82`](https://github.com/Folks-Finance/folks-staking-contracts/blob/main/src/Staking.sol#L77C1-L82C6) — `setMigrationPermit`, the role check gates both grant and revoke
* [`Staking.sol:166-199`](https://github.com/Folks-Finance/folks-staking-contracts/blob/main/src/Staking.sol#L166) — `migratePositionsFrom`, reads `migrationPermits` to validate consent

## Proof of Concept

Place the following unit test in `Staking.t.sol` and run with this command:

`forge test --mt test_revokePermissionRevertsForOldMigrator`

```js
function test_revokePermissionRevertsForOldMigrator() public {
    // Step 1: user grants permission to a legitimate migrator
    vm.prank(alice);
    staking.setMigrationPermit(migrator, true);
    assertEq(staking.migrationPermits(migrator, alice), true);

    // Step 2: admin revokes the migrator's role (e.g. bad terms discovered)
    bytes32 migratorRole = staking.MIGRATOR_ROLE();
    vm.prank(admin);
    staking.revokeRole(migratorRole, migrator);
    assertEq(staking.hasRole(migratorRole, migrator), false);

    // Step 3: user tries to revoke their permission as a precaution — reverts
    vm.expectRevert(abi.encodeWithSelector(IStakingV1.MigratorNotFound.selector, migrator));
    vm.prank(alice);
    staking.setMigrationPermit(migrator, false);

    // Permit is still true despite the user's attempt to revoke it
    assertEq(staking.migrationPermits(migrator, alice), true);
}
```
