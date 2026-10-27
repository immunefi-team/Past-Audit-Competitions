# #37391 \[SC-High] Early Reward Accrual Undermines Validator Group Performance Incentives

**Submitted on Dec 3rd 2024 at 18:24:38 UTC by @jovi for** [**Audit Comp | Celo**](https://immunefi.com/audit-competition/audit-comp-celo)

* **Report ID:** #37391
* **Report Type:** Smart Contract
* **Report severity:** High
* **Target:** <https://github.com/celo-org/celo-monorepo/blob/release/core-contracts/12/packages/protocol/contracts-0.8/common/EpochManager.sol>
* **Impacts:**
  * Manipulation of governance voting result deviating from voted outcome and resulting in a direct change from intended effect of original results
  * Temporary freezing of funds
  * Misaligned performance incentives resulting in diminished validator participation or performance.

## Description

## Vulnerability Details

The `EpochManager` contract allows for premature accrual of epoch rewards, potentially undermining the incentive structure designed to maintain high-quality validator performance throughout an epoch. Due to the lack of restrictions on when `setToProcessGroups()` can be called within an epoch. This function, which calculates and locks in rewards based on the current group score, can be invoked immediately after a new epoch starts. The key issues are:

1. **No Cooldown Period**: Unlike starting a new epoch, there's no enforced cooldown or waiting period for calling `setToProcessGroups()`. This allows for premature reward accrual.
2. **Reward Calculation Timing**: Rewards are calculated based on the group's score at the time `setToProcessGroups()` is called, which can be at the very beginning of an epoch.
3. **Misaligned Incentives**: Once rewards are calculated and locked in, a group's performance for the remainder of the epoch becomes irrelevant to their rewards for that epoch.
4. **Potential for Exploitation**: Groups intending to leave or underperform can exploit this by ensuring `setToProcessGroups()` is called early, then behaving poorly for the rest of the epoch without financial consequences.

The relevant code snippet from `EpochManager.sol`:

```solidity
function setToProcessGroups() external {
    require(isOnEpochProcess(), "Epoch process is not started");
}
```

This function can be called by anyone, at any time during an epoch, as long as the epoch process has started. The lack of additional timing constraints allows for the described vulnerability.

## Impact

This vulnerability could lead to:

1. Reduced validator performance and network security
2. Misaligned incentives for validator groups
3. Potential exploitation by groups planning to quit or underperform

## Proof of Concept

## Proof of Concept

The following proof of concept rely mainly on executing function calls at specific periods and the vulnerability lies on the context in which those calls are invoked. Furthermore, all the functions involved are permissionless and only have epoch statuses/timing to fulfill. For this reason, I have opted to avoid making a coded POC as such is already available at the EpochManager contract tests at `EpochManager.t.sol` and the most important aspect of this submission is the semantical repercussions for the governance process.

1. An epoch begins with `startNextEpochProcess()`.
2. Immediately after, anyone can call `setToProcessGroups()`. Which will in turn calculate the epoch rewards for a group based on the group's score:

```solidity
 function setToProcessGroups() external {
    require(isOnEpochProcess(), "Epoch process is not started");
...
      if (processedGroups[group] == 0) {
        ...
        uint256 groupScore = scoreReader.getGroupScore(group);
        // We need to precompute epoch rewards for each group since computation depends on total active votes for all groups.
        uint256 epochRewards = election.getGroupEpochRewardsBasedOnScore(
          group,
          _epochProcessing.totalRewardsVoter,
          groupScore
        );
        processedGroups[group] = epochRewards == 0 ? type(uint256).max : epochRewards;
      }
    }
  }
```

3. This locks in rewards based on the group's score at that specific moment of the epoch.
4. The group's performance for the rest of the epoch becomes irrelevant to their rewards as it is locked in for the whole duration of the epoch.
5. The contract will wait till the condition for `isTimeForNextEpoch()` is fulfilled before beginning a new epoch. In practice, this means a preemptive execution of `setToProcessGroups()` can make an epoch's rewards be distributed to groups regardless of how well they perform.
