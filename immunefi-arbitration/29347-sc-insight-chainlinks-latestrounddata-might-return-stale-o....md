# 29347 - \[SC - Insight] Chainlinks latestRoundData might return stale o...

Submitted on Mar 14th 2024 at 16:48:24 UTC by @OxRizwan for [Boost | Immunefi Arbitration](https://immunefi.com/bounty/immunefiarbitration-boost/)

Report ID: #29347

Report type: Smart Contract

Report severity: Insight

Target: <https://github.com/immunefi-team/vaults/blob/main/src/oracles/PriceConsumer.sol>

Impacts:

* Contract fails to deliver promised returns, but doesn't lose value

## Description

## Brief/Intro

Chainlink's latestRoundData might return stale or incorrect results leading to incorrect prices

## Vulnerability Details

`PriceConsumer.getChainlinkPrice18Decimals()` fetches the asset price from a Chainlink aggregator using the `latestRoundData` function. However, there are no checks on roundID, resulting in stale prices. The oracle wrapper calls out to a chainlink oracle receiving the latestRoundData().

```solidity
    function getChainlinkPrice18Decimals(
        address base,
        address quote
    ) public view returns (PriceResponse memory response) {
@>      try registry.latestRoundData(base, quote) returns (
            uint80 roundId,
            int256 answer,
            uint256 /* startedAt */,
            uint256 updatedAt,
            uint80 /* answeredInRound */
        ) {
            response.success = true;
            response.roundId = roundId;
            response.answer = _convertTo18Decimals(answer, registry.decimals(base, quote));
            response.updatedAt = updatedAt;
        } catch {}
    }
```

Stale prices could put funds at risk. According to Chainlink's documentation, This function does not error if no answer has been reached but returns 0, causing an incorrect price fed to the PriceOracle. The external Chainlink oracle, which provides index price information to the system, introduces risk inherent to any dependency on third-party data sources.

For example, the oracle could fall behind or otherwise fail to be maintained, resulting in outdated data being fed to the index price calculations. Oracle reliance has historically resulted in crippled on-chain systems, and complications that lead to these outcomes can arise from things as simple as network congestion.

## Impact Details

The Oracle can return stale data which does not reflect the most recent price of the asset, which could leas to incorrect calculations, and users or the protocol could end up losing money because of the discrepancy between the returned price and the real current value of the asset.

## References

<https://consensys.io/diligence/audits/2021/09/fei-protocol-v2-phase-1/#chainlinkoraclewrapper-latestrounddata-might-return-stale-results>

<https://github.com/code-423n4/2021-05-fairside-findings/issues/70>

## Recommendation to fix

Consider adding missing checks for stale data.

For example:

```solidity
(uint80 roundID, int256 feedPrice, , uint256 timestamp, uint80 answeredInRound) = feed.latestRoundData();
require(feedPrice > 0, "Chainlink price <= 0"); 
require(answeredInRound >= roundID, "Stale price");
require(timestamp != 0, "Round not complete");
```

## Proof of Concept
