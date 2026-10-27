# 68955 sc low unconditional hasrole check in setmigrationpermit authorization entrapment

*Submitted on Mar 12th 2026 at 04:38:52 UTC by @hyuunn for* [*Audit Comp | Folks Finance: Staking Contracts*](https://immunefi.com/audit-competition/audit-comp-folks-finance-staking-contracts)

* **Report ID:** #68955
* **Report Type:** Smart Contract
* **Report severity:** Low
* **Target:** <https://github.com/Folks-Finance/folks-staking-contracts/blob/main/src/Staking.sol>
* **Impacts:**

## Description

### Brief/Intro

When an admin revokes the `MIGRATOR_ROLE` from a migrator address, users who previously granted a migration permit to that address cannot revoke their permit. The permit persists indefinitely in storage. If the role is later re-granted to the same address, the stale permit automatically reactivates without the user's knowledge or fresh authorization.

### Vulnerability Details

The `setMigrationPermit()` function applies the `hasRole` check to both grant and revoke operations:

```solidity
// File: src/Staking.sol:84-87
function setMigrationPermit(address _migrator, bool _isMigrationPermitted) external {
    if (!hasRole(MIGRATOR_ROLE, _migrator)) revert MigratorNotFound(_migrator);

    migrationPermits[_migrator][msg.sender] = _isMigrationPermitted;
    emit MigrationPermitUpdated(_migrator, msg.sender, _isMigrationPermitted);
}
```

When an admin revokes `MIGRATOR_ROLE`, users cannot call `setMigrationPermit(migrator, false)` — the check reverts. The permit remains `true` in storage indefinitely. This violates the design principle documented in `IMigratorV1.sol`: "migrator should not be able to migrate stakes if not approved by user" — user approval must always be revocable at their discretion.

### Impact Details

* **Authorization Entrapment:** Users cannot revoke stale permits after role revocation, losing control over their migration consent
* **Surprise Re-Migration Risk:** If the role is re-granted to the same address (or if the address is an upgradeable proxy), the permit silently reactivates without fresh user authorization
* **No Immediate Fund Loss:** Impact is authorization integrity, not fund theft — but violates core security principle that users must control their own approvals

**Conditions for Insight Value:** Admin revokes a migrator role while users still hold permits; user later wants to revoke; role could potentially be re-granted

### References

* `src/Staking.sol:84-87` — `setMigrationPermit()` with unconditional `hasRole` check
* `src/interfaces/IMigratorV1.sol` — design intent: "migrator should not be able to migrate stakes if not approved by user"

## Proof of Concept

**Scenario Where Authorization Control Is Lost:**

1. User calls `setMigrationPermit(migratorAddress, true)` → permit is granted, stored as `true`
2. Admin calls `revokeRole(MIGRATOR_ROLE, migratorAddress)` → migrator loses the role
3. User discovers the migrator no longer needs permission and calls `setMigrationPermit(migratorAddress, false)` to clean up
4. **Transaction reverts:** `MigratorNotFound(migratorAddress)` — the `hasRole` check fails on revocation
5. User has no way to clear their permit — it remains `true` in storage indefinitely
6. Later: Admin re-grants `MIGRATOR_ROLE` to the same address (legitimate reason: replacement migrator)
7. The stale permit becomes active again without the user's knowledge or fresh authorization
8. If the user created new stakes after the role was revoked, these new stakes are now vulnerable to migration without fresh consent
