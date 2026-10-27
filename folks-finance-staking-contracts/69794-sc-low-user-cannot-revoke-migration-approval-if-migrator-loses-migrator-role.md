# 69794 sc low user cannot revoke migration approval if migrator loses migrator role

**Submitted on Mar 16th 2026 at 21:29:50 UTC by @CarlosMB for** [**Audit Comp | Folks Finance: Staking Contracts**](https://immunefi.com/audit-competition/audit-comp-folks-finance-staking-contracts)

* **Report ID:** #69794
* **Report Type:** Smart Contract
* **Report severity:** Low
* **Target:** <https://github.com/Folks-Finance/folks-staking-contracts/blob/main/src/Staking.sol>

## Description

## Intro

The function `setMigrationPermit` requires the target address to currently have `MIGRATOR_ROLE`. If the role is revoked from a migrator, users can no longer call `setMigrationPermit(migrator, false)` to revoke their previously granted approval. This leaves stale migration approvals that users cannot clear themselves.

## Vulnerability Details

The function `setMigrationPermit` performs the following check:

```solidity
if (!hasRole(MIGRATOR_ROLE, _migrator))
    revert MigratorNotFound(_migrator);
```

This check is executed regardless of whether the user is granting or revoking permission.

If a migrator previously had `MIGRATOR_ROLE` and users granted permission via `setMigrationPermit(migrator, true);`

and the admin later revokes the role: `revokeRole(MIGRATOR_ROLE, migrator);`

users are unable to revoke the permission because calling `setMigrationPermit(migrator, false);` will revert due to the role check.

As a result, the contract can contain stale approvals that users cannot clear.

## Impact Details

Users cannot revoke migration approvals once a migrator loses `MIGRATOR_ROLE`, leaving stale permissions stored in `migrationPermits`.

This does not directly lead to loss of funds because migration requires the caller to have `MIGRATOR_ROLE`. However, it creates a situation where users cannot clean up previously granted permissions.

Category: Insight → Code Optimizations and Enhancements

## Proof of Concept

1. User authorizes a migrator:

   `setMigrationPermit(migrator, true);`
2. Admin revokes the migrator role:

   `revokeRole(MIGRATOR_ROLE, migrator);`
3. User attempts to revoke approval:

   `setMigrationPermit(migrator, false);`
4. Transaction reverts due to:

   ```solidity
   if (!hasRole(MIGRATOR_ROLE, _migrator))
       revert MigratorNotFound(_migrator);
   ```

Result:

`migrationPermits[migrator][user]` remains `true` and cannot be cleared by the user.

## Recommendation

Allow users to revoke permissions even if the address no longer has `MIGRATOR_ROLE`. For example:

```solidity
if (_isMigrationPermitted && !hasRole(MIGRATOR_ROLE, _migrator))
    revert MigratorNotFound(_migrator);
```
