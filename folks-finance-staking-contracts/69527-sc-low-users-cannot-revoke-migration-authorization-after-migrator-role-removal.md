# 69527 sc low users cannot revoke migration authorization after migrator role removal

**Submitted on Mar 15th 2026 at 12:18:55 UTC by @gneiss for** [**Audit Comp | Folks Finance: Staking Contracts**](https://immunefi.com/audit-competition/audit-comp-folks-finance-staking-contracts)

* **Report ID:** #69527
* **Report Type:** Smart Contract
* **Report severity:** Low
* **Target:** <https://github.com/Folks-Finance/folks-staking-contracts/blob/main/src/Staking.sol>
* **Impacts:**
  * Contract fails to deliver promised returns, but doesn't lose value

## Description

### Brief/Intro

The `setMigrationPermit` function in the `Staking` contract allows users to grant or revoke permission for a migrator address that can move their staking positions. However, the current implementation requires the target address to hold the `MIGRATOR_ROLE` for **both enabling and disabling** the permit.

If an administrator later removes the role from a previously authorized migrator, users who had granted permission to that address are no longer able to revoke it. Any attempt to call `setMigrationPermit(migrator, false)` will revert with `MigratorNotFound`.

As a result, the approval entry remains permanently stored as `true`, leaving users without a direct method to clear it on-chain. This contradicts the documented behavior which states that migration permissions can be revoked at any time.

### Vulnerability Details

Offer a detailed explanation of the vulnerability itself. Do not leave out any relevant information. Code snippets should be supplied whenever helpful, as long as they don’t overcrowd the report with unnecessary details. This section should make it obvious that you understand exactly what you’re talking about, and more importantly, it should be clear by this point that the vulnerability does exist.

## Impact Details

The problem originates from the unconditional role validation inside `setMigrationPermit`.

**Affected Code**

```solidity
function setMigrationPermit(address _migrator, bool _isMigrationPermitted) external {
    if (!hasRole(MIGRATOR_ROLE, _migrator)) revert MigratorNotFound(_migrator);
    migrationPermits[_migrator][msg.sender] = _isMigrationPermitted;
    emit MigrationPermitUpdated(_migrator, msg.sender, _isMigrationPermitted);
}
```

The function verifies that `_migrator` currently holds `MIGRATOR_ROLE` before performing any update. This check executes even when `_isMigrationPermitted` is `false`, which corresponds to a user attempting to revoke permission.

If the migrator role has already been revoked by an administrator, the role check fails and the transaction reverts. Consequently, users cannot remove the stored permit.

## Scenario

A realistic sequence demonstrating the issue:

1. A user grants migration permission to an address that currently holds `MIGRATOR_ROLE`.
2. At a later time, an administrator removes the role from that address (for example after a migration phase ends).
3. The user attempts to disable the authorization by calling `setMigrationPermit(migrator, false)`.
4. The call reverts because the migrator no longer satisfies the role requirement.

The permit therefore remains stored as `true`.

Although migrations cannot be executed while the role is missing, the stored authorization remains in the contract state. If the same address receives the `MIGRATOR_ROLE` again in the future, the previous approvals automatically become valid again without any new user action.

## Impact

This issue primarily affects permission management rather than directly causing fund loss.

Potential consequences include:

* **Irreversible approvals** – Users may be unable to remove migration permissions they previously granted.
* **Stale permissions remaining in storage** – Authorization entries persist even after the migrator is removed.
* **Automatic reactivation of privileges** – If the address receives `MIGRATOR_ROLE` again later, all previously granted permissions instantly become active.

While migrations still require the role, the inability to revoke permissions contradicts expected behavior and introduces long-term authorization risk.

## References

<https://github.com/Folks-Finance/folks-staking-contracts/blob/3131a2d46b5afa76f606bf08adfd85452a47e2d8/src/Staking.sol#L77-L82>

## Proof of Concept

The following Foundry test demonstrates that once a migrator’s role is removed, a user cannot revoke a previously granted permit.

Run:

```bash
forge test --mt test_UserStuckWithPermitAfterMigratorRoleRevoked -vvv
```

```solidity
function test_UserStuckWithPermitAfterMigratorRoleRevoked() public {
    // provide token balances to staking contract and test user
    deal(address(token), address(staking), 1_000 ether);
    deal(address(token), alice, 100 ether);

    // manager creates a staking configuration
    uint8 periodId = addStakingPeriodByManager(50 ether, 20, 10, 5000, true);

    // alice deposits tokens into the staking contract
    approveAndStake(alice, periodId, 10 ether, 20, 10, 5000, address(0));

    // user authorizes the migrator address
    vm.prank(alice);
    staking.setMigrationPermit(migrator, true);

    // verify the permission is recorded
    bool permitEnabled = staking.migrationPermits(migrator, alice);
    assertTrue(permitEnabled);

    // retrieve role identifier for the migrator role
    bytes32 migratorRoleId = staking.MIGRATOR_ROLE();

    // admin removes migrator privileges
    vm.prank(admin);
    staking.revokeRole(migratorRoleId, migrator);

    // confirm role removal
    assertFalse(staking.hasRole(migratorRoleId, migrator));

    // user attempts to remove the authorization
    vm.startPrank(alice);
    vm.expectRevert(
        abi.encodeWithSelector(IStakingV1.MigratorNotFound.selector, migrator)
    );
    staking.setMigrationPermit(migrator, false);
    vm.stopPrank();

    // permission entry remains enabled
    bool permitStillActive = staking.migrationPermits(migrator, alice);
    assertTrue(permitStillActive);
}
```

The test confirms that the permit remains set to `true` even though the user attempted to revoke it.
