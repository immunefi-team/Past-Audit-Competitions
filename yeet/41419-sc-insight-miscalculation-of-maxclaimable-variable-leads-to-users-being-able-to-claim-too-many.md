# #41419 \[SC-Insight] Miscalculation of \`maxClaimable\` variable leads to users being able to claim too many or too few reward tokens

**Submitted on Mar 15th 2025 at 02:14:26 UTC by @Exp10its for** [**Audit Comp | Yeet**](https://immunefi.com/audit-competition/audit-comp-yeet)

* **Report ID:** #41419
* **Report Type:** Smart Contract
* **Report severity:** Insight
* **Target:** <https://github.com/immunefi-team/audit-comp-yeet/blob/main/src/Reward.sol>
* **Impacts:**
  * Permanent freezing of unclaimed royalties
  * Theft of unclaimed royalties

## Description

## Brief/Intro

The `maxClaimable` variable in `getClaimableAmount()` is calculated incorrectly. As a result, users may be able to claim significantly more or less rewards than intended depending on the `RewardSettings` configuration.

## Vulnerability Details

The `maxClaimable` variable in `getClaimableAmount()` is calculated as follows:

```
uint256 maxClaimable = (epochRewards[epoch] /
                rewardsSettings.MAX_CAP_PER_WALLET_PER_EPOCH_FACTOR());
```

rewardsSettings.MAX\_CAP\_PER\_WALLET\_PER\_EPOCH\_FACTOR is the percentage of the total rewards in an epoch allowed to be distributed to one user. This is initialised to 30% by default aligning with the protocol docs linked below.

However, instead of multiplying by the value and dividing by 100, the contract currently divides by the value, leading to an incorrect `maxClaimable` value. More specifcally, the `maxClaimable` value becomes:\
(100/rewardsSettings.MAX\_CAP\_PER\_WALLET\_PER\_EPOCH\_FACTOR)%\
instead of:\
rewardsSettings.MAX\_CAP\_PER\_WALLET\_PER\_EPOCH\_FACTOR%

## Impact Details

As a result of this issue, users may be able to claim more or less funds than intended depending on configuration.

If rewardsSettings.MAX\_CAP\_PER\_WALLET\_PER\_EPOCH\_FACTOR is configured to be greater than 10, users owed funds above a certain threshold will permanently not be able to claim some of their funds.

For example, taking the default value of 30, the `maxClaimable` value becomes \~3.33% (100/30). Hence, a user having yeeted enough funds to be owed 10% of the rewards for the epoch will only be able to claim \~3.33% losing \~6.66% of their owed rewards.

If rewardsSettings.MAX\_CAP\_PER\_WALLET\_PER\_EPOCH\_FACTOR is configured to be less than 10, users will be able to claim more funds than they are owed, resulting in the theft of protocol funds or unclaimed funds of other users if the balance of the pool reduces below the total claimable rewards.

For example, taking the current onchain value of 5, the `maxClaimable` value becomes 20% (100/5). Hence, users can claim up to 20% of the rewards for the epoch, far exceeding the intended 5% limit.

## References

<https://docs.yeetit.xyz/yeet/yeet-game/mechanics>

## Proof of Concept

## Proof of Concept

The below two tests are derived from the existing test case (which only tests at 10%, an edge case where the current calculation method is equivalent to the correct one). The first test case demonstrates the issue at 30% whereas the second demonstrates it at 5%.

```
pragma solidity ^0.8.19;

import "forge-std/Test.sol";
import "../src/YeetGameSettings.sol";
import {Reward} from "../src/Reward.sol";
import {MockERC20} from "./mocks/MockERC20.sol";
import {RewardSettings} from "../src/RewardSettings.sol";

contract Reward_Rewards_CappedPerUserPerEpochOver10 is Test {
    Reward private reward;

    function setUp() public {
        MockERC20 token = new MockERC20("TEST", "TEST", 18);
        RewardSettings settings = new RewardSettings();
        settings.setYeetRewardsSettings(30);
        reward = new Reward(token, settings);
        reward.setYeetContract(address(this));

        token.mint(address(reward), 40_000_000 ether);
    }

    function test_shouldCapRewardsAt30Percent() public {
        reward.addYeetVolume(address(0xaa), 1000);

        skip(86402);
        reward.addYeetVolume(address(0xbb), 1000);

        uint256 totalRewards = reward.getEpochRewardsForCurrentEpoch();
        uint256 claimableReward = reward.getClaimableAmount(address(0xaa));

        // These logs are intended to better illustrate the miscalculation
        console.log("Claimable Reward: ", claimableReward); // the actual claimable amount as returned by the contract
        console.log(
            "~3.333% of total rewards: ",
            (3333 * totalRewards) / 100000
        ); // approximately the actual claimable amount as calculated
        console.log("30% of total rewards: ", (30 * totalRewards) / 100); // the expected claimable amount

        // This assertion should pass when the bug is resolved
        assertEq(
            claimableReward,
            56263285714285714285714 // 30% * (1_312_810 ether / 7)
        );
    }
}

contract Reward_Rewards_CappedPerUserPerEpochUnder10 is Test {
    Reward private reward;

    function setUp() public {
        MockERC20 token = new MockERC20("TEST", "TEST", 18);
        RewardSettings settings = new RewardSettings();
        settings.setYeetRewardsSettings(5);
        reward = new Reward(token, settings);
        reward.setYeetContract(address(this));

        token.mint(address(reward), 40_000_000 ether);
    }

    function test_shouldCapRewardsAt5Percent() public {
        reward.addYeetVolume(address(0xaa), 1000);

        skip(86402);
        reward.addYeetVolume(address(0xbb), 1000);

        uint256 totalRewards = reward.getEpochRewardsForCurrentEpoch();
        uint256 claimableReward = reward.getClaimableAmount(address(0xaa));

        // These logs are intended to better illustrate the miscalculation
        console.log("Claimable Reward: ", claimableReward); // the actual claimable amount as returned by the contract
        console.log("20% of total rewards: ", (20 * totalRewards) / 100); // the actual claimable amount as calculated
        console.log("5% of total rewards: ", (5 * totalRewards) / 100); // the expected claimable amount

        // This assertion should pass when the bug is resolved
        assertEq(
            claimableReward,
            9377214285714285714285 // 5% * (1_312_810 ether / 7)
        );
    }
}
```
