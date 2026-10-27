# #41664 \[SC-Low] Users may receive fewer rewards due to the change in reward limits

**Submitted on Mar 17th 2025 at 12:22:20 UTC by @zaevlad for** [**Audit Comp | Yeet**](https://immunefi.com/audit-competition/audit-comp-yeet)

* **Report ID:** #41664
* **Report Type:** Smart Contract
* **Report severity:** Low
* **Target:** <https://github.com/immunefi-team/audit-comp-yeet/blob/main/src/Reward.sol>
* **Impacts:**
  * Permanent freezing of unclaimed yield

## Description

## Brief/Intro

The max reward amount is based on the calculation of epoch rewards and MAX\_CAP\_PER\_WALLET\_PER\_EPOCH\_FACTOR value. If its value will be changed, users may get less rewards than expected.

## Vulnerability Details

After every game users can claim thier rewards by clalling claim() in the REwards contract.

The function defines the max reward amount and max aount that can be claimed by a user:

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

@>          uint256 maxClaimable = (epochRewards[epoch] / rewardsSettings.MAX_CAP_PER_WALLET_PER_EPOCH_FACTOR());
            uint256 claimable = (userShare * epochRewards[epoch]) / scalingFactor;

            if (claimable > maxClaimable) {
@>             claimable = maxClaimable;
            }

            totalClaimable += claimable;
        }

        return totalClaimable;
    }
```

MAX\_CAP\_PER\_WALLET\_PER\_EPOCH\_FACTOR is a base value for each epoch that can be changed in the RewardsSettings contract. The default value is 30. This means that the maximum claimable amount for each user cannot exceed 30% of the total amount for the epoch.

From a functional point of view, there is no mechanism in the code to carry over 'surpluses' to the next epoch. So any unclaimed rewards are lost.

Also there is no requirement for the user to call the claim() function after the epoch ends. He can wait for several epochs to end to get rewards from all of them.

If MAX\_CAP\_PER\_WALLET\_PER\_EPOCH\_FACTOR is changed to a lower value, such as 25, users will lose a portion of their rewards forever.

## Impact Details

Users may receive fewer rewards due to the change in reward limits, and these rewards will be lost.

## References

<https://github.com/immunefi-team/audit-comp-yeet/blob/main/src/RewardSettings.sol#L41\\>
<https://github.com/immunefi-team/audit-comp-yeet/blob/main/src/Reward.sol#L187>

## Proof of Concept

## Proof of Concept

Let's make a simple example:

```
Alice: 500 YEET
Bob: 300 YEET
Rob: 200 YEET

epochRewards[epoch] = 1000 tokens
MAX_CAP_PER_WALLET_PER_EPOCH_FACTOR = 2 (maxClaimable = 500)
```

For Alice the amount of rewards can be the max amount and she will not loose anything:

```
Alice:
userShare = (500 * 1e18) / 1000 = 0.5e18
claimable = (0.5e18 * 1000) / 1e18 = 500 tokens
maxClaimable = 1000 / 2 = 500 tokens

Finaly Alice gets 500 tokens 
```

However if the MAX\_CAP\_PER\_WALLET\_PER\_EPOCH\_FACTOR will be cahnged to 3:

```
maxClaimable = 1000 / 3 = 333 tokens

Alice will get 333 tokens instead of 500. So she looses 167 reward tokens.
```

And this amount will be lost forever. If she's trying to claim rewards for more than one epoch, the amount lost may be higher.

Consider storing the MAX\_CAP\_PER\_WALLET\_PER\_EPOCH\_FACTOR separately for each epoch.
