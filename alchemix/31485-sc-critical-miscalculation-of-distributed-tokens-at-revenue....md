# 31485 - \[SC - Critical] Miscalculation of distributed tokens at revenue...

Submitted on May 20th 2024 at 06:40:34 UTC by @MahdiKarimi for [Boost | Alchemix](https://immunefi.com/bounty/alchemix-boost/)

Report ID: #31485

Report type: Smart Contract

Report severity: Critical

Target: <https://github.com/alchemix-finance/alchemix-v2-dao/blob/main/src/RevenueHandler.sol>

Impacts:

* Theft of unclaimed yield

## Description

## Brief/Intro

Revenue handler uses contract balance to calculate distribution amount, so if users didn't claim their rewards from the last distribution, unclaimed amount is mistakenly considered as newly distributed rewards

## Vulnerability Details

Every time the checkpoint is called at revenue handler to distribute revenues, it uses the contract balance as the amount to be distributed. However, if some users haven't claimed their rewards from previous distributions, those unclaimed rewards are mistakenly considered as newly distributed rewards so some users can receive more rewards while others can't receive their rewards.

## Impact Details

Inconsistency between contract balance and user claimable amount enables some users to receive more rewards while some users are not able to receive any rewards

## References

<https://github.com/alchemix-finance/alchemix-v2-dao/blob/f1007439ad3a32e412468c4c42f62f676822dc1f/src/RevenueHandler.sol#L245-L264>

## Proof of Concept

```
      function testClaimAfterNextCheckpoint() external {
        uint256 revAmt = 1000e18;
        uint256 tokenId = _setupClaimableNonAlchemicRevenue(revAmt, bal);
        uint256 tokenId2 = _setupClaimableNonAlchemicRevenue(revAmt, bal);

        uint256 claimable = revenueHandler.claimable(tokenId, bal);
        uint256 claimable2 = revenueHandler.claimable(tokenId2, bal);
    
        // as we see contract balance is not sufficient for claimable amount 
        assert(claimable > IERC20(bal).balanceOf(address(revenueHandler)));
        assert(claimable + claimable2 > IERC20(bal).balanceOf(address(revenueHandler)));
        
    }
```
