# 69390 sc low users cannot revoke migration permit at any time breaking documented guarantee

**Submitted on Mar 14th 2026 at 16:12:33 UTC by @go4ko for** [**Audit Comp | Folks Finance: Staking Contracts**](https://immunefi.com/audit-competition/audit-comp-folks-finance-staking-contracts)

* **Report ID:** #69390
* **Report Type:** Smart Contract
* **Report severity:** Low
* **Target:** <https://github.com/Folks-Finance/folks-staking-contracts/blob/main/src/Staking.sol>
* **Impacts:**
  * Contract fails to deliver promised returns, but doesn't lose value

## Description

## Brief/Intro

The `setMigrationPermit` function enforces a `hasRole(MIGRATOR_ROLE, _migrator)` check on both granting **and** revoking permission. When an admin revokes a migrator's role, any user who previously granted that migrator permission is unable to call `setMigrationPermit(migrator, false)` - the transaction reverts with `MigratorNotFound`.

This directly contradicts the README's documented guarantee that "The permission can be revoked at any time." If the same address later regains `MIGRATOR_ROLE`, the migrator can intentionally ot unintentionally migrate the user's funds before the user revokes which the user might not want anymore but had no chance to remove the permission.

## Vulnerability Details

In `Staking.sol:77-82`, the `setMigrationPermit` function applies the `hasRole` check unconditionally:

```solidity
function setMigrationPermit(address _migrator, bool _isMigrationPermitted) external {
    if (!hasRole(MIGRATOR_ROLE, _migrator)) revert MigratorNotFound(_migrator);
    migrationPermits[_migrator][msg.sender] = _isMigrationPermitted;
    emit MigrationPermitUpdated(_migrator, msg.sender, _isMigrationPermitted);
}
```

The `hasRole` check is appropriate when **granting** permission (`true`) - users shouldn't authorize non-migrators. However, it also blocks **revocation** (`false`), which has no security reason to be gated behind a role check.

{% stepper %}
{% step %}

### Alice grants migration permission

Alice calls `setMigrationPermit(migrator, true)` - succeeds, migrator has the role.
{% endstep %}

{% step %}

### Admin revokes the migrator role

Admin revokes `MIGRATOR_ROLE` from the migrator for any operational reason.
{% endstep %}

{% step %}

### Alice tries to revoke the permit

Alice changes her mind about migration (which is in her rights to do) and calls `setMigrationPermit(migrator, false)` - **reverts** with `MigratorNotFound` because `hasRole` returns false.
{% endstep %}

{% step %}

### The permit remains stored

Alice's permit remains `true` in storage and she has no way to clear it.
{% endstep %}

{% step %}

### Admin re-grants the role

At some later point, admin re-grants `MIGRATOR_ROLE` to the same address.
{% endstep %}

{% step %}

### Migration can proceed using stale consent

The migrator calls `migratePositionsFrom(alice)` - succeeds using the stale permit from step 1, without Alice ever re-consenting.
{% endstep %}
{% endstepper %}

This is distinct from the known issues which acknowledge that stale permit state can exist ("State `migrationPermits` may contain migrator which had its `MIGRATOR_ROLE` later revoked"). The known issues frame this as residual data - they do not address the fact that users are **blocked from cleaning it up**. The user's inability to revoke is the root cause that transforms harmless residual state into an irrevocable consent problem.

## Impact Details

This falls under: **"Contract fails to deliver promised returns, but doesn't lose value" (Low).**

* **Broken documented guarantee**: The README explicitly states "The permission can be revoked at any time by calling `setMigrationPermit(migratorAddress, false)`." This is provably false when the migrator's role has been revoked.
* **Loss of user sovereignty over migration consent**: A user who granted permission and later decided against migration has no recourse. They must monitor the protocol and hope the address never regains the role.
* **Forced migration without fresh consent**: If the address regains `MIGRATOR_ROLE`, the migrator (acting honestly within its privileges) can migrate the user's stakes to a V2 contract the user never agreed to. The user's principal and rewards are moved atomically with no way to prevent it.
* **No funds are lost directly** - the stakes are transferred to V2, not stolen. However, the user loses control over where their assets reside.

## References

* `setMigrationPermit` implementation: `Staking.sol:77-82`
* `migratePositionsFrom` (consumes the permit): `Staking.sol:166-209`
* README documentation guarantee: `README.md:140` — "The permission can be revoked at any time"

## Proof of Concept

Please create a file `IrrevocableMigrationPermit.t.sol` inside the `test` folder and paste the code below.

Run with:

```js
forge test --match-test test_UserCannotRevokeMigrationPermitWhenRoleRevoked -vvv
```

Code:

```js
// SPDX-License-Identifier: BUSL-1.1

pragma solidity ^0.8.23;

import {ERC20Permit, ERC20} from "openzeppelin-contracts/contracts/token/ERC20/extensions/ERC20Permit.sol";
import {Test} from "forge-std/Test.sol";
import {Staking} from "../src/Staking.sol";
import {IStakingV1} from "../src/interfaces/IStakingV1.sol";

contract Token is ERC20Permit {
    constructor() ERC20Permit("TestToken") ERC20("TestToken", "TTKN") {}
}

contract IrrevocableMigrationPermitTest is Test {
    Staking public staking;
    Token public token;

    address public admin = address(2);
    address public manager = address(3);
    address public migrator = address(4);
    address public pauser = address(5);
    address public alice = address(6);

    bytes32 public constant MIGRATOR_ROLE = keccak256("MIGRATOR");

    function setUp() public {
        token = new Token();
        staking = new Staking(admin, manager, pauser, address(token));

        vm.prank(admin);
        staking.grantRole(MIGRATOR_ROLE, migrator);
    }

    function test_UserCannotRevokeMigrationPermitWhenRoleRevoked() public {
        // Step 1: Alice grants migration permit to the migrator
        vm.prank(alice);
        staking.setMigrationPermit(migrator, true);
        assertTrue(staking.migrationPermits(migrator, alice));

        // Step 2: Admin revokes MIGRATOR_ROLE from the migrator (for any reason)
        vm.prank(admin);
        staking.revokeRole(MIGRATOR_ROLE, migrator);
        assertFalse(staking.hasRole(MIGRATOR_ROLE, migrator));

        // Step 3: Alice decides she no longer wants to migrate and tries to revoke her permit
        //         This reverts with MigratorNotFound because the hasRole check blocks revocation
        vm.prank(alice);
        vm.expectRevert(abi.encodeWithSelector(IStakingV1.MigratorNotFound.selector, migrator));
        staking.setMigrationPermit(migrator, false);

        // Step 4: The permit is still true — Alice could not revoke it
        assertTrue(staking.migrationPermits(migrator, alice));

        // Step 5: Admin re-grants MIGRATOR_ROLE to the same address at a later time
        vm.prank(admin);
        staking.grantRole(MIGRATOR_ROLE, migrator);

        // Step 6: The migrator can now migrate Alice's stakes without her fresh consent
        //         The stale permit from Step 1 is still active
        assertTrue(staking.migrationPermits(migrator, alice));
        assertTrue(staking.hasRole(MIGRATOR_ROLE, migrator));
    }
}
```

Result:

```js
Ran 1 test for test/IrrevocableMigrationPermit.t.sol:IrrevocableMigrationPermitTest
[PASS] test_UserCannotRevokeMigrationPermitWhenRoleRevoked() (gas: 74592)
Suite result: ok. 1 passed; 0 failed; 0 skipped; finished in 8.21ms (594.91µs CPU time)
```
