# 69345 sc low migration permits cannot be revoked after migrator role is revoked despite readme claiming revocation is possible at any time&#x20;

**Submitted on Mar 14th 2026 at 11:09:40 UTC by @Razkky for** [**Audit Comp | Folks Finance: Staking Contracts**](https://immunefi.com/audit-competition/audit-comp-folks-finance-staking-contracts)

* **Report ID:** #69345
* **Report Type:** Smart Contract
* **Report severity:** Low
* **Target:** <https://github.com/Folks-Finance/folks-staking-contracts/blob/main/src/Staking.sol>
* **Impacts:**

## Description

The README states that a user can revoke a migrator permission at any time by calling:

```solidity
staking.setMigrationPermit(migratorAddress, false);
```

Source:

* <https://github.com/Folks-Finance/folks-staking-contracts/blob/main/README.md#L140>

However, the implementation of `setMigrationPermit` first checks that the target address currently holds `MIGRATOR_ROLE`:

* <https://github.com/Folks-Finance/folks-staking-contracts/blob/main/src/Staking.sol#L77-L80>

```solidity
function setMigrationPermit(address _migrator, bool _isMigrationPermitted) external {
    if (!hasRole(MIGRATOR_ROLE, _migrator)) revert MigratorNotFound(_migrator);

    migrationPermits[_migrator][msg.sender] = _isMigrationPermitted;
    emit MigrationPermitUpdated(_migrator, msg.sender, _isMigrationPermitted);
}
```

Because of this, once a migrator loses `MIGRATOR_ROLE`, users can no longer clear the stored approval for that address. The old approval remains in `migrationPermits`, and if the same address later regains `MIGRATOR_ROLE`, the prior approval becomes active again.

This appears to be an informational / documentation mismatch rather than a direct asset-loss issue, since exploitation depends on privileged role management. Still, the current behavior is weaker than the user-facing statement that revocation is possible "at any time".

## Impact

Informational.

The issue does not directly cause theft or denial of funds on its own, but it means:

* users cannot actually revoke a previously granted migration approval after role revocation, despite documentation saying they can;
* stale approvals remain stored and can become effective again if the same address is re-granted `MIGRATOR_ROLE`.

## Proof of Concept

{% stepper %}
{% step %}

### Add the following test cases to `test/Staking.t.sol`.

```solidity
function test_CannotRevokePermitAfterMigratorRoleRevoked() public {
    vm.prank(alice);
    staking.setMigrationPermit(migrator, true);
    assertEq(staking.migrationPermits(migrator, alice), true);

    vm.prank(admin);
    staking.revokeRole(keccak256("MIGRATOR"), migrator);

    vm.expectRevert(abi.encodeWithSelector(IStakingV1.MigratorNotFound.selector, migrator));
    vm.prank(alice);
    staking.setMigrationPermit(migrator, false);

    assertEq(staking.migrationPermits(migrator, alice), true);
}

function test_StalePermitBecomesActiveAgainIfRoleReGranted() public {
    vm.prank(alice);
    staking.setMigrationPermit(migrator, true);

    vm.prank(admin);
    staking.revokeRole(keccak256("MIGRATOR"), migrator);

    vm.prank(admin);
    staking.grantRole(keccak256("MIGRATOR"), migrator);

    assertEq(staking.migrationPermits(migrator, alice), true);
}
```

{% endstep %}

{% step %}

### Run the following command:

```sh
forge test --match-test test_Audit_Migration_CannotRevokePermitAfterMigratorRoleRevoked

forge test --match-test test_Audit_Migration_RevokedMigratorPermitBecomesActiveAgainIfRoleReGranted
```

{% endstep %}

{% step %}

### Output

```bash
Ran 1 test for test/Staking.t.sol:StakingTest
[PASS] test_Audit_Migration_CannotRevokePermitAfterMigratorRoleRevoked() (gas: 59383)
Suite result: ok. 1 passed; 0 failed; 0 skipped; finished in 4.77ms (251.93µs CPU time)

Ran 1 test suite in 27.59ms (4.77ms CPU time): 1 tests passed, 0 failed, 0 skipped (1 total tests)
[⠊] Compiling...
No files changed, compilation skipped

Ran 1 test for test/Staking.t.sol:StakingTest
[PASS] test_Audit_Migration_RevokedMigratorPermitBecomesActiveAgainIfRoleReGranted() (gas: 60781)
Suite result: ok. 1 passed; 0 failed; 0 skipped; finished in 4.59ms (237.62µs CPU time)

Ran 1 test suite in 20.98ms (4.59ms CPU time): 1 tests passed, 0 failed, 0 skipped (1 total tests)
```

{% endstep %}

{% step %}

### Observe the results:

* `test_CannotRevokePermitAfterMigratorRoleRevoked` shows that once the migrator loses `MIGRATOR_ROLE`, the user can no longer revoke the stored approval because `setMigrationPermit(migrator, false)` reverts with `MigratorNotFound`.
* `test_StalePermitBecomesActiveAgainIfRoleReGranted` shows that the stale approval remains stored and becomes active again if the same address later regains `MIGRATOR_ROLE`.
  {% endstep %}
  {% endstepper %}
