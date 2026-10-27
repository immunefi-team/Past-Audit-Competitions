# 31584 - \[SC - Critical] Loss Of Boosted Weight When Poking In The Same ...

Submitted on May 21st 2024 at 14:48:26 UTC by @Limbooo for [Boost | Alchemix](https://immunefi.com/bounty/alchemix-boost/)

Report ID: #31584

Report type: Smart Contract

Report severity: Critical

Target: <https://github.com/alchemix-finance/alchemix-v2-dao/blob/main/src/Voter.sol>

Impacts:

* Permanent freezing of unclaimed yield

## Description

## Brief/Intro

The `Voter` contract has an issue where users lose their boosted voting weight if they "poke" their vote within the same boosted vote period. This leads to a loss of boosted voting power and can potentially result in unclaimed bribe rewards being permanently frozen in the contract.

## Vulnerability Details

The `poke` function is intended to update the user's voting weight. However, it resets the boost to zero when it re-calculates the vote, as seen in the following snippet:

```solidity
src/Voter.sol:
  194:     /// @inheritdoc IVoter
  195:     function poke(uint256 _tokenId) public {
  196:         // Previous boost will be taken into account with weights being pulled from the votes mapping
@>197:         uint256 _boost = 0;
  ....
  205:         uint256[] memory _weights = new uint256[](_poolCnt);
  206: 
  207:         for (uint256 i = 0; i < _poolCnt; i++) {
  208:             _weights[i] = votes[_tokenId][_poolVote[i]];
  209:         }
  210: 
@>211:         _vote(_tokenId, _poolVote, _weights, _boost);
  212:     }


  412:     function _vote(uint256 _tokenId, address[] memory _poolVote, uint256[] memory _weights, uint256 _boost) internal {
  413:         _reset(_tokenId);
  ....
  419:         for (uint256 i = 0; i < _poolCnt; i++) {
  420:             _totalVoteWeight += _weights[i];
  421:         }
  422: 
  423:         IFluxToken(FLUX).accrueFlux(_tokenId);
@>424:         uint256 totalPower = (IVotingEscrow(veALCX).balanceOfToken(_tokenId) + _boost);
  425: 
  426:         for (uint256 i = 0; i < _poolCnt; i++) {
  427:             address _pool = _poolVote[i];
  428:             address _gauge = gauges[_pool];
  429: 
  430:             require(isAlive[_gauge], "cannot vote for dead gauge");
  431: 
@>432:             uint256 _poolWeight = (_weights[i] * totalPower) / _totalVoteWeight;
  ....
  439:             weights[_pool] += _poolWeight;
  440:             votes[_tokenId][_pool] += _poolWeight;
  441:             IBribe(bribes[_gauge]).deposit(uint256(_poolWeight), _tokenId);
  442:             _totalWeight += _poolWeight;
  443:             emit Voted(msg.sender, _pool, _tokenId, _poolWeight);
  444:         }
  ....
  455:     }
```

When the `poke` function is called (which users may interact with when they want to update the voting power after depositing more token into veALCX), it sets `_boost` to 0 before calling `_vote`, leading to a recalculation of voting power without considering the previously accumulated boost (see line 424).

Additionally, the `pokeTokens` function, which can be called by the admin, also resets the boost, causing the same issue.

## Impact Details

* **Loss of Boosted Voting Weight**: Users lose their boosted voting power if they poke their vote within the same boosted period, which can lead to a reduction in their voting influence.
* **Potential Unclaimed Yield**: The loss of boosted weight can result in unclaimed bribe rewards remaining in the contract.

## Mitigation Analysis

To mitigate this issue, the `poke` function should be updated to correctly account for the accumulated boost. The `_boost` variable should be calculated and passed appropriately to the `_vote` function. Since boost is only valid for the epoch it was used in, ensure that this information is properly retained and used when recalculating votes within the same epoch.

## Proof of Concept

The test can be added to a new file under the current test suite `src/test/VotingPoC.t.sol`, then specify the file name in `FILE` flag under `Makefile` configuration. Run using `make test_file`

```solidity
// SPDX-License-Identifier: UNLICENSED
pragma solidity ^0.8.15;

import "./BaseTest.sol";

contract VotingPoCTest is BaseTest {
    address public alice;

    function setUp() public {
        setupContracts(block.timestamp);

        // Setup addresses
        alice = _makeAddr('Alice');
    }

    function testLossOfBoostedWeightAfterPookingInTheSamePeriod() public {
        uint256 period = minter.activePeriod();

        // First epoch started
        hevm.warp(period + 1);

        // Alice start a veALCX postion.
        uint256 tokenId = _initializeVeALCXPosition(alice, TOKEN_1);

        address bribeAddress = voter.bribes(address(sushiGauge));
        address[] memory pools = new address[](1);
        pools[0] = sushiPoolAddress;
        uint256[] memory weights = new uint256[](1);
        weights[0] = 10000;

        address[] memory bribes = new address[](1);
        bribes[0] = address(bribeAddress);
        address[][] memory tokens = new address[][](1);
        tokens[0] = new address[](1);
        tokens[0][0] = bal;

        uint256 boost = veALCX.claimableFlux(tokenId) + flux.getUnclaimedFlux(tokenId);
        
        // Alice Vote
        hevm.prank(alice);
        voter.vote(tokenId, pools, weights, boost);

        // Reward amount
        uint256 rewardAmount = TOKEN_100K;
        // Notify bribe for reward amount
        createThirdPartyBribe(bribeAddress, bal, rewardAmount);

        // Current balance of vote
        uint256 voteBalanceBeforePoke = Bribe(bribeAddress).balanceOf(tokenId);
        assertEq(
            voteBalanceBeforePoke,
            veALCX.balanceOfToken(tokenId) + boost
        );
        uint256 voteTotalSup = IBribe(bribeAddress).totalSupply();

        // Wrap to half of current period.
        hevm.warp(period + 1 weeks + 1);
        hevm.roll(block.number + (1 weeks / 12));

        // Alice want to deposit more to his lock
        deal(address(bpt), alice, 1);
        hevm.startPrank(alice);
        IERC20(bpt).approve(address(veALCX), 1);
        veALCX.depositFor(tokenId, 1);
        // Since his token balance grows, he want to poke his vote to grow his vote balance too
        voter.poke(tokenId);
        hevm.stopPrank();

        // Now his vote has become less than before poke
        uint256 voteBalanceAfterPoke = Bribe(bribeAddress).balanceOf(tokenId);
        assertLe(voteBalanceAfterPoke, voteBalanceBeforePoke);
        // After poking, it only accounts for the balance of token without the boost amount for this period
        assertEq(
            voteBalanceAfterPoke,
            veALCX.balanceOfToken(tokenId)
        );

        // Next epoch started
        hevm.warp(period + 2 weeks + 1);
        hevm.roll(block.number + (1 weeks / 12));
        voter.distribute();

        // After Alice claim his bribes
        hevm.prank(alice);
        voter.claimBribes(bribes, tokens, tokenId);

        // There is unclaimable balance left in the contract 
        assertGt(IERC20(bal).balanceOf(bribeAddress), 0);
    }

    // Helper functions
    function _makeAddr(string memory name) internal returns(address addr){
        addr = hevm.addr(uint256(keccak256(abi.encodePacked(name))));
        vm.label(addr, name);
    }

    function _initializeVeALCXPosition(address owner, uint256 amount) internal returns (uint256 tokenId) {
        veALCX.checkpoint();
        tokenId = _lockVeALCX(owner, amount);
    }

    function _lockVeALCX(address owner, uint256 amount) internal returns (uint256 tokenId) {
        deal(address(bpt), owner, amount);
        hevm.startPrank(owner);
        IERC20(bpt).approve(address(veALCX), amount);
        tokenId = veALCX.createLock(amount, MAXTIME, false);
        hevm.stopPrank();
    }
}
```
