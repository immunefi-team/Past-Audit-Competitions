# #42462 \[SC-Low] Potential loss of unclaimed rewards due to updating setting \`MAX\_CAP\_PER\_WALLET\_PER\_EPOCH\_FACTOR\`

**Submitted on Mar 24th 2025 at 06:10:31 UTC by @trtrth for** [**Audit Comp | Yeet**](https://immunefi.com/audit-competition/audit-comp-yeet)

* **Report ID:** #42462
* **Report Type:** Smart Contract
* **Report severity:** Low
* **Target:** <https://github.com/immunefi-team/audit-comp-yeet/blob/main/src/Reward.sol>
* **Impacts:**
  * Permanent freezing of unclaimed yield

## Description

## Brief/Intro

The claimable reward amount of an user is dependant on the global setting `MAX_CAP_PER_WALLET_PER_EPOCH_FACTOR`, which is not the max cap at the specified epoch. This can cause user's unclaimed reward unexpectedly changed when the global setting `MAX_CAP_PER_WALLET_PER_EPOCH_FACTOR` is updated.

## Vulnerability Details

The function `Reward::getClaimableAmount()` calculates the amount of tokens that a user can claim, such that it calculates all rewards from user's last claimed epoch to current epoch considering the max cap per wallet per epoch.

```solidity
    function getClaimableAmount(address user) public view returns (uint256) {
        uint256 totalClaimable;

        // Fixed-point arithmetic for more precision
        uint256 scalingFactor = 1e18;

        for (uint256 epoch = lastClaimedForEpoch[user] + 1; epoch < currentEpoch; epoch++) {
            if (totalYeetVolume[epoch] == 0) continue; // Avoid division by zero

            uint256 userVolume = userYeetVolume[epoch][user];
            uint256 totalVolume = totalYeetVolume[epoch];

            uint256 userShare = (userVolume * scalingFactor) / totalVolume;

>> consider max cap >>            uint256 maxClaimable = (epochRewards[epoch] / rewardsSettings.MAX_CAP_PER_WALLET_PER_EPOCH_FACTOR());
            uint256 claimable = (userShare * epochRewards[epoch]) / scalingFactor;

            if (claimable > maxClaimable) {
                claimable = maxClaimable;
            }

            totalClaimable += claimable;
        }

        return totalClaimable;
    }
```

Here, there exists scenario that user can lose rewards when the setting `MAX_CAP_PER_WALLET_PER_EPOCH_FACTOR` is updated.

For example:

1. The current `MAX_CAP_PER_WALLET_PER_EPOCH_FACTOR = 5`, meaning the wallet cap is 20%
2. At epoch 10, `Alice` has 25% volume shares and with the current max cap setting, `Alice` can claim up to 20% epoch reward
3. At epoch 20, the Yeet governance decides to update the setting `MAX_CAP_PER_WALLET_PER_EPOCH_FACTOR` to new value `10`, meaning the wallet cap is 10%.
4. Assume that `Alice` has not claimed rewards since the epoch 10. Now at epoch 30, `Alice` decides to claim rewards. Now, the claimable reward amount at epoch 10 is only 10% of the epoch reward. In other words, `Alice` loses 10% of the epoch reward at epoch 10.

Indeed, updating the max cap setting should have effects for the epochs in the future, not the epochs in the past.

## Impact Details

* Users can lose unclaimed rewards in past epochs when `MAX_CAP_PER_WALLET_PER_EPOCH_FACTOR` is updated to a higher value (decreasing max cap per wallet per epoch)
* It can be unfair for users who claims rewards before the setting is updated and the ones who claims rewards after the setting is updated.

## References

<https://github.com/immunefi-team/audit-comp-yeet/blob/da15231cdefd8f385fcdb85c27258b5f0d0cc270/src/Reward.sol#L173C1-L198C6>

## Proof of Concept

## Proof of Concept

* Add the test function below to file `test/Reward.Test.sol`, and within the test contract `Reward_runFor208weeks`.

```solidity
    function test_lossOfUnclaimRewards() public {
      // max cap 20%
      reward.rewardsSettings().setYeetRewardsSettings(5);

      address alice = makeAddr('alice');

      // Alice takes 100% volume shares
      reward.addYeetVolume(alice, 1000);

      // this is to end epoch and transition to next epoch
      skip(1 days + 1);
      reward.addYeetVolume(alice, 1000);

      // claimable amount before updating setting
      uint claimableBefore = reward.getClaimableAmount(alice);

      skip(1 days + 1);

      // update setting -> cap is 10%
      reward.rewardsSettings().setYeetRewardsSettings(10);

      // claimable amount after updating setting
      uint claimableAfter = reward.getClaimableAmount(alice);

      assertEq(claimableAfter, claimableBefore);
    }
```

* Run the test and console shows

```bash
Failing tests:
Encountered 1 failing test in test/Reward.Test.sol:Reward_runFor208weeks
[FAIL: assertion failed: 18754428571428571428571 != 37508857142857142857142] test_lossOfUnclaimRewards() (gas: 202467)
```

It means that the claimable is reduced after updating the global setting `MAX_CAP_PER_WALLET_PER_EPOCH_FACTOR` to a lower value
