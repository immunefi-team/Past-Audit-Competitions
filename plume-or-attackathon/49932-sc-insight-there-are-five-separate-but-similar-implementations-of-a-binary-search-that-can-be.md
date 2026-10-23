# 49932 sc insight there are five separate but similar implementations of a binary search that can be condensed into one function

* Submitted on Jul 20th 2025 at 15:52:52 UTC by @Vanshika for [Attackathon | Plume Network](https://immunefi.com/audit-competition/plume-network-attackathon)
* Report ID: #49932
* Report Type: Smart Contract
* Report severity: Insight
* Target: <https://github.com/immunefi-team/attackathon-plume-network/blob/main/plume/src/lib/PlumeRewardLogic.sol>

## Brief / Intro

In the Plume repo, there are five separate implementations of a binary search, three of them in the same file. These serve a similar purpose and can be condensed into one function.

## Vulnerability Details

There are five separate implementations of a binary search in the Plume repo, three of them in the same file, two of them fully identical except for their input values. These are in the following functions:

* `RewardsFacet.getUserLastCheckpointIndex()`
* `PlumeRewardLogic.findCheckpointIndex()`
* `PlumeRewardLogic.findRewardRateCheckpointIndexAtOrBefore()`
* `PlumeRewardLogic.findCommissionCheckpointIndexAtOrBefore()`
* `Raffle.handleWinnerSelection()`

The binary search serves the same purpose in each function and has similar implementations throughout. `PlumeRewardLogic.findRewardRateCheckpointIndexAtOrBefore()` and `PlumeRewardLogic.findCommissionCheckpointIndexAtOrBefore()` are identical except for their input. This logic should be distilled into its own function so it doesn't have to be repeated. One of the implementations of the code is shown below:

```solidity
function findCommissionCheckpointIndexAtOrBefore(
        PlumeStakingStorage.Layout storage $,
        uint16 validatorId,
        uint256 timestamp
    ) internal view returns (uint256) {
        PlumeStakingStorage.RateCheckpoint[] storage checkpoints = $.validatorCommissionCheckpoints[validatorId];
        uint256 len = checkpoints.length;
        if (len == 0) {
            return 0; // No checkpoints, caller uses current validator.commission
        }

        uint256 low = 0;
        uint256 high = len - 1;
        uint256 ans = 0;
        bool foundSuitable = false;
        //BINARY SEARCH
        while (low <= high) {
            uint256 mid = low + (high - low) / 2;

            if (checkpoints[mid].timestamp <= timestamp) {
                ans = mid;
                foundSuitable = true;
                low = mid + 1;
            } else {
                if (mid == 0) {
                    break;
                }
                high = mid - 1;
            }
        }
        return ans;
    }
```

## Impact Details

Repeating identical logic multiple times throughout the contract increases contract size and interferes with readability and maintainability of the code.

## Proof of Concept

The binary search logic can be separated into its own function to improve code readability and size. E.g., for `PlumeRewardLogic.sol`, this can be done as below.

```solidity
function binarySearch(PlumeStakingStorage.RateCheckpoint[] memory checkpoints) returns (uint256 ans, bool foundAns) {};
```

(End of report)
