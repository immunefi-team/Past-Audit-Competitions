# 31519 - \[SC - Low] Lack of revert statement in Votersolpoke result...

Submitted on May 21st 2024 at 00:31:49 UTC by @gladiator111 for [Boost | Alchemix](https://immunefi.com/bounty/alchemix-boost/)

Report ID: #31519

Report type: Smart Contract

Report severity: Low

Target: <https://github.com/alchemix-finance/alchemix-v2-dao/blob/main/src/Voter.sol>

Impacts:

* Permanent freezing of unclaimed yield

## Description

## Brief/Intro

There is no revert statement in `Voter.sol::poke` for first time callers resulting in loss of yield for the epoch

## Vulnerability Details

`Note - For Immunefi Triage - Do not close this , if you have any doubt then directly ask me through comments, as the boost period is ending it would not be great for this to get closed. It is a legitimate issue, I request you to read the report and POC carefully. Thanks!`

The `Voter.sol::poke` function doesn't revert when \_poolCnt = 0 (\_poolVote.length = 0), i.e for first time voters.

```solidity
function poke(uint256 _tokenId) public {
        // Previous boost will be taken into account with weights being pulled from the votes mapping
        uint256 _boost = 0;

        if (msg.sender != admin) {
            require(IVotingEscrow(veALCX).isApprovedOrOwner(msg.sender, _tokenId), "not approved or owner");
        }

        address[] memory _poolVote = poolVote[_tokenId];
@>      uint256 _poolCnt = _poolVote.length;  //Ideally should revert if 0
        uint256[] memory _weights = new uint256[](_poolCnt);

        for (uint256 i = 0; i < _poolCnt; i++) {
            _weights[i] = votes[_tokenId][_poolVote[i]];
        }

        _vote(_tokenId, _poolVote, _weights, _boost);
    }
```

So, if a first time voter calls poke function then he will be marked as voted without even voting for anyone because of empty `poolVote[_tokenId]` array. This will prevent him from getting any yield. The situation becomes even worse because he can't even call vote function also because of onlyNewEpoch modifier.

```solidity
    function vote(
        uint256 _tokenId,
        address[] calldata _poolVote,
        uint256[] calldata _weights,
        uint256 _boost
    ) external onlyNewEpoch(_tokenId) {          //onlyNewEpoch modifier restricts the user from using this function 
```

He will only be able to vote in the next epoch and all his yield will be lost permanently. This can easily be prevented with a revert statement.

## Impact Details

User will lose his yield for the entire epoch.

## Recommendation / Suggestion

Modify the function as follows

```diff
function poke(uint256 _tokenId) public {
        // Previous boost will be taken into account with weights being pulled from the votes mapping
        uint256 _boost = 0;

        if (msg.sender != admin) {
            require(IVotingEscrow(veALCX).isApprovedOrOwner(msg.sender, _tokenId), "not approved or owner");
        }

        address[] memory _poolVote = poolVote[_tokenId];
        uint256 _poolCnt = _poolVote.length;
+       require(_poolCnt > 0, "Vote first" );
        uint256[] memory _weights = new uint256[](_poolCnt);

        for (uint256 i = 0; i < _poolCnt; i++) {
            _weights[i] = votes[_tokenId][_poolVote[i]];
        }

        _vote(_tokenId, _poolVote, _weights, _boost);
    }
```

## References

<https://github.com/alchemix-finance/alchemix-v2-dao/blob/f1007439ad3a32e412468c4c42f62f676822dc1f/src/Voter.sol#L195-212>

## Proof of Concept

Paste the following code in `Voting.t.sol` and run the test with the following command

```bash
forge test --match-test testYieldFreeze -vvvv --fork-url $FORK_URL
```

```solidity
function testYieldFreeze() public {
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

        address[] memory bribes = new address[](2);
        bribes[0] = address(bribeAddress1); 
        bribes[1] = address(bribeAddress2); 

        address[][] memory tokens = new address[][](1);
        tokens[0] = new address[](2);                
        tokens[0][0] = bal;
        tokens[0][1] = aura;

        hevm.prank(admin);
        voter.poke(tokenId1);              // using poke first (ideally should revert but doesn't revert)
        hevm.prank(admin);
        vm.expectRevert("TOKEN_ALREADY_VOTED_THIS_EPOCH");  //can't vote
        voter.vote(tokenId1, pools, weights, 0);
        hevm.warp(block.timestamp + nextEpoch);
        hevm.prank(admin);
        vm.expectRevert("no rewards to claim");   // can't claim yield
        voter.claimBribes(bribes, tokens, tokenId1);
```
