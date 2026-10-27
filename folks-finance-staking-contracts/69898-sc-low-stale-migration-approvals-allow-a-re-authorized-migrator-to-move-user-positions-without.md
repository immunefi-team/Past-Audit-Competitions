# 69898 sc low stale migration approvals allow a re authorized migrator to move user positions without renewed consent

**Submitted on Mar 17th 2026 at 10:48:53 UTC by @IronsideSec for** [**Audit Comp | Folks Finance: Staking Contracts**](https://immunefi.com/audit-competition/audit-comp-folks-finance-staking-contracts)

* **Report ID:** #69898
* **Report Type:** Smart Contract
* **Report severity:** Low
* **Target:** <https://github.com/Folks-Finance/folks-staking-contracts/blob/main/src/Staking.sol>
* **Impacts:**
  * Contract fails to deliver promised returns, but doesn't lose value

## Description

### Brief/Intro

`setMigrationPermit()` only allows users to update permissions for addresses that currently hold `MIGRATOR_ROLE`. If a migrator was previously approved, then loses the role, the old approval remains stored and the user can no longer revoke it. If the same address later receives `MIGRATOR_ROLE` again, it immediately regains the ability to migrate that user's positions without fresh consent.

### Vulnerability Details

The bug is caused by the mismatch between permission storage and permission revocation rules. `setMigrationPermit()` reverts when `_migrator` does not currently hold `MIGRATOR_ROLE`, but `migrationPermits[_migrator][user]` is not cleared when the role is revoked. As a result, a stale `true` value can survive role removal and become active again after the role is regranted.

* Alice calls `setMigrationPermit(migrator, true)`.
* The admin revokes `MIGRATOR_ROLE` from `migrator`.
* Alice then tries to revoke the approval with `setMigrationPermit(migrator, false)`, but the call reverts with `MigratorNotFound`, so the stored approval stays `true`.
* The admin grants `MIGRATOR_ROLE` back to the same address.
* The migrator can now call `migratePositionsFrom(alice)` successfully and receive `10 ether + 3,170,979,198,376 wei` (check poc), even though Alice attempted to revoke approval before the role was restored.

This proves the approval is not truly revocable once the role is removed, and that old consent can be resurrected later by admin action.

## Impact Details

`Contract fails to deliver promised returns, but doesn't lose value` :

This is a broken authorization / stale permission issue. A previously approved migrator can regain migration power over users without those users providing fresh consent. In the PoC, the migrator is able to pull Alice's full live position worth `10.000003170979198376` tokens. In production, the impact scales to the sum of all still-open positions and reserved rewards belonging to users who once approved that migrator address and were later unable to revoke it after role removal.

## References

<https://github.com/Folks-Finance/folks-staking-contracts/blob/3131a2d46b5afa76f606bf08adfd85452a47e2d8/src/Staking.sol#L77-L83>

```solidity
File: folks-staking-contracts/src/Staking.sol
    function setMigrationPermit(address _migrator, bool _isMigrationPermitted) external {
  >>>   if (!hasRole(MIGRATOR_ROLE, _migrator)) revert MigratorNotFound(_migrator);

        migrationPermits[_migrator][msg.sender] = _isMigrationPermitted;
        emit MigrationPermitUpdated(_migrator, msg.sender, _isMigrationPermitted);
    }
```

## Proof of Concept

Paste into `test/staking.t.sol`.

```solidity
    function test_Migration_StaleApprovalCanBeReusedAfterRoleRegrant() public {
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

        assertEq(staking.migrationPermits(migrator, alice), true);

        vm.prank(admin);
        staking.grantRole(keccak256("MIGRATOR"), migrator);

        vm.prank(migrator);
        IStakingV1.UserStake[] memory migratedStakes = staking.migratePositionsFrom(alice);

        assertEq(migratedStakes.length, 1);
        assertEq(migratedStakes[0].amount, 10 ether);
        assertEq(staking.getUserStakes(alice).length, 0);
    }
```
