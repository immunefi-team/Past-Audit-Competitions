# 69717 sc low users are unable to revoke migration permits for deprecated or demoted migrators

**Submitted on Mar 16th 2026 at 13:16:19 UTC by @OxAnmol for** [**Audit Comp | Folks Finance: Staking Contracts**](https://immunefi.com/audit-competition/audit-comp-folks-finance-staking-contracts)

* **Report ID:** #69717
* **Report Type:** Smart Contract
* **Report severity:** Low
* **Target:** <https://github.com/Folks-Finance/folks-staking-contracts/blob/main/src/Staking.sol>
* **Impacts:**

## Description

## Brief/Intro

The `Staking` contract allows users to approve specific migrator addresses to move their funds via `setMigrationPermit()`. However, the function enforces that the target address currently holds the `MIGRATOR_ROLE`. If a migrator is deprecated and has its role revoked by the admin, users are permanently blocked from revoking their approval for that specific migrator.

## Vulnerability Details

In `Staking.sol`, the `setMigrationPermit` function includes a strict access control check on the target migrator address before updating the user's permit state:

```solidity
function setMigrationPermit(address _migrator, bool _isMigrationPermitted) external {
    if (!hasRole(MIGRATOR_ROLE, _migrator)) revert MigratorNotFound(_migrator);
    migrationPermits[_migrator][msg.sender] = _isMigrationPermitted;
    emit MigrationPermitUpdated(_migrator, msg.sender, _isMigrationPermitted);
}
```

If an active migrator is deprecated (maybe, if it experiences a security incident?), the protocol admin will naturally revoke its `MIGRATOR_ROLE`.

If a user wanted to delist such a migrator by manually clearing their approval (`_isMigrationPermitted = false`) for the now-defunct migrator, the transaction will revert because `hasRole(MIGRATOR_ROLE, _migrator)` evaluates to false. This forcibly locks the user's storage state into an approved status for a deprecated address.

## Impact Details

While the immediate risk is low because a demoted migrator cannot call `migratePositionsFrom()` anyway, this breaks the expected allowance flow. Users should always retain their ability to revoke permissions to migrators, regardless of that migrator's current standing with the protocol admin.

## Proof of Concept

Add the following to the `Staking.t.sol`:

```solidity
function test_Migration_RevertWhen_DemotedMigratorRevokePermit() public {
    // 1. Alice grants permission to the active migrator
    vm.prank(alice);
    staking.setMigrationPermit(migrator, true);
    assertEq(staking.migrationPermits(migrator, alice), true);

    // 2. Admin revokes the MIGRATOR_ROLE from the migrator due to deprecation/compromise
    vm.prank(admin);
    staking.revokeRole(keccak256("MIGRATOR"), migrator);
    assertEq(staking.hasRole(keccak256("MIGRATOR"), migrator), false);

    // 3. Alice tries to practice good hygiene and revoke her permission for the deprecated migrator
    // However, the transaction reverts because the target is no longer a migrator!
    vm.prank(alice);
    vm.expectRevert(abi.encodeWithSelector(IStakingV1.MigratorNotFound.selector, migrator));
    staking.setMigrationPermit(migrator, false);
    
    // As a result, the obsolete migrator is still permanently marked as "permitted" by Alice
    assertEq(staking.migrationPermits(migrator, alice), true);
}
```

### Recommendation:

Modify the function so that the `MIGRATOR_ROLE` check is only enforced when a user is granting permission, allowing users to freely revoke permission from any address at any time.

```diff
function setMigrationPermit(address _migrator, bool _isMigrationPermitted) external {
-   if (!hasRole(MIGRATOR_ROLE, _migrator)) revert MigratorNotFound(_migrator);
+   if (_isMigrationPermitted && !hasRole(MIGRATOR_ROLE, _migrator)) revert MigratorNotFound(_migrator);
    migrationPermits[_migrator][msg.sender] = _isMigrationPermitted;
    emit MigrationPermitUpdated(_migrator, msg.sender, _isMigrationPermitted);
}
```
