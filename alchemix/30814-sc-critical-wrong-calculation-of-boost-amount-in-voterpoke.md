# 30814 - \[SC - Critical] Wrong calculation of boost amount in Voterpoke

Submitted on May 6th 2024 at 16:29:36 UTC by @cryptoticky for [Boost | Alchemix](https://immunefi.com/bounty/alchemix-boost/)

Report ID: #30814

Report type: Smart Contract

Report severity: Critical

Target: <https://github.com/alchemix-finance/alchemix-v2-dao/blob/main/src/Voter.sol>

Impacts:

* Contract fails to deliver promised returns, but doesn't lose value

## Description

## Brief/Intro

The Voter.poke function votes again with the same weights, effectively renewing the previous vote. However, the boost value reflected in the previous weights will not be taken into account when the poke function is called. The boost is always 0.

## Vulnerability Details

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

`// Previous boost will be taken into account with weights being pulled from the votes mapping` Will the calculation be done like this explanation? The answer is `No`.

```
_weights[i] = votes[_tokenId][_poolVote[i]];
```

This only reflects the previous weight and does not reproduce the previous boost.

## Recommendation

In order to reproduce the boost in the Poke function, it is recommended to record the previous boost amount.

## Proof of Concept

```
// SPDX-License-Identifier: GPL-3
pragma solidity ^0.8.15;

import "./BaseTest.sol";

contract BugPokePoC is BaseTest {

    function setUp() public {
        setupContracts(block.timestamp);
    }

    function testBugPokeBoost() public {
        uint256 tokenId = createVeAlcx(admin, TOKEN_1, MAXTIME, false);
        address bribeAddress = voter.bribes(address(sushiGauge));

        // Add BAL bribes to sushi pool
        createThirdPartyBribe(bribeAddress, bal, TOKEN_100K);

        address[] memory pools = new address[](1);
        pools[0] = sushiPoolAddress;
        uint256[] memory weights = new uint256[](1);
        weights[0] = 10000;

        address[] memory bribes = new address[](1);
        bribes[0] = address(bribeAddress);
        address[][] memory tokens = new address[][](2);
        tokens[0] = new address[](1);
        tokens[0][0] = bal;

        uint256 fluxAccessable = veALCX.claimableFlux(tokenId) + flux.getUnclaimedFlux(tokenId);


        // Vote with the max boost amount
        hevm.prank(admin);
        voter.vote(tokenId, pools, weights, fluxAccessable / 2);


        // Used weight should be greater when boosting with unused flux
        assertGt(voter.usedWeights(tokenId), veALCX.balanceOfToken(tokenId), "should be greater when boosting");


        // Reach the end of the epoch
        hevm.warp(block.timestamp + nextEpoch);

        // Vote with the max boost amount
        hevm.prank(admin);
        voter.poke(tokenId);


        // If the code is correct, used weight should be greater in calling poke function.
        // But in this poke, the boost is not calculated. That is 0.
        assertEq(voter.usedWeights(tokenId), veALCX.balanceOfToken(tokenId), "if this is failed, that means the code is correct");
    }
}
```
