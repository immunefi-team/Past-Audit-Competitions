# 29445 - \[SC - Insight] latestRoundData Call May Result Stale

Submitted on Mar 19th 2024 at 05:06:55 UTC by @caglankaan for [Boost | Immunefi Arbitration](https://immunefi.com/bounty/immunefiarbitration-boost/)

Report ID: #29445

Report type: Smart Contract

Report severity: Insight

Target: <https://github.com/immunefi-team/vaults/blob/main/src/oracles/FeedRegistryL2.sol>

Impacts:

* Stale Data Can Be Used

## Description

## Brief/Intro

Data returned from chainlink might be old.

## Vulnerability Details

The contract calls out to a Chainlink oracle receiving the latestRoundData(). If there is a problem with Chainlink starting a new round and finding consensus on the new value for the oracle (e.g. Chainlink nodes abandon the oracle, chain congestion, vulnerability/attacks on the chainlink system) consumers of this contract may continue using outdated stale or incorrect data (if oracles are unable to submit no new round is started).Take a look at the [Chainlink documentation](https://docs.chain.link/data-feeds/price-feeds/historical-data#getrounddata-return-values)

## Impact Details

The latestRoundData() could return stale price data for the underlying asset.

## References

```solidity
Path: ./src/oracles/FeedRegistryL2.sol

100:        (, int256 answer, uint256 startedAt, , ) = SEQUENCER_UPTIME_FEED.latestRoundData();	// @audit-issue

```

[100](https://github.com/immunefi-team/vaults/blob/49c1de26cda19c9e8a4aa311ba3b0dc864f34a25/./src/oracles/FeedRegistryL2.sol#L100-L100)

## Recommendation

Implement comprehensive checks to validate the freshness of the data returned by Chainlink's `latestRoundData()` in your smart contracts. This includes verifying the timestamp of the latest round against a permissible time window to ensure the data's relevance and accuracy. Additionally, consider using Chainlink's `getRoundData()` function with specific round IDs for historical data checks and to verify data continuity. Add the following checks:

```solidity
...
( roundId, rawPrice, , updateTime, answeredInRound ) = AggregatorV3Interface(XXXXX).latestRoundData();
require(rawPrice > 0, "Chainlink price <= 0");
require(updateTime != 0, "Incomplete round");
require(answeredInRound >= roundId, "Stale price");
...
```

## Proof of Concept
