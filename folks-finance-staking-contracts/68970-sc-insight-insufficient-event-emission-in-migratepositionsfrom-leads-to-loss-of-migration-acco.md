# 68970 sc insight insufficient event emission in migratepositionsfrom leads to loss of migration accounting visibility

**Submitted on Mar 12th 2026 at 06:34:21 UTC by @ghufran for** [**Audit Comp | Folks Finance: Staking Contracts**](https://immunefi.com/audit-competition/audit-comp-folks-finance-staking-contracts)

* **Report ID:** #68970
* **Report Type:** Smart Contract
* **Report severity:** Insight
* **Target:** <https://github.com/Folks-Finance/folks-staking-contracts/blob/main/src/Staking.sol>

## Description

### Brief/Intro

The `migratePositionsFrom` function emits the `MigrateFrom` event without including the migrated stake and reward amounts, even though the function transfers these funds to the migrator. This incomplete event emission prevents off-chain services such as indexers, analytics platforms, and monitoring tools from accurately tracking the value of assets migrated from the contract. As a result, migration accounting may become inconsistent, making it difficult to audit fund movements, verify correct migration behavior, or detect anomalies during or after a migration process.

```solidity
function migratePositionsFrom(address user)
    external
    nonReentrant
    onlyRole(MIGRATOR_ROLE)
    returns (UserStake[] memory)
{
    if (!migrationPermits[msg.sender][user]) revert MigratorNotPermitted(msg.sender, user);

    UserStake[] memory stakes = userStakes[user];

    uint256 stakesToMigrateCount;
    // Count migratedStakes array size
    for (uint256 i = 0; i < stakes.length; i++) {
        if (stakes[i].claimedAmount + stakes[i].claimedReward < stakes[i].amount + stakes[i].reward) {
            stakesToMigrateCount++;
        }
    }
    UserStake[] memory migratedStakes = new UserStake[](stakesToMigrateCount);
    delete userStakes[user];

    uint256 migratedCount;
    uint256 unclaimedUserAmount;
    uint256 unclaimedUserRewards;
    for (uint256 i = 0; i < stakes.length; i++) {
        if (stakes[i].claimedAmount + stakes[i].claimedReward >= stakes[i].amount + stakes[i].reward) {
            userStakes[user].push(stakes[i]);
            continue;
        }
        unclaimedUserAmount += stakes[i].amount - stakes[i].claimedAmount;
        unclaimedUserRewards += stakes[i].reward - stakes[i].claimedReward;

        migratedStakes[migratedCount] = stakes[i];
        migratedCount++;
    }

    // The capUsed is intentionally not decremented for migrated positions. Migration is a terminal operation:
    // the manager will deactivate all staking periods or pauser will pause the contract before migration begins
    activeTotalStaked -= unclaimedUserAmount;
    activeTotalRewards -= unclaimedUserRewards;

    TOKEN.safeTransfer(msg.sender, unclaimedUserAmount + unclaimedUserRewards);

    emit MigrateFrom(msg.sender, user);
    return migratedStakes;
}
```

## Vulnerability Details

The `migratePositionsFrom` function is responsible for migrating a user's active staking positions to another contract that has the `MIGRATOR_ROLE`. During this process, the function calculates the total unclaimed principal and rewards for all active stakes of a user and transfers these tokens to the migrator. However, the function emits an event that does not include the amounts of tokens migrated, resulting in incomplete event logging.

```solidity
function migratePositionsFrom(address user)
    external
    nonReentrant
    onlyRole(MIGRATOR_ROLE)
    returns (UserStake[] memory)
{
    if (!migrationPermits[msg.sender][user]) revert MigratorNotPermitted(msg.sender, user);

    UserStake[] memory stakes = userStakes[user];

    uint256 stakesToMigrateCount;
    // Count migratedStakes array size
    for (uint256 i = 0; i < stakes.length; i++) {
        if (stakes[i].claimedAmount + stakes[i].claimedReward < stakes[i].amount + stakes[i].reward) {
            stakesToMigrateCount++;
        }
    }
    UserStake[] memory migratedStakes = new UserStake[](stakesToMigrateCount);
    delete userStakes[user];

    uint256 migratedCount;
    uint256 unclaimedUserAmount;
    uint256 unclaimedUserRewards;
    for (uint256 i = 0; i < stakes.length; i++) {
        if (stakes[i].claimedAmount + stakes[i].claimedReward >= stakes[i].amount + stakes[i].reward) {
            userStakes[user].push(stakes[i]);
            continue;
        }
        unclaimedUserAmount += stakes[i].amount - stakes[i].claimedAmount;
        unclaimedUserRewards += stakes[i].reward - stakes[i].claimedReward;

        migratedStakes[migratedCount] = stakes[i];
        migratedCount++;
    }

    // The capUsed is intentionally not decremented for migrated positions. Migration is a terminal operation:
    // the manager will deactivate all staking periods or pauser will pause the contract before migration begins
    activeTotalStaked -= unclaimedUserAmount;
    activeTotalRewards -= unclaimedUserRewards;

    TOKEN.safeTransfer(msg.sender, unclaimedUserAmount + unclaimedUserRewards);

    emit MigrateFrom(msg.sender, user);
    return migratedStakes;
}
```

During migration, the contract aggregates the total amounts to be transferred:

```solidity
unclaimedUserAmount += stakes[i].amount - stakes[i].claimedAmount;
unclaimedUserRewards += stakes[i].reward - stakes[i].claimedReward;
```

These totals represent the actual value moved from the contract to the migrator:

```solidity
TOKEN.safeTransfer(msg.sender, unclaimedUserAmount + unclaimedUserRewards);
```

However, the event emitted afterward does not include these values:

```solidity
emit MigrateFrom(msg.sender, user);
```

As a result, the event log only records the addresses involved in the migration but omits the critical financial data associated with the transfer.

## Impact Details

The incomplete event emission prevents off-chain services such as indexers, analytics platforms, and monitoring tools from accurately tracking the value of assets migrated from the contract.

## References

<https://github.com/Folks-Finance/folks-staking-contracts/blob/main/src/Staking.sol#L208>

## Mitigation

Replace the following event in `IStakingV1.sol`

```solidity
event MigrateFrom(address indexed migrator, address indexed user);
```

with

```solidity
event StakeMigrated(
    address indexed migrator,
    address indexed user,
    uint256 amount,
    uint256 reward
);
```

and replace this in `Staking.sol`

```solidity
emit MigrateFrom(msg.sender, user);
```

with

```solidity
emit StakeMigrated(msg.sender, user, unclaimedUserAmount, unclaimedUserRewards);
```

This will help in better tracking user amount and user rewards alongside for off-chain trackers and explorers

## Proof of Concept

Please add the below test `test_Migration_IncompleteEventEmission()` in `Staking.t.sol` and run `forge test --match-test test_Migration_IncompleteEventEmission` as it describes that the `MigrateFrom` event is missing critical financial data, making it impossible for off-chain services to track the actual value being migrated. It only tracks who performed the migration and whose stakes were migrated only.

```solidity
function test_Migration_IncompleteEventEmission() public {

    deal(address(token), address(staking), 1000 ether);
    deal(address(token), alice, 100 ether);
    uint8 periodIndex = addStakingPeriodByManager(50 ether, 20, 10, 5000, true);
    uint256 stakeAmount = 10 ether;
    approveAndStake(alice, periodIndex, stakeAmount, 20, 10, 5000, address(0));
    uint256 expectedReward = calculateReward(stakeAmount, 20, 5000);
    vm.prank(alice);
    staking.setMigrationPermit(migrator, true);
    vm.recordLogs();
    vm.prank(migrator);
    IStakingV1.UserStake[] memory migratedStakes = staking.migratePositionsFrom(alice);
    Vm.Log[] memory entries = vm.getRecordedLogs();
    bool eventFound = false;
    for (uint256 i = 0; i < entries.length; i++) {
        if (entries[i].topics[0] == keccak256("MigrateFrom(address,address)")) {
            eventFound = true;
            (address emittedMigrator, address emittedUser) = abi.decode(entries[i].data, (address, address));
            assertEq(emittedMigrator, migrator);
            assertEq(imedUser, alice);
            assertEq(entries[i].data.length, 64, "Event data should be 64 bytes for two addresses");
            vm.expectRevert();
            abi.decode(entries[i].data, (address, address, uint256, uint256));
        }
    }
    assertTrue(eventFound, "MigrateFrom event should have been emitted");
}
```
