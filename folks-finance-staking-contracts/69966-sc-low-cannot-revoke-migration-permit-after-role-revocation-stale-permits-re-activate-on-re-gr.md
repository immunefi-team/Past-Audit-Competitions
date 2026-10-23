# 69966 sc low cannot revoke migration permit after role revocation stale permits re activate on re grant&#x20;

Submitted on Mar 17th 2026 at 14:53:43 UTC by @rlp\_xdaem0n for [Audit Comp | Folks Finance: Staking Contracts](https://immunefi.com/audit-competition/audit-comp-folks-finance-staking-contracts)

* **Report ID:** #69966
* **Report Type:** Smart Contract
* **Report severity:** Low
* **Target:** <https://github.com/Folks-Finance/folks-staking-contracts/blob/main/src/Staking.sol>
* **Impacts:**
  * Griefing (e.g. no profit motive for an attacker, but damage to the users or the protocol)

## Description

## Brief/Intro

`setMigrationPermit` checks `hasRole(MIGRATOR_ROLE, _migrator)` before allowing **any** permit change — including revocation. When an admin revokes `MIGRATOR_ROLE` from a migrator address (e.g., because it was compromised), all users who had previously approved that migrator are **permanently locked into their approval**: calling `setMigrationPermit(_migrator, false)` reverts with `MigratorNotFound`. The permit persists in storage with no user-accessible remediation path.

If the admin later re-grants `MIGRATOR_ROLE` to the same address — whether intentionally (believing the address re-secured) or accidentally — the stale permit is immediately exploitable: the migrator can call `migratePositionsFrom(user)` and extract the user's full unclaimed principal + rewards **without the user ever re-consenting**.

***

## Vulnerability Details

### The Role Check Blocks Both Granting and Revoking

```solidity
// Staking.sol L77-L82
function setMigrationPermit(address _migrator, bool _isMigrationPermitted) external {
    if (!hasRole(MIGRATOR_ROLE, _migrator)) revert MigratorNotFound(_migrator); // @audit blocks revocation too
    migrationPermits[_migrator][msg.sender] = _isMigrationPermitted;
    emit MigrationPermitUpdated(_migrator, msg.sender, _isMigrationPermitted);
}
```

The `hasRole` check serves a legitimate purpose when **granting** a permit: it ensures the user only approves addresses that actually hold `MIGRATOR_ROLE`. However, the same check is applied unconditionally when **revoking** (`_isMigrationPermitted = false`), where it serves no security purpose and actively harms users.

### Attack Sequence

1. **Normal operation:** Migrator `M` holds `MIGRATOR_ROLE`. Alice calls `setMigrationPermit(M, true)`.
2. **Role revocation:** Admin revokes `MIGRATOR_ROLE` from `M` (e.g., `M` is compromised, or migration phase ends).
3. **User trapped:** Alice calls `setMigrationPermit(M, false)` — **reverts** with `MigratorNotFound(M)`. Alice has no way to clean up her permit.
4. **Role re-grant:** Admin re-grants `MIGRATOR_ROLE` to `M` at a later time.
5. **Exploitation:** `M` calls `migratePositionsFrom(alice)`. The stale permit from step 1 satisfies the check at L172. Alice's unclaimed stake and rewards are transferred to `M` without Alice's re-consent.

### Why This Is Distinct From Known Issues

The project's known issues include: *"State `migrationPermits` may contain migrator which had its MIGRATOR\_ROLE later revoked."*

This acknowledges that stale permit **state exists** but does not address:

1. The user's **inability to revoke** that stale permit (the actual code bug)
2. The **attack vector** when the role is re-granted to the same address

The stale state is a symptom. The bug is that `setMigrationPermit` applies the role check symmetrically to both granting and revoking, when the security requirement is asymmetric.

***

## Impact

* **Loss of user sovereignty:** Users cannot revoke permissions they granted. This violates the principle that users should always be able to reduce their own exposure.
* **Unauthorized migration on role re-grant:** If the admin re-grants `MIGRATOR_ROLE` to the same address, the migrator can extract the user's full unclaimed principal + rewards without re-consent. The migrator contract receives the tokens via `TOKEN.safeTransfer(msg.sender, ...)` at L206.
* **No user remediation path:** Between role revocation and re-grant, the user has zero ability to protect themselves. They cannot call `setMigrationPermit` (reverts), cannot withdraw faster (linear unlock), and cannot cancel the permit through any other function.

***

## Recommended Fix

Only require `hasRole` when granting a permit. Revocation should always be permitted:

```solidity
function setMigrationPermit(address _migrator, bool _isMigrationPermitted) external {
    if (_isMigrationPermitted && !hasRole(MIGRATOR_ROLE, _migrator)) {
        revert MigratorNotFound(_migrator);
    }
    migrationPermits[_migrator][msg.sender] = _isMigrationPermitted;
    emit MigrationPermitUpdated(_migrator, msg.sender, _isMigrationPermitted);
}
```

Single-line change. Preserves the granting validation while restoring user sovereignty over revocation.

***

## References

* Root cause in `Staking.sol`: <https://github.com/Folks-Finance/folks-staking-contracts/blob/3131a2d/src/Staking.sol#L77-L82>
* Commit audited: <https://github.com/Folks-Finance/folks-staking-contracts/tree/3131a2d46b5afa76f606bf08adfd85452a47e2d8>

## Link to Proof of Concept

<https://gist.github.com/wsam07/a7454521c87df36d8d81cc0706e94fa5>

## Proof of Concept

This PoC proves (1) users cannot revoke a previously granted migration permit after `MIGRATOR_ROLE` is revoked, and (2) the stale permit becomes effective again if the same migrator address is later re-granted the role. A standalone copy of the PoC (including the commands and sample output) is available below.

**Check Full PoC:** <https://gist.github.com/wsam07/a7454521c87df36d8d81cc0706e94fa5>
