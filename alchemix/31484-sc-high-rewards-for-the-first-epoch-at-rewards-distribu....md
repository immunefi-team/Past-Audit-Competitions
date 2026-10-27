# 31484 - \[SC - High] Rewards for the first epoch at rewards distribu...

Submitted on May 20th 2024 at 06:25:57 UTC by @MahdiKarimi for [Boost | Alchemix](https://immunefi.com/bounty/alchemix-boost/)

Report ID: #31484

Report type: Smart Contract

Report severity: High

Target: <https://github.com/alchemix-finance/alchemix-v2-dao/blob/main/src/RewardsDistributor.sol>

Impacts:

* Permanent freezing of unclaimed yield

## Description

## Brief/Intro

Some parts of rewards at rewards distributor would be lost

## Vulnerability Details

RewardsDistributor distributes ALCX rewards to veALCX holders, rewards are distributed based on the balance of users at the end of an epoch, for the first time that the minter distributes rewards, some part of it is being allocated to the first epoch which can be withdrawn only if there was a user at the start of the first epoch ( end of last epoch ) but since rewards distributor and voting escrow ( mints veALCX ) are being deployed in the same time so there is no user at that timestamp so no one can claim those rewards and rewards gets stuck in contract forever

## Impact Details

loss of rewards at rewards distributor in the first epoch

## References

<https://github.com/alchemix-finance/alchemix-v2-dao/blob/f1007439ad3a32e412468c4c42f62f676822dc1f/src/RewardsDistributor.sol#L244-L248>

## Proof of Concept

```
        function testLostRewardsFirstEpoch() public {
        initializeVotingEscrow();

        // assert alcx balance of rewards distributor is zero
        uint256 distributorBalance = alcx.balanceOf(address(distributor));
        assertEq(distributorBalance, 0);

        // Fast forward 1/2 epoch
        // create a lock
        hevm.warp(block.timestamp + nextEpoch / 2);
        hevm.roll(block.number + 1);
        uint256 tokenId1 = createVeAlcx(admin, TOKEN_1, MAXTIME, false);

        // Finish the epoch and distribute some rewards 
        hevm.warp(newEpoch());
        voter.distribute();
        
        // assert alcx balance of rewards distributor is greater than zero which means some tokens has been transffered as reward 
        uint256 distributorBalanceEnd = alcx.balanceOf(address(distributor));
        assertGt(distributorBalanceEnd, 0);

        // calculate claimable amount by token1 
        uint256 claimable1 = distributor.claimable(tokenId1);
        // as we see token1 can't claim all alcx tokens despite that he's only user (on one can claim it)
        assert(distributorBalanceEnd > claimable1);
        // to ensure diffrence is high we check freezed amount is at least 3 time more than claimable amount  
        assert(distributorBalanceEnd > 3 * claimable1);

    }
```
