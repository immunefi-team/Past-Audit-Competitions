# 30683 - \[SC - Critical] User can increase their unclaimed Flux token wi...

Submitted on May 4th 2024 at 11:47:30 UTC by @jecikpo for [Boost | Alchemix](https://immunefi.com/bounty/alchemix-boost/)

Report ID: #30683

Report type: Smart Contract

Report severity: Critical

Target: <https://github.com/alchemix-finance/alchemix-v2-dao/blob/main/src/VotingEscrow.sol>

Impacts:

* unbound minting of token
* Manipulation of governance voting result deviating from voted outcome and resulting in a direct change from intended effect of original results

## Description

## Brief/Intro

A user can increase their unclaimed Flux token amount by the abuse of `VotedEscrow.merge()` and `Voter.reset()`.

## Vulnerability Details

The `Voter.reset()` function calls `FluxToken.accrueFlux(_tokenId)` which increases the `unclaimed[_tokenId]` balance according the `_tokenId` voting power. The `Voter.reset()` function can be called only once per epoch due to the `onlyNewEpoch(_tokenId)` modifier preventing the user from accruing excess unclaimed Flux. This however can be abused if the voting power of a token is transferred to a new `tokenId` using `VotedEscrow.merge()`.

After calling `VotedEscrow.merge()` the voting power shall be transfered to a new token and the `Voter.reset()` can be called again on the new `tokenId` within the same epoch.

## Impact Details

The impact is that the user can accrue potentially unlimited amount of unclaimed Flux. The unclaimed Flux could be used to execute unfair voting by increasing the user's voting power. It could also be claimed and sold on the open market to suppress the Flux price which will allow other users to unlock their veALCX tokens at lower prices hence destroying the entire voting system credibility.

## References

<https://github.com/alchemix-finance/alchemix-v2-dao/blob/f1007439ad3a32e412468c4c42f62f676822dc1f/src/VotingEscrow.sol#L618>

## Proof of Concept

Add to the `VotingEscrow.t.sol` the following code:

```solidity
function testAbuseResetFlux() public {
        hevm.startPrank(admin);

        uint256 tokenIdLarge = veALCX.createLock(TOKEN_100K, THREE_WEEKS, false);
        uint256 tokenIdSmall = veALCX.createLock(TOKEN_1, THREE_WEEKS, false);


        voter.reset(tokenIdLarge);
        console.log("Unclaimed Flux on tokenIdLarge: %d", flux.unclaimedFlux(tokenIdLarge));
        veALCX.merge(tokenIdLarge, tokenIdSmall);
        voter.reset(tokenIdSmall);
        // we can see that the user amassed double of the unclaimedFlux during single epoch, this pattern could be repeated with more smaller vALCX locks
        console.log("Unclaimed Flux on tokenIdSmall: %d", flux.unclaimedFlux(tokenIdSmall));

        hevm.stopPrank();
    }
```
