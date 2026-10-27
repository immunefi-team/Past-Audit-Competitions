# 31078 - \[SC - High] withdraw doesnt claim all rewards before burnin...

Submitted on May 12th 2024 at 11:07:38 UTC by @Holterhus for [Boost | Alchemix](https://immunefi.com/bounty/alchemix-boost/)

Report ID: #31078

Report type: Smart Contract

Report severity: High

Target: <https://github.com/alchemix-finance/alchemix-v2-dao/blob/main/src/VotingEscrow.sol>

Impacts:

* Permanent freezing of unclaimed yield

## Description

## Brief/Intro

The `claim()` function in the `RewardsDistributor` does not always claim all of the pending rewards for a given `tokenId`. This is because the `_claimable()` function has a finite `for` loop that does 50 iterations. However, the `withdraw()` function in `veALCX` only calls `claim()` once before permanently burning the token. This can lead to a permanent freezing of unclaimed yield if the 50 iterations are not sufficient.

## Vulnerability Details

The `_claimable()` function in the `RewardsDistributor` has the following main loop:

```solidity
for (uint256 i = 0; i < 50; i++) {
    if (weekCursor >= _lastTokenTime) break;

    if (weekCursor >= userPoint.ts && userEpoch <= maxUserEpoch) {
        userEpoch += 1;
        oldUserPoint = userPoint;
        if (userEpoch > maxUserEpoch) {
            userPoint = IVotingEscrow.Point(0, 0, 0, 0);
        } else {
            userPoint = IVotingEscrow(_ve).getUserPointHistory(_tokenId, userEpoch);
        }
    } else {
        int256 dt = int256(weekCursor - oldUserPoint.ts);
        int256 calc = oldUserPoint.bias - dt * oldUserPoint.slope > int256(0)
            ? oldUserPoint.bias - dt * oldUserPoint.slope
            : int256(0);
        uint256 balanceOf = uint256(calc);
        if (balanceOf == 0 && userEpoch > maxUserEpoch) break;
        if (balanceOf != 0) {
            toDistribute += (balanceOf * tokensPerWeek[weekCursor]) / veSupply[weekCursor];
        }
        weekCursor += WEEK;
    }
}
```

Notice that this loop only does a maximum of 50 iterations. This is fine on its own, because calling `claim()` multiple times will progress the rewards claiming in multiples of 50 iterations.

However, consider the following code in the `withdraw()` function of the `veALCX` contract:

```solidity
// Claim any unclaimed ALCX rewards and FLUX
IRewardsDistributor(distributor).claim(_tokenId, false);
IFluxToken(FLUX).claimFlux(_tokenId, IFluxToken(FLUX).getUnclaimedFlux(_tokenId));

// Burn the token
_burn(_tokenId, value);
```

Since this only calls `claim()` once, this will only do 50 iterations within the `_claimable()` calculation, which can be insufficient for claiming all of the user's unclaimed `ALCX` rewards.

## Impact Details

Yield is permanently frozen whenever a user calls `withdraw()` on a `tokenId` that needs more than 50 iterations to claim remaining rewards. This can happen if the `tokenId` has been deposited and left alone for a long time (perhaps with `maxLockEnabled == true`). It can also happen if the user has checkpointed many times before (notice that `weekCursor += WEEK` only happens in the `else` case of the main loop, so `claim()` may not even progress a single week).

## References

See the PoC below.

## Proof of Concept

I have created the following test file and added it to the `tests/` directory:

```solidity
// SPDX-License-Identifier: GPL-3.0
pragma solidity ^0.8.15;

import "./BaseTest.sol";

contract NotFullClaimBugTest is BaseTest {

    constructor() {
        setupContracts(block.timestamp);
    }

    function testNotFullClaimBug() public {

        vm.startPrank(admin);

        uint256 tokenId1 = createVeAlcx(admin, TOKEN_100K, MAXTIME, true);
        uint256 tokenId2 = createVeAlcx(admin, TOKEN_100K, MAXTIME, true);

        for (uint256 i; i < 100; ++i) {
            vm.warp(newEpoch());
            voter.distribute();
        }

        veALCX.updateUnlockTime(tokenId1, 365 days, false);
        veALCX.updateUnlockTime(tokenId2, 365 days, false);

        for (uint256 i; i < 30; ++i) {
            vm.warp(newEpoch());
            voter.distribute();
        }

        veALCX.startCooldown(tokenId1);
        veALCX.startCooldown(tokenId2);

        vm.warp(block.timestamp + 1 weeks);

        address rewardsToken = distributor.rewardsToken();
        uint256 balBefore;
        uint256 balAfter;


        console.log("claimable 1:", distributor.claimable(tokenId1));
        console.log("claimable 2:", distributor.claimable(tokenId2));

        /*************************************************************
            Amount from withdraw()
        *************************************************************/

        balBefore = IERC20(rewardsToken).balanceOf(admin);
        veALCX.withdraw(tokenId1);
        balAfter = IERC20(rewardsToken).balanceOf(admin);
        console.log("claimed from withdraw():", balAfter - balBefore);

        /*************************************************************
            Amount from claim()
        *************************************************************/

        balBefore = IERC20(rewardsToken).balanceOf(admin);
        for (uint256 i; i < 20; ++i) distributor.claim(tokenId2, false);
        balAfter = IERC20(rewardsToken).balanceOf(admin);
        console.log("claimed from claim():", balAfter - balBefore);

        vm.stopPrank();
    }

}
```

Running the command `forge test -vvv --match-test testNotFullClaimBug --rpc-url $ETH_RPC_URL` gives the following result:

```
[PASS] testNotFullClaimBug() (gas: 109527243)
Logs:
  claimable 1: 85548935342535580780653
  claimable 2: 85548935342535580780653
  claimed from withdraw(): 85548935342535580780653
  claimed from claim(): 113317958706345737195718
```

which shows that calling `withdraw()` will permanently freeze unclaimed yield.
