# 69756 sc low staking setmigrationpermit unnecessary hasrole check on revocation blocks users from managing own permits

**Submitted on Mar 16th 2026 at 17:27:00 UTC by @zbugs for** [**Audit Comp | Folks Finance: Staking Contracts**](https://immunefi.com/audit-competition/audit-comp-folks-finance-staking-contracts)

* **Report ID:** #69756
* **Report Type:** Smart Contract
* **Report severity:** Low
* **Target:** <https://github.com/Folks-Finance/folks-staking-contracts/blob/main/src/Staking.sol>
* **Impacts:**
  * Contract fails to deliver promised returns, but doesn't lose value

## Description

## Brief/Intro

The `setMigrationPermit` function in `Staking.sol` applies a `hasRole(MIGRATOR_ROLE, _migrator)` check unconditionally, blocking users from revoking their own migration permits once the migrator's role is revoked. No privileged action is required to trigger this bug: a regular user calls a public function to manage their own mapping entry, and the transaction reverts. The program's known issues list acknowledges that "migrationPermits may contain revoked migrators" as a data-level observation about stale state, but this finding identifies the root cause: a code-level bug where the `hasRole` check is applied to revocations (`false`), not just grants (`true`). There is no security or design rationale for blocking permit revocation. The check serves a valid purpose only when granting permits (preventing approval of non-migrator addresses), but not when revoking, since `migratePositionsFrom` already independently enforces `onlyRole(MIGRATOR_ROLE)`. The result is that users are stripped of the ability to manage their own authorization state with no defensive alternative: locked stakes cannot be withdrawn (staking period not ended), and the revocation itself reverts. If the admin later re-grants `MIGRATOR_ROLE` to the same address for legitimate operational reasons (e.g., patching and redeploying a migrator), the irrevocable stale permit allows any third party to force-migrate the user's locked positions to a V2 contract without the user's current consent.

## Vulnerability Details

The `setMigrationPermit` function \[1] requires that the `_migrator` address currently holds the `MIGRATOR_ROLE`:

```solidity
function setMigrationPermit(address _migrator, bool _isMigrationPermitted) external {
    if (!hasRole(MIGRATOR_ROLE, _migrator)) revert MigratorNotFound(_migrator);

    migrationPermits[_migrator][msg.sender] = _isMigrationPermitted;
    emit MigrationPermitUpdated(_migrator, msg.sender, _isMigrationPermitted);
}
```

This check does not distinguish between granting and revoking. When a migrator's role is revoked, `setMigrationPermit(_migrator, false)` reverts with `MigratorNotFound`, preventing the user from revoking their own permit. This is a user-callable function operating on the caller's own data. No elevated privileges are required to reach the bug: any user who previously granted a permit and now wants to revoke it will hit this revert.

The check is also redundant for access control. The `migratePositionsFrom` function \[2] already enforces role-based access independently:

```solidity
function migratePositionsFrom(address user)
    external
    nonReentrant
    onlyRole(MIGRATOR_ROLE)
    returns (UserStake[] memory)
{
    if (!migrationPermits[msg.sender][user]) revert MigratorNotPermitted(msg.sender, user);
```

Because `migratePositionsFrom` checks `onlyRole(MIGRATOR_ROLE)` before reading the permit, a stale permit for a de-roled migrator is inert. The role check in `setMigrationPermit` adds no security value. Its only effect is to block users from revoking their own permits.

The downstream consequence turns this into a security issue when combined with locked stakes:

1. User stakes tokens in a period with a long staking duration (tokens locked).
2. User grants migration permit to a migrator address.
3. Admin revokes `MIGRATOR_ROLE` from that address (security concern, contract rotation, etc.).
4. User attempts to revoke their permit. Transaction reverts with `MigratorNotFound`. No privileged action was involved.
5. User cannot withdraw locked stakes as an alternative defense (staking period has not ended).
6. Admin later re-grants `MIGRATOR_ROLE` to the same address (legitimate operational action, e.g., after patching the migrator contract).
7. The stale permit is active again. Any third party that can invoke the migrator contract triggers migration of the user's locked positions without the user's current consent.

The fix is a single conditional: only apply the role check when the user is granting permission, not when revoking:

```solidity
function setMigrationPermit(address _migrator, bool _isMigrationPermitted) external {
    if (_isMigrationPermitted && !hasRole(MIGRATOR_ROLE, _migrator)) revert MigratorNotFound(_migrator);
    migrationPermits[_migrator][msg.sender] = _isMigrationPermitted;
    emit MigrationPermitUpdated(_migrator, msg.sender, _isMigrationPermitted);
}
```

## Impact Details

This vulnerability affects any user who has previously granted a migration permit. The bug requires no elevated privileges, no special conditions, and no external dependencies to trigger. A regular user calls `setMigrationPermit(migrator, false)` and the transaction reverts.

For users with locked stakes, the consequences compound. Both defensive options are unavailable: withdrawal reverts (staking period not yet complete) and revocation reverts (`MigratorNotFound`). If `MIGRATOR_ROLE` is later re-granted to the same address for legitimate operational reasons, the stale permit becomes immediately exploitable. In the reference `MigratorV1` implementation, the `migrate(user)` function is permissionless, meaning any third party can trigger forced migration of the user's locked principal and earned rewards to a V2 contract the user did not consent to interact with under the current circumstances.

## References

\[1] <https://github.com/Folks-Finance/folks-staking-contracts/blob/3131a2d46b5afa76f606bf08adfd85452a47e2d8/src/Staking.sol#L77-L82>

\[2] <https://github.com/Folks-Finance/folks-staking-contracts/blob/3131a2d46b5afa76f606bf08adfd85452a47e2d8/src/Staking.sol#L166-L172>

## Proof of Concept

{% stepper %}
{% step %}

### Create a new file `test/StalePermitPOC.t.sol` with the following content:

```solidity
// SPDX-License-Identifier: BUSL-1.1
pragma solidity ^0.8.23;

import {ERC20Permit, ERC20} from "openzeppelin-contracts/contracts/token/ERC20/extensions/ERC20Permit.sol";
import {Test, console} from "forge-std/Test.sol";
import {Staking} from "../src/Staking.sol";
import {IStakingV1} from "../src/interfaces/IStakingV1.sol";

contract Token is ERC20Permit {
    constructor() ERC20Permit("FOLKS") ERC20("Folks Finance", "FOLKS") {}
    function mint(address to, uint256 amount) external { _mint(to, amount); }
}

contract StalePermitPOCTest is Test {
    Staking staking;
    Token token;

    address admin = makeAddr("admin");
    address manager = makeAddr("manager");
    address pauser = makeAddr("pauser");
    address migrator = makeAddr("migrator");
    address alice = makeAddr("alice");

    bytes32 constant MIGRATOR_ROLE = keccak256("MIGRATOR");

    function setUp() public {
        token = new Token();
        staking = new Staking(admin, manager, pauser, address(token));

        // Setup: fund the contract with reward tokens, create a staking period,
        // and register a migrator address.
        token.mint(address(staking), 10_000 ether);

        vm.prank(manager);
        staking.addStakingPeriod(
            1_000_000 ether,  // cap
            365 days,         // staking duration
            30 days,          // unlock duration
            1000,             // 10% APR
            true              // active
        );

        vm.prank(admin);
        staking.grantRole(MIGRATOR_ROLE, migrator);
    }

    /// @notice CORE BUG: A user who granted a migration permit cannot revoke it
    ///         after the migrator's role is revoked. setMigrationPermit reverts
    ///         on revocation with MigratorNotFound.
    function test_UserBlockedFromRevokingOwnPermit() public {
        // Alice grants migration permit while migrator has the role
        vm.prank(alice);
        staking.setMigrationPermit(migrator, true);
        assertTrue(staking.migrationPermits(migrator, alice));

        // Admin revokes MIGRATOR_ROLE
        vm.prank(admin);
        staking.revokeRole(MIGRATOR_ROLE, migrator);

        // Alice tries to revoke her own permit - reverts
        vm.expectRevert(abi.encodeWithSelector(IStakingV1.MigratorNotFound.selector, migrator));
        vm.prank(alice);
        staking.setMigrationPermit(migrator, false);

        // Permit remains set - Alice cannot clear her own authorization
        assertTrue(staking.migrationPermits(migrator, alice));
    }

    /// @notice DOWNSTREAM CONSEQUENCE: Shows what can happen as a result of the bug
    ///         above. If the role is later re-granted, the stale permit that Alice could
    ///         not revoke is immediately exploitable. Alice's locked stakes are migrated
    ///         out of the contract without her current consent.
    function test_StalePermitExploitedAfterRoleRegrant() public {
        // Alice stakes 1000 FOLKS (locked for 365 days)
        token.mint(alice, 1000 ether);
        vm.startPrank(alice);
        token.approve(address(staking), 1000 ether);
        staking.stake(0, 1000 ether, IStakingV1.StakeParams({
            maxStakingDurationSeconds: type(uint64).max,
            maxUnlockDurationSeconds: type(uint64).max,
            minAprBps: 0,
            referrer: address(0)
        }));
        staking.setMigrationPermit(migrator, true);
        vm.stopPrank();

        // Admin revokes MIGRATOR_ROLE
        vm.prank(admin);
        staking.revokeRole(MIGRATOR_ROLE, migrator);

        // Alice tries to revoke her permit - blocked
        vm.expectRevert(abi.encodeWithSelector(IStakingV1.MigratorNotFound.selector, migrator));
        vm.prank(alice);
        staking.setMigrationPermit(migrator, false);

        // Alice tries to withdraw her locked stakes - blocked
        vm.expectRevert();
        vm.prank(alice);
        staking.withdraw(0);

        // Alice has no defensive action available at this point.

        // Admin re-grants MIGRATOR_ROLE to the same address
        vm.prank(admin);
        staking.grantRole(MIGRATOR_ROLE, migrator);

        // Migrator calls migratePositionsFrom using Alice's stale permit
        uint256 contractBalBefore = token.balanceOf(address(staking));
        vm.prank(migrator);
        IStakingV1.UserStake[] memory migrated = staking.migratePositionsFrom(alice);
        uint256 tokensTransferred = contractBalBefore - token.balanceOf(address(staking));

        // Alice's locked position was migrated without her current consent
        assertEq(migrated.length, 1);
        assertEq(staking.getUserStakes(alice).length, 0);
        assertGt(tokensTransferred, 0);

        console.log("Alice position migrated without current consent");
        console.log("Principal + rewards transferred out:", tokensTransferred);
    }

    /// @notice PROOF OF REDUNDANCY: migratePositionsFrom already enforces
    ///         onlyRole(MIGRATOR_ROLE). The role check in setMigrationPermit adds
    ///         no security value - its only effect is to block revocation.
    function test_MigratePositionsAlreadyEnforcesRole() public {
        // A non-migrator calling migratePositionsFrom reverts on onlyRole,
        // before the permit mapping is ever read.
        address notMigrator = makeAddr("notMigrator");
        vm.expectRevert();
        vm.prank(notMigrator);
        staking.migratePositionsFrom(alice);
    }
}
```

{% endstep %}

{% step %}

### Compilation and Execution

```bash
forge test --match-path test/StalePermitPOC.t.sol -vvv
```

{% endstep %}

{% step %}

### Test Output

```
Compiling 1 files with Solc 0.8.33
Solc 0.8.33 finished in 621.90ms
Compiler run successful!

Ran 3 tests for test/StalePermitPOC.t.sol:StalePermitPOCTest
[PASS] test_MigratePositionsAlreadyEnforcesRole() (gas: 23971)
[PASS] test_StalePermitExploitedAfterRoleRegrant() (gas: 307260)
Logs:
  Alice position migrated without current consent
  Principal + rewards transferred out: 1100000000000000000000

[PASS] test_UserBlockedFromRevokingOwnPermit() (gas: 59336)
Suite result: ok. 3 passed; 0 failed; 0 skipped; finished in 4.58ms (1.31ms CPU time)

Ran 1 test suite in 24.16ms (4.58ms CPU time): 3 tests passed, 0 failed, 0 skipped (3 total tests)
```

{% endstep %}
{% endstepper %}
