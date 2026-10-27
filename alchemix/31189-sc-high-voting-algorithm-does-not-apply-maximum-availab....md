# 31189 - \[SC - High] Voting algorithm does not apply maximum availab...

Submitted on May 14th 2024 at 15:36:44 UTC by @xBentley for [Boost | Alchemix](https://immunefi.com/bounty/alchemix-boost/)

Report ID: #31189

Report type: Smart Contract

Report severity: High

Target: <https://github.com/alchemix-finance/alchemix-v2-dao/blob/main/src/Voter.sol>

Impacts:

* Contract fails to deliver promised returns, but doesn't lose value

## Description

## Brief/Intro

When voting, the algorithm used to allocate weights to pools does not use up all available voting power for the token. This can disadvantage some voters leading to skewed voting.

## Vulnerability Details

When voting, weights for each pool are allocated proportionally, <https://github.com/alchemix-finance/alchemix-v2-dao/blob/f1007439ad3a32e412468c4c42f62f676822dc1f/src/Voter.sol#L432>:

```solidity
 uint256 _poolWeight = (_weights[i] * totalPower) / _totalVoteWeight;
```

This calculation leaves out some amounts unallocated since Solidity will round down the calculation making \_totalWeight to be less than totalPower. Actually the code does not check that the totalPower available has been used up. Consider this scenario:

totalPower = 500 weight1 = 10 weight2 = 50 weight3 = 75

poolWeight1 = (10 \* 500)/135 = 37 poolWeight2 = (50 \* 500)/135 = 185 poolWeight3 = (75 \* 500)/135 = 277

total voting power used = 499.

## Impact Details

Voters who will be affected by the rounding down might not be able to apply all available voting power, compared to other voters who, for example, pass in single parameters. This might lead to skewed voting results where the final tally is determined by a small difference between nays and ayes.

## References

<https://github.com/alchemix-finance/alchemix-v2-dao/blob/f1007439ad3a32e412468c4c42f62f676822dc1f/src/Voter.sol#L432>

\##Recommendation I would recommend that the weight for the last pool be allocated via Subtraction and not division, thus src/Voter.sol::Ln432:

```solidity
 if(i == _poolCnt - 1)
uint256 _poolWeight = totalPower - _totalWeight;
```

## Proof of Concept

Add this test to src/test/Voting.t.sol:

```solidity
    function testMultiPoolVote() public {
        uint256 tokenId = createVeAlcx(admin, TOKEN_1, MAXTIME, false);

        hevm.startPrank(admin);

        hevm.warp(block.timestamp + nextEpoch);

        uint256 maxPower = veALCX.balanceOfToken(tokenId);
        console.log(maxPower);
        address[] memory pools = new address[](3);
        pools[0] = alETHPool;
        pools[1] = sushiPoolAddress;
        pools[2] = balancerPoolAddress;
        uint256[] memory weights = new uint256[](3);
        weights[0] = 1;
        weights[1] = 50;
        weights[2] = 75;
        voter.vote(tokenId, pools, weights, 0);

        uint256 weightAlETH = voter.weights(alETHPool);
        uint256 weightSushi = voter.weights(sushiPoolAddress);
        uint256 weightBalancer = voter.weights(balancerPoolAddress);
        assertGt(maxPower,weightBalancer + weightSushi + weightAlETH);
    }

```

Due to Solidity rounding down, the total power applied is less than available for the token.
