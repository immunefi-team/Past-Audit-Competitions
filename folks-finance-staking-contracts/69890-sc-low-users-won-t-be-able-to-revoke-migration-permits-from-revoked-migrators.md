# 69890 sc low users won t be able to revoke migration permits from revoked migrators

Submitted on Mar 17th 2026 at 09:40:57 UTC by @AasifUsmani for [Audit Comp | Folks Finance: Staking Contracts](https://immunefi.com/audit-competition/audit-comp-folks-finance-staking-contracts)

* **Report ID:** #69890
* **Report Type:** Smart Contract
* **Report severity:** Low
* **Target:** <https://github.com/Folks-Finance/folks-staking-contracts/blob/main/src/Staking.sol>
* **Impacts:**
  * Contract fails to deliver promised returns, but doesn't lose value

## Description

### Brief/Intro

The `setMigrationPermit()` function before setting the migration flag checks if the migrator being used by caller is a valid migrator, if not, then the function reverts.\
So if a user permits a migrator, and later the migrator is revoked from the `migrator role` by the admin (maybe because of malicious behaviour or account takeover), then the user won't be able to revoke the permit because the `setMigrationPermit()` function will require the provided address to be a migrator.

### Vulnerability Details

The `setMigrationPermit` function in `Staking.sol` allows users to approve an address to migrate their positions. However, it enforces that the `_migrator` address must currently hold the `MIGRATOR_ROLE`.

```solidity
function setMigrationPermit(address _migrator, bool _isMigrationPermitted) external {
    if (!hasRole(MIGRATOR_ROLE, _migrator)) revert MigratorNotFound(_migrator); // @audit-issue Check prevents revocation if role is removed

    migrationPermits[_migrator][msg.sender] = _isMigrationPermitted;
    emit MigrationPermitUpdated(_migrator, msg.sender, _isMigrationPermitted);
}
```

If a migrator is compromised or acts maliciously, the admin would revoke their `MIGRATOR_ROLE`. Once this happens, the check `if (!hasRole(MIGRATOR_ROLE, _migrator))` will cause any call to `setMigrationPermit` to revert.

This prevents users who had previously approved this migrator from revoking their approval. Their approval remains pending `true` in the `migrationPermits` mapping forever, unless the role is granted back.

### Impact Details

If the `MIGRATOR_ROLE` is ever re-granted to the removed address (e.g., accidental admin action, or operational oversight), the previously removed actor would immediately regain the ability to migrate user positions without obtaining new consent. Users are essentially denied the ability to "clean up" their approvals and proactively secure their positions against a specific address.

### Recommendations

Modify `setMigrationPermit` to allow users to set `_isMigrationPermitted` to `false` even if the `_migrator` does not have the role. The check should only be enforced when granting permission (setting it to `true`).

```solidity
function setMigrationPermit(address _migrator, bool _isMigrationPermitted) external {
    // Only require the role if we are enabling the permit
    if (_isMigrationPermitted && !hasRole(MIGRATOR_ROLE, _migrator)) revert MigratorNotFound(_migrator);

    migrationPermits[_migrator][msg.sender] = _isMigrationPermitted;
    emit MigrationPermitUpdated(_migrator, msg.sender, _isMigrationPermitted);
}
```

## References

<https://github.com/Folks-Finance/folks-staking-contracts/blob/3131a2d46b5afa76f606bf08adfd85452a47e2d8/src/Staking.sol#L77>

## Proof of Concept

* To run the PoC, create a new file named `MigrationPermit.t.sol` in **test/** folder.
* copy/paste the given test suite in the newly created file.
* Run the PoC with command `forge test --mt test_CannotRevokePermitAfterMigratorRemoved -vvvv`.

```solidity
// SPDX-License-Identifier: BUSL-1.1
pragma solidity ^0.8.23;

import {Test, console2} from "forge-std/Test.sol";
import {Staking} from "../src/Staking.sol";
import {ERC20} from "openzeppelin-contracts/contracts/token/ERC20/ERC20.sol";
import {IStakingV1} from "../src/interfaces/IStakingV1.sol";

contract Token is ERC20 {
    constructor() ERC20("TestToken", "TTKN") {}
}

contract MigrationPermitIssueTest is Test {
    Staking public staking;
    Token public token;

    address public admin = address(1);
    address public manager = address(2);
    address public pauser = address(3);
    address public user = address(5);
    address public maliciousMigrator = address(6);

    function setUp() public {
        token = new Token();
        
        vm.startPrank(admin);
        staking = new Staking(admin, manager, pauser, address(token));
        vm.stopPrank();
    }

    function test_CannotRevokePermitAfterMigratorRemoved() public {
        // 1. Admin grants migrator role to maliciousMigrator
        vm.startPrank(admin);
        staking.grantRole(staking.MIGRATOR_ROLE(), maliciousMigrator);
        vm.stopPrank();

        // 2. User permits the migrator
        vm.startPrank(user);
        staking.setMigrationPermit(maliciousMigrator, true);
        vm.stopPrank();

        // Check permit is true
        assertTrue(staking.migrationPermits(maliciousMigrator, user));

        // 3. Admin detects malicious behavior and revokes role
        vm.startPrank(admin);
        staking.revokeRole(staking.MIGRATOR_ROLE(), maliciousMigrator);
        vm.stopPrank();

        // 4. User tries to revoke permit to be safe
        vm.startPrank(user);
        
        // This should fail with the current implementation
        vm.expectRevert(
            abi.encodeWithSelector(IStakingV1.MigratorNotFound.selector, maliciousMigrator)
        );
        staking.setMigrationPermit(maliciousMigrator, false);
        vm.stopPrank();
    }
}
```

<details>

<summary>LOGS</summary>

```solidity
    ├─ [25339] Staking::setMigrationPermit(ECAdd: [0x0000000000000000000000000000000000000006], true)
    │   ├─ emit MigrationPermitUpdated(migrator: ECAdd: [0x0000000000000000000000000000000000000006], user: ModExp: [0x0000000000000000000000000000000000000005], isMigrationPermitted: true)
    │   └─ ← [Stop]
    ├─ [0] VM::stopPrank()
    │   └─ ← [Return]
    ├─ [1186] Staking::migrationPermits(ECAdd: [0x0000000000000000000000000000000000000006], ModExp: [0x0000000000000000000000000000000000000005]) [staticcall]
    │   └─ ← [Return] true
    ├─ [0] VM::startPrank(ECRecover: [0x0000000000000000000000000000000000000001])
    │   └─ ← [Return]
    ├─ [459] Staking::MIGRATOR_ROLE() [staticcall]
    │   └─ ← [Return] 0x9fd6ebe44f314ab6ca41249424c1bf3bbe9476c31fbbd7cdb0549ff8e49c8e01
    ├─ [4405] Staking::revokeRole(0x9fd6ebe44f314ab6ca41249424c1bf3bbe9476c31fbbd7cdb0549ff8e49c8e01, ECAdd: [0x0000000000000000000000000000000000000006])
    │   ├─ emit RoleRevoked(role: 0x9fd6ebe44f314ab6ca41249424c1bf3bbe9476c31fbbd7cdb0549ff8e49c8e01, account: ECAdd: [0x0000000000000000000000000000000000000006], sender: ECRecover: [0x0000000000000000000000000000000000000001])
    │   └─ ← [Stop]
    ├─ [0] VM::stopPrank()
    │   └─ ← [Return]
    ├─ [0] VM::startPrank(ModExp: [0x0000000000000000000000000000000000000005])
    │   └─ ← [Return]
    ├─ [0] VM::expectRevert(custom error 0xf28dceb3: 00000000000000000000000000000000000000000000000000000000000000200000000000000000000000000000000000000000000000000000000000000024881da127000000000000000000000000000000000000000000000000000000000000000600000000000000000000000000000000000000000000000000000000)
    │   └─ ← [Return]
    ├─ [1268] Staking::setMigrationPermit(ECAdd: [0x0000000000000000000000000000000000000006], false)
    │   └─ ← [Revert] MigratorNotFound(0x0000000000000000000000000000000000000006)
    ├─ [0] VM::stopPrank()
    │   └─ ← [Return]
    └─ ← [Stop]

Suite result: ok. 1 passed; 0 failed; 0 skipped; finished in 5.88ms (172.90µs CPU time)

Ran 1 test suite in 20.05ms (5.88ms CPU time): 1 tests passed, 0 failed, 0 skipped (1 total tests)
```

</details>
