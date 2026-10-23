# 69136 sc low missing revocation condition in setmigrationpermit prevents users from revoking stale migration permissions violating documented protocol guarantee

**Submitted on Mar 13th 2026 at 01:31:04 UTC by @CaptSinbad for** [**Audit Comp | Folks Finance: Staking Contracts**](https://immunefi.com/audit-competition/audit-comp-folks-finance-staking-contracts)

* **Report ID:** #69136
* **Report Type:** Smart Contract
* **Report severity:** Low
* **Target:** <https://github.com/Folks-Finance/folks-staking-contracts/blob/main/src/Staking.sol>
* **Impacts:**
  * Contract fails to deliver promised returns, but doesn't lose value

## Description

The `setMigrationPermit` function in `Staking.sol` applies a `hasRole(MIGRATOR_ROLE, _migrator)` check unconditionally — for both granting and revoking a permit. This means a user cannot call `setMigrationPermit(migrator, false)` to revoke a previously granted permit if the migrator's role has since been revoked by an admin.

The vulnerable code:

```solidity
function setMigrationPermit(address _migrator, bool _isMigrationPermitted) external {
    if (!hasRole(MIGRATOR_ROLE, _migrator)) revert MigratorNotFound(_migrator);
    migrationPermits[_migrator][msg.sender] = _isMigrationPermitted;
    emit MigrationPermitUpdated(_migrator, msg.sender, _isMigrationPermitted);
}
```

The role check is appropriate when granting a permit — there is no reason to permit a non-existent migrator. However, the same check incorrectly blocks revocation. A user who wants to clear a stale permit cannot do so until the admin re-grants the role to that address, which may never happen.

The project's own known issues list acknowledges that `migrationPermits` can contain entries for migrators whose role has been revoked. That state is only reachable because this check blocks the user from clearing it themselves.

The `README.md` (line 140) explicitly states:

> "The permission can be revoked at any time by calling `setMigrationPermit(migratorAddress, false)`."

This guarantee is false in the scenario the known issues list acknowledges as possible. The contract fails to deliver a documented and expected security property.

**Recommended fix:**

Apply the role check only when granting a permit, not when revoking one:

```solidity
function setMigrationPermit(address _migrator, bool _isMigrationPermitted) external {
    if (_isMigrationPermitted && !hasRole(MIGRATOR_ROLE, _migrator)) {
        revert MigratorNotFound(_migrator);
    }
    migrationPermits[_migrator][msg.sender] = _isMigrationPermitted;
    emit MigrationPermitUpdated(_migrator, msg.sender, _isMigrationPermitted);
}
```

## Proof of Concept

```solidity
// SPDX-License-Identifier: BUSL-1.1
pragma solidity ^0.8.23;
import {Test} from "forge-std/Test.sol";
import {Staking} from "../../src/Staking.sol";
import {IStakingV1} from "../../src/interfaces/IStakingV1.sol";
import {Token} from "../Staking.t.sol";

contract MigrationPermitRevocationTest is Test {
    Staking public staking;
    Token public token;

    address public admin = address(0xAD);
    address public alice = address(0xA1);
    address public migratorAddr = address(0x11);

    function setUp() public {
        token = new Token();
        staking = new Staking(admin, address(1), address(2), address(token));
    }

    function test_Revocation_Blocked_When_Role_Revoked() public {
        bytes32 migratorRole = staking.MIGRATOR_ROLE();

        // 1. Admin grants MIGRATOR_ROLE
        vm.prank(admin);
        staking.grantRole(migratorRole, migratorAddr);

        // 2. Alice grants permit to migrator
        vm.prank(alice);
        staking.setMigrationPermit(migratorAddr, true);
        assertTrue(staking.migrationPermits(migratorAddr, alice));

        // 3. Admin revokes MIGRATOR_ROLE (routine maintenance)
        vm.prank(admin);
        staking.revokeRole(migratorRole, migratorAddr);

        // 4. Alice attempts to revoke her permit to secure her account
        // BUG: reverts — Alice cannot clear her own permit
        vm.prank(alice);
        vm.expectRevert(
            abi.encodeWithSelector(IStakingV1.MigratorNotFound.selector, migratorAddr)
        );
        staking.setMigrationPermit(migratorAddr, false);

        // 5. Permit remains true despite Alice's attempt to revoke
        assertTrue(
            staking.migrationPermits(migratorAddr, alice),
            "Permit stuck at true - user cannot clear it"
        );
    }
}
```

Terminal Output:

```bash
$ forge test --match-path test/poc/MigrationPermitRevocation.t.sol -vv
Ran 1 test for test/poc/MigrationPermitRevocation.t.sol:MigrationPermitRevocationTest
[PASS] test_Revocation_Blocked_When_Role_Revoked() (gas: 68511)
Suite result: ok. 1 passed; 0 failed; 0 skipped; finished in 11.85ms
```

The test demonstrates that after a role revocation, a user's attempt to call `setMigrationPermit(migrator, false)` reverts with `MigratorNotFound`, leaving the permit permanently set to `true` with no user-accessible remedy. This directly contradicts the README guarantee that revocation is available at any time.
