# 31486 - \[SC - High] getClaimableFlux miscalculates claimable FLUX f...

Submitted on May 20th 2024 at 09:53:00 UTC by @Holterhus for [Boost | Alchemix](https://immunefi.com/bounty/alchemix-boost/)

Report ID: #31486

Report type: Smart Contract

Report severity: High

Target: <https://github.com/alchemix-finance/alchemix-v2-dao/blob/main/src/FluxToken.sol>

Impacts:

* Permanent freezing of unclaimed royalties

## Description

## Brief/Intro

When NFT holders claim FLUX, the base `claimableFlux` amount is calculated based on `veALCX` values, but the formula is implemented incorrectly which leads to the wrong results.

## Vulnerability Details

`claimableFlux` is intended to calculate the amount of FLUX that would be earned in 1 year, assuming max lock was enabled.

This should be calculated as follows:

`amount * fluxPerveALCX * maxLockMultiplier * numEpochsInOneYear`

(This is because, each epoch, it would earn `amount * fluxPerVeALCX * maxLockMultiplier`.)

This can be expressed in Solidity as:

```solidity
claimableFlux = (_amount * fluxPerVe * veMul * veMax) / (BPS * WEEK * 2)
```

Instead, it is calculated as follows:

```solidity
claimableFlux = (((bpt * veMul) / veMax) * veMax * (fluxPerVe + BPS)) / BPS / fluxMul;
```

There are a number of issues with this calculation (ignoring that `bpt` is used instead of `_amount`, as that is covered in a separate submission):

1. We multiply and divide by `veMax`, rather than dividing by the length of time of one epoch in order to get the number of epochs in `MAXTIME`.
2. We do `(fluxPerVe + BPS) / BPS` instead of simply `fluxPerVe / BPS`, which gives us a 150% payout per Ve rather than the 50% that is intended.
3. We divide by `fluxMul`, which does not appear to be relevant.

## Impact Details

This will result in the wrong amount of FLUX being calculated when NFT holders claim FLUX. This leads to a permanent loss of value to the claimers of the FLUX.

## References

`FluxToken.sol`

## Proof of Concept

The following test can be added to `FluxToken.t.sol`. It should show that different values arise for the calculated claimable FLUX amount and the actual amount that is accrued for a year.

```
function test_IncorrectClaimableFlux() external {
    vm.startPrank(admin);
    uint256 amount = 1e18;
    uint256 bptAmount = flux.calculateBPT(amount);

    uint256 claimableFluxCalc = flux.getClaimableFlux(amount, patronNFT);
    uint256 tokenId = createVeAlcx(admin, bptAmount, veALCX.MAXTIME(), false);

    // Claim 365 days of flux with this lock
    for (uint256 i; i < 365; ++i) {
        uint256 currentEpoch = (block.timestamp / 2 weeks) * 2 weeks;
        if (currentEpoch > voter.lastVoted(tokenId)) voter.reset(tokenId);
        vm.roll(block.number + 1);
        vm.warp(block.timestamp + 1 days);
    }

    console.log("FLUX balance at end of year:", flux.getUnclaimedFlux(tokenId));
    console.log("Calculated FLUX:", claimableFluxCalc);

    vm.stopPrank();
}
```
