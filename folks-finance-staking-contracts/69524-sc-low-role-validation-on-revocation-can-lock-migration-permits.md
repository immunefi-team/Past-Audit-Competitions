# 69524 sc low role validation on revocation can lock migration permits

**Submitted on Mar 15th 2026 at 12:15:38 UTC by @theboiledcorn for** [**Audit Comp | Folks Finance: Staking Contracts**](https://immunefi.com/audit-competition/audit-comp-folks-finance-staking-contracts)

* **Report ID:** #69524
* **Report Type:** Smart Contract
* **Report severity:** Low
* **Target:** <https://github.com/Folks-Finance/folks-staking-contracts/blob/main/src/Staking.sol>
* **Impacts:**
  * Contract fails to deliver promised returns, but doesn't lose value

## Description

### Brief/Intro

The `setMigrationPermit` function in the `Staking` contract allows users to authorize or revoke a migrator address that can migrate their staking positions. However, the implementation enforces that the specified migrator must currently hold the `MIGRATOR_ROLE` **for both granting and revoking permissions**.

If the role is later removed from the migrator by an administrator, users who previously granted a permit to that address lose the ability to revoke it. Any attempt to call `setMigrationPermit(migrator, false)` reverts with `MigratorNotFound`.

As a result, the stored permit remains permanently set to `true` with no direct on-chain method for the user to clear it. This behavior conflicts with the documented guarantee that migration permissions **“can be revoked at any time.”**

### Vulnerability Details

```solidity
function setMigrationPermit(address _migrator, bool _isMigrationPermitted) external {
    if (!hasRole(MIGRATOR_ROLE, _migrator)) revert MigratorNotFound(_migrator);
    migrationPermits[_migrator][msg.sender] = _isMigrationPermitted;
    emit MigrationPermitUpdated(_migrator, msg.sender, _isMigrationPermitted);
}
```

The role verification is applied regardless of whether the user is **granting** (`true`) or **revoking** (`false`) permission.

When the migrator’s role has already been revoked, the `hasRole` check fails and the transaction reverts. This prevents users from removing previously granted authorization.

### Scenario

1. Alice grants a migration permit to `migrator` while the address holds the `MIGRATOR_ROLE`.
2. At a later time, an admin removes the migrator’s role.
3. Alice attempts to revoke the authorization by calling `setMigrationPermit(migrator, false)`.
4. The transaction reverts because the migrator no longer satisfies the `MIGRATOR_ROLE` requirement.

The permit therefore remains stored as `true`. Alice has no way to clear this entry through the contract interface.

Although the migrator cannot execute migrations while the role is absent, the permit effectively persists indefinitely. If the role is granted again in the future, the authorization automatically becomes active again without Alice explicitly re-approving it.

## Impact Details

* **Irrevocable Authorization:** Users may be unable to revoke migration permissions that were previously granted, contradicting documented behavior.
* **Persistent Latent Permissions:** Old approvals remain recorded even after a migrator loses its role.
* **Potential Future Risk:** If the same address regains the `MIGRATOR_ROLE` later, all existing permits instantly become valid again without user interaction.

There is **no immediate loss of funds**, since migration calls still require the `MIGRATOR_ROLE`. However, the inability to revoke approvals breaks expected permission management and might introduce long-term authorization risk.

## References

[Staking.sol#L77](https://github.com/Folks-Finance/folks-staking-contracts/blob/3131a2d46b5afa76f606bf08adfd85452a47e2d8/src/Staking.sol#L77)

## Recommended Mitigation

The role check should only apply when **granting** a migration permit. Revocation should always be allowed, regardless of the migrator’s current role status.

```solidity
function setMigrationPermit(address _migrator, bool _isMigrationPermitted) external {
    if (_isMigrationPermitted && !hasRole(MIGRATOR_ROLE, _migrator)) {
        revert MigratorNotFound(_migrator);
    }

    migrationPermits[_migrator][msg.sender] = _isMigrationPermitted;
    emit MigrationPermitUpdated(_migrator, msg.sender, _isMigrationPermitted);
}
```

This modification ensures that:

* New permits can only be granted to valid migrators.
* Users retain the ability to revoke previously granted permissions at any time.

## Proof of Concept

Insert into `Staking.t.sol`

```solidity
function test_UserCannotRevokePermitIfMigratorRoleRemoved() public {
    // fund contracts and users
    deal(address(token), address(staking), 1_000 ether);
    deal(address(token), alice, 100 ether);

    // create staking configuration
    uint8 pid = addStakingPeriodByManager(50 ether, 20, 10, 5000, true);

    // alice stakes
    approveAndStake(alice, pid, 10 ether, 20, 10, 5000, address(0));

    // alice authorizes migrator
    vm.startPrank(alice);
    staking.setMigrationPermit(migrator, true);
    vm.stopPrank();

    assertEq(staking.migrationPermits(migrator, alice), true);

    // store role identifier
    bytes32 role = staking.MIGRATOR_ROLE();

    // admin removes migrator privileges
    vm.prank(admin);
    staking.revokeRole(role, migrator);

    assertFalse(staking.hasRole(role, migrator));

    // alice now attempts to disable the permit
    vm.startPrank(alice);
    vm.expectRevert(
        abi.encodeWithSelector(IStakingV1.MigratorNotFound.selector, migrator)
    );
    staking.setMigrationPermit(migrator, false);
    vm.stopPrank();

    // the approval is stuck as true
    bool stillApproved = staking.migrationPermits(migrator, alice);
    assertTrue(stillApproved);
}
```
