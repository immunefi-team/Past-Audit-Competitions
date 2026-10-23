# #46982 \[SC-Insight] Spread calculation discrepancy allows wildly divergent prices to be accepted

**Submitted on Jun 7th 2025 at 08:38:29 UTC by @dawn for** [**Audit Comp | Flare | FAssets**](https://immunefi.com/audit-competition/audit-comp-flare-fassets)

* **Report ID:** #46982
* **Report Type:** Smart Contract
* **Report severity:** Insight
* **Target:** <https://github.com/flare-foundation/fassets/blob/main/contracts/assetManager/implementation/FtsoV2PriceStore.sol>
* **Impacts:**
  * Protocol insolvency

## Description

## Brief/Intro

A significant flaw exists in the `_calculateMedian` function within `FtsoV2PriceStore.sol` that compromises its ability to accurately assess price stability. The issue stems from an inconsistent calculation of the spread (how dispersed prices are) depending on whether there's an odd or even number of price inputs. When there's an even number of prices, the system only considers the two middle values to determine the spread, effectively ignoring extreme price fluctuations. This allows a bypass of the crucial spread check, potentially leading to a misleading median price that doesn't reflect the true market value, especially during volatile periods. This inaccurate price can then be exploited, either by manipulation or error, to artificially inflate or deflate collateral ratios for agents. This could prevent necessary liquidations of undercollateralized agents, risking F-Asset holders, or trigger unwarranted liquidations.

## Vulnerability Details

Below is a snippet of the median and spread calculation of the `_calculateMedian` function in `FtsoV2PriceStore.sol`:

```solidity
   function _calculateMedian(bytes memory _prices) internal view returns (uint256 _medianPrice, bool _priceOk) {
        //...
        uint256 spread = 0;
        uint256 middleIndex = length / 2;
        if (length % 2 == 1) {
            _medianPrice = prices[middleIndex];
            if (length >= 3) {
                spread = prices[middleIndex + 1] - prices[middleIndex - 1];
            }
        } else {
            // if median is "in the middle", take the average price of the two consecutive prices
            _medianPrice = (prices[middleIndex - 1] + prices[middleIndex]) / 2;
            spread = prices[middleIndex] - prices[middleIndex - 1];
        }
        // check if spread is within the limit
        _priceOk = spread <= maxSpreadBIPS * _medianPrice / MAX_BIPS; // no overflow
    }
```

For odd length it sets the `_medianPrice` to the middle element (`prices[middleIndex]`) and calculate the `spread` as the difference between the price just above and below the median (`prices[middleIndex + 1] - prices[middleIndex - 1]`);

For even length it calculates the `_medianPrice` as the average of the two middle prices (`(prices[middleIndex - 1] + prices[middleIndex]) / 2`) and calculate the `spread` as their difference (`prices[middleIndex] - prices[middleIndex - 1]`);

This inconsistency allows the check to be bypassed. The even-length case is significantly weaker, as it completely ignores the values at the extremes of the dataset. For even lengths, `_priceOk` may result to true when prices are widely dispersed, as long as the two middle prices are close, undermining the spread check’s purpose.

## Impact Details

The result of the `_calculateMedian` function could be based on a median of widely divergent prices. This median itself might not be representative of the true market price during high market volatility, or it could be easily manipulated if a few trusted providers collude or submit erroneous data. This result is used in the `publishPrices` function, it will be stored on the `priceStore.trustedValue` variable. This variable is accessed by the `_getPriceFromTrustedProviders` function which is relied heavily by an `AssetManager` for accurate price feeds (both FTSO scaling and trusted prices) to calculate agent collateral ratios (CR). The function `Liquidation.getCollateralRatioBIPS` uses the maximum of the FTSO-derived CR and the trusted-price-derived CR. This could make an agent's collateral appear more valuable than it is, or the F-Assets less valuable, their calculated CR could be artificially inflated. This might prevent necessary liquidations or CCB triggers, allowing undercollateralized agents to continue operating, putting F-Asset holders at risk. Or vice versa which makes collateral seem less valuable or F-Assets more valuable, an agent's CR could be artificially deflated, this could trigger unwarranted liquidations.

## Proof of Concept

For brevity, the variable initializations and sort logic step by step calculations are skipped.

`_prices`: \[0, 100, 101, 1000]

`length`: 4

`maxSpreadBIPS`: 100 (1%)

Median and spread calculation:

```solidity
        uint256 spread = 0;
        uint256 middleIndex = length / 2;
        if (length % 2 == 1) {
            _medianPrice = prices[middleIndex];
            if (length >= 3) {
                spread = prices[middleIndex + 1] - prices[middleIndex - 1];
            }
        } else {
            // if median is "in the middle", take the average price of the two consecutive prices
            _medianPrice = (prices[middleIndex - 1] + prices[middleIndex]) / 2;
            spread = prices[middleIndex] - prices[middleIndex - 1];
        }
```

`middleIndex` = length / 2 = 4 / 2 = 2

`length` is 4 so the else block is entered:

`_medianPrice` = (prices\[middleIndex - 1] + prices\[middleIndex]) / 2

* `_medianPrice` = (100 + 101) / 2 = 201 / 2 = 100 (integer division)

`spread` = prices\[middleIndex] - prices\[middleIndex - 1]

* `spread` = 101 - 100 = 1

`_priceOk` = spread <= maxSpreadBIPS \* \_medianPrice / MAX\_BIPS;

* `_priceOk` = 1 <= 100 \* 100 / 10000;
* `_priceOk` = 1 <= 10000 / 10000;
* `_priceOk` = 1 <= 1;
* `_priceOk` = true;

Final Output:

`_medianPrice` = 100

`_priceOk` = true

The spread is only 1 which is very small and barely pass the `_priceOk` check, while actually the prices contain 0 and 1000 and the range of the dataset is 1000 which is extremely large compared to the median of 100.
