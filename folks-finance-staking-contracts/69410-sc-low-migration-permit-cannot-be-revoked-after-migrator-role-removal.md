# 69410 sc low migration permit cannot be revoked after migrator role removal

**Submitted on Mar 14th 2026 at 18:48:58 UTC by @silverologist for** [**Audit Comp | Folks Finance: Staking Contracts**](https://immunefi.com/audit-competition/audit-comp-folks-finance-staking-contracts)

* **Report ID:** #69410
* **Report Type:** Smart Contract
* **Report severity:** Low
* **Target:** <https://github.com/Folks-Finance/folks-staking-contracts/blob/main/src/Staking.sol>
* **Impacts:**
  * Contract fails to deliver promised returns, but doesn't lose value

## Description

## Brief/Intro

The user side migration approval cannot be removed for a migrator whose migrator role is currently not active.

## Vulnerability Details

`setMigrationPermit` validates that the target address holds MIGRATOR\_ROLE for both granting and revoking permission:

```solidity
function setMigrationPermit(address _migrator, bool _isMigrationPermitted) external {
    if (!hasRole(MIGRATOR_ROLE, _migrator)) revert MigratorNotFound(_migrator);
    migrationPermits[_migrator][msg.sender] = _isMigrationPermitted;
}
```

When admin revokes MIGRATOR\_ROLE from an address, users who previously granted migration permits to that address are unable to revoke them. The `hasRole` check causes the revocation call to revert with `MigratorNotFound`.

While the stale permit is currently inert (since `migratePositionsFrom` independently requires `onlyRole(MIGRATOR_ROLE)`), the permit persists in storage. If MIGRATOR\_ROLE is later regranted to the same address, the previous permit becomes active again without the user's consent who might have wanted to remove the permit.

## Impact Details

Users cannot freely remove migrator permits for their positions.

## Proof of Concept

* The MIGRATOR\_ROLE is granted to an address `migrator`
* Alice approves the migration permit for `migrator`
* The MIGRATOR\_ROLE is removed
* Alice wants to remove the permit for `migrator` but cannot do so
* Later, if the MIGRATOR\_ROLE is re-added to `migrator`, they will be able to perform the migration for Alice immediately despite Alice having changed her mind
