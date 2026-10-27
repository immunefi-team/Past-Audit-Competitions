# 69673 sc low users cannot revoke a migration permit after role removal

**Submitted on Mar 16th 2026 at 07:47:51 UTC by @Athenea for** [**Audit Comp | Folks Finance: Staking Contracts**](https://immunefi.com/audit-competition/audit-comp-folks-finance-staking-contracts)

* **Report ID:** #69673
* **Report Type:** Smart Contract
* **Report severity:** Low
* **Target:** <https://github.com/Folks-Finance/folks-staking-contracts/blob/main/src/Staking.sol>

## Description

### Brief/Intro

This insight highlights a user-control issue in the migration permission system. Once a migrator loses `MIGRATOR_ROLE`, users who had previously approved it can no longer explicitly revoke that approval, leaving stale permissions in storage.

### Vulnerability Details

`setMigrationPermit()` requires the target address to currently hold `MIGRATOR_ROLE` for both enabling and disabling a permit:

```solidity
function setMigrationPermit(address _migrator, bool _isMigrationPermitted) external {
    if (!hasRole(MIGRATOR_ROLE, _migrator)) revert MigratorNotFound(_migrator);

    migrationPermits[_migrator][msg.sender] = _isMigrationPermitted;
    emit MigrationPermitUpdated(_migrator, msg.sender, _isMigrationPermitted);
}
```

As a result, once a migrator loses `MIGRATOR_ROLE`, users who previously approved it can no longer explicitly revoke that approval.

### Impact Details

This is not an immediate exploit, because a role-revoked migrator cannot call `migratePositionsFrom()` while the role is absent.\
Still, the behavior weakens user control and leaves stale approvals in place until role configuration changes again.

### References

<https://github.com/Folks-Finance/folks-staking-contracts/blob/3131a2d46b5afa76f606bf08adfd85452a47e2d8/src/Staking.sol#L166-L210>

### Recommendation

Require `MIGRATOR_ROLE` only when enabling a permit, not when disabling one:

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

### Steps

1. Deploy `Staking`.
2. Grant `MIGRATOR_ROLE` to `migrator`.
3. User enables migration permit for `migrator`.
4. Admin revokes `MIGRATOR_ROLE` from `migrator`.
5. User tries to disable the permit.

### Code

Git Url: <https://github.com/breakAIDev/fork-staking-poc.git>

```solidity
pragma solidity ^0.8.23;

import {console2} from "forge-std/console2.sol";
import {BaseInsightScript, PoCToken} from "../src/InsightPoCHelpers.sol";
import {Staking} from "repo-src/Staking.sol";

contract RevokePermitAfterRoleRemoval is BaseInsightScript {
    function run() external {
        _selectForkIfConfigured();

        console2.log("=== Insight 1: Revoke permit fails after role removal ===");

        (Staking staking, PoCToken token) = _deployStakingFixture();
        token;

        bytes32 migratorRole = staking.MIGRATOR_ROLE();

        vm.prank(USER_A);
        staking.setMigrationPermit(MIGRATOR, true);
        console2.log("Step 1: USER_A enabled migration permit for MIGRATOR");

        vm.prank(ADMIN);
        staking.revokeRole(migratorRole, MIGRATOR);
        console2.log("Step 2: ADMIN revoked MIGRATOR_ROLE");

        vm.prank(USER_A);
        (bool success, bytes memory lowLevelData) =
            address(staking).call(abi.encodeCall(staking.setMigrationPermit, (MIGRATOR, false)));

        if (success) {
            console2.log("Unexpected: permit revocation succeeded");
        } else {
            console2.log("Step 3: USER_A tried to disable the permit and the call reverted");
            console2.logBytes(lowLevelData);
        }
    }
}
```

### Result

Script ran successfully.

Gas used: 5623713

```
== Logs ==

  Fork selected from BSC_RPC_URL

  === Insight 1: Revoke permit fails after role removal ===

  Step 1. USER_A enabled migration permit for MIGRATOR

  Step 2. ADMIN revoked MIGRATOR_ROLE

  Step 3. USER_A tried to disable the permit and the call reverted
  
0x881da127000000000000000000000000000000000000000000000000000000000000000e
```
