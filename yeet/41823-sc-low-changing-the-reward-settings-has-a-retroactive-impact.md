# #41823 \[SC-Low] Changing the reward settings has a retroactive impact

**Submitted on Mar 18th 2025 at 17:06:18 UTC by @pontifex for** [**Audit Comp | Yeet**](https://immunefi.com/audit-competition/audit-comp-yeet)

* **Report ID:** #41823
* **Report Type:** Smart Contract
* **Report severity:** Low
* **Target:** <https://github.com/immunefi-team/audit-comp-yeet/blob/main/src/Reward.sol>
* **Impacts:**
  * Theft of unclaimed yield

## Description

## Brief/Intro

Changing `RewardSettings.MAX_CAP_PER_WALLET_PER_EPOCH_FACTOR` variable has an retroactive impact and cause users with the same volume in the same epoch can receive different rewards just depending on the date of the claiming.

## Vulnerability Details

The `RewardSettings.MAX_CAP_PER_WALLET_PER_EPOCH_FACTOR` variable caps the max rewards a wallet can get per epoch. This variable is used for all epochs after the user's `lastClaimedForEpoch` epoch.

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

>>          uint256 maxClaimable = (epochRewards[epoch] / rewardsSettings.MAX_CAP_PER_WALLET_PER_EPOCH_FACTOR());
            uint256 claimable = (userShare * epochRewards[epoch]) / scalingFactor;

>>          if (claimable > maxClaimable) {
                claimable = maxClaimable;
            }

            totalClaimable += claimable;
        }

        return totalClaimable;
    }
```

Since the protocol owner can change the `MAX_CAP_PER_WALLET_PER_EPOCH_FACTOR` the `maxClaimable` reward can also be changed for users who have not claimed rewards yet.

```solidity
    function setYeetRewardsSettings(uint256 _maxCapPerWalletPerEpochFactor) external onlyOwner {
        require(
            _maxCapPerWalletPerEpochFactor >= 1,
            "YeetRewardSettings: maxCapPerWalletPerEpochFactor must be greater than 1"
        ); // 1/1 of the total rewards
        require(
            _maxCapPerWalletPerEpochFactor <= 100,
            "YeetRewardSettings: maxCapPerWalletPerEpochFactor must be less than 100"
        ); // 1/100 of the total rewards

        MAX_CAP_PER_WALLET_PER_EPOCH_FACTOR = _maxCapPerWalletPerEpochFactor;

        emit YeetRewardSettingsChanged(MAX_CAP_PER_WALLET_PER_EPOCH_FACTOR);
    }
```

Consider tracking the `MAX_CAP_PER_WALLET_PER_EPOCH_FACTOR` value for each epoch in a separate mapping and using the values for the `maxClaimable` reward calculation.

## Impact Details

The size of the group of users that can be impacted by the issue depends on the `MAX_CAP_PER_WALLET_PER_EPOCH_FACTOR`. The default value of the parameter is 30. This means that only 1/30 part of an epoch's emission can be claimed. So all users with `userShare` which exceeds 3,33% are capped by the parameter. This way changing the `MAX_CAP_PER_WALLET_PER_EPOCH_FACTOR` can retroactively increase or decrease the `maxClaimable` value for a big group of users for a sufficient value. This can cause unexpected rewards distribution, users losses and breaking tokenomic.

## References

<https://github.com/immunefi-team/audit-comp-yeet/blob/da15231cdefd8f385fcdb85c27258b5f0d0cc270/src/Reward.sol#L187\\>
<https://github.com/immunefi-team/audit-comp-yeet/blob/da15231cdefd8f385fcdb85c27258b5f0d0cc270/src/RewardSettings.sol#L41-L54>

## Proof of Concept

## Proof of Concept

1. Alice and Bob yeet during several epochs with the same `userYeetVolume`:

```solidity
    function addYeetVolume(address user, uint256 amount) external onlyYeetOwner {
        require(amount > 0, "Amount must be greater than 0");
        require(user != address(0), "Invalid user address");

        if (_shouldEndEpoch()) {
            _endEpoch();
        }

>>      userYeetVolume[currentEpoch][user] += amount;
        totalYeetVolume[currentEpoch] += amount;
    }
```

2. Since Alice and Bob have the same `userYeetVolume` they also have the same `userShare` per epoch and `claimable` amount respectively:

```solidity
    function getClaimableAmount(address user) public view returns (uint256) {
        uint256 totalClaimable;

        // Fixed-point arithmetic for more precision
        uint256 scalingFactor = 1e18;

        for (uint256 epoch = lastClaimedForEpoch[user] + 1; epoch < currentEpoch; epoch++) {
            if (totalYeetVolume[epoch] == 0) continue; // Avoid division by zero

            uint256 userVolume = userYeetVolume[epoch][user];
            uint256 totalVolume = totalYeetVolume[epoch];

>>          uint256 userShare = (userVolume * scalingFactor) / totalVolume;

            uint256 maxClaimable = (epochRewards[epoch] / rewardsSettings.MAX_CAP_PER_WALLET_PER_EPOCH_FACTOR());
>>          uint256 claimable = (userShare * epochRewards[epoch]) / scalingFactor;

            if (claimable > maxClaimable) {
                claimable = maxClaimable;
            }

            totalClaimable += claimable;
        }

        return totalClaimable;
    }
```

3. Suppose the current `MAX_CAP_PER_WALLET_PER_EPOCH_FACTOR` caps the `claimable` amount.
4. Alice claims rewards every day while Bob decided to claim much rarely.
5. Then the protocol decides to change the `MAX_CAP_PER_WALLET_PER_EPOCH_FACTOR` value and the new `maxClaimable` variable does not cap the `claimable` amount anymore.
6. Bob claims rewards for all previous epochs and receives more rewards than Alice.
