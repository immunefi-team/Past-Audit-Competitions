# 68994 sc low users cannot revoke migration permits after migrator role is removed

**Submitted on Mar 12th 2026 at 09:24:35 UTC by @godwinudo for** [**Audit Comp | Folks Finance: Staking Contracts**](https://immunefi.com/audit-competition/audit-comp-folks-finance-staking-contracts)

* **Report ID:** #68994
* **Report Type:** Smart Contract
* **Report severity:** Low
* **Target:** <https://github.com/Folks-Finance/folks-staking-contracts/blob/main/src/Staking.sol>

## Description

## Brief/Intro

The `setMigrationPermit` function validates that the target address currently holds the `MIGRATOR_ROLE` before allowing any change to the permit, including revocation. This means that if an admin removes the `MIGRATOR_ROLE` from a migrator address, users who previously granted that migrator permission can no longer revoke their own authorization. The permit remains `true` in storage permanently, and if the role is ever re-granted to the same address, the old permit becomes active again without the user's re-consent.

## Vulnerability Details

The `Staking.sol` contract implements a migration system where users must explicitly authorize specific migrator contracts to move their staking positions. This authorization is managed through the `setMigrationPermit` function, which writes to the `migrationPermits` mapping:

```solidity
mapping(address migrator => mapping(address user => bool isAuthorized)) public migrationPermits;
```

When a user calls `setMigrationPermit`, the function first checks whether the target address currently holds the `MIGRATOR_ROLE`:

```solidity
function setMigrationPermit(address _migrator, bool _isMigrationPermitted) external {
    if (!hasRole(MIGRATOR_ROLE, _migrator)) revert MigratorNotFound(_migrator);

    migrationPermits[_migrator][msg.sender] = _isMigrationPermitted;
    emit MigrationPermitUpdated(_migrator, msg.sender, _isMigrationPermitted);
}
```

The problem is that this role check applies to both granting (`true`) and revoking (`false`) the permit.

{% stepper %}
{% step %}

#### The admin grants `MIGRATOR_ROLE` to address `MigratorA`.

A user calls `setMigrationPermit(MigratorA, true)`, setting `migrationPermits[MigratorA][user] = true`.
{% endstep %}

{% step %}

#### The admin later revokes `MIGRATOR_ROLE` from `MigratorA`.

This may happen because the migrator contract was found to have a flaw, or because the migration window has closed.
{% endstep %}

{% step %}

#### The user now wants to clean up their authorization and calls `setMigrationPermit(MigratorA, false)`.

The function executes `hasRole(MIGRATOR_ROLE, MigratorA)`, which returns `false` because the role was revoked.
{% endstep %}

{% step %}

#### The function reverts with `MigratorNotFound(MigratorA)`.

At this point, the storage value `migrationPermits[MigratorA][user]` is permanently stuck at `true`.
{% endstep %}
{% endstepper %}

The migrator can call `migratePositionsFrom(user)` and the permit check at line 173 passes:

```solidity
if (!migrationPermits[msg.sender][user]) revert MigratorNotPermitted(msg.sender, user);
```

The user never re-consented to this authorization. They may not even be aware that their old permit is active.

## Impact Details

A user permanently loses the ability to revoke their migration authorization for a specific migrator address once that address's `MIGRATOR_ROLE` is removed. The permit remains `true` in storage with no way for anyone to clear it. If the same address later regains `MIGRATOR_ROLE`, it can migrate the user's positions without fresh consent.

## Proof of Concept

{% stepper %}
{% step %}

#### The protocol admin calls `grantRole(MIGRATOR_ROLE, MigratorA)` to authorize a migrator contract.

{% endstep %}

{% step %}

#### Alice has active staking positions.

She calls `setMigrationPermit(MigratorA, true)`. The function checks `hasRole(MIGRATOR_ROLE, MigratorA)` which returns `true`, so the transaction succeeds. Storage now holds `migrationPermits[MigratorA][Alice] = true`.
{% endstep %}

{% step %}

#### The admin calls `revokeRole(MIGRATOR_ROLE, MigratorA)` to disable it.

MigratorA can no longer call `migratePositionsFrom` because the `onlyRole(MIGRATOR_ROLE)` modifier blocks it.
{% endstep %}

{% step %}

#### Alice, being prudent, wants to revoke her permit.

She calls `setMigrationPermit(MigratorA, false)`. The function hits `hasRole(MIGRATOR_ROLE, MigratorA)` returns `false`. The transaction reverts with `MigratorNotFound(MigratorA)`. Alice cannot revoke her permit.
{% endstep %}
{% endstepper %}
