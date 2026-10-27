# 31420 - \[SC - Insight] No array lengths check in VotersolclaimBribes

Submitted on May 18th 2024 at 21:57:38 UTC by @gladiator111 for [Boost | Alchemix](https://immunefi.com/bounty/alchemix-boost/)

Report ID: #31420

Report type: Smart Contract

Report severity: Insight

Target: <https://github.com/alchemix-finance/alchemix-v2-dao/blob/main/src/Voter.sol>

Impacts:

* Contract fails to deliver promised returns, but doesn't lose value

## Description

## Brief/Intro

`_bribes` and `_tokens` length are not checked to be equal in `Voter.sol::claimBribes` which can lead to problems as discussed below

## Vulnerability Details

In `Voter.sol::claimBribes`, `_bribes` and `_tokens` length are not checked to be equal.

```solidity
    // @audit No check for _bribes.length == _tokens.length
    /// @inheritdoc IVoter
    function claimBribes(address[] memory _bribes, address[][] memory _tokens, uint256 _tokenId) external {
        require(IVotingEscrow(veALCX).isApprovedOrOwner(msg.sender, _tokenId));

        for (uint256 i = 0; i < _bribes.length; i++) {
            IBribe(_bribes[i]).getRewardForOwner(_tokenId, _tokens[i]);
        }
    }
```

Similar length equal checks are there in all other parts of the protocol but is absent in this function.

```solidity
  // Similar checks are there in other functions
    function vote(
        uint256 _tokenId,
        address[] calldata _poolVote,
        uint256[] calldata _weights,
        uint256 _boost
    ) external onlyNewEpoch(_tokenId) {
        require(IVotingEscrow(veALCX).isApprovedOrOwner(msg.sender, _tokenId), "not approved or owner");
@>      require(_poolVote.length == _weights.length, "pool vote and weights mismatch");
        require(_poolVote.length > 0, "no pools voted for");
@>     require(_poolVote.length <= pools.length, "invalid pools");
        require(
            IVotingEscrow(veALCX).claimableFlux(_tokenId) + IFluxToken(FLUX).getUnclaimedFlux(_tokenId) >= _boost,
            "insufficient FLUX to boost"
        );
        require(
            (IVotingEscrow(veALCX).balanceOfToken(_tokenId) + _boost) <= maxVotingPower(_tokenId),
            "cannot exceed max boost"
        );
        require(block.timestamp < IVotingEscrow(veALCX).lockEnd(_tokenId), "cannot vote with expired token");

        _vote(_tokenId, _poolVote, _weights, _boost);
    }
```

This can create problem in a scenerio where \_bribes length is less than \_tokens length as the user will lose on the funds on the bribe which is not set in the array. Such checks are essential in order to validate that valid transactions are going through and there is no discrepancy between bribes and tokens.

`Note - Although the contract should check for these things in order to not procure any loss to users but it is also the responsibility of the user hence this is a low issue and not any medium or higher severity issue`

## Impact Details

Lack of this validation can result in user putting wrong lengths of bribes and tokens. This can create problem in a scenerio where \_bribes length is less than \_tokens length as the user will lose on the funds on the bribe which is not set in the array.

## Suggestion/ Recommendation

Modify the function as follows

```diff
    function claimBribes(address[] memory _bribes, address[][] memory _tokens, uint256 _tokenId) external {
 +      require(_bribes.length == _tokens.length, "token and bribes are not the same length");
        require(IVotingEscrow(veALCX).isApprovedOrOwner(msg.sender, _tokenId));

        for (uint256 i = 0; i < _bribes.length; i++) {
            IBribe(_bribes[i]).getRewardForOwner(_tokenId, _tokens[i]);
        }
    }
```

## References

<https://github.com/alchemix-finance/alchemix-v2-dao/blob/f1007439ad3a32e412468c4c42f62f676822dc1f/src/Voter.sol#L332-L338>

## Proof of Concept

In the following POC we have given vote through 2 bribe Address but at the time of claimBribe only one bribe address is (unintentionally) given and 2 token arrays are given but the transaction still goes through resulting in user losing out on bribe from the second bribe address. Paste the following code in `Voting.t.sol` and run with the following command

```bash
forge test --match-test testBribesUnequalLength -vvvv --fork-url $FORK_URL
```

```solidity
    function testBribesUnequalLength() public {
        uint256 tokenId1 = createVeAlcx(admin, TOKEN_1, MAXTIME, false);
        address bribeAddress1 = voter.bribes(address(sushiGauge));
        address bribeAddress2 = voter.bribes(address(balancerGauge));
        createThirdPartyBribe(bribeAddress1, bal, TOKEN_100K);
        createThirdPartyBribe(bribeAddress1, aura, TOKEN_100K);
        createThirdPartyBribe(bribeAddress2, bal, TOKEN_100K);
        createThirdPartyBribe(bribeAddress2, aura, TOKEN_100K);

        address[] memory pools = new address[](2);
        pools[0] = sushiPoolAddress;
        pools[1] = balancerPoolAddress;
        uint256[] memory weights = new uint256[](2);
        weights[0] = 5000;
        weights[1] = 5000;

        hevm.prank(admin);
        voter.vote(tokenId1, pools, weights, 0);


        address[] memory bribes = new address[](1);
        bribes[0] = address(bribeAddress1);           // adding only one bribeAddress
        address[][] memory tokens = new address[][](2);
        tokens[0] = new address[](2);                // adding two tokens
        tokens[0][0] = bal;
        tokens[0][1] = aura;
        tokens[1] = new address[](2);
        tokens[1][0] = bal;
        tokens[1][1] = aura;

        hevm.warp(block.timestamp + nextEpoch);

        hevm.prank(admin);
        voter.claimBribes(bribes, tokens, tokenId1);   // transaction goes through without revert and user only gets claims for one bribe only

    }
```
