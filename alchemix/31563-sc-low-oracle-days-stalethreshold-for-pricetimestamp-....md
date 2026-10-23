# 31563 - \[SC - Low] Oracle days staleThreshold for priceTimestamp ...

Submitted on May 21st 2024 at 11:15:40 UTC by @SAAJ for [Boost | Alchemix](https://immunefi.com/bounty/alchemix-boost/)

Report ID: #31563

Report type: Smart Contract

Report severity: Low

Target: <https://github.com/alchemix-finance/alchemix-v2-dao/blob/main/src/RewardsDistributor.sol>

Impacts:

* Protocol insolvency
* Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield

## Description

## Brief/Intro

60 days is more than enough for threshold on oracle time updated causes stale price to be accepted

## Vulnerability Details

The `RewardsDistributor` contract have `amountToCompound` function makes call to Chainlink oracle receiving the update price. However, `staleThreshold` variable define the `priceTimestamp` to be checked every `60 days` which is way more than enough to validate price of any token.

In a high volatile market like predicted for this year the token price greatly fluctuates causing the threshold to have price that are stale. This will impact old prices to be accepted based on the `staleThreshold` define for the timelimit.

```
        require(block.timestamp - priceTimestamp < staleThreshold, "Price is stale");

```

According to chainlink [docs](https://docs.chain.link/architecture-overview/architecture-decentralized-model#aggregator) for using price feed the oracle `priceTimestamp` is based on `heartbeat` which defaults to threshold of `86400` or `1 day`.

The other factor consider for threshold according to each token is deviation from price which is set to `2%` for [`Alchemix`](https://docs.chain.link/data-feeds/price-feeds/addresses?network=ethereum\&page=1\&search=alch).

```
Asset name:
Alchemix

Asset type:
Crypto

Market hours:
Crypto

Pair:	
🟡ALCX / ETH

Deviation	:
2%	
Heartbeat	:
86400s

Dec	:
18

Address:
 0x194a9AaF2e0b67c35915cD01101585A33Fe25CAa

```

## Impact Details

`amountToCompound` function is called in `claim` method at L#175 where passing `alcxAmount` will give value in `weth`. Value of `weth` is highly dependent on `ETH` which is very volatile will result in either depositing amount greater or less than `alcxAmount`.

Having a longer point in time till which the oracle price is accepted, will cause to include transactions when prices have completely changed due to market conditions.

Stale prices of asset will be accepted as the current price, causing wrong/stale prices be fetched as if they were the latest causing loss to either protocol or the claimer.

## References

<https://github.com/alchemix-finance/alchemix-v2-dao/blob/f1007439ad3a32e412468c4c42f62f676822dc1f/src/RewardsDistributor.sol#L123>

## Proof of Concept

The `RewardsDistributor` contract have `amountToCompound` function makes call to Chainlink oracle receiving the update price. The `staleThreshold` impact is clearly to give out stale price which is accepted and deposited in the `claim` function.

```
    function amountToCompound(uint256 _alcxAmount) public view returns (uint256, uint256[] memory) {
        // Increased for testing since tests go into future
        uint256 staleThreshold = 60 days;

        (uint80 roundId, int256 alcxEthPrice,, uint256 priceTimestamp, uint80 answeredInRound) =
            priceFeed.latestRoundData();

        require(answeredInRound >= roundId, "Stale price");
        require(block.timestamp - priceTimestamp < staleThreshold, "Price is stale");
        require(alcxEthPrice > 0, "Chainlink answer reporting 0");

        uint256[] memory normalizedWeights = IManagedPool(address(balancerPool)).getNormalizedWeights();

        uint256 amount =
            (((_alcxAmount * uint256(alcxEthPrice)) / 1 ether) * normalizedWeights[0]) / normalizedWeights[1];

        return (amount, normalizedWeights);
    }
```
