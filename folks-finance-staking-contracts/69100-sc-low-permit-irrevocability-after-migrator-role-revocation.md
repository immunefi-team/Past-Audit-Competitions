# 69100 sc low permit irrevocability after migrator role revocation

**Submitted on Mar 12th 2026 at 20:25:12 UTC by @Petrate for** [**Audit Comp | Folks Finance: Staking Contracts**](https://immunefi.com/audit-competition/audit-comp-folks-finance-staking-contracts)

* **Report ID:** #69100
* **Report Type:** Smart Contract
* **Report severity:** Low
* **Target:** <https://github.com/Folks-Finance/folks-staking-contracts/blob/main/src/Staking.sol>
* **Impacts:**
  * Griefing (e.g. no profit motive for an attacker, but damage to the users or the protocol)
  * Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield

## Description

### Permit Irrevocable When Migrator Role Is Revoked

The user cannot protect themselves (cannot revoke consent) even though the documentation guarantees they can. No direct theft occurs at this step alone, but the user suffers loss of control over their funds with no profit motive required from the attacker. The re-grant attack path escalates this to Direct theft of any user funds if MIGRATOR\_ROLE is re-granted.

**Contract:** `Staking.sol` → `setMigrationPermit()`

`setMigrationPermit()` in `Staking.sol` applies a `hasRole(MIGRATOR_ROLE, _migrator)` check to **both** the grant (`true`) and revoke (`false`) paths. Once a migrator's role is revoked, users can no longer call `setMigrationPermit(migrator, false)` — it always reverts with `MigratorNotFound`. The permit is permanently stuck as `true` with no way for the user to clear it.

The documentation makes a direct, verifiable false promise:

> *"The permission can be revoked at any time by calling `setMigrationPermit(migratorAddress, false)`"*

## Root Cause

```solidity
function setMigrationPermit(address _migrator, bool _isMigrationPermitted) external {
    if (!hasRole(MIGRATOR_ROLE, _migrator)) revert MigratorNotFound(_migrator);
    // ↑ This guard fires even when _isMigrationPermitted = false
    migrationPermits[_migrator][msg.sender] = _isMigrationPermitted;
}
```

The `hasRole` check is unconditional — it applies regardless of whether the caller is granting or revoking. Revoking should never require the migrator to still hold the role, since the user is only modifying their own consent record.

## Impact

A user who granted a migration permit to Migrator M cannot revoke it once M's `MIGRATOR_ROLE` is revoked. If the admin later re-grants `MIGRATOR_ROLE` to M (e.g., after resolving a compromised key incident), M's stale permit immediately reactivates and M can call `migratePositionsFrom(user)` to drain the user's stakes — without any new consent from the user.

## Attack Path

{% stepper %}
{% step %}
Migrator M holds `MIGRATOR_ROLE`; user grants permit via `setMigrationPermit(M, true)`
{% endstep %}

{% step %}
M's role is revoked (compromised key, admin action)
{% endstep %}

{% step %}
User calls `setMigrationPermit(M, false)` to protect themselves → **REVERTS** with `MigratorNotFound`
{% endstep %}

{% step %}
Admin re-grants `MIGRATOR_ROLE` to M after resolving the incident
{% endstep %}

{% step %}
M's permit for the user is live again — no new user action required
{% endstep %}

{% step %}
M calls `migratePositionsFrom(user)` and drains all user stakes
{% endstep %}
{% endstepper %}

## Known Issue Distinction

The published known issue states:

> *"State `migrationPermits` may contain migrator which had its `MIGRATOR_ROLE` later revoked"*

This acknowledges the stale state as an observation. It does **not** acknowledge that users lose the ability to clean it up — which is what the documentation explicitly guarantees they can do.

## Recommended Fix

```solidity
function setMigrationPermit(address _migrator, bool _isMigrationPermitted) external {
    // Only require role check when GRANTING permission, not revoking
    if (_isMigrationPermitted && !hasRole(MIGRATOR_ROLE, _migrator))
        revert MigratorNotFound(_migrator);
    migrationPermits[_migrator][msg.sender] = _isMigrationPermitted;
}
```

## Link to Proof of Concept

<https://gist.github.com/devpetrate/02b3234b4403ea993ebc66f7dad7f766>

## Proof of Concept

{% stepper %}
{% step %}
Migrator M holds `MIGRATOR_ROLE`; user calls `setMigrationPermit(M, true)` to grant permission
{% endstep %}

{% step %}
Admin calls `revokeRole(MIGRATOR_ROLE, M)` — M's role is removed
{% endstep %}

{% step %}
User calls `setMigrationPermit(M, false)` to protect themselves
{% endstep %}

{% step %}
Transaction **reverts with `MigratorNotFound`** — user cannot revoke
{% endstep %}

{% step %}
Admin calls `grantRole(MIGRATOR_ROLE, M)` to restore the role
{% endstep %}

{% step %}
`migrationPermits[M][user]` is still `true` — no new user action needed
{% endstep %}

{% step %}
M calls `migratePositionsFrom(user)` and drains all user stakes without any new consent from the user
{% endstep %}
{% endstepper %}

## PoC Output (verified locally)

```
[Step 1] User stakes 1,000 FOLKS on V1
         Stake created ✓
[Step 2] User grants migration permit to migratorEOA
         Permit granted ✓
[Step 3] Admin revokes MIGRATOR_ROLE
         Role revoked ✓
[Step 4] Permit after role revocation: true
         ⚠️  Permit should auto-clear but persists
[Step 5] User attempts setMigrationPermit(migrator, false)
         Docs promise: "can be revoked at any time"
         Result: REVERTED — MigratorNotFound ✓
         User is permanently stuck. Cannot protect themselves.
[Step 6] Admin re-grants MIGRATOR_ROLE
         Role restored: true
         Permit ACTIVE (no new user action): true

VULNERABILITY CONFIRMED
  Doc: 'Permission can be revoked at any time'
  Fact: setMigrationPermit(M, false) REVERTS after role is revoked
  Risk: If role re-granted, stale permit enables drain of user stakes
```

## Doc Contradiction

The README explicitly states:

> "The permission can be revoked at any time by calling `setMigrationPermit(migratorAddress, false)`"

This is demonstrably false once the migrator's role has been revoked.

## Known Issue Distinction

The published known issue states:

> "State migrationPermits may contain migrator which had its MIGRATOR\_ROLE later revoked"

This acknowledges the stale state only as an observation. It does not acknowledge that users lose the ability to clean it up — which is what the documentation explicitly guarantees they can do.

## Full Runnable PoC

Full source code, contracts, and test files available at: <https://gist.github.com/devpetrate/02b3234b4403ea993ebc66f7dad7f766>

Run:

```bash
npx hardhat --config hardhat.config.js test test/PoC1_PermitIrrevocability.test.js
```
