# 30826 - \[SC - High] ALCK rewards are lost when merging tokens becau...

Submitted on May 6th 2024 at 22:25:53 UTC by @Jonnes for [Boost | Alchemix](https://immunefi.com/bounty/alchemix-boost/)

Report ID: #30826

Report type: Smart Contract

Report severity: High

Target: <https://github.com/alchemix-finance/alchemix-v2-dao/blob/main/src/VotingEscrow.sol>

Impacts:

* Permanent freezing of unclaimed yield

## Description

## Brief/Intro

ALCK rewards are lost when merging tokens because the rewards are not claimed before burning the token.

## Vulnerability Details

Merging or withdrawing tokens require burning the token. When merging tokens, unclaimed rewards must be claimed before burning the token. This prevents users from losing their rewards when the tokens are burnt. This isn't the case however as unclaimed rewards are not claimed before burning the token. This makes the user's unclaimed ALCX rewards to become lost and unclaimable when the tokens are burnt.

```
        _checkpoint(_from, _locked0, LockedBalance(0, 0, false, 0));
        _burn(_from, value0);
        _depositFor(_to, value0, end, _locked1.maxLockEnabled, _locked1, DepositType.MERGE_TYPE);

```

In contrast to the merge function, the withdraw function first claims all unclaimed rewards before burning the token. This prevents users from losing their rewards when the tokens are burnt.

```
        // Claim any unclaimed ALCX rewards and FLUX
        IRewardsDistributor(distributor).claim(_tokenId, false);
        IFluxToken(FLUX).claimFlux(_tokenId, IFluxToken(FLUX).getUnclaimedFlux(_tokenId));

        // Burn the token
        _burn(_tokenId, value);
```

Hence, users will lose their ALCX rewards when merging tokens because the ALCX rewards are not claimed before burning the token. This leads to a permanent freezing of unclaimed rewards as the ALCX rewards are lost and unclaimable.

## Impact Details

Permanent freezing of unclaimed rewards

## References

<https://github.com/alchemix-finance/alchemix-v2-dao/blob/f1007439ad3a32e412468c4c42f62f676822dc1f/src/VotingEscrow.sol#L649>

<https://github.com/alchemix-finance/alchemix-v2-dao/blob/f1007439ad3a32e412468c4c42f62f676822dc1f/src/VotingEscrow.sol#L767C1-L772C32>

## Proof of Concept

The following test can be added to VotingEscrow\.t.sol to show the described scenario.

```
    function testMergeTokens() public {
        uint256 tokenId1 = createVeAlcx(beef, TOKEN_1, MAXTIME, false);
        uint256 tokenId2 = createVeAlcx(beef, TOKEN_100K, MAXTIME / 2, false);
        uint256 tokenId3 = createVeAlcx(beef, TOKEN_100K, MAXTIME / 2, false);

        hevm.startPrank(beef);

        uint256 lockEnd1 = veALCX.lockEnd(tokenId1);

        assertEq(lockEnd1, ((block.timestamp + MAXTIME) / ONE_WEEK) * ONE_WEEK);
        assertEq(veALCX.lockedAmount(tokenId1), TOKEN_1);

        // Vote to trigger flux accrual
        hevm.warp(newEpoch());

        address[] memory pools = new address[](1);
        pools[0] = alETHPool;
        uint256[] memory weights = new uint256[](1);
        weights[0] = 5000;
        voter.vote(tokenId1, pools, weights, 0);
        voter.vote(tokenId2, pools, weights, 0);

        voter.distribute();

        hevm.warp(newEpoch());

        // Reset to allow merging of tokens
        voter.reset(tokenId1);
        voter.reset(tokenId2);

        uint256 unclaimedFluxBefore1 = flux.getUnclaimedFlux(tokenId1);
        uint256 unclaimedFluxBefore2 = flux.getUnclaimedFlux(tokenId2);

        uint256 unclaimedALCX1 = distributor.claimable(tokenId1);
        // There are unclaimed ALCX rewards
        assertGt(unclaimedALCX1, 0);

        // merging the two tokens.
        veALCX.merge(tokenId1, tokenId2);

        uint256 unclaimedALCXAfter1 = distributor.claimable(tokenId1);

        // These rewards are lost forever.
        assertGt(unclaimedALCXAfter1, 0);

        uint256 unclaimedFluxAfter1 = flux.getUnclaimedFlux(tokenId1);
        uint256 unclaimedFluxAfter2 = flux.getUnclaimedFlux(tokenId2);

        // After merge unclaimed flux should consolidate into one token
        assertEq(unclaimedFluxAfter2, unclaimedFluxBefore1 + unclaimedFluxBefore2, "unclaimed flux not consolidated");
        assertEq(unclaimedFluxAfter1, 0, "incorrect unclaimed flux");
        assertGt(unclaimedFluxAfter2, 0);
        // Merged token should take longer of the two lock end dates
        assertEq(veALCX.lockEnd(tokenId2), lockEnd1);

        // Merged token should have sum of both token locked amounts
        assertEq(veALCX.lockedAmount(tokenId2), TOKEN_1 + TOKEN_100K);

        // The token is burnt and the unclaimed ALCX rewards are lost.
        assertEq(veALCX.ownerOf(tokenId1), address(0));

        hevm.stopPrank();
    }
```
