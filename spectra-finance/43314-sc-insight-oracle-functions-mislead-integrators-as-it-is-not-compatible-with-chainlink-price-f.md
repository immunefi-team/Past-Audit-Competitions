# #43314 \[SC-Insight] Oracle functions mislead integrators as it is not compatible with Chainlink Price feed behaviour

**Submitted on Apr 4th 2025 at 14:14:13 UTC by @holydevoti0n for** [**Audit Comp | Spectra Finance**](https://immunefi.com/audit-competition/audit-comp-spectra-finance)

* **Report ID:** #43314
* **Report Type:** Smart Contract
* **Report severity:** Insight
* **Target:** <https://github.com/immunefi-team/Spectra-Audit-Competition/blob/main/src/spectra-oracles/oracles/BaseOracleCurveLPT.sol>
* **Impacts:**

## Description

## Vulnerability Details

All the new oracles from Spectra inherits from `AggregatorV3Interface` which is the Chainlink interface for price feeds. Problem is all oracles from Spectra that implement this interface do not behave as expected for a Chainlink Price Feed Integration.

First issue is:

* The `getRoundData` and `latestRoundData` will always return the same data, as `getRoundData` completely ignores the `roundId` passed as a parameter. Proof here: <https://github.com/immunefi-team/Spectra-Audit-Competition/blob/1cebdc67a9276fd87105d13f302fd77d000d0c0b/src/spectra-oracles/oracles/BaseOracle.sol#L46>

```solidity
    function getRoundData(
        uint80
    )
        external
        view
        returns (
            uint80 roundId,
            int256 answer,
            uint256 startedAt,
            uint256 updatedAt,
            uint80 answeredInRound
        )
    {          // @audit - `latestRoundData` return same output, regardless of the input passed in `getRoundData`
@>        return (0, int256(_getQuoteAmount()), 0, 0, 0);
    }


    /** @dev See {AggregatorV3Interface-latestRoundData}. */
    function latestRoundData()
        external
        view
        returns (
            uint80 roundId,
            int256 answer,
            uint256 startedAt,
            uint256 updatedAt,
            uint80 answeredInRound
        )
    {        // @audit - `latestRoundData` and `getRoundData` return same output, regardless of the input passed in `getRoundData`
@>        return (0, int256(_getQuoteAmount()), 0, 0, 0);
    }
```

* Second issue is that: the returned variables `roundId`, `startedAt`, `updatedAt` and `answeredInRound` are always zero. Integrators use those variables to check whether the current price is still valid. As Spectra always returns those variables with the value `0`, this means the current price being returned is **INVALID**.

You can see this is the Chainlink recommendation for consuming valid price from the price feed oracles: <https://docs.chain.link/data-feeds#check-the-timestamp-of-the-latest-answer>

So Spectra implements Chainlink Price Feed interface for an oracle that do not reflect the Chainlink Implementation's behaviour.

## Impact Details

* The current implementation always returns an "invalid price" according to Chainlink, which will mislead integrators to avoid using Spectra's oracles.
* `getRoundData` will mislead integrators that wants to check specific rounds for prices as this function works exactly as `latestRoundData` returning the latest price.

## Recommendation

Spectra should not use `AggregatorV3Interface` to its oracles as they are not reflecting a Chainlink Price Feed.

Spectra should create a specific interface based on its current implementation and remove the Chainlink functions completely.

## Proof of Concept

## Proof of Concept
