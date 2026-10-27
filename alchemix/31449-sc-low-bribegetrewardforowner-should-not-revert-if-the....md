# 31449 - \[SC - Low] BribegetRewardForOwner should not revert if the...

Submitted on May 19th 2024 at 13:39:38 UTC by @OxAnmol for [Boost | Alchemix](https://immunefi.com/bounty/alchemix-boost/)

Report ID: #31449

Report type: Smart Contract

Report severity: Low

Target: <https://github.com/alchemix-finance/alchemix-v2-dao/blob/main/src/Voter.sol>

Impacts:

* Contract fails to deliver promised returns, but doesn't lose value

## Description

## Brief/Intro

Bribe:getRewardForOwner should not revert if there are no bribe rewards. But it should continue looping.

## Vulnerability Details

A bribe contract permits a maximum of 16 reward tokens, and anyone wishing to give a bribe can offer any of these tokens as a reward using the `notifyRewardAmount` function.

Given these 16 tokens, it's unpredictable how many reward tokens get deposited in one epoch. At times, the bribe contract may involve all 16 tokens as rewards, while other times, only a few may be present.

Users who participate in gauge voting can claim their bribe rewards using the `Voter::claimBribes` function.

```solidity
 function claimBribes(address[] memory _bribes, address[][] memory _tokens, uint256 _tokenId) external {
        require(IVotingEscrow(veALCX).isApprovedOrOwner(msg.sender, _tokenId));

        for (uint256 i = 0; i < _bribes.length; i++) {
            IBribe(_bribes[i]).getRewardForOwner(_tokenId, _tokens[i]);
        }
    }
```

The `address[][] memory tokens` will typically contain 16 whitelisted tokens due to the confusion discussed previously.

In the `getRewardForOwner` function, it iterates through all the provided tokens. If it doesn't find a reward, it simply reverts without checking all tokens. Consequently, any remaining token that might have a reward for the user is not accounted for.

```solidity
function getRewardForOwner(uint256 tokenId, address[] memory tokens) external lock {
        require(msg.sender == voter, "not voter");
        address _owner = IVotingEscrow(veALCX).ownerOf(tokenId);
        uint256 length = tokens.length;
        for (uint256 i = 0; i < length; i++) {
            uint256 _reward = earned(tokens[i], tokenId);
            
            require(_reward > 0, "no rewards to claim");
   //@audit-issue should not revert if there is no reward for token instead should continue             require(_reward > 0, "no rewards to claim"); 

            lastEarn[tokens[i]][tokenId] = block.timestamp;

            _writeCheckpoint(tokenId, balanceOf[tokenId]);

            IERC20(tokens[i]).safeTransfer(_owner, _reward);

            emit ClaimRewards(_owner, tokens[i], _reward);
        }
    }
```

For reference please have a look at how Velodrome have done it

<https://github.com/velodrome-finance/contracts/blob/888617832644f073f331d6576da3f4bd987be982/contracts/rewards/Reward.sol#L230>

<https://github.com/velodrome-finance/contracts/blob/888617832644f073f331d6576da3f4bd987be982/contracts/rewards/VotingReward.sol#L23C4-L29C6>

## Recommendation

The loop should continue if there is no reward earned and distribute all other remaining rewards.

```diff
function getRewardForOwner(uint256 tokenId, address[] memory tokens) external lock {
        require(msg.sender == voter, "not voter");
        address _owner = IVotingEscrow(veALCX).ownerOf(tokenId);
        uint256 length = tokens.length;
        for (uint256 i = 0; i < length; i++) {
            uint256 _reward = earned(tokens[i], tokenId);
            
-            require(_reward > 0, "no rewards to claim");
+             if(_reward == 0) continue;

            lastEarn[tokens[i]][tokenId] = block.timestamp;

            _writeCheckpoint(tokenId, balanceOf[tokenId]);

            IERC20(tokens[i]).safeTransfer(_owner, _reward);

            emit ClaimRewards(_owner, tokens[i], _reward);
        }
    }
```

## Impact Details

The function will assuredly revert without yielding any bribe reward for the user.

## References

<https://github.com/alchemix-finance/alchemix-v2-dao/blob/f1007439ad3a32e412468c4c42f62f676822dc1f/src/Bribe.sol#L290>

## Proof of Concept

paste this test in `Voting.t.sol`

```solidity
function testRevertOnNoReward() public {
        uint256 tokenId1 = createVeAlcx(admin, TOKEN_1, MAXTIME, false);
        uint256 tokenId2 = createVeAlcx(beef, TOKEN_1, MAXTIME, false);

        address bribeAddress = voter.bribes(address(sushiGauge));

        // Add BAL bribes to sushiGauge
        createThirdPartyBribe(bribeAddress, bal, TOKEN_100K);

        uint256 balanceStart = IERC20(bal).balanceOf(bribeAddress);

        address[] memory pools = new address[](1);
        pools[0] = sushiPoolAddress;
        uint256[] memory weights = new uint256[](1);
        weights[0] = 5000;

        address[] memory bribes = new address[](1);
        bribes[0] = address(bribeAddress);
        address[][] memory tokens = new address[][](2);
        tokens[0] = new address[](2);
        tokens[0][0] = bal;
        tokens[0][1] = aura; // aura is whitelisted

        hevm.prank(admin);
        voter.vote(tokenId1, pools, weights, 0);

        // Fast forward epochs
        hevm.warp(block.timestamp + nextEpoch * 5);
        voter.distribute();

        hevm.prank(admin);

        hevm.expectRevert(abi.encodePacked("no rewards to claim"));
        voter.claimBribes(bribes, tokens, tokenId1);
    }
```
