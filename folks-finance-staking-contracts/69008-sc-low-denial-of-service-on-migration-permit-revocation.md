# 69008 sc low denial of service on migration permit revocation

**Submitted on Mar 12th 2026 at 11:15:24 UTC by @Afriauditor for** [**Audit Comp | Folks Finance: Staking Contracts**](https://immunefi.com/audit-competition/audit-comp-folks-finance-staking-contracts)

* **Report ID:** #69008
* **Report Type:** Smart Contract
* **Report severity:** Low
* **Target:** <https://github.com/Folks-Finance/folks-staking-contracts/blob/main/src/Staking.sol>
* **Impacts:**
  * Smart contract unable to operate due to lack of token funds

## Description

## Brief/Intro

The known issues acknowledge that `migrationPermits` may contain a migrator which had its `MIGRATOR_ROLE` later revoked. However, the actual impact extends beyond stale state users who actively want to set their permit to `false` for that revoked address are denied the ability to do so.

Users who have previously called `setMigrationPermit(migrator, true)` are permanently denied the ability to revoke that approval if the migrator address subsequently loses its `MIGRATOR_ROLE`. The function enforces `hasRole(MIGRATOR_ROLE, _migrator)` unconditionally, causing any attempt to call `setMigrationPermit(migrator, false)` to revert with `MigratorNotFound`. This is a denial of service on a user safety action the user is blocked from managing their own permissions, and the stale `true` permit remains in storage indefinitely with no alternative path to clear it.

## Vulnerability Details

The `setMigrationPermit` function applies the same `hasRole` validation regardless of whether the user is granting or revoking:

```solidity
function setMigrationPermit(address _migrator, bool _isMigrationPermitted) external {
    if (!hasRole(MIGRATOR_ROLE, _migrator)) revert MigratorNotFound(_migrator);

    migrationPermits[_migrator][msg.sender] = _isMigrationPermitted;
    emit MigrationPermitUpdated(_migrator, msg.sender, _isMigrationPermitted);
}
```

When `_isMigrationPermitted` is `false`, the user is attempting to protect themselves by removing a previous approval. This is a defensive action that should always be available to the user unconditionally. However, the `hasRole` check blocks it if the migrator no longer holds the role.

Consider the following sequence:

1. Admin grants `MIGRATOR_ROLE` to address `0xMIG`
2. Alice calls `setMigrationPermit(0xMIG, true)` — succeeds
3. Admin revokes `MIGRATOR_ROLE` from `0xMIG`
4. Alice calls `setMigrationPermit(0xMIG, false)` — reverts with `MigratorNotFound`
5. `migrationPermits[0xMIG][alice]` is stuck as `true` permanently

The known issue covers the existence of stale state, but does not address that the contract actively prevents users from cleaning it up. There is no admin function to batch-clear permits, no expiry mechanism, and no alternative function for Alice to remove her stale approval. The `setMigrationPermit` function is the only path to modify the `migrationPermits` mapping, and it is permanently blocked for this address.

## Impact Details

Users are denied the ability to revoke their own migration permits after a migrator role rotation

## Proof of Concept

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.23;

import {Test} from "forge-std/Test.sol";
import {Staking} from "../src/Staking.sol";
import {IStakingV1} from "../src/interfaces/IStakingV1.sol";
import {ERC20} from "openzeppelin-contracts/contracts/token/ERC20/ERC20.sol";

contract Token is ERC20 {
    constructor() ERC20("Token", "TKN") {}
}

contract MigrationPermitDOSTest is Test {
    Staking public staking;
    Token public token;

    address admin = address(1);
    address manager = address(2);
    address pauser = address(3);
    address migrator = address(4);
    address alice = address(5);

    function setUp() public {
        token = new Token();
        staking = new Staking(admin, manager, pauser, address(token));

        vm.prank(admin);
        staking.grantRole(keccak256("MIGRATOR"), migrator);
    }

    function test_CannotRevokeMigrationPermitAfterRoleRemoval() public {
        // Alice approves the migrator
        vm.prank(alice);
        staking.setMigrationPermit(migrator, true);
        assertEq(staking.migrationPermits(migrator, alice), true);

        // Admin revokes MIGRATOR_ROLE
        vm.prank(admin);
        staking.revokeRole(keccak256("MIGRATOR"), migrator);

        // Alice tries to revoke her approval — reverts
        vm.prank(alice);
        vm.expectRevert(abi.encodeWithSelector(IStakingV1.MigratorNotFound.selector, migrator));
        staking.setMigrationPermit(migrator, false);

        // Stale permit remains true — Alice cannot clear it
        assertEq(staking.migrationPermits(migrator, alice), true);
    }
}
```

Run with `forge test --match-test test_CannotRevokeMigrationPermitAfterRoleRemoval -vvv`. The test proves Alice's `setMigrationPermit(migrator, false)` reverts after the role is removed, leaving her permit permanently stuck as `true`.
