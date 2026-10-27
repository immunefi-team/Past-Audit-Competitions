# 69382 sc low irrevocable migration permit users cannot revoke permit after migrator role revocation

**Submitted on Mar 14th 2026 at 15:01:55 UTC by @M1S00 for** [**Audit Comp | Folks Finance: Staking Contracts**](https://immunefi.com/audit-competition/audit-comp-folks-finance-staking-contracts)

* **Report ID:** #69382
* **Report Type:** Smart Contract
* **Report severity:** Low
* **Target:** <https://github.com/Folks-Finance/folks-staking-contracts/blob/main/src/Staking.sol>
* **Impacts:**
  * Griefing (e.g. no profit motive for an attacker, but damage to the users or the protocol)

## Description

The `setMigrationPermit()` function in `Staking.sol` enforces a `hasRole(MIGRATOR_ROLE, _migrator)` check on every call, including when a user is trying to **revoke** their previously granted permit. If the admin revokes `MIGRATOR_ROLE` from a migrator address (e.g., due to suspected key compromise), all users who previously granted that migrator a permit are permanently locked in. They cannot call `setMigrationPermit(migrator, false)` because the function reverts with `MigratorNotFound`. This directly violates the contract's documented design intent that *"migrator should not be able to migrate stakes if not approved by user"*, since users lose the ability to withdraw their approval. If the role is ever re-granted to that address (governance error, CREATE2 redeployment, or admin key compromise), the stale permit enables unauthorized migration of user funds.

## Vulnerable Code

**File:** `src/Staking.sol`, `setMigrationPermit()` (Lines 85-90)

```solidity
function setMigrationPermit(address _migrator, bool _isMigrationPermitted) external {
    if (!hasRole(MIGRATOR_ROLE, _migrator)) revert MigratorNotFound(_migrator);

    migrationPermits[_migrator][msg.sender] = _isMigrationPermitted;
    emit MigrationPermitUpdated(_migrator, msg.sender, _isMigrationPermitted);
}
```

## Why This Is Vulnerable

The `hasRole` check is applied unconditionally. It does not distinguish between granting (`true`) and revoking (`false`) a permit. When the admin revokes `MIGRATOR_ROLE` from a migrator, the role check causes all subsequent calls to `setMigrationPermit` targeting that address to revert, regardless of whether the user is trying to grant or revoke. This means a user's previously granted `migrationPermits[migrator][user] = true` becomes permanently stuck in storage with no way to set it back to `false`. The user's own safety mechanism, the ability to control who can migrate their funds, is completely broken.

## Impact

Users who previously approved a migrator permanently lose the ability to revoke that approval once the migrator's role is removed by admin. This creates a latent, irrevocable authorization in contract storage. If the `MIGRATOR_ROLE` is ever re-granted to the same address, whether through a governance mistake, a deliberate re-assignment, a CREATE2 redeployment to the same address, or admin key compromise, the migrator can immediately call `migratePositionsFrom(user)` and transfer all of the user's unclaimed staked funds and rewards without the user's current consent. This is a direct violation of the protocol's documented trust model where users must explicitly approve migrations.

## Attack Scenario

1. **Alice** stakes 10 ETH into a staking period
2. **Alice** calls `setMigrationPermit(migrator, true)`, grants the migrator permission to migrate her position
3. **Admin** revokes `MIGRATOR_ROLE` from the migrator (e.g., migrator key suspected compromised)
4. **Alice** tries to protect herself by revoking: `setMigrationPermit(migrator, false)` → **REVERTS** with `MigratorNotFound`
5. **Alice's permit is stuck as `true`**, irrevocable, permanently stored in contract state
6. If `MIGRATOR_ROLE` is ever re-granted to that address, the migrator can migrate Alice's funds without her current consent

## Proof of Concept

Paste the following test inside the `StakingTest` contract in `test/Staking.t.sol`:

```solidity
/// @notice Migration permit becomes irrevocable when migrator role is revoked
/// User cannot call setMigrationPermit(migrator, false) because hasRole check reverts
/// If the role is ever re-granted, the stale permit allows unauthorized migration
function test_Exploit_IrrevocableMigrationPermit() public {
    deal(address(token), address(staking), 1000 ether);
    uint8 periodIndex = addStakingPeriodByManager(100 ether, 30 days, 10 days, 1000, true);

    // Step 1: Alice stakes 10 ether
    deal(address(token), alice, 10 ether);
    vm.startPrank(alice);
    token.approve(address(staking), 10 ether);
    staking.stake(
        periodIndex,
        10 ether,
        IStakingV1.StakeParams({
            maxStakingDurationSeconds: 30 days,
            maxUnlockDurationSeconds: 10 days,
            minAprBps: 1000,
            referrer: address(0)
        })
    );
    vm.stopPrank();

    // Step 2: Alice grants migration permit to the migrator
    vm.prank(alice);
    staking.setMigrationPermit(migrator, true);
    assertEq(staking.migrationPermits(migrator, alice), true);

    // Step 3: Admin revokes MIGRATOR_ROLE (e.g., migrator key suspected compromised)
    vm.prank(admin);
    staking.revokeRole(keccak256("MIGRATOR"), migrator);
    assertEq(staking.hasRole(keccak256("MIGRATOR"), migrator), false);

    // Step 4: Alice tries to revoke her migration permit, REVERTS!
    // She CANNOT protect herself because setMigrationPermit checks hasRole first
    vm.prank(alice);
    vm.expectRevert(abi.encodeWithSelector(IStakingV1.MigratorNotFound.selector, migrator));
    staking.setMigrationPermit(migrator, false);

    // The permit is STUCK as true, irrevocable
    assertEq(staking.migrationPermits(migrator, alice), true);

    // Step 5: If admin ever re-grants MIGRATOR_ROLE to the same address
    // (governance error, re-deployment at same address, or admin compromise)
    vm.prank(admin);
    staking.grantRole(keccak256("MIGRATOR"), migrator);

    // Step 6: Migrator can now migrate Alice's funds WITHOUT her current consent
    // The stale permit from before the role revocation is still active
    vm.prank(migrator);
    IStakingV1.UserStake[] memory migratedStakes = staking.migratePositionsFrom(alice);

    // Alice's funds have been transferred to the migrator address
    assertEq(migratedStakes.length, 1);
    assertEq(migratedStakes[0].amount, 10 ether);
    // Alice has no more active stakes
    IStakingV1.UserStake[] memory aliceStakes = staking.getUserStakes(alice);
    assertEq(aliceStakes.length, 0);
}
```

### Run

```bash
forge test --mt test_Exploit_IrrevocableMigrationPermit -vvv
```

### Output

```
┌──(m1s0㉿M1S0)-[~/Desktop/BugBounty/Web3/Smart Contract/folks-staking-contracts]
└─$ forge test --mt test_Exploit_IrrevocableMigrationPermit
[⠊] Compiling...
No files changed, compilation skipped

Ran 1 test for test/Staking.t.sol:StakingTest
[PASS] test_Exploit_IrrevocableMigrationPermit() (gas: 746664)
Suite result: ok. 1 passed; 0 failed; 0 skipped; finished in 3.10ms (1.48ms CPU time)

Ran 1 test suite in 11.40ms (3.10ms CPU time): 1 tests passed, 0 failed, 0 skipped (1 total tests)
┌──(m1s0㉿M1S0)-[~/Desktop/BugBounty/Web3/Smart Contract/folks-staking-contracts]
```

## Mitigation

The `hasRole` check should only be enforced when **granting** a permit, not when revoking. Users must always be able to revoke their own permissions regardless of the migrator's current role status:

```solidity
function setMigrationPermit(address _migrator, bool _isMigrationPermitted) external {
    if (_isMigrationPermitted && !hasRole(MIGRATOR_ROLE, _migrator)) 
        revert MigratorNotFound(_migrator);

    migrationPermits[_migrator][msg.sender] = _isMigrationPermitted;
    emit MigrationPermitUpdated(_migrator, msg.sender, _isMigrationPermitted);
}
```

This ensures:

* **Granting** (`true`): Still validates the migrator has the role, no change in security
* **Revoking** (`false`): Always succeeds, users can always protect themselves
