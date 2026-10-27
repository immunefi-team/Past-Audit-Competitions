# 30920 - \[SC - Low] User loses access to claims after merging of to...

Submitted on May 8th 2024 at 05:28:35 UTC by @jecikpo for [Boost | Alchemix](https://immunefi.com/bounty/alchemix-boost/)

Report ID: #30920

Report type: Smart Contract

Report severity: Low

Target: <https://github.com/alchemix-finance/alchemix-v2-dao/blob/main/src/VotingEscrow.sol>

Impacts:

* Contract fails to deliver promised returns, but doesn't lose value

## Description

## Brief/Intro

When two veALCX tokens are merged the assets and voting power is transfered from tokenA to tokenB, however claims are not. Upon merging tokenA is burned and claims that were associatied with it are lost.

## Vulnerability Details

When the user calls `VotingEscrow.merge()` to merge tokenA and tokenB, the tokenA is burned and it no longer exists, however certains claims in other contracts (e.g. in `Bribe`) are still linked to the old tokenA. Those claims cannot be further accessed, because the verification of ownership of tokenA cannot be passed as it is removed from the necessary storage variables.

## Impact Details

If the user does not claim explicitly his claims on tokenA before merging, they are all becoming inaccessible.

## References

The `merge()` function: <https://github.com/alchemix-finance/alchemix-v2-dao/blob/f1007439ad3a32e412468c4c42f62f676822dc1f/src/VotingEscrow.sol#L618>

The `burn()` function: <https://github.com/alchemix-finance/alchemix-v2-dao/blob/f1007439ad3a32e412468c4c42f62f676822dc1f/src/VotingEscrow.sol#L1558>

The `_isApprovedOrOwner()` function: <https://github.com/alchemix-finance/alchemix-v2-dao/blob/f1007439ad3a32e412468c4c42f62f676822dc1f/src/VotingEscrow.sol#L826>

## Proof of Concept

Paste the following code into `Voting.t.sol` file:

```solidity
    function testMergeClaimsLost() public {
        uint256 tokenId1 = createVeAlcx(admin, TOKEN_1, 156 days, false);
        uint256 tokenId2 = createVeAlcx(admin, TOKEN_1, 156 days, false);
        address bribeAddress = voter.bribes(address(sushiGauge));

        createThirdPartyBribe(bribeAddress, bal, TOKEN_1);
        //voter.distribute();

        address[] memory pools = new address[](1);
        pools[0] = sushiPoolAddress;
        uint256[] memory weights = new uint256[](1);
        weights[0] = 5000;

        address[] memory bribes = new address[](1);
        bribes[0] = address(bribeAddress);
        address[][] memory tokens = new address[][](2);
        tokens[0] = new address[](1);
        tokens[0][0] = bal;

        hevm.prank(admin);
        voter.vote(tokenId1, pools, weights, 0);

        hevm.prank(admin);
        voter.vote(tokenId2, pools, weights, 0);

        //console.log("Bribe lastEarned: ", )
        uint256 adminBribesToken1;
        uint256 adminBribesToken2;

        hevm.warp(block.timestamp + 6 days);

        // epoch 2
        hevm.warp(block.timestamp + nextEpoch);
        createThirdPartyBribe(bribeAddress, bal, TOKEN_1);
        voter.distribute();
        hevm.prank(admin);
        voter.vote(tokenId1, pools, weights, 0);
        hevm.prank(admin);
        voter.vote(tokenId2, pools, weights, 0);

        hevm.warp(block.timestamp + nextEpoch);

        hevm.prank(admin);
        voter.reset(tokenId1);
        hevm.prank(admin);
        voter.reset(tokenId2);

        // accrued bribes for both of our tokens
        adminBribesToken1 = IBribe(bribeAddress).earned(bal, tokenId1);
        adminBribesToken2 = IBribe(bribeAddress).earned(bal, tokenId2);
        console.log("[admin] earned bribes before merge on tokenId1: %d", adminBribesToken1);
        console.log("[admin] earned bribes before merge on tokenId2: %d", adminBribesToken2);

        // we are merging tokenId1 -> tokenId2
        hevm.prank(admin);
        veALCX.merge(tokenId1, tokenId2);

        // we can claim on tokenId2, but that's just half of what we own.
        hevm.prank(admin);
        voter.claimBribes(bribes, tokens, tokenId2);

        // claim on the old token fails
        hevm.expectRevert();
        hevm.prank(admin);
        voter.claimBribes(bribes, tokens, tokenId1);

        console.log("[admin] total bribes claimed: %d", ERC20(bal).balanceOf(admin));
    }
```
