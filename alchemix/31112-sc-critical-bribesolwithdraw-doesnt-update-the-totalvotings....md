# 31112 - \[SC - Critical] Bribesolwithdraw doesnt update the totalVotings...

Submitted on May 12th 2024 at 23:48:52 UTC by @crazy\_squirrel for [Boost | Alchemix](https://immunefi.com/bounty/alchemix-boost/)

Report ID: #31112

Report type: Smart Contract

Report severity: Critical

Target: <https://github.com/alchemix-finance/alchemix-v2-dao/blob/main/src/Bribe.sol>

Impacts:

* The totalVoting tracker can be inflated

## Description

## Brief/Intro

> Bribe.sol The Bribe.sol contract distributes bribes for a given Gauge. Each Gauge had a Bribe contract attached to it, and each Bribe can accept multiple (up to 16) different tokens as bribes. During each epoch, veALCX stakers can collect bribes for a given gauge if they voted on that gauge in the previous epoch.
>
> * the total amount of bribe `b` on pool `p` claimable by a veALCX NFT with token-ID `i` during a given epoch `n` is equal to the proportion of total veALCX power that that NFT used to vote on pool `p`

## Vulnerability Details

In the `Bribe.sol` contract, the `deposit` function increases the `totalVoting` tracker's amount. However during the `withdraw`al's execution, the `totalVoting`'s value is **NOT** decreased.

This missing `totalVoting` decrease directly impacts the voting power checkpointing and calculations by writing the wrong amount:

```solidity
function _writeVotingCheckpoint() internal {
        uint256 _nCheckPoints = votingNumCheckpoints;
        uint256 _timestamp = block.timestamp;

        if (_nCheckPoints > 0 && votingCheckpoints[_nCheckPoints - 1].timestamp == _timestamp) {
            votingCheckpoints[_nCheckPoints - 1].votes = totalVoting;
        } else {
            votingCheckpoints[_nCheckPoints] = VotingCheckpoint(_timestamp, totalVoting);
            votingNumCheckpoints = _nCheckPoints + 1;
        }
    }
```

## Impact Details

Medium.

By impacting the variable used in the voting result calculations, this

## References

```solidity
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
```

```solidity
function withdraw(uint256 amount, uint256 tokenId) external {
        require(msg.sender == voter);

        totalSupply -= amount;
        balanceOf[tokenId] -= amount;

        _writeCheckpoint(tokenId, balanceOf[tokenId]);
        _writeSupplyCheckpoint();

        emit Withdraw(msg.sender, tokenId, amount);
    }
```

## Proof of Concept

See and compare the following PoC's `console.log`s.

These will be:

```bash
Running 1 test for src/test/PassthroughGauge.t.sol:PassthroughGaugeTest
[PASS] testPassthroughGaugeRewards() (gas: 1406037)
Logs:
  totalVoting BEFORE the withdrawal 300000000000000000000
  totalVoting AFTER the withdrawal 300000000000000000000
```

Priot to running `forge test --match-contract PassthroughGauge -vvvv --fork-url https://eth-mainnet.alchemyapi.io/v2/{YOUR_ALCHEMY_API_KEY}`, paste the following code to the `src/test/PassthroughGauge.t.sol` file:

```solidity
// SPDX-License-Identifier: GPL-3
pragma solidity ^0.8.15;

import "./BaseTest.sol";

contract PassthroughGaugeTest is BaseTest {
    uint256 snapshotWeek = 17120807;

    uint256 platformFee = 400; // 4%
    uint256 DENOMINATOR = 10_000; // denominates weights 10_000 = 100%

    function setUp() public {
        setupContracts(block.timestamp);
    }

    // Rewards should be passed through to external gauges
    // Add tests for gauges as they are added
    function testPassthroughGaugeRewards() public {
        uint256 tokenId = createVeAlcx(admin, TOKEN_1, MAXTIME, false);

        hevm.startPrank(admin);

        uint256 period = minter.activePeriod();

        hevm.warp(period);

        assertEq(sushiGauge.rewardToken(), address(alcx), "incorrect reward token");
        uint256 sushiBalanceBefore = alcx.balanceOf(sushiPoolAddress);

        address[] memory pools = new address[](1);
        pools[0] = sushiPoolAddress;
        uint256[] memory weights = new uint256[](1);
        weights[0] = 5000;

        // Move forward epoch
        hevm.warp(period + 1 weeks);

        IBribe bribeTest = IBribe(voter.bribes(voter.gauges(pools[0])));

        vm.stopPrank();

        vm.startPrank(address(voter));

        bribeTest.deposit(300 * 1e18, tokenId);

        for (uint256 i = 0; i < pools.length; i++) {
            console.log("totalVoting BEFORE the withdrawal", bribeTest.totalVoting());
        }

        // voter.vote(tokenId, pools, weights, 0);


        bribeTest.withdraw(300 * 1e18, tokenId);

        for (uint256 i = 0; i < pools.length; i++) {
            console.log("totalVoting AFTER the withdrawal", bribeTest.totalVoting());
        }
    }
}
```

**Notice that the `totalVoting` amount HAS NOT decreased after the withdrawal.**
