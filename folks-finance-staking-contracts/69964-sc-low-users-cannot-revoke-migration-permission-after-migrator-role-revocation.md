# 69964 sc low users cannot revoke migration permission after migrator role revocation

**Submitted on Mar 17th 2026 at 14:53:34 UTC by @psb01 for** [**Audit Comp | Folks Finance: Staking Contracts**](https://immunefi.com/audit-competition/audit-comp-folks-finance-staking-contracts)

* **Report ID:** #69964
* **Report Type:** Smart Contract
* **Report severity:** Low
* **Target:** <https://github.com/Folks-Finance/folks-staking-contracts/blob/main/src/Staking.sol>
* **Impacts:**
  * Contract fails to deliver promised returns, but doesn't lose value

## Description

## Brief/Intro

The `setMigrationPermit` function enforces that the `_migrator` must currently hold `MIGRATOR_ROLE` for both granting and revoking permissions.

As a result, if a migrator’s role is revoked, users are unable to revoke previously granted migration permissions, since any attempt to call:

```solidity
setMigrationPermit(migrator, false)
```

will revert.

This creates a scenario where user approvals become non-revocable, leading to a loss of control over previously granted permissions.

## Vulnerability Details

```solidity
if (!hasRole(MIGRATOR_ROLE, _migrator)) revert MigratorNotFound(_migrator);
```

This check applies universally, regardless of whether the user is:

* granting permission (`true`), or
* revoking permission (`false`)

Thus, revocation is blocked when the migrator does not currently hold the role.

Additionally, migration permissions are not cleared after migration, meaning approvals persist even after a user’s positions have already been migrated. Combined with the inability to revoke permissions when the role is revoked, this leads to long-lived and user-uncontrollable approvals.

## Impact Details

* Users cannot revoke previously granted migration permissions once a migrator loses its role
* Migration permissions may persist indefinitely, including after migration is completed
* Users lose the ability to manage or invalidate their approvals
* If the same migrator address later regains `MIGRATOR_ROLE`, previously granted permissions may still be active without any opportunity for the user to revoke them beforehand

This results in persistent, user-uncontrollable approvals, which can lead to unexpected migration behavior in the future.

## References

Code: <https://github.com/Folks-Finance/folks-staking-contracts/blob/main/src/Staking.sol?utm\\_source=immunefi#L77-L82>

## Proof of Concept

1. Admin grants `MIGRATOR_ROLE` to a migrator.
2. User stakes and opts in: `setMigrationPermit(migrator, true)`
3. Admin revokes `MIGRATOR_ROLE` from migrator.
4. User attempts to revoke: `setMigrationPermit(migrator, false)`\
   → Transaction reverts.

Below is coded POC:

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.23;

import {ERC20Permit, ERC20} from "openzeppelin-contracts/contracts/token/ERC20/extensions/ERC20Permit.sol";
import {Test} from "forge-std/Test.sol";
import {Staking} from "src/Staking.sol";
import {IStakingV1} from "src/interfaces/IStakingV1.sol";

contract Token is ERC20Permit {
    constructor() ERC20Permit("TestToken") ERC20("TestToken", "TTKN") {}
}

contract StaleMigrationPermitTest is Test {
    Staking private staking;
    Token private token;

    bytes32 private constant MIGRATOR_ROLE = keccak256("MIGRATOR");

    address private admin = address(1);
    address private manager = address(2);
    address private pauser = address(3);
    address private migrator = address(4);
    address private alice = address(5);

    function setUp() public {
        token = new Token();
        staking = new Staking(admin, manager, pauser, address(token));
        vm.prank(admin);
        staking.grantRole(MIGRATOR_ROLE, migrator);
    }

    function test_StaleMigrationPermitAfterRoleRevocation() public {
        vm.prank(manager);
        uint8 periodIndex = staking.addStakingPeriod(100 ether, 10 days, 1 days, 5000, true);

        deal(address(token), address(staking), 1000 ether);
        deal(address(token), alice, 10 ether);

        vm.startPrank(alice);
        token.approve(address(staking), 10 ether);
        staking.stake(
            periodIndex,
            10 ether,
            IStakingV1.StakeParams({
                maxStakingDurationSeconds: 10 days,
                maxUnlockDurationSeconds: 1 days,
                minAprBps: 5000,
                referrer: address(0)
            })
        );
        staking.setMigrationPermit(migrator, true);
        vm.stopPrank();

        vm.prank(admin);
        staking.revokeRole(MIGRATOR_ROLE, migrator);

        vm.startPrank(alice);
        vm.expectRevert(abi.encodeWithSelector(IStakingV1.MigratorNotFound.selector, migrator));
        staking.setMigrationPermit(migrator, false);
        vm.stopPrank();
    }
}
```
