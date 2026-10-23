# 69146 sc low readme states migration permission can be revoked at any time but revocation becomes impossible after migrator role is removed

**Submitted on Mar 13th 2026 at 03:48:14 UTC by @Rhaydden for** [**Audit Comp | Folks Finance: Staking Contracts**](https://immunefi.com/audit-competition/audit-comp-folks-finance-staking-contracts)

* **Report ID:** #69146
* **Report Type:** Smart Contract
* **Report severity:** Low
* **Target:** <https://github.com/Folks-Finance/folks-staking-contracts/blob/main/src/Staking.sol>
* **Impacts:**

## Description

### Vulnerability description

The ReadME describes migration approval as a permission that users can revoke at any time. But really, the contract doesnt allow that once the approved migrator no longer holds `MIGRATOR_ROLE`.

The issue comes from the way `setMigrationPermit` is implemented in [Staking.sol](https://github.com/Folks-Finance/folks-staking-contracts/blob/3131a2d46b5afa76f606bf08adfd85452a47e2d8/src/Staking.sol#L77-L82). Before updating the mapping, the function requires the provided address to currently hold the migrator role:

```solidity
function setMigrationPermit(address _migrator, bool _isMigrationPermitted) external {
    if (!hasRole(MIGRATOR_ROLE, _migrator)) revert MigratorNotFound(_migrator);

    migrationPermits[_migrator][msg.sender] = _isMigrationPermitted;
    emit MigrationPermitUpdated(_migrator, msg.sender, _isMigrationPermitted);
}
```

This means a user can successfully approve an address while it is an active migrator, but once governance or an admin revokes `MIGRATOR_ROLE` from that address, the same user can no longer clear the stored approval. Any attempt to call `setMigrationPermit(migrator, false)` reverts with `MigratorNotFound`.

The problem is not limited to a stale UI state. The old approval remains stored in `migrationPermits`, and `migratePositionsFrom` only checks whether the caller currently has the role and whether the stored permit is `true`:

```solidity
function migratePositionsFrom(address user)
    external
    nonReentrant
    onlyRole(MIGRATOR_ROLE)
    returns (UserStake[] memory)
{
    if (!migrationPermits[msg.sender][user]) revert MigratorNotPermitted(msg.sender, user);

    UserStake[] memory stakes = userStakes[user];
    ...
    TOKEN.safeTransfer(msg.sender, unclaimedUserAmount + unclaimedUserRewards);

    emit MigrateFrom(msg.sender, user);
    return migratedStakes;
}
```

As a result, the implementation behaves differently from the documented flow:

* The README says the permission can be revoked at any time.
* The contract actually requires the target address to still be an active migrator before the user can revoke it.
* The stored approval survives role revocation.
* If the same address is granted `MIGRATOR_ROLE` again later, the old approval becomes usable again without any fresh user action.

This is more of a doc mismatch, with a concrete authorization lifecycle problem behind it. Users are led to believe they retain continuous control over migration consent, but that is not true once the role has been revoked from the target address.

## Recommended mitigation steps

Allow revocation regardless of current role status. That is permit users to set `false` even if the target address no longer holds `MIGRATOR_ROLE`. But if the current behavior is intentional, the README should not state that revocation is available at any time.

## Proof of Concept

Add these 2 tests in `test/Staking.t.sol` to reproduce the issue.

### 1. Revocation becomes impossible after role revocation

```solidity
function test_PoC_Info_MigrationPermitCannotBeRevokedAfterRoleRevocation() public {
    vm.prank(alice);
    staking.setMigrationPermit(migrator, true);
    assertEq(staking.migrationPermits(migrator, alice), true);

    vm.prank(admin);
    staking.revokeRole(keccak256("MIGRATOR"), migrator);

    vm.expectRevert(abi.encodeWithSelector(IStakingV1.MigratorNotFound.selector, migrator));
    vm.prank(alice);
    staking.setMigrationPermit(migrator, false);

    assertEq(staking.migrationPermits(migrator, alice), true);
}
```

### 2. The stale approval becomes usable again if the role is re-granted

```solidity
function test_PoC_Info_StaleMigrationPermitCanBeUsedAgainAfterRoleRegrant() public {
    deal(address(token), address(staking), 1000 ether);
    deal(address(token), alice, 100 ether);

    uint8 periodIndex = addStakingPeriodByManager(50 ether, 20, 10, 5000, true);
    approveAndStake(alice, periodIndex, 10 ether, 20, 10, 5000, address(0));

    vm.prank(alice);
    staking.setMigrationPermit(migrator, true);
    assertEq(staking.migrationPermits(migrator, alice), true);

    vm.prank(admin);
    staking.revokeRole(keccak256("MIGRATOR"), migrator);

    vm.expectRevert(abi.encodeWithSelector(IStakingV1.MigratorNotFound.selector, migrator));
    vm.prank(alice);
    staking.setMigrationPermit(migrator, false);

    vm.prank(admin);
    staking.grantRole(keccak256("MIGRATOR"), migrator);

    vm.prank(migrator);
    IStakingV1.UserStake[] memory migratedStakes = staking.migratePositionsFrom(alice);
    assertEq(migratedStakes.length, 1);
    assertEq(migratedStakes[0].amount, 10 ether);
    assertEq(staking.getUserStakes(alice).length, 0);
}
```

Run with:

```bash
forge test --match-test test_PoC_Info_ -vv
```
