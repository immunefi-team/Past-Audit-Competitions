# 69663 sc low users cannot revoke previously granted migration permit after migrator role is revoked

**Submitted on Mar 16th 2026 at 06:52:27 UTC by @Oxb4b for** [**Audit Comp | Folks Finance: Staking Contracts**](https://immunefi.com/audit-competition/audit-comp-folks-finance-staking-contracts)

* **Report ID:** #69663
* **Report Type:** Smart Contract
* **Report severity:** Low
* **Target:** <https://github.com/Folks-Finance/folks-staking-contracts/blob/main/src/Staking.sol>
* **Impacts:**
  * Griefing (e.g. no profit motive for an attacker, but damage to the users or the protocol)

## Description

### Brief/Intro

The `setMigrationPermit()` function in `Staking.sol` prevents users from revoking previously granted migration permissions once the migrator's role has been revoked by the admin. This creates a **consent-revocation deadlock** where stale permissions persist indefinitely and can become active again if the same address is later re-granted the `MIGRATOR_ROLE`, allowing migrations without fresh user consent.

### Vulnerability Details

#### Current Implementation

```solidity
function setMigrationPermit(address _migrator, bool _isMigrationPermitted) external {
    if (!hasRole(MIGRATOR_ROLE, _migrator)) revert MigratorNotFound(_migrator);
    
    migrationPermits[_migrator][msg.sender] = _isMigrationPermitted;
    emit MigrationPermitUpdated(_migrator, msg.sender, _isMigrationPermitted);
}
```

The function enforces `hasRole(MIGRATOR_ROLE, _migrator)` check **regardless** of whether the user is granting or revoking permission. This design has an unintended consequence:

* **Granting permission (`true`)**: The role check is appropriate — users should only grant permits to active migrators.
* **Revoking permission (`false`)**: The role check creates a **deadlock** — users cannot revoke previously granted permissions if the migrator no longer holds the role.

#### Root Cause

The `migrationPermits` mapping is never cleared when a migrator loses the `MIGRATOR_ROLE`. The single gate (`hasRole` check) applies to both grant and revoke operations, preventing users from cleaning up stale consent after role revocation.

#### Consent-Revocation Deadlock Scenario

1. Alice grants migration permission to migrator `M`: `setMigrationPermit(M, true)` → `migrationPermits[M][Alice] = true`
2. Admin revokes `M`'s role: `revokeRole(MIGRATOR_ROLE, M)`
3. Alice attempts to revoke her old permit: `setMigrationPermit(M, false)`
4. **Call reverts** with `MigratorNotFound(M)` because `hasRole(MIGRATOR_ROLE, M)` now returns `false`
5. Months later, admin re-grants role: `grantRole(MIGRATOR_ROLE, M)`
6. `M` can now migrate Alice's positions using the **stale permit** from step 1, without any fresh consent from Alice

***

### Impact and Likelihood Details

#### Impact: Low

* **No Direct Fund Loss:** Funds are not stolen; migration is a legitimate protocol operation.
* **Loss of User Autonomy:** Users lose the ability to revoke previously granted migration consent, violating the principle that **consent should always be user-controlled**.
* **Unexpected Migration:** Users who attempted to revoke consent (and were blocked) may be surprised when old permissions become active again after role re-grant.

#### Likelihood: Low

* Requires specific operational sequence: user grants permit → role revoked → user attempts revocation → role re-granted to same address.
* However, the bug is **deterministic** once the sequence occurs — every user who granted a permit before role revocation will experience this issue.

## Link to Proof of Concept

<https://gist.github.com/tharunbethina/36b3cf2705b65633fc2fd05fe6d8e599>

## Proof of Concept

{% stepper %}
{% step %}

#### Alice grants permission to migrator `M`

`setMigrationPermit(M, true)` → `migrationPermits[M][Alice] = true`
{% endstep %}

{% step %}

#### Admin revokes `MIGRATOR_ROLE` from `M`

`revokeRole(MIGRATOR_ROLE, M)`
{% endstep %}

{% step %}

#### Alice tries to revoke old permit

`setMigrationPermit(M, false)`
{% endstep %}

{% step %}

#### Call reverts

**Call reverts** with `MigratorNotFound(M)` (role check blocks revocation)
{% endstep %}

{% step %}

#### Stale permit persists in storage

`migrationPermits[M][Alice]` remains `true`
{% endstep %}

{% step %}

#### Admin later re-grants `MIGRATOR_ROLE` to `M`

`grantRole(MIGRATOR_ROLE, M)`
{% endstep %}

{% step %}

#### `M` migrates Alice's positions

`M` migrates Alice's positions via `migratePositionsFrom(Alice)` **despite Alice's explicit revocation attempt** - her consent intent (`false`) was ignored due to the deadlock
{% endstep %}
{% endstepper %}
