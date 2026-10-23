# 30711 - \[SC - Low] The result of the AggregatorVInterface is not v...

Submitted on May 5th 2024 at 14:39:56 UTC by @infosec\_us\_team for [Boost | Alchemix](https://immunefi.com/bounty/alchemix-boost/)

Report ID: #30711

Report type: Smart Contract

Report severity: Low

Target: <https://github.com/alchemix-finance/alchemix-v2-dao/blob/main/src/RewardsDistributor.sol>

Impacts:

* Protocol insolvency

## Description

## Vulnerability Details

The *RewardsDistributor* (`alchemix-v2-dao/src/RewardsDistributor.sol`) calculates the amount to compound based on the **alcxEthPrice**.

```
function amountToCompound(uint256 _alcxAmount) public view returns (uint256, uint256[] memory) {
    // Increased for testing since tests go into future
    uint256 staleThreshold = 60 days;

    (uint80 roundId, int256 alcxEthPrice, , uint256 priceTimestamp, uint80 answeredInRound) = priceFeed
        .latestRoundData();

    require(answeredInRound >= roundId, "Stale price");
    require(block.timestamp - priceTimestamp < staleThreshold, "Price is stale");
    require(alcxEthPrice > 0, "Chainlink answer reporting 0");

    uint256[] memory normalizedWeights = IManagedPool(address(balancerPool)).getNormalizedWeights();

    uint256 amount = (((_alcxAmount * uint256(alcxEthPrice)) / 1 ether) * normalizedWeights[0]) /
        normalizedWeights[1];

    return (amount, normalizedWeights);
}
```

> Code snippet from <https://github.com/alchemix-finance/alchemix-v2-dao/blob/main/src/RewardsDistributor.sol#L116-L133>

The variable "staleThreshold" inside the function `amountToCompound(..)` was hard-coded to 60 days to make running foundry tests that go forward in time easier. Still, if the smart contract is deployed with the hard-coded value, it will accept outdated prices for up to 60 days.

> If there is a problem with chainlink starting a new round and finding consensus on the new value for the oracle (e.g. chainlink nodes abandon the oracle, chain congestion, vulnerability/attacks on the chainlink system) the ***RewardsDistributor*****&#x20;will continue using an outdated price for 60 days** (if oracles are unable to submit no new round is started).

The *RewardsDistributor* is not a test file and is meant to be deployed.

We must audit all in-scope smart contracts as "ready to be deployed if no bugs are found", therefore, we have to report this as a bug.

### Recommendation

**1-** Update the "*staleThreshold*" to 24 hours.

**2-** Instead of hardcoding the "*staleThreshold*" inside a function, make it a global variable that an admin can update by executing a permissioned function.

> This allows the Alchemix team to develop a *RewardsDistributor* that is easy to run time-based tests on and can be safely deployed without any modification, removing the risk of forgetting to update "x value" inside the "function y" inside the "smart contract z" before deployment.

## Impact Details

Using stale prices results in wrong calculations for the amount to compound, which can lead to loss of funds and insolvency.

## Severity

The report is technically valid and bugs that affect the solvency of the protocol are of critical severity. Still, the prerequisite decreases the severity of this finding to medium.

## Proof of Concept

The finding is straightforward to understand, but the boost's policy requires creating a proof of concept, so here's one that can be quickly run in chisel.

In the shell run: `chisel`

Then paste this function and press enter:

```javascript

function amountToCompound() public view returns (bool) {

    uint256 staleThreshold = 60 days;

    // example value for block.timestamp to quickly run this test inside chisel
    uint256 current_timestamp = 1714897791;

    // priceTimestamp is a 10-days old price
    uint256 priceTimestamp = current_timestamp - 10 days;

    // check if the "price is stale" does not revert for a 10-days old price
    require(current_timestamp - priceTimestamp < staleThreshold, "Price is stale");

    // Returns true
    return true;
}
```

Now call it by typing this and pressing enter: `amountToCompound()`
