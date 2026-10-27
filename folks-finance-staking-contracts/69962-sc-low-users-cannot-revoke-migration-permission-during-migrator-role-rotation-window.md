# 69962 sc low users cannot revoke migration permission during migrator role rotation window

**Submitted on Mar 17th 2026 at 14:52:22 UTC by @x0t0wt1w for** [**Audit Comp | Folks Finance: Staking Contracts**](https://immunefi.com/audit-competition/audit-comp-folks-finance-staking-contracts)

* **Report ID:** #69962
* **Report Type:** Smart Contract
* **Report severity:** Low
* **Target:** <https://github.com/Folks-Finance/folks-staking-contracts/blob/main/src/Staking.sol>
* **Impacts:**
  * Contract fails to deliver promised returns, but doesn't lose value

## Description

## Brief/Intro

The `setMigrationPermit()` function unconditionally requires the migrator to hold `MIGRATOR_ROLE` regardless of whether the user is granting or revoking permission. If a migrator's role is temporarily revoked, during a key rotation or infrastructure upgrade users are locked out from revoking their migration consent during that window. If the role is subsequently re-granted to the same address, the migrator can immediately call `migratePositionsFrom()` and transfer all of the user's staked principal and unclaimed rewards without any renewed consent, effectively ignoring the user's expressed intent to revoke. The permission system provides a false sense of control over a critical authorization.

## Vulnerability Details

The setMigrationPermit function enforces a MIGRATOR\_ROLE check unconditionally, regardless of whether the user is granting or revoking permission:

```solidity
function setMigrationPermit(address _migrator, bool _isMigrationPermitted) external {
    if (!hasRole(MIGRATOR_ROLE, _migrator)) revert MigratorNotFound(_migrator);

    migrationPermits[_migrator][msg.sender] = _isMigrationPermitted;
    emit MigrationPermitUpdated(_migrator, msg.sender, _isMigrationPermitted);
}
```

This creates the following scenario:

1. User calls `setMigrationPermit(migrator, true)` -> permission granted successfully.
2. Admin revokes `MIGRATOR_ROLE` from the migrator (key rotation, compromise, upgrade, etc.).
3. User attempts to revoke their permission by calling `setMigrationPermit(migrator, false)`.
4. The call reverts with `MigratorNotFound` -> the user is now locked out of revoking.
5. Admin re-grants `MIGRATOR_ROLE` to the same address at a later date.
6. `migrationPermits[migrator][user]` is still true in storage -> the migrator can now call `migratePositionsFrom(user)` without any renewed consent from the user.

The permission persists indefinitely in storage with no expiration mechanism and no TTL. There is no event or on-chain signal that would alert the user that their previously-blocked revocation attempt is now effectively void. The user has no recourse once this state is reached. The root cause is that the check `hasRole(MIGRATOR_ROLE, _migrator)` should only apply when `_isMigrationPermitted == true`. Revoking consent must always be unconditionally available to the user. The fix is straightforward:

```solidity
function setMigrationPermit(address _migrator, bool _isMigrationPermitted) external {
    if (_isMigrationPermitted && !hasRole(MIGRATOR_ROLE, _migrator)) {
        revert MigratorNotFound(_migrator);
    }
    migrationPermits[_migrator][msg.sender] = _isMigrationPermitted;
    emit MigrationPermitUpdated(_migrator, msg.sender, _isMigrationPermitted);
}
```

## Impact Details

A user who attempts to revoke their migration permission during a `MIGRATOR_ROLE` rotation window is silently locked out, with no on-chain indication of when revocation will become possible again. If the role is re-granted to the same address before the user retries, the migrator can immediately call `migratePositionsFrom()` and transfer all of the user's staked principal and unclaimed rewards, without any renewed consent. The user's expressed intent to revoke is effectively ignored, and the permission system provides a false sense of control.

## References

<https://github.com/Folks-Finance/folks-staking-contracts/blob/main/src/Staking.sol?utm\\_source=immunefi#L77>

## Proof of Concept

Add the following test in `Staking.t.sol` and run `forge test --match-test test_SetMigrationPermit_UserCannotRevokeWhenMigratorRoleRevoked`

```solidity
function test_SetMigrationPermit_UserCannotRevokeWhenMigratorRoleRevoked()
    public
{
    assertEq(staking.migrationPermits(migrator, alice), false);
    vm.startPrank(alice);

    staking.setMigrationPermit(migrator, true);
    assertEq(staking.migrationPermits(migrator, alice), true);
    vm.stopPrank();

    vm.prank(admin);
    staking.revokeRole(keccak256("MIGRATOR"), migrator);

    //Alice tries to revoke the migration permit but it reverts with MigratorNotFound
    vm.expectRevert(
        abi.encodeWithSelector(
            IStakingV1.MigratorNotFound.selector,
            migrator
        )
    );
    vm.startPrank(alice);
    staking.setMigrationPermit(migrator, false);
    vm.stopPrank();
}
```
