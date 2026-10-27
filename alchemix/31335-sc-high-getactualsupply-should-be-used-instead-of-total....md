# 31335 - \[SC - High] getActualSupply should be used instead of total...

Submitted on May 17th 2024 at 11:44:53 UTC by @OxAnmol for [Boost | Alchemix](https://immunefi.com/bounty/alchemix-boost/)

Report ID: #31335

Report type: Smart Contract

Report severity: High

Target: <https://github.com/alchemix-finance/alchemix-v2-dao/blob/main/src/RewardsDistributor.sol>

Impacts:

* Theft of unclaimed yield

## Description

## Brief/Intro

The `rewardDistributor:_depositIntoBalancerPool` uses totalSupply from the balance pool to calculate the expected BPT amount out. But the balancer docs recommend using getActualSupply instead of totalSupply.

## Vulnerability Details

`totalSupply` function in the balancer doesn’t account for protocol fees and unminted tokens which means the totalSupply doesn't' correctly reflect the actual supply of BPT. The balancer recommends using `getActualSupply` to get the correct total supply of the BPT tokens.

<https://docs.balancer.fi/concepts/advanced/valuing-bpt/valuing-bpt.html#getting-bpt-supply>

<https://github.com/balancer/balancer-v2-monorepo/blob/ac63d64018c6331248c7d77b9f317a06cced0243/pkg/pool-weighted/contracts/WeightedPool.sol#L332>

In our case, the BPT total supply function is used to calculate the `bptAmountOut`

<https://github.com/alchemix-finance/alchemix-v2-dao/blob/f1007439ad3a32e412468c4c42f62f676822dc1f/src/RewardsDistributor.sol#L410>

```solidity
 uint256 bptAmountOut = WeightedMath._calcBptOutGivenExactTokensIn(
            balances,
            _normalizedWeights,
            amountsIn,
            IERC20(address(balancerPool)).totalSupply(), //@audit should use getActualSupply
            balancerPool.getSwapFeePercentage()
        );

```

bptAmountOut here acts like a slippage protection for add liquidity and this parameter is very important for sandwich protection. If the total supply is used instead of getActualSupply then the bptAmountOut can be significantly low and this can be vulnerable to sandwich attacks resulting in the loss of BPT for a user.

## Impact Details

Users can receive less BPT because of sandwich attacks resulting in the loss of unclaimed yield for the user. Please follow [this](https://docs.balancer.fi/guides/builders/join-pool.html#building-a-join-transaction) and

[this](https://solodit.xyz/issues/m-6-balancer-lp-valuation-methodologies-use-the-incorrect-supply-metric-sherlock-olympus-rbs-20-git) for further clarity in the issue.

## References

<https://github.com/alchemix-finance/alchemix-v2-dao/blob/f1007439ad3a32e412468c4c42f62f676822dc1f/src/RewardsDistributor.sol#L414>

<https://github.com/balancer/balancer-v2-monorepo/blob/ac63d64018c6331248c7d77b9f317a06cced0243/pkg/pool-weighted/contracts/WeightedPool.sol#L325C1-L345C1>

<https://docs.balancer.fi/concepts/advanced/valuing-bpt/valuing-bpt.html#getting-bpt-supply>

\##Recommedation use getActualSupply instead of totalSupply

## Proof of Concept

Here I have added a new interface `IBalancerPool`, edited the RewardDistributor:\_depositIntoBalancerPool in RewardDistributor and used getTotalSupply to get bptAmountOut2 The console log is used to show the difference between the two outputs.

```solidity
 

interface IBalancerPool {
    function getActualSupply() external view returns (uint256);
}

...SNIP..
 function _depositIntoBalancerPool(
        uint256 _wethAmount,
        uint256 _alcxAmount,
        uint256[] memory _normalizedWeights
    ) internal {
        (, uint256[] memory balances, ) = balancerVault.getPoolTokens(balancerPoolId);

        uint256[] memory amountsIn = new uint256[](2);
        amountsIn[0] = _wethAmount;
        amountsIn[1] = _alcxAmount;

        uint256 bptAmountOut = WeightedMath._calcBptOutGivenExactTokensIn(
            balances,
            _normalizedWeights,
            amountsIn,
            IERC20(address(balancerPool)).totalSupply(),
            balancerPool.getSwapFeePercentage()
        );

        uint256 bptAmountOut2 = WeightedMath._calcBptOutGivenExactTokensIn(
            balances,
            _normalizedWeights,
            amountsIn,
            IBalancerPool(address(balancerPool)).getActualSupply(),
            balancerPool.getSwapFeePercentage()
        );

        if (bptAmountOut > bptAmountOut2) {
            console2.log("bptAmountOut > bptAmountOut2 and diff is ", bptAmountOut - bptAmountOut2);
        } else if (bptAmountOut < bptAmountOut2) {
            console2.log("bptAmountOut < bptAmountOut2 and diff is ", bptAmountOut2 - bptAmountOut);
            //@audit-issue this is a potential issue
        } else {
            console2.log("bptAmountOut == bptAmountOut2");
        }
        bytes memory _userData = abi.encode(
            WeightedPoolUserData.JoinKind.EXACT_TOKENS_IN_FOR_BPT_OUT,
            amountsIn,
            bptAmountOut2
        );

        IVault.JoinPoolRequest memory request = IVault.JoinPoolRequest({
            assets: poolAssets,
            maxAmountsIn: amountsIn,
            userData: _userData,
            fromInternalBalance: false
        });

        balancerVault.joinPool(balancerPoolId, address(this), address(this), request);
    }

```

add this test in `Voting.t.sol`

```solidity
function testGetActualSupply() public {
        uint256 period = minter.activePeriod();

        // Create a veALCX token and vote to trigger voter rewards
        uint256 tokenId = createVeAlcx(admin, TOKEN_1, MAXTIME, false);
        address bribeAddress = voter.bribes(address(sushiGauge));
        createThirdPartyBribe(bribeAddress, bal, TOKEN_100K);

        address[] memory pools = new address[](1);
        pools[0] = sushiPoolAddress;
        uint256[] memory weights = new uint256[](1);
        weights[0] = 5000;

        address[] memory bribes = new address[](1);
        bribes[0] = address(bribeAddress);
        address[][] memory tokens = new address[][](2);
        tokens[0] = new address[](1);
        tokens[0][0] = bal;

        hevm.startPrank(admin);
        veALCX.approve(beef, tokenId);
        hevm.stopPrank();

        hevm.startPrank(beef);
        voter.vote(tokenId, pools, weights, 0);

        // Move forward a week relative to period
        hevm.warp(period + nextEpoch);
        voter.distribute();

        hevm.deal(beef, 10e18); // Sendt 10 ether to admin
        // 10 ether should be enough to pair with ALCX
        distributor.claim{ value: 10e18 }(tokenId, true); // Opt for compounding

        hevm.stopPrank();
    }
```

### Console Outputs

```bash
Ran 1 test for src/test/Voting.t.sol:VotingTest
[PASS] testGetActualSupply() (gas: 4931022)
Logs:
  bptAmountOut < bptAmountOut2 and diff is  130532624887077104

Suite result: ok. 1 passed; 0 failed; 0 skipped; finished in 142.81s (116.42s CPU time)

Ran 1 test suite in 143.73s (142.81s CPU time): 1 tests passed, 0 failed, 0 skipped (1 total tests)
```

As we can see the bptAmountOut2 calculated using getTotalSupply is greater than bptAmountOut. The difference here might seem small but this will largely depend on the trading volume of the balancer pool.
