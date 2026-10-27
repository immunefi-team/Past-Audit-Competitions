# 69505 sc low user cannot revoke migration permit after migrator role is revoked

**Submitted on Mar 15th 2026 at 09:35:04 UTC by @hcrlen for** [**Audit Comp | Folks Finance: Staking Contracts**](https://immunefi.com/audit-competition/audit-comp-folks-finance-staking-contracts)

* **Report ID:** #69505
* **Report Type:** Smart Contract
* **Report severity:** Low
* **Target:** <https://github.com/Folks-Finance/folks-staking-contracts/blob/main/src/Staking.sol>
* **Impacts:**
  * Contract fails to deliver promised returns, but doesn't lose value

## Description

### Brief/Intro

The `setMigrationPermit` function enforces that the target address must currently hold `MIGRATOR_ROLE` for both granting and revoking permits. This means if an admin revokes a migrator's role, any user who previously granted that migrator a permit is permanently unable to revoke it, breaking the protocol's explicit documentation guarantee.

README.md:

> 1. **Grant permission** to a specific migrator contract:
>
> ```
> staking.setMigrationPermit(migratorAddress, true);
> ```
>
> The migrator must hold the `MIGRATOR_ROLE` in the staking contract. The permission can be revoked at any time by calling `setMigrationPermit(migratorAddress, false)`.

## Vulnerability Details

`setMigrationPermit` checks the migrator role unconditionally:

```solidity
function setMigrationPermit(address _migrator, bool _isMigrationPermitted) external {
        if (!hasRole(MIGRATOR_ROLE, _migrator)) revert MigratorNotFound(_migrator);

        migrationPermits[_migrator][msg.sender] = _isMigrationPermitted;
        emit MigrationPermitUpdated(_migrator, msg.sender, _isMigrationPermitted);
    }
```

The role check makes sense when granting a permit. You don't want users granting permits to random addresses. But when revoking, the check is unnecessary and harmful. If the admin revokes `MIGRATOR_ROLE` from a migrator, any user attempting to call `setMigrationPermit(migrator, false)` will revert with `MigratorNotFound`.

The protocol documentation explicitly states:

> "The permission can be revoked at any time by calling `setMigrationPermit(migratorAddress, false)`."

## Impact Details

The stale `true` permit persists in `migrationPermits[migrator][user]` mapping. If the admin ever re-grants `MIGRATOR_ROLE` to the same migrator address, that migrator can immediately call `migratePositionsFrom(user)` and migrate the user's positions, without the user's current consent, since they were unable to revoke during the window when the role was inactive.

## References

Add any relevant links to documentation or code

## Proof of Concept

Paste the following code snippet to `staking.t.sol`:

```solidity
function test_migratorRevokePermit() external {
    //setup
    deal(address(token), address(staking), 1000 ether);
    deal(address(token), alice, 100 ether);

    uint8 periodIndex = addStakingPeriodByManager(50 ether, 20, 5, 5000, true);

    //alice stake
    vm.prank(alice);
    token.approve(address(staking), 10 ether);

    vm.expectEmit(true, true, true, true);
    emit IStakingV1.Staked(alice, periodIndex, address(0), 0, 10 ether);
    stake(alice, periodIndex, 10 ether, 20, 5, 5000, address(0));

    Staking.UserStake[] memory aliceStakes = staking.getUserStakes(alice);
    assertEq(aliceStakes.length, 1);
    assertEq(aliceStakes[0].amount, 10 ether);
    assertEq(token.balanceOf(address(staking)), 1000 ether + 10 ether);

    //alice set migrator permit
    assertEq(staking.migrationPermits(migrator, alice), false);
    vm.startPrank(alice);

    staking.setMigrationPermit(migrator, true);
    assertEq(staking.migrationPermits(migrator, alice), true);
    vm.stopPrank();

    //admin revoke migrator role
    vm.prank(admin);
    staking.revokeRole(keccak256("MIGRATOR"), migrator);

    //alice want to remove migrator permit but she cant
    vm.expectRevert(abi.encodeWithSelector(IStakingV1.MigratorNotFound.selector, migrator));
    vm.prank(alice);
    staking.setMigrationPermit(migrator, false);
}
```
