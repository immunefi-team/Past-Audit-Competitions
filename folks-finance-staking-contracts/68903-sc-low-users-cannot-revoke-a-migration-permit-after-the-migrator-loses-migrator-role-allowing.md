# 68903 sc low users cannot revoke a migration permit after the migrator loses migrator role allowing stale approval to reactivate if the same address is re granted the role

**Submitted on Mar 11th 2026 at 21:30:32 UTC by @iam0x04 for** [**Audit Comp | Folks Finance: Staking Contracts**](https://immunefi.com/audit-competition/audit-comp-folks-finance-staking-contracts)

* **Report ID:** #68903
* **Report Type:** Smart Contract
* **Report severity:** Low
* **Target:** <https://github.com/Folks-Finance/folks-staking-contracts/blob/main/src/Staking.sol>
* **Impacts:**
  * a stale approval can later be reused to migrate a user’s active position without fresh consent.

## Description

## Brief/Intro

`setMigrationPermit()` blocks both granting and revoking permission behind a live `MIGRATOR_ROLE` check. As a result, if a user previously approved a migrator, and that migrator later loses its role, the user can no longer clear the stored approval. If the same address is granted `MIGRATOR_ROLE` again in the future, the stale approval becomes active again and the migrator can move the user’s still-open staking position without fresh consent, breaking the intended “user-controlled migration” model.

## Vulnerability Details

```solidity
function setMigrationPermit(address _migrator, bool _isMigrationPermitted) external {
    if (!hasRole(MIGRATOR_ROLE, _migrator)) revert MigratorNotFound(_migrator);

    migrationPermits[_migrator][msg.sender] = _isMigrationPermitted;
    emit MigrationPermitUpdated(_migrator, msg.sender, _isMigrationPermitted);
}
```

<https://github.com/Folks-Finance/folks-staking-contracts/blob/3131a2d46b5afa76f606bf08adfd85452a47e2d8/src/Staking.sol#L77-L82>

This function applies the same `hasRole(MIGRATOR_ROLE, _migrator)` gate to both:

* enabling a permit with `true`
* disabling a permit with `false`

That creates the following broken state transition:

1. A user grants migration permission to migrator M by calling `setMigrationPermit(M, true)`.
2. Later, M loses `MIGRATOR_ROLE`.
3. The user attempts to revoke the old approval with `setMigrationPermit(M, false)`.
4. The call reverts with `MigratorNotFound(M)` because M no longer has the role.
5. The old value in `migrationPermits[M][user]` remains `true`.
6. If M is re-granted `MIGRATOR_ROLE` later, the stale approval is immediately valid again.

The stale approval is then consumed by `migratePositionsFrom()`

```solidity
if (!migrationPermits[msg.sender][user]) revert MigratorNotPermitted(msg.sender, user);
...
TOKEN.safeTransfer(msg.sender, unclaimedUserAmount + unclaimedUserRewards);
```

Once the role is re-granted, the old permit is enough for the migrator to pass authorization and receive the user’s remaining principal and rewards from the staking contract.

This is also inconsistent with the documentation in `README.md` (line 136), which states that migration permission "can be revoked at any time."

<https://github.com/Folks-Finance/folks-staking-contracts/blob/3131a2d46b5afa76f606bf08adfd85452a47e2d8/README.md?plain=1#L134-L140>

## Impact Details

The impact is that a stale migration approval can later be reused to migrate a user’s active position without fresh consent if the same address is re-granted `MIGRATOR_ROLE`.

## Proof of Concept

Put this file under the `test` folder.

```solidity
// SPDX-License-Identifier: BUSL-1.1

pragma solidity ^0.8.23;

import "./Staking.t.sol";

contract StaleMigrationPermitRegrantPoC is StakingTest {
    function test_poc_staleMigrationPermitCanBeUsedAfterRoleRegrant() public {
        deal(address(token), address(staking), 1000 ether);
        deal(address(token), alice, 100 ether);

        uint8 periodIndex = addStakingPeriodByManager(50 ether, 20, 10, 5000, true);
        approveAndStake(alice, periodIndex, 10 ether, 20, 10, 5000, address(0));

        vm.prank(alice);
        staking.setMigrationPermit(migrator, true);
        assertTrue(staking.migrationPermits(migrator, alice));

        vm.prank(admin);
        staking.revokeRole(keccak256("MIGRATOR"), migrator);

        vm.expectRevert(abi.encodeWithSelector(IStakingV1.MigratorNotFound.selector, migrator));
        vm.prank(alice);
        staking.setMigrationPermit(migrator, false);

        assertTrue(staking.migrationPermits(migrator, alice));

        vm.prank(admin);
        staking.grantRole(keccak256("MIGRATOR"), migrator);

        uint256 expectedTransferredBalance = 10 ether + calculateReward(10 ether, 20, 5000);

        vm.prank(migrator);
        IStakingV1.UserStake[] memory migratedStakes = staking.migratePositionsFrom(alice);

        assertEq(migratedStakes.length, 1);
        assertEq(migratedStakes[0].amount, 10 ether);
        assertEq(staking.getUserStakes(alice).length, 0);
        assertEq(token.balanceOf(migrator), expectedTransferredBalance);
    }
}
```

* Alice stakes `10 ether` into the staking contract.
* Alice explicitly grants migration permission to `migrator`.
* Admin revokes `MIGRATOR_ROLE` from `migrator`.
* Alice tries to revoke the old permission, but the call reverts with `MigratorNotFound`.
* The stored approval remains `true` even though the migrator was deauthorized.
* Admin grants `MIGRATOR_ROLE` back to the same `migrator` address.
* The migrator successfully calls `migratePositionsFrom(alice)` using the stale approval.
* The test confirms Alice’s stake is removed and the migrator receives Alice’s principal plus reward.

```bash
forge test --mt test_poc_staleMigrationPermitCanBeUsedAfterRoleRegrant
```
