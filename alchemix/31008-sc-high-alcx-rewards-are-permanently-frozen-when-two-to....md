# 31008 - \[SC - High] Alcx rewards are permanently frozen when two to...

Submitted on May 10th 2024 at 19:51:14 UTC by @Adrianx for [Boost | Alchemix](https://immunefi.com/bounty/alchemix-boost/)

Report ID: #31008

Report type: Smart Contract

Report severity: High

Target: <https://github.com/alchemix-finance/alchemix-v2-dao/blob/main/src/VotingEscrow.sol>

Impacts:

* Permanent freezing of unclaimed yield

## Description

## Brief/Intro

Alcx rewards are not claimed before burning the merged token, which leads to a permanent freezing of unclaimed Alcx rewards.

## Vulnerability Details

When two tokens are merged, the from token is burnt but the unclaimed alcx rewards of the from token are not claimed before burning it. This causes the unclaimed alcx rewards of the from token to be permanently frozen when the token is burnt.

```
        IFluxToken(FLUX).mergeFlux(_from, _to);

        // If max lock is enabled end is the max lock time, otherwise it is the greater of the two end times
        uint256 end = _locked1.maxLockEnabled
            ? ((block.timestamp + MAXTIME) / WEEK) * WEEK
            : _locked0.end >= _locked1.end
            ? _locked0.end
            : _locked1.end;

        locked[_from] = LockedBalance(0, 0, false, 0);
        _checkpoint(_from, _locked0, LockedBalance(0, 0, false, 0));
        _burn(_from, value0);
        _depositFor(_to, value0, end, _locked1.maxLockEnabled, _locked1, DepositType.MERGE_TYPE);

```

As seen above, the alcx rewards are not claimed before burning the merged token. This leads to a permanent freezing of unclaimed rewards.

## Impact Details

Alcx rewards are permanently frozen when the from tokens are burnt during token merging.

## References

<https://github.com/alchemix-finance/alchemix-v2-dao/blob/f1007439ad3a32e412468c4c42f62f676822dc1f/src/VotingEscrow.sol#L649>

## Proof of Concept

Add the test below to VotingEscrow\.t.sol and check the logs to see the frozen rewards.

```
    function testFrozenRewards() public {
        uint256 tokenId1 = createVeAlcx(David, TOKEN_1, MAXTIME, false);
        uint256 tokenId2 = createVeAlcx(David, TOKEN_100K, MAXTIME / 2, false);
        hevm.startPrank(David);
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
        veALCX.merge(tokenId1, tokenId2);
        uint256 frozenAlcxReward = distributor.claimable(tokenId1);

        // The unclaimed rewards are permanently frozen
        vm.expectRevert();
        distributor.claim(tokenId1, false);
        console.log("frozen AlcxReward", frozenAlcxReward);
        assertEq(veALCX.ownerOf(tokenId1), address(0));
        hevm.stopPrank();
    }
```

```
  frozen AlcxReward: 492146894738584492
```
