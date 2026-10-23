# #41283 \[SC-Low] Contract fails to deliver promised returns, due to changed \`MAX\_CAP\_PER\_WALLET\_PER\_EPOCH\_FACTOR\`

**Submitted on Mar 13th 2025 at 10:34:06 UTC by @Oxl33 for** [**Audit Comp | Yeet**](https://immunefi.com/audit-competition/audit-comp-yeet)

* **Report ID:** #41283
* **Report Type:** Smart Contract
* **Report severity:** Low
* **Target:** <https://github.com/immunefi-team/audit-comp-yeet/blob/main/src/Reward.sol>
* **Impacts:**
  * Contract fails to deliver promised returns, but doesn't lose value

## Description

**Description:**

In the documentation it is stated:

`There is a cap each day on what percentage of the daily emissions that an individual address can receive, set at 30%`

But in reality `MAX_CAP_PER_WALLET_PER_EPOCH_FACTOR` is initialized as `30` and acts as a denominator, so `maxClaimable` is only \~3.3% of epoch rewards, not 30%.

This is where the value gets set initially:

```solidity
contract RewardSettings is Ownable2Step {
    /// @dev The max rewards a wallet can get per epoch
    uint256 public MAX_CAP_PER_WALLET_PER_EPOCH_FACTOR;

    event YeetRewardSettingsChanged(uint256 indexed maxCapPerWalletPerEpoch);

    constructor() Ownable(msg.sender) {
        /// @dev this is in percentage, 1/10 of the total rewards
@>      MAX_CAP_PER_WALLET_PER_EPOCH_FACTOR = 30;
    }
```

This is how the value gets used:

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

@>          uint256 maxClaimable = (epochRewards[epoch] / rewardsSettings.MAX_CAP_PER_WALLET_PER_EPOCH_FACTOR()); // max cap initially set to 30, so `maxClaimable` will be ~3.3% of epoch rewards
            uint256 claimable = (userShare * epochRewards[epoch]) / scalingFactor;

            if (claimable > maxClaimable) {
                claimable = maxClaimable;
            }

            totalClaimable += claimable;
        }

        return totalClaimable;
    }
```

Of course the owner is able to change the value by calling `RewardSettings::setYeetRewardsSettings`, but still I am letting you know about it, because code and docs don't match.

Another issue arises due to the fact, that `MAX_CAP_PER_WALLET_PER_EPOCH_FACTOR` value is the same for every epoch when calculating `maxClaimable` amount.

Consider this scenario:

* current max cap is 30%, user yeets big amounts of BERA in several different epochs, expecting to receive 30% of total rewards for each epoch they participated in
* user waits a few days before claiming, which can happen due to various reasons (it is not mentioned in docs that users can get punished for waiting before they claim)
* owner changes the max cap to e.g. 10%. This can be done due to various reasons, such as community/protocol team decision, rewarding smaller game participants, etc.
* when user tries to claim, `claimable` will get set to `maxClaimable`, which is 10% of epoch rewards, even though the user has shares that represent 30%+ of the rewards
* user will get less rewards than they expected, due to owner being able to set the cap at any time and this cap being used for all previous epochs

**Impact:**

Users will get smaller than expected rewards.

**Recommended Mitigation:**

Consider implementing a mechanism where each epoch can have its own `MAX_CAP_PER_WALLET_PER_EPOCH_FACTOR` value, or at least make it clear in the documentation that this value can be changed at any time and that it affects all previous epochs.

## Proof of Concept

N/A
