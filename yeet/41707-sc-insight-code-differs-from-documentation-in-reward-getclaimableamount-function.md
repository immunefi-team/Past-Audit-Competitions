# #41707 \[SC-Insight] Code differs from documentation in \`Reward::getClaimableAmount\` function

**Submitted on Mar 17th 2025 at 17:32:38 UTC by @Oxl33 for** [**Audit Comp | Yeet**](https://immunefi.com/audit-competition/audit-comp-yeet)

* **Report ID:** #41707
* **Report Type:** Smart Contract
* **Report severity:** Insight
* **Target:** <https://github.com/immunefi-team/audit-comp-yeet/blob/main/src/Reward.sol>
* **Impacts:**

## Description

**Description:**

Statement from documentation:

`- There is a cap each day on what percentage of the daily emissions that an individual address can receive, set at 30%`\
`- Surplus token are burned`

Source: <https://docs.yeetit.xyz/yeet/yeet-game/mechanics>

Now take a look at the code:

```solidity
    function getClaimableAmount(address user) public view returns (uint256) {
        uint256 totalClaimable;

        uint256 scalingFactor = 1e18;

        for (uint256 epoch = lastClaimedForEpoch[user] + 1; epoch < currentEpoch; epoch++) {
            if (totalYeetVolume[epoch] == 0) continue;

            uint256 userVolume = userYeetVolume[epoch][user];
            uint256 totalVolume = totalYeetVolume[epoch];

            uint256 userShare = (userVolume * scalingFactor) / totalVolume;

            uint256 maxClaimable = (epochRewards[epoch] / rewardsSettings.MAX_CAP_PER_WALLET_PER_EPOCH_FACTOR());
            uint256 claimable = (userShare * epochRewards[epoch]) / scalingFactor;

            if (claimable > maxClaimable) {
                claimable = maxClaimable;
@>              // @audit info: surplus tokens are not burned, but in docs said otherwise
            }

            totalClaimable += claimable;
        }

        return totalClaimable;
    }
```

As you can see, `claimable` is set to `maxClaimable`, but the surplus tokens are not burned and they remain in the contract.

**Recommended Mitigation:**

Consider actually burning the surplus tokens or remove the `Surplus token are burned` statement from documentation, to avoid misleading users.

## Proof of Concept

**Proof of Concept:**

Surplus tokens are not burned, but documentation states that they are.

Source: <https://github.com/immunefi-team/audit-comp-yeet/blob/da15231cdefd8f385fcdb85c27258b5f0d0cc270/src/Reward.sol#L190>
