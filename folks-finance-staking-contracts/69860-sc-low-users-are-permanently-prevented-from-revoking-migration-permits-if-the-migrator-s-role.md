# 69860 sc low users are permanently prevented from revoking migration permits if the migrator s role is temporarily or permanently revoked

Submitted on Mar 17th 2026 at 06:39:10 UTC by @OadeHack for [Audit Comp | Folks Finance: Staking Contracts](https://immunefi.com/audit-competition/audit-comp-folks-finance-staking-contracts)

* **Report ID:** #69860
* **Report Type:** Smart Contract
* **Report severity:** Low
* **Target:** <https://github.com/Folks-Finance/folks-staking-contracts/blob/main/src/Staking.sol>
* **Impacts:**
  * Contract fails to deliver promised returns, but doesn't lose value

## Description

### Summary

The `setMigrationPermit` function tightly couples a user's ability to modify their own permission state with the target migrator's current active role. If a user grants permission to a Migrator, and the Admin subsequently revokes the `MIGRATOR_ROLE` from that contract, the user is completely blocked from revoking their permission. Their consent is permanently trapped at `true`. In decentralized systems, a user must always retain the unilateral right to revoke an allowance or permission; tying the ability to *revoke* consent to the protocol's external role management violates core user asset sovereignty.

### Description

*Why it bypasses the Tradeoff List:* The sponsor noted that `"State 'migrationPermits' may contain migrator which had its MIGRATOR_ROLE later revoked"`. This simply acknowledges that a user's `true` state persists after an admin action. However, the tradeoff list **does not** state that it is intentional for a user to be actively blocked from clearing their own state back to `false`.

In standard smart contract architecture (analogous to ERC20 `approve`), verifying the validity of a spender is required when *granting* an allowance, but should never be required when *revoking* an allowance.

In `Staking.sol`, the check is strictly enforced for both:

```solidity
function setMigrationPermit(address _migrator, bool _isMigrationPermitted) external {
    if (!hasRole(MIGRATOR_ROLE, _migrator)) revert MigratorNotFound(_migrator);

    migrationPermits[_migrator][msg.sender] = _isMigrationPermitted;
    emit MigrationPermitUpdated(_migrator, msg.sender, _isMigrationPermitted);
}
```

**The Logic Trap:**

{% stepper %}
{% step %}

### The Logic Trap

The Admin grants `MIGRATOR_ROLE` to `Migrator_V1`.
{% endstep %}

{% step %}
Alice explicitly opts-in by calling `setMigrationPermit(Migrator_V1, true)`.
{% endstep %}

{% step %}
The Admin revokes `MIGRATOR_ROLE` from `Migrator_V1` (e.g., due to a paused migration or deprecated contract).
{% endstep %}

{% step %}
Alice decides she no longer wishes to have an active approval associated with that address. She attempts to clear her state by calling `setMigrationPermit(Migrator_V1, false)`.
{% endstep %}

{% step %}
**The transaction reverts.** Because `Migrator_V1` no longer has the role, the `hasRole` check fails. Alice is permanently locked out of her own permission state.
{% endstep %}
{% endstepper %}

### Impact

* **Loss of User Sovereignty:** Users are mathematically denied the fundamental right to revoke approvals on their own assets.
* **Violation of Allowance Standards:** The contract incorrectly applies a "granting" security check to a "revoking" action, permanently trapping user state on the blockchain.

## Recommendation

Decouple the role verification from the revocation action. The contract should only enforce that the target is a valid migrator when the user is *granting* permission (`_isMigrationPermitted == true`). Users must always be allowed to turn their permission to `false`.

```diff
    function setMigrationPermit(address _migrator, bool _isMigrationPermitted) external {
-       if (!hasRole(MIGRATOR_ROLE, _migrator)) revert MigratorNotFound(_migrator);
+       if (_isMigrationPermitted) {
+           if (!hasRole(MIGRATOR_ROLE, _migrator)) revert MigratorNotFound(_migrator);
+       }

        migrationPermits[_migrator][msg.sender] = _isMigrationPermitted;
        emit MigrationPermitUpdated(_migrator, msg.sender, _isMigrationPermitted);
    }
```

## Proof of Concept

Drop the following test into `StakingTest.sol`. It mathematically proves that Alice is blocked from revoking her own permission once the admin revokes the migrator's role.

```solidity
function test_Low_CannotRevokePermitWhenRoleRevoked() public {
    // 1. Alice safely opts-in to the migration using the active migrator
    vm.prank(alice);
    staking.setMigrationPermit(migrator, true);

    // Sanity check: Alice's permit is successfully set to true
    assertTrue(staking.migrationPermits(migrator, alice), "Setup: Permit not set");

    // 2. Admin revokes the role (e.g., stopping the migration due to a bug in the proxy)
    vm.prank(admin);
    staking.revokeRole(keccak256("MIGRATOR"), migrator);

    // Sanity check: The migrator definitely lost its role
    assertFalse(staking.hasRole(keccak256("MIGRATOR"), migrator), "Setup: Role not revoked");

    // 3. Alice attempts to revoke her consent to protect her assets.
    vm.prank(alice);

    // This MUST revert due to the strict role check, trapping Alice's state
    vm.expectRevert(abi.encodeWithSelector(IStakingV1.MigratorNotFound.selector, migrator));
    staking.setMigrationPermit(migrator, false);

    // 4. Assert the Bug: Alice's permission is permanently trapped at TRUE
    assertTrue(staking.migrationPermits(migrator, alice), "Bug: Alice's permit was not trapped!");
}
```
