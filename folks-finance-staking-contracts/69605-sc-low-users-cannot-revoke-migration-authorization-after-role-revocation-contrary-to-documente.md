# 69605 sc low users cannot revoke migration authorization after role revocation contrary to documented behavior

**Submitted on Mar 15th 2026 at 21:48:51 UTC by @Paludo0x for** [**Audit Comp | Folks Finance: Staking Contracts**](https://immunefi.com/audit-competition/audit-comp-folks-finance-staking-contracts)

* **Report ID:** #69605
* **Report Type:** Smart Contract
* **Report severity:** Low
* **Target:** <https://github.com/Folks-Finance/folks-staking-contracts/blob/main/src/Staking.sol>
* **Impacts:**
  * Contract fails to deliver promised returns, but doesn't lose value

## Description

### Brief/Intro

The contract documentation states that a user can revoke a migration permission at any time, but the implementation does not uphold that guarantee. Once a user has approved a migrator and that migrator later loses `MIGRATOR_ROLE`, the user can no longer revoke the stored permission. If the same address is granted the role again in the future, the historical permit becomes active again without any fresh user action.

### Vulnerability Details

The migration flow in the README explicitly says:

> The migrator must hold the `MIGRATOR_ROLE` in the staking contract. The permission can be revoked at any time by calling `setMigrationPermit(migratorAddress, false)`.

The implementation of `setMigrationPermit()` is:

```solidity
function setMigrationPermit(address _migrator, bool _isMigrationPermitted) external {
    if (!hasRole(MIGRATOR_ROLE, _migrator)) revert MigratorNotFound(_migrator);

    migrationPermits[_migrator][msg.sender] = _isMigrationPermitted;
    emit MigrationPermitUpdated(_migrator, msg.sender, _isMigrationPermitted);
}
```

The `hasRole(MIGRATOR_ROLE, _migrator)` gate is unconditional. It applies both when the user is trying to:

* grant a permit with `_isMigrationPermitted = true`
* revoke a permit with `_isMigrationPermitted = false`

This creates the following lifecycle bug:

1. User calls `setMigrationPermit(M, true)`.
2. Admin revokes `MIGRATOR_ROLE` from `M`.
3. The stored value `migrationPermits[M][user]` remains `true`.
4. User calls `setMigrationPermit(M, false)`.
5. The call reverts with `MigratorNotFound(M)` because `M` no longer has the role.

So the user loses the ability to manage their own authorization, despite the documentation promising revocation at any time.

This stale authorization is not cleaned up anywhere else:

* `revokeRole()` does not clear `migrationPermits`
* `migratePositionsFrom()` only reads `migrationPermits[msg.sender][user]` and never consumes or clears it

### Impact Details

The main impact is loss of user control over migration authorization.

As a result, if the same migrator address is granted `MIGRATOR_ROLE` again later, the old stored permit becomes usable again immediately, even though the user was previously unable to revoke it.

There is no explicit NatSpec on `setMigrationPermit()` itself that describes revocation semantics differently. The contradiction is between the actual code path and the README's user-facing guarantee.

## Proof of Concept

The PoC demonstrates that:

{% stepper %}
{% step %}
Alice grants a migration permit to `migrator`.
{% endstep %}

{% step %}
Admin revokes `MIGRATOR_ROLE` from `migrator`.
{% endstep %}

{% step %}
Alice can no longer call `setMigrationPermit(migrator, false)` because it reverts with `MigratorNotFound`.
{% endstep %}

{% step %}
Admin grants `MIGRATOR_ROLE` back to the same address.
{% endstep %}

{% step %}
The old permit is still `true`, and `migratePositionsFrom(alice)` succeeds without any fresh approval from Alice.
{% endstep %}
{% endstepper %}

```solidity
// SPDX-License-Identifier: BUSL-1.1

pragma solidity ^0.8.23;

import {IStakingV1} from "../src/interfaces/IStakingV1.sol";
import {StakingTest} from "./Staking.t.sol";

contract PoCStalePermitReactivationTest is StakingTest {
    function test_PoC_UserCannotRevokePermitAfterRoleRevocation_AndOldPermitReactivatesOnRegrant() public {
        bytes32 migratorRole = keccak256("MIGRATOR");

        deal(address(token), address(staking), 1000 ether);
        deal(address(token), alice, 100 ether);

        uint8 periodIndex = addStakingPeriodByManager(100 ether, 20, 10, 5000, true);
        approveAndStake(alice, periodIndex, 10 ether, 20, 10, 5000, address(0));

        vm.prank(alice);
        staking.setMigrationPermit(migrator, true);
        assertEq(staking.migrationPermits(migrator, alice), true);

        vm.prank(admin);
        staking.revokeRole(migratorRole, migrator);
        assertEq(staking.hasRole(migratorRole, migrator), false);
        assertEq(staking.migrationPermits(migrator, alice), true);

        vm.expectRevert(abi.encodeWithSelector(IStakingV1.MigratorNotFound.selector, migrator));
        vm.prank(alice);
        staking.setMigrationPermit(migrator, false);

        vm.prank(admin);
        staking.grantRole(migratorRole, migrator);
        assertEq(staking.hasRole(migratorRole, migrator), true);
        assertEq(staking.migrationPermits(migrator, alice), true);

        vm.prank(migrator);
        staking.migratePositionsFrom(alice);

        assertEq(staking.getUserStakes(alice).length, 0);
    }
}
```
