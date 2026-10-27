# #46999 \[SC-Insight] Absence of event emission in critical functions

**Submitted on Jun 7th 2025 at 16:59:03 UTC by @dldLambda for** [**Audit Comp | Flare | FAssets**](https://immunefi.com/audit-competition/audit-comp-flare-fassets)

* **Report ID:** #46999
* **Report Type:** Smart Contract
* **Report severity:** Insight
* **Target:** <https://github.com/flare-foundation/fassets/blob/main/contracts/assetManager/implementation/FtsoV2PriceStore.sol>
* **Impacts:**
  * Contract fails to deliver promised returns, but doesn't lose value

## Description

## Brief/Intro

The FtsoV2PriceStore contract omits event emissions for two significant actions: (1) when variable "priceOk" is returned - indicating whether the change in median was successful (2) when governance updates critical settings in updateSettings without notifying external observers. These omissions reduce transparency and traceability.

With strong fluctuations in prices, which is absolutely normal for cryptocurrency, this can cause financial losses for users who may rely on the old price.

It is necessary to emit events about "priceOk" (function publishPrices; emit PricesPublished - is not enough, it is necessary to notify about the fact of successful change of median) and it is highly desirable when changing important settings (function updateSettings).

## Vulnerability Details

The vulnerabilities are related to the absence of event emissions in two key functions:

The publishPrices function processes price feeds and trusted prices, calling \_calculateMedian to compute the median price from trusted providers' submissions (submittedTrustedPrices). The \_calculateMedian function checks if the spread around the median is within maxSpreadBIPS:

```
function _calculateMedian(bytes memory _prices) internal view returns (uint256 _medianPrice, bool _priceOk) {
    // ... (price sorting logic)
    uint256 spread = 0;
    uint256 middleIndex = length / 2;
    if (length % 2 == 1) {
        _medianPrice = prices[middleIndex];
        if (length >= 3) {
            spread = prices[middleIndex + 1] - prices[middleIndex - 1];
        }
    } else {
        _medianPrice = (prices[middleIndex - 1] + prices[middleIndex]) / 2;
        spread = prices[middleIndex] - prices[middleIndex - 1];
    }
    _priceOk = spread <= maxSpreadBIPS * _medianPrice / MAX_BIPS;
}
```

In publishPrices, if \_priceOk is false (spread too large), the median is not stored, and submittedTrustedPrices is deleted without emitting an event:

```
if (trustedPrices.length > 0 && trustedPrices.length >= 4 * trustedProvidersThreshold) {
    (uint256 medianPrice, bool priceOk) = _calculateMedian(trustedPrices);
    if (priceOk) {
        priceStore.trustedVotingRoundId = votingRoundId;
        priceStore.trustedValue = uint32(medianPrice);
        priceStore.numberOfSubmits = uint8(trustedPrices.length / 4);
    }
    delete submittedTrustedPrices[feedId][votingRoundId];
}
```

The only event emitted is PricesPublished, which does not indicate whether the trusted price (median) update failed:

```
emit PricesPublished(votingRoundId);
```

This lack of an event for failed updates obscures price rejections, especially during high volatility when trusted providers submit divergent prices.

Settings Update in updateSettings:

The updateSettings function, callable only by governance, updates critical parameters: feedIds, symbols, trustedDecimals, and maxSpreadBIPS.

```
function updateSettings(
    bytes21[] calldata _feedIds,
    string[] calldata _symbols,
    int8[] calldata _trustedDecimals,
    uint16 _maxSpreadBIPS
) external onlyGovernance {
    require(_feedIds.length == _symbols.length && _feedIds.length == _trustedDecimals.length, "length mismatch");
    require(_maxSpreadBIPS <= MAX_BIPS, "max spread too big");
    maxSpreadBIPS = _maxSpreadBIPS;
    feedIds = _feedIds;
    for (uint256 i = 0; i < _feedIds.length; i++) {
        bytes21 feedId = _feedIds[i];
        symbolToFeedId[_symbols[i]] = feedId;
        feedIdToSymbol[feedId] = _symbols[i];
        PriceStore storage latestPrice = latestPrices[feedId];
        if (latestPrice.trustedDecimals != _trustedDecimals[i]) {
            latestPrice.trustedDecimals = _trustedDecimals[i];
            latestPrice.trustedValue = 0;
            latestPrice.trustedVotingRoundId = 0;
            for (uint32 j = lastPublishedVotingRoundId + 1; j <= _getPreviousVotingEpochId(); j++) {
                delete submittedTrustedPrices[feedId][j];
            }
        }
    }
}
```

No event is emitted to signal these changes, despite their impact on price calculations, supported assets, and system behavior. Changes to maxSpreadBIPS or feedIds can alter price update reliability, while trustedDecimals resets affect price validity.

As an example:

parameter "maxSpreadBIPS" influences whether the new median will be accepted or not. If it changes, then the value of "\_priceOk" will be affected. However, the event is not emitted in any of these functions, which is very undesirable and can lead to financial losses in conditions of high volatility, since external systems remain unaware.

## Impact Details

The absence of events for these actions has the following impacts:

Transparency Loss: External systems (e.g., monitoring tools, UI) cannot detect failed price updates, complicating diagnostics during volatile markets.

Downstream Risks: F-asset protocols can rely on outdated prices, leading to: Incorrect collateral calculations, increasing default risks. Errors in redemptions, causing financial losses for users.

Settings Update: Configuration Oversight: Governance changes to feedIds, symbols, trustedDecimals, or maxSpreadBIPS go unnoticed by external systems, risking misconfiguration.

Price Disruption: Resetting trustedDecimals clears trustedValue, potentially disrupting price feeds until new submissions, without alerting dependent systems.

## Recommendations

Add an event in publishPrices to log failed/successful median price updates due to changing spread, and a SettingsUpdated event in updateSettings to track governance changes to feedIds, symbols, trustedDecimals, or maxSpreadBIPS. These events will enhance transparency and enable better monitoring for f-asset systems.

## References

<https://github.com/flare-foundation/fassets/blob/fc727ee70a6d36a3d8dec81892d76d01bb22e7f1/contracts/assetManager/implementation/FtsoV2PriceStore.sol#L88> - function publishPrices

<https://github.com/flare-foundation/fassets/blob/fc727ee70a6d36a3d8dec81892d76d01bb22e7f1/contracts/assetManager/implementation/FtsoV2PriceStore.sol#L171> - function updateSettings

## Proof of Concept

## Proof of Concept

Add and run this test:

```
        it("dl_dLambda_test2", async () => {
            const newTrustedProviders = [accounts[1], accounts[2], accounts[3], accounts[4]];
            await priceStore.setTrustedProviders(newTrustedProviders, 2, { from: governance });

            await time.increaseTo(startTs + 2 * votingEpochDurationSeconds);
            const feeds0 = [];
            const feeds1 = [];
            const feeds2 = [];
            const feeds3 = [];
            for (let i = 0; i < feedIds.length; i++) {
                feeds0.push({ id: feedIds[i], value: 1000000, decimals: feedDecimals[i] });
                feeds1.push({ id: feedIds[i], value: 999900, decimals: feedDecimals[i] });
                feeds2.push({ id: feedIds[i], value: 1000100, decimals: feedDecimals[i] });
                feeds3.push({ id: feedIds[i], value: 1000050, decimals: feedDecimals[i] });
            }
            await priceStore.submitTrustedPrices(1, feeds0, { from: newTrustedProviders[0] });
            await priceStore.submitTrustedPrices(1, feeds1, { from: newTrustedProviders[1] });
            await priceStore.submitTrustedPrices(1, feeds2, { from: newTrustedProviders[2] });
            await priceStore.submitTrustedPrices(1, feeds3, { from: newTrustedProviders[3] });

            await publishPrices();

            const { 0: initialPrice, 1: initialTimestamp, 2: initialDecimals, 3: noOfSubmits } =
                await priceStore.getPriceFromTrustedProvidersWithQuality(feedSymbols[1]);

            assertWeb3Equal(initialPrice, 1000025);
            assertWeb3Equal(initialTimestamp, startTs + 2 * votingEpochDurationSeconds);
            assertWeb3Equal(initialDecimals, feedDecimals[1]);
            assertWeb3Equal(noOfSubmits, 4);

            // ~20% price drop
            await time.increaseTo(startTs + 3 * votingEpochDurationSeconds);

            feeds0[1].value = 800000;
            feeds1[1].value = 790000;
            feeds2[1].value = 810000;
            feeds3[1].value = 900000;
            await priceStore.submitTrustedPrices(2, feeds0, { from: newTrustedProviders[0] });
            await priceStore.submitTrustedPrices(2, feeds1, { from: newTrustedProviders[1] });
            await priceStore.submitTrustedPrices(2, feeds2, { from: newTrustedProviders[2] });
            await priceStore.submitTrustedPrices(2, feeds3, { from: newTrustedProviders[3] });

            // Capture transaction for event checking
            const tx = await publishPrices(true, 2, 2);


            // Note: No other events are emitted, as confirmed by state check 

            const { 0: newPrice, 1: newTimestamp, 2: newDecimals, 3: newNoOfSubmits } =
                await priceStore.getPriceFromTrustedProvidersWithQuality(feedSymbols[1]);

            assertWeb3Equal(newPrice, initialPrice); // 1000025
            assertWeb3Equal(newTimestamp, initialTimestamp); // startTs + 2 * votingEpochDurationSeconds
            assertWeb3Equal(newDecimals, initialDecimals);
            assertWeb3Equal(newNoOfSubmits, 4);

            const { 0: flrPrice, 1: flrTimestamp } = await priceStore.getPriceFromTrustedProviders(feedSymbols[0]);
            assert(flrTimestamp.gt(initialTimestamp));
            assertWeb3Equal(flrPrice, 1000025);
        });
```

This test shows that the median remains old under certain conditions - however, the code makes it clear that there is no event emission and no one will be notified whether a change has occurred.
