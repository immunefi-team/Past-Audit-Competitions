# 69956 sc low users cannot revoke migration permits after migrator role is revoked stale permits enable unconsented future migrations

## #69956 \[SC-Low] Users Cannot Revoke Migration Permits After MIGRATOR\_ROLE Is Revoked - Stale Permits Enable Unconsented Future Migrations

*Submitted on Mar 17th 2026 at 14:35:05 UTC by @dev3e3 for* [*Audit Comp | Folks Finance: Staking Contracts*](https://immunefi.com/audit-competition/audit-comp-folks-finance-staking-contracts)

* **Report ID:** #69956
* **Report Type:** Smart Contract
* **Report severity:** Low
* **Target:** <https://github.com/Folks-Finance/folks-staking-contracts/blob/main/src/Staking.sol>
* **Impacts:**
  * Griefing (e.g. no profit motive for an attacker, but damage to the users or the protocol)

### Description

## Users Cannot Revoke Migration Permits After MIGRATOR\_ROLE Is Revoked — Stale Permits Enable Unconsented Future Migrations

**Platform:** Immunefi\
**Program:** Folks Finance Staking (Audit Competition)\
**Severity:** Medium — Smart contract operational failure\
**Asset:** Staking.sol (line 78)

### Bug Description

`setMigrationPermit()` gates both **granting** and **revoking** migration permits behind the same `hasRole(MIGRATOR_ROLE, _migrator)` check (line 78). When the admin revokes `MIGRATOR_ROLE` from a migrator address — a standard operational step after migration completes — all users who previously approved that migrator become unable to revoke their permits. The `setMigrationPermit(revokedMigrator, false)` call reverts with `MigratorNotFound`.

The `migrationPermits` mapping has no alternative write path: no admin setter, no batch clear function, no expiration mechanism. The stale permits persist indefinitely in storage. If the admin later re-grants `MIGRATOR_ROLE` to the same address (for a new migration cycle, contract reuse, or by mistake), the migrator can migrate those users' new stakes without fresh consent.

This violates the interface contract stated in `IMigratorV1.sol` lines 9–10: *"Migrator require permission from user to transfer their position to the new staking contract. (migrator should not be able to migrate stakes if not approved by user)"*

```solidity
// Staking.sol lines 77-82
function setMigrationPermit(address _migrator, bool _isMigrationPermitted) external {
    if (!hasRole(MIGRATOR_ROLE, _migrator)) revert MigratorNotFound(_migrator);
    //    ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
    //    This check blocks REVOCATION when the role has been revoked.
    //    Users cannot set _isMigrationPermitted = false for a de-roled migrator.

    migrationPermits[_migrator][msg.sender] = _isMigrationPermitted;
    emit MigrationPermitUpdated(_migrator, msg.sender, _isMigrationPermitted);
}
```

### Impact

**Scenario:**

1. Admin grants `MIGRATOR_ROLE` to MigratorV1 contract
2. Users approve migration: `setMigrationPermit(MigratorV1, true)`
3. V1→V2 migration completes normally
4. Admin revokes `MIGRATOR_ROLE` from MigratorV1 (standard post-migration cleanup)
5. Users attempt to revoke stale permits: `setMigrationPermit(MigratorV1, false)` — **reverts** with `MigratorNotFound`
6. Users stake new tokens in a new staking cycle
7. Admin re-grants `MIGRATOR_ROLE` to MigratorV1 (V2→V3 migration, contract reuse, or operational error)
8. MigratorV1 calls `migratePositionsFrom(user)` — **succeeds** using the stale permit from step 2

**Consequences:**

* Users lose control over which migrator can move their funds
* The stale permit is irrevocable without admin re-granting the role specifically to allow revocation
* Users' new stakes can be migrated to an arbitrary destination contract without current consent
* The user has no on-chain mechanism to protect themselves during the role-revoked window

### Anticipated Objections

<details>

<summary>"This falls under 'Migration operational risks' (excluded)"</summary>

**Rebuttal**: "Migration operational risks" refers to risks inherent in the migration *process* — gas costs, ordering dependencies, data transfer failures. This finding is an access control defect in `setMigrationPermit` at line 78. The function's `hasRole` check incorrectly gates *revocation* behind the same role requirement as *granting*. This is a code-level logic error in an in-scope contract, not an operational risk of performing a migration. The bug manifests even if no migration ever occurs — a user who grants a permit and then the role is revoked is stuck regardless of whether `migratePositionsFrom` is ever called.

</details>

<details>

<summary>"This requires compromised admin keys (excluded)"</summary>

**Rebuttal**: No key compromise is needed. The scenario involves two standard admin operations: (1) revoking `MIGRATOR_ROLE` after migration completes (standard cleanup), and (2) re-granting the role later for a new migration cycle (legitimate operational action). Both are expected lifecycle operations documented in the contract's own design — the `AccessControlDefaultAdminRules` inheritance exists precisely to manage role grants and revocations safely. The vulnerability is that `setMigrationPermit` breaks under this normal lifecycle, not that the admin acts maliciously.

</details>

<details>

<summary>"This is just 'State persistence after withdrawals' (excluded)"</summary>

**Rebuttal**: That exclusion refers specifically to `capUsed` not decrementing on withdrawal, which is explicitly documented as intentional at line 201: *"The capUsed is intentionally not decremented for migrated positions."* In contrast, `migrationPermits` being irrevocable is NOT documented as intentional anywhere in the codebase. The team's own Exploit Attempt 5 (line 357) states *"User must explicitly revoke via `setMigrationPermit(migrator, false)`"* — proving they expected revocation to always work. The irrevocability is an unintended side effect of the `hasRole` check, not a design choice.

</details>

<details>

<summary>"The user consented originally, so the permit is valid"</summary>

**Rebuttal**: The user consented to a specific migration in a specific context. The `IMigratorV1` interface contract (lines 9–10) states: *"migrator should not be able to migrate stakes if not approved by user."* Approval implies ongoing, revocable consent — not a permanent, irrevocable authorization. The inability to withdraw consent violates the documented interface guarantee. A user who approved migration of 100 tokens in January did not consent to migration of 50 different tokens staked in December.

</details>

### Aggravating Factors

#### 1. Circular dependency — no safe fix path exists without contract upgrade

The `migrationPermits` mapping is only written at line 80 via `setMigrationPermit`, which uses `msg.sender` as the user key. **No admin function, batch clear, or alternative write path exists.** Only the affected user can clear their own permit — and they can't, because the role check blocks them.

This creates a circular dependency:

* **To clear the permit:** user must call `setMigrationPermit(migrator, false)` → requires `hasRole(MIGRATOR_ROLE, migrator)` to be true
* **When the role is active:** `migratePositionsFrom(user)` is also callable → migrator can front-run the user's revocation

Both states (role active / role inactive) leave users vulnerable:

| Role State   | User Can Revoke?                  | Migrator Can Exploit?              |
| ------------ | --------------------------------- | ---------------------------------- |
| **Active**   | Yes                               | **Yes — can front-run revocation** |
| **Inactive** | **No — reverts MigratorNotFound** | No                                 |

There is **no sequence of admin actions** that can safely clear stale permits without exposing users to migration. The admin cannot call `setMigrationPermit` on behalf of users (it sets `migrationPermits[_migrator][msg.sender]`, where `msg.sender` is the admin, not the affected user). Even a helper contract calling `setMigrationPermit` would set the helper's permit, not the user's.

**The only true fix is deploying a patched contract.** Proven in PoC test `test_exploit_noSafeFixPath`.

#### 2. Affects all users simultaneously

A single admin action (revoking `MIGRATOR_ROLE`) locks the permits of ALL users who approved that migrator. If 100 users approved migration, all 100 are stuck with irrevocable permits after one `revokeRole` call.

#### 3. No expiration — permits persist indefinitely

The `migrationPermits` mapping has no timestamp, no TTL, no expiration mechanism. A permit granted today is exploitable years later if the role is re-granted. Proven in PoC test `test_exploit_permitPersistsIndefinitely` (2-year gap between approval and exploitation).

### Root Cause

Line 78 of `Staking.sol` applies `hasRole(MIGRATOR_ROLE, _migrator)` unconditionally, regardless of whether the user is granting (`true`) or revoking (`false`) their permit. When the role is revoked from the migrator address, both grant AND revoke operations revert with `MigratorNotFound`.

The `migrationPermits` mapping (line 38) is only written at line 80. No other function, admin role, or mechanism can modify it. Users have no path to clear stale permits when the migrator's role is revoked.

### Recommendation

Only require the `MIGRATOR_ROLE` check when **granting** permits, not when **revoking**. Users should always be able to revoke their own consent:

```solidity
function setMigrationPermit(address _migrator, bool _isMigrationPermitted) external {
    if (_isMigrationPermitted && !hasRole(MIGRATOR_ROLE, _migrator)) {
        revert MigratorNotFound(_migrator);
    }

    migrationPermits[_migrator][msg.sender] = _isMigrationPermitted;
    emit MigrationPermitUpdated(_migrator, msg.sender, _isMigrationPermitted);
}
```

This preserves the original safety property (users can only grant permits to active migrators) while ensuring revocation is always available.

### Link to Proof of Concept

<https://gist.github.com/3e3dev/ceda26c1ebca74c11d587d8c6e26633a>

### Proof of Concept

```solidity
// SPDX-License-Identifier: BUSL-1.1
pragma solidity ^0.8.23;

import {Test} from "forge-std/Test.sol";
import {ERC20Permit, ERC20} from "openzeppelin-contracts/contracts/token/ERC20/extensions/ERC20Permit.sol";
import {Staking} from "../src/Staking.sol";
import {IStakingV1} from "../src/interfaces/IStakingV1.sol";

contract MockToken is ERC20Permit {
    constructor() ERC20Permit("MockToken") ERC20("MockToken", "MTK") {}
    function mint(address to, uint256 amount) external { _mint(to, amount); }
}

contract FS001_IrrevocablePermitTest is Test {
    Staking public staking;
    MockToken public token;

    address admin    = address(0xAD01);
    address manager  = address(0xBE02);
    address pauser   = address(0xCE03);
    address migrator = address(0xDE04);
    address alice    = address(0xA11C);

    bytes32 constant MIGRATOR_ROLE = keccak256("MIGRATOR");

    function setUp() public {
        token = new MockToken();
        staking = new Staking(admin, manager, pauser, address(token));

        // Setup roles and staking period
        vm.prank(admin);
        staking.grantRole(MIGRATOR_ROLE, migrator);

        vm.prank(manager);
        staking.addStakingPeriod(1000 ether, 30 days, 10 days, 1000, true);

        // Fund contract and user
        token.mint(address(staking), 50_000 ether);
        token.mint(alice, 10_000 ether);
        vm.prank(alice);
        token.approve(address(staking), type(uint256).max);
    }

    function test_irrevocablePermitAfterRoleRevoke() public {
        IStakingV1.StakingPeriod memory sp = staking.getStakingPeriod(0);
        IStakingV1.StakeParams memory params = IStakingV1.StakeParams({
            maxStakingDurationSeconds: sp.stakingDurationSeconds,
            maxUnlockDurationSeconds: sp.unlockDurationSeconds,
            minAprBps: sp.aprBps,
            referrer: address(0)
        });

        // Step 1: Alice stakes and grants migration permit
        vm.prank(alice);
        staking.stake(0, 100 ether, params);

        vm.prank(alice);
        staking.setMigrationPermit(migrator, true);
        assertTrue(staking.migrationPermits(migrator, alice));

        // Step 2: Migration occurs normally
        vm.prank(migrator);
        staking.migratePositionsFrom(alice);

        // Step 3: Admin revokes MIGRATOR_ROLE (standard cleanup)
        vm.prank(admin);
        staking.revokeRole(MIGRATOR_ROLE, migrator);
        assertFalse(staking.hasRole(MIGRATOR_ROLE, migrator));

        // Step 4: Alice tries to revoke her permit — REVERTS
        vm.expectRevert(
            abi.encodeWithSelector(IStakingV1.MigratorNotFound.selector, migrator)
        );
        vm.prank(alice);
        staking.setMigrationPermit(migrator, false);

        // Permit is stuck as TRUE — irrevocable
        assertTrue(staking.migrationPermits(migrator, alice));

        // Step 5: Alice stakes new tokens
        vm.prank(alice);
        staking.stake(0, 50 ether, params);

        // Step 6: Admin re-grants MIGRATOR_ROLE (new migration cycle)
        vm.prank(admin);
        staking.grantRole(MIGRATOR_ROLE, migrator);

        // Step 7: Migrator migrates Alice's NEW stakes without fresh consent
        uint256 migratorBalBefore = token.balanceOf(migrator);
        vm.prank(migrator);
        IStakingV1.UserStake[] memory migrated = staking.migratePositionsFrom(alice);

        // Proof: migration succeeded using stale permit
        assertEq(migrated.length, 1, "New stake migrated without re-consent");
        assertGt(
            token.balanceOf(migrator) - migratorBalBefore, 0,
            "Tokens transferred using stale permit"
        );
    }
}
```

**To run:** `forge test --match-test test_irrevocablePermitAfterRoleRevoke -vvv`
