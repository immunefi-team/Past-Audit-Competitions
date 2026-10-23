# 31377 - \[SC - Critical] Stucked yield tokens upon withdrawal of votes f...

Submitted on May 17th 2024 at 18:16:22 UTC by @Saediek for [Boost | Alchemix](https://immunefi.com/bounty/alchemix-boost/)

Report ID: #31377

Report type: Smart Contract

Report severity: Critical

Target: <https://github.com/alchemix-finance/alchemix-v2-dao/blob/main/src/Bribe.sol>

Impacts:

* Permanent freezing of unclaimed yield

## Description

## Brief/Intro

The withdraw() method in the Bribe contract doesn't update the totalVoting if a staker/voter decides to withdraw his/her stake this would lead to a scenario where a percentage of rewardToken meant for the staker that withdraws his/her tokens gets stucked in the pool instead of distributing it among other participants.

## Vulnerability Details

Stucked rewards tokens upon withdrawal of votes from Bribe contract. The Bribe module is an essential part of the whole system it mainly functions as a distributor for reward tokens to voters for a particular guage and it accounts for votes per epoch,reward tokens per epoch among so many other things. In the CONTRACT.md it says and i quote "the total amount of bribe b on pool p claimable by a veALCX NFT with token-ID i during a given epoch n is equal to the proportion of total veALCX power that that NFT used to vote on pool \`p",which translates to BribePerNFT(n)=(NFTVoteAmount(n)\*REWARDS)/TotalVotes(n)(where BribePerNFT refers to the amount of bribe a tokenId is expected to receive).This equation doesn't always hold and i would prove it to you with an illustration.Lets say we have 5 entities or voters(A,B,C,D,E) at the current epoch they all hold 20 votes each and all of them wagered their votes on the guage so according to the Guage Bribe the totalVotes =100 and each entity has a 20% stake on the rewardTokens for that epoch but towards the end of that epoch A decides to withdraw from the votes so there are four stakers left which implies that each staker should be entitled to 25% of the totalRewards for that epoch,but this isn't true because whenever a withdrawal occurs the totalVote isn't deducted and the bribes Owed to a tokenId=votes/totalVotes \*rewardTokens which means the bribes owed to each entity remains 20% each and since there are 4 entities left it sums up to 80% and the remaining 20% remains unclaimable and stuck in the pool forever and ever.

The bug is spotted in the deposit method: whenever a deposit is made the totalVoting variable is increased and whenever a withdrawal is made the totalVoting variable should also be decreased. ##Code Snippet function deposit(uint256 amount, uint256 tokenId) external { require(msg.sender == voter);

```
    totalSupply += amount;
    balanceOf[tokenId] += amount;

    totalVoting += amount;

    _writeCheckpoint(tokenId, balanceOf[tokenId]);
    _writeSupplyCheckpoint();
    _writeVotingCheckpoint();

    emit Deposit(msg.sender, tokenId, amount);
}
```

function withdraw(uint256 amount, uint256 tokenId) external { require(msg.sender == voter);

```
    totalSupply -= amount;
    balanceOf[tokenId] -= amount;

    _writeCheckpoint(tokenId, balanceOf[tokenId]);
    _writeSupplyCheckpoint();

    emit Withdraw(msg.sender, tokenId, amount);
}
```

but the totalVoting isn't decreased in withdraw() which means totalVoting before and after withdrawal would always stay the same but since the newVoting power is less than totalVoting the portion of rewards previously entitled to the user who withdrew his/her tokens are stucked in the pool.

## Impact Details

The ripple effect of this issue is that a portion of reward tokens whenever a withdrawal occurs would be lost in the bribe contract forever.

## References

Withdraw():\[<https://github.com/alchemix-finance/alchemix-v2-dao/blob/f1007439ad3a32e412468c4c42f62f676822dc1f/src/Bribe.sol#L319>] ##Recommendation: Modify the withdraw method from : function withdraw(uint256 amount, uint256 tokenId) external { require(msg.sender == voter);

```
    totalSupply -= amount;
    balanceOf[tokenId] -= amount;

    _writeCheckpoint(tokenId, balanceOf[tokenId]);
    _writeSupplyCheckpoint();

    emit Withdraw(msg.sender, tokenId, amount);
}
```

to: function withdraw(uint256 amount, uint256 tokenId) external { require(msg.sender == voter);

```
    totalSupply -= amount;
     balanceOf[tokenId] -= amount;
    _writeVotingCheckpoint();
       totalVoting-=amount;
    _writeCheckpoint(tokenId, balanceOf[tokenId]);
    _writeSupplyCheckpoint();

    emit Withdraw(msg.sender, tokenId, amount);
}
```

And also they should be a recovery mode for cases where all the voters withdraw their votes so that rewardTokens wouldn't just get lost in the pool.

## Proof of Concept

//SPDX-License-Identifier:UNLICENSED //@author:<Saediek@proton.me> /\*\*

* Steps to run file
* Create a file called {file}.t.sol in the test directory of the alchemix-v2-dao directory
* Paste code in the newly created file
* create a remappings.txt in your root directory.
* paste the following
* foundry-libs/=alchemix/lib/forge-std/src
* alchemix/=src/
* openzeppelin/=lib/openzeppelin-contracts/ \*/ pragma solidity ^0.8; import "foundry-libs/Test.sol"; import { Bribe } from "alchemix/Bribe.sol"; import "openzeppelin/contracts/token/ERC20/ERC20.sol";

contract BribeTest is Test { /\*\* \*Addres of the users or voters \*/

```
address private bob = makeAddr("BOB");
address private ryan = makeAddr("RYAN");
address private alice = makeAddr("ALICE");
address private sarah = makeAddr("SALAH");
address private daniel = makeAddr("DANIEL");
//The mock voter contract represents the Voter contract.
MockVoter private voter;
//The Bribe contract
Bribe private bribe;
//An allowed ERC20 token.
MockRewardToken rewardToken;

modifier onlyVoter() {
    vm.startPrank(address(voter));
    _;
    vm.stopPrank();
}

/**
 *Deploy all necessary contracts
 */
constructor() {
    rewardToken = new MockRewardToken();
    voter = new MockVoter(address(new veALCX()));
    bribe = new Bribe(address(voter));
    //Issue 100e18 of the reward token to adress(this) so that address(this)
    //could call {notifyRewardAmount depositing the reward Tokens}
    rewardToken.mint(address(this), 100e18);
    rewardToken.approve(address(bribe), 100e18);
}

/**
 * This test is meant to illustrate a scenario whereby a voter that casts
 * his vote for a guage choose to withdraw it and the amount previously allocated
 * to the staker is stucked in the pool instead of being distributed to the other stakers
 * It also breaks the equation which states that [amountOwedToTokenId(n)=voteCastedByTokenId(n)*rewardsPerEpoch(n)/totalVotes]
 * The issue is solely caused due to totalVoting not deducted at withdrawals of votes
 * I am also exploring the possibilty that if all the voters withdraws the rewardsAmount for that epoch is loast forever
 */
function testStuckTokensForWithdrawal() external {
    //The time is kinda random
    vm.warp(1715861236);
    uint256 startOfCurrentEpoch = bribe.getEpochStart(block.timestamp);
    console.log("Start-time of current epoch:[%s]", startOfCurrentEpoch);
    //Reward tokens are deposited for the currentEpoch
    bribe.notifyRewardAmount(address(rewardToken), 100e18);
    console.log("Amount deposited to bribe-contract:[%s]", rewardToken.balanceOf(address(bribe)));
    /**
   * Five of the stakers which holds equal voting power of 20 Votes decided to vote for a guage
   *and after sometime a particular staker[bob] decides for some reason that he is going to withdraw his vote || uncast his vote
   *This should also imply that bob is not entitled to a portion of the rewardAmount for that epoch and also means distribution of the rewardTokens should 
   *be shared among the other four stakers which means they should all hold 25% of the totalReward after bob's withdrawal but this isn't so 
   *because the totakVoting is not updated after bobs withdrawal so the  real-voting-power[80 i.e 20 per user] is not equal to the totalVotingPower which stays 
   *the same [100]
   
   */
    _vote(alice, 20);
    _vote(daniel, 20);
    _vote(bob, 20);
    _vote(sarah, 20);
    _vote(ryan, 20);
    assertEq(100, bribe.totalVoting());
    //bob pulls out his votes
    _resetVote(bob, 20);
    //@notice the totalVoting doesn't change after bobs out
    assertEq(100, bribe.totalVoting());
    //fast forward to the next epoch
    //when the rewards are claimable
    vm.warp(block.timestamp + 2 weeks);
    //since bob withdrew alice is entitled to 25% of the whole rewards
    _claimRewards(alice);
    _claimRewards(ryan);
    _claimRewards(daniel);
    _claimRewards(sarah);
    assertEq(20e18, rewardToken.balanceOf(address(bribe)));
    //and bob can't also claim those tokens..
    //if he tries the call reverts which is expected but this means bob's former share is stucked in the pool forever.
    vm.expectRevert("no rewards to claim");
    _claimRewards(bob);
}

/**
 * Helper for casting an amount of votes in  the bribe contract
 * @param _user Owner of tokenId
 * @param _amount  the amount of votes to withdraw
 */
function _vote(address _user, uint256 _amount) internal {
    vm.startPrank(address(voter));
    bribe.deposit(_amount, uint256(uint160(_user)));
    vm.stopPrank();
}

/**
 * Helper for withdrawing an amount of votes from the bribe contract
 * @param _user Owner of tokenId
 * @param _amount  the amount of votes to withdraw
 */
function _resetVote(address _user, uint256 _amount) internal {
    vm.startPrank(address(voter));
    bribe.withdraw(_amount, uint256(uint160(_user)));
    vm.stopPrank();
}

/**
 *helper function to claim rewardToken for the previous epoch
 * @param _user  The owner of the tokenId
 */
function _claimRewards(address _user) internal {
    vm.startPrank(address(voter));
    uint256 _tokenId = _getTokenId(_user);
    address[] memory _tokens = new address[](1);
    _tokens[0] = address(rewardToken);
    bribe.getRewardForOwner(_tokenId, _tokens);
    vm.stopPrank();
}

function _getTokenId(address _user) internal pure returns (uint256 _tokenId) {
    _tokenId = uint256(uint160(_user));
}
```

}

contract MockVoter { address public veALCX;

```
constructor(address _token) {
    veALCX = _token;
}

function isWhitelisted(address) external pure returns (bool) {
    return true;
}
```

} //Mock contract for the veALCX contract to represent the owner of token by the address contract veALCX { function ownerOf(uint256 \_tokenId) external view returns (address) { return address(uint160(\_tokenId)); } } //Mock ERC20 contract which serves as reward tokens contract MockRewardToken is ERC20 { constructor() ERC20("MOCK-TOKEN", "MCK") {}

```
function mint(address _target, uint256 _amount) external {
    _mint(_target, _amount);
}
```

}
