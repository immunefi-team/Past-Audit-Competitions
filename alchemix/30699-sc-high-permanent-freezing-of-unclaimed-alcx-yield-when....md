# 30699 - \[SC - High] Permanent freezing of unclaimed ALCX yield when...

Submitted on May 5th 2024 at 03:53:39 UTC by @infosec\_us\_team for [Boost | Alchemix](https://immunefi.com/bounty/alchemix-boost/)

Report ID: #30699

Report type: Smart Contract

Report severity: High

Target: <https://github.com/alchemix-finance/alchemix-v2-dao/blob/main/src/VotingEscrow.sol>

Impacts:

* Permanent freezing of unclaimed yield

## Description

There are 2 ways of burning a `veALCX` position (the NFT that is minted in *VotingEscrow\.sol* and accrues rewards):

* When withdrawing a position
* When merging two positions

veALCX holders accrue rewards in "`FLUX`" (in the *FluxToken.sol*) and rewards in "`ALCX`" (in the *RewardsDistributor.sol*).

In the *VotingEscrow* when withdrawing a position, there's a segment of the code that makes sure to claim all FLUX and ALCX earned, and distribute it:

```
    /**
     * @notice Withdraw all tokens for `_tokenId`
     * @dev Only possible if the lock has expired
     */
    function withdraw(uint256 _tokenId) public nonreentrant {

         // .......... some code here removed for the sake of simplicity

        // Claim any unclaimed ALCX rewards and FLUX
        IRewardsDistributor(distributor).claim(_tokenId, false);
        IFluxToken(FLUX).claimFlux(_tokenId, IFluxToken(FLUX).getUnclaimedFlux(_tokenId));

        // Burn the token
        _burn(_tokenId, value);

        emit Withdraw(msg.sender, _tokenId, value, block.timestamp);
```

> Source code at: <https://github.com/alchemix-finance/alchemix-v2-dao/blob/main/src/VotingEscrow.sol?#L741-L775>

Users could manually send a transaction to the blockchain to claim the ALCX, another to claim the FLUX, and a 3rd transaction to withdraw the position, but, wisely, the implementation of the withdraw function saves them from manually having to claim each reward token.

When merging two positions by calling `merge(uint256 _from, uint256 _to)` the code transfers any accrued FLUX from one token to another before burning the NFT, but it forgets to claim the accrued ALCX.

> Source code at: <https://github.com/alchemix-finance/alchemix-v2-dao/blob/main/src/VotingEscrow.sol?utm\\_source=immunefi#L618-L651>

Therefore, merging two positions burns one of the NFTs and permanently freezes the unclaimed ALCX yield accrued by that NFT.

## Impact Details

Permanent freezing of unclaimed ALCX yield when merging veALCX positions

## Proof of Concept

Using Alchemix's test suite we created a foundry test inside "`alchemix-v2-dao/src/test/VotingEscrow.t.sol`" proving how all unclaimed ALCX yield is lost when merging two **veALCX** positions.

Include the following function in `VotingEscrow.t.sol`:

```
    function testClamingRewardsOnMerge() public {
        hevm.startPrank(admin);

        uint256 tokenId1 = veALCX.createLock(TOKEN_1, THREE_WEEKS, true);
        uint256 tokenId2 = veALCX.createLock(TOKEN_1, THREE_WEEKS, true);

        voter.reset(tokenId1);
        voter.reset(tokenId2);

        hevm.warp(newEpoch());

        voter.distribute();

        uint256 unclaimedAlcx1 = distributor.claimable(tokenId1);
        uint256 unclaimedFlux1 = flux.getUnclaimedFlux(tokenId1);

        uint256 unclaimedAlcx2 = distributor.claimable(tokenId2);
        uint256 unclaimedFlux2 = flux.getUnclaimedFlux(tokenId2);

        console2.log("-BEFORE MERGING--------------------------------------------------------");
        console2.log("Unclaimed ALCX before merge for token1", unclaimedAlcx1);
        console2.log("Unclaimed FLUX before merge for token1", unclaimedFlux1);
        console2.log("-----------------------------------------------------------------------");
        console2.log("Unclaimed ALCX before merge for token2", unclaimedAlcx2);
        console2.log("Unclaimed FLUX before merge for token2", unclaimedFlux2);
        console2.log("-----------------------------------------------------------------------");

        hevm.warp(newEpoch());

        veALCX.merge(tokenId1, tokenId2);

        uint256 unclaimedAlcx_afterMerge1 = distributor.claimable(tokenId1);
        uint256 unclaimedFlux_afterMerge1 = flux.getUnclaimedFlux(tokenId1);

        uint256 unclaimedAlcx_afterMerge2 = distributor.claimable(tokenId2);
        uint256 unclaimedFlux_afterMerge2 = flux.getUnclaimedFlux(tokenId2);

        console2.log("-AFTER MERGING---------------------------------------------------------");
        console2.log("Unclaimed ALCX after merge for token1", unclaimedAlcx_afterMerge1);
        console2.log("Unclaimed FLUX after merge for token1", unclaimedFlux_afterMerge1);
        console2.log("-----------------------------------------------------------------------");
        console2.log("Unclaimed ALCX after merge for token2", unclaimedAlcx_afterMerge2);
        console2.log("Unclaimed FLUX after merge for token2", unclaimedFlux_afterMerge2);
        console2.log("-----------------------------------------------------------------------");

        assertEq(veALCX.balanceOfToken(tokenId1), 0);
        assertEq(veALCX.ownerOf(tokenId1), address(0));

        hevm.stopPrank();
    }

```
