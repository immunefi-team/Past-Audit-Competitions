# Boost \_ IDEX 34437 - \[Smart Contract - Insight] User positions could be unfairly liquidated due to s

Submitted on Mon Aug 12 2024 15:50:10 GMT-0400 (Atlantic Standard Time) by @marchev for [Boost | IDEX](https://immunefi.com/bounty/boost-idex/)

Report ID: #34437

Report type: Smart Contract

Report severity: Insight

Target: <https://github.com/idexio/idex-contracts-ikon/blob/main/contracts/oracle-price-adapters/PythOraclePriceAdapter.sol>

Impacts:

* Permanent freezing of funds

## Description

## Brief/Intro

A vulnerability exists in the protocol where positions could be erroneously liquidated using outdated price data from the Pyth oracle. This issue arises because the protocol accepts any valid Pyth price data, regardless of its age, when updating the last index price of a market as long as it is fresher than the last known market index price. This issue could lead to unfair liquidations, potentially causing users to lose funds.

## Vulnerability Details

The vulnerability is rooted in the way the protocol handles price data from the Pyth oracle. Specifically, positions below maintenance margin are liquidated using the `Exchange#liquidateWalletInMaintenance()` function, which calls `WalletInMaintenanceLiquidation.liquidate_delegatecall()`. This function further invokes `_validateQuoteQuantitiesAndLiquidatePositions()`, which is responsible for ensuring a wallet has fallen below the maintenance threshold.

The protocol calculates the total account value and margin requirements using the `IndexPriceMargin.loadTotalAccountValueInDoublePipsAndMaintenanceMarginRequirementInTriplePips()` function. The total account value is computed as the sum of the USD balance and the value of any open positions. The value of open positions is determined by multiplying the asset price by the last index price, as shown in the following code snippet:

```solidity
totalAccountValueInDoublePips +=
    int256(balanceTracking.loadBalanceFromMigrationSourceIfNeeded(wallet, market.baseAssetSymbol)) *
    Math.toInt64(market.lastIndexPrice);
```

Prices for any given market are updated via `Exchange#publishIndexPrices()`, which calls `MarketAdmin.publishIndexPrices_delegatecall()`. This function validates the price against using `IIndexPriceAdapter#validateIndexPricePayload()`. As per the project documentation, under normal conditions prices are sourced from the Pyth oracle and thus the `PythIndexPriceAdapter` in this context. `PythIndexPriceAdapter#validateIndexPricePayload()` validates price data against the Pyth oracle through the `Pyth#parsePriceFeedUpdates()` function. Notably, this function allows developers to specify a time range within which the price data should be considered valid. However, the `minPublishTime` argument is hardcoded to `0`, meaning any price published since the inception of the Pyth oracle is considered valid, regardless of its age. This flaw could lead to positions being liquidated based on outdated or stale price data, resulting in unfair liquidations.

Although there is a check in `publishIndexPrices_delegatecall()` that ensures the price is fresher than the last one:

```solidity
require(market.lastIndexPriceTimestampInMs < indexPrice.timestampInMs, "Outdated index price");
```

this check is insufficient under certain conditions, such as outages of the off-chain component or other force majeure circumstances. In these cases, the check only ensures the new price is more recent than the last one, but it does not guarantee the price is sufficiently fresh to avoid using stale data.

## Impact Details

The potential impact of this vulnerability includes the erroneous liquidation of user positions due to possible reliance on outdated price data. This could lead to substantial financial losses for users whose positions are unfairly liquidated. While the likelihood of such an event is low due to the generally reliable nature of the Pyth oracle and the protocol’s off-chain component, the consequences of the issue ocuring could be severe.

According to Immunefi’s Vulnerability Severity Classification System, such an issue involving user loss of funds typically warrants a Critical severity rating. However, given the specific circumstances and low likelihood of occurrence, the assessed severity of this vulnerability is Medium.

## Recommendation

It is recommended to introduce a sensible default value for the `minPublishTime` parameter in the `Pyth#parsePriceFeedUpdates()` function. This default value should be set to a timeframe that ensures only fresh and recent price data is accepted when publishing index price data by the protocol. By enforcing a reasonable `minPublishTime`, the protocol would mitigate the risk of using stale or outdated price data, thereby reducing the likelihood of erroneous liquidations and ensuring more accurate and reliable market operations.

## References

* <https://github.com/idexio/idex-contracts-ikon/blob/a4bfee2cb80daec8ba22ee926a13884807d0a94a/contracts/index-price-adapters/PythIndexPriceAdapter.sol#L146-L151>
* <https://docs.pyth.network/price-feeds/api-reference/evm/parse-price-feed-updates>

## Proof of concept

## Proof of Concept

The following coded PoC demonstrates how `Pyth#parsePriceFeedUpdates()` would validate a 1-month old price when `minPublishTime = 0`:

```solidity
// SPDX-License-Identifier: UNLICENSED
pragma solidity ^0.8.13;

import {Test, console} from "forge-std/Test.sol";

contract OutOfGasPythTest is Test {

    Pyth public pyth;

    function setUp() public {
        vm.createSelectFork("https://rpc.ankr.com/arbitrum");
        pyth = Pyth(0xff1a0f4744e8582DF1aE09D5611b887B6a12925C);
    }

    function test_parsePriceFeedUpdates_with_minPublishTime_of_0_would_validate_any_price_since_pyth_inception() public {
        address alice = makeAddr("alice");
        vm.deal(alice, 2 ether);

        vm.startPrank(alice);

        bytes[] memory updateData = new bytes[](1);
        updateData[0] = hex"504e41550100000003b801000000040d00e37edcfcb76086197e97ca974e899399d12833929aadd9e437cc1f2e70e929423072cd954e1fa527b99e3adc50c15923b5658f67a8d5be3be11e224435ed696f0002db47c6d7bdace7afeab893fdfe76a5d626cea0bf3a21dc8c937b4dabc7d1713d7cabe45232ec71874bc3b40f8bf753040f7101b607008cf400d048a9015d8f3a00037ca1ce42b6dcde8a495aeaee2e9a4c4198f6e082da67f522ce85207473480d1d04404199ceef6ae81ebe99223dd72f6161eaf4fe8f3e7f31eec6eb59eb07658301043d2f9fd3f2e54933bff1d45bc1cc8630556dfa397d7079464ca6e4394e3a38e249b07e1dbc67012c388ad25952d16dac5fb27761d1112ba244ee0a9cba9b7226000695a30c124cf8bbb08265020c2438f4c908e29a5dff84c32adffbd9e61f481202049579ad626dfbf6b754f3b1427de1b6602e97179ddbb1154ff2dfcb7deb11ad01082ed2598ffbda99b48998b96cf782367e3ed6c907742baf2b072f4cdf8c79273475dd9700f128e02e11067ce62cf5db71c781c7779ab3ec50d88bf02d3a1ec588000adf531a9c17c9916b753162cde40529d057bb460babd8796d50d6573d7a3c1fdb18fb36e9a08cf4045c6f609438fc61ad7b4ff77b3af7cf267e842daaf9bf622e010bf0cc220d47a083daf38d5cf3d98408469bf0632426d377ebf25fc47b6e0a80f17cbf4e9db0be30b066ef5e9e496930e3268e06eef2b4433eddda16e3a63139af010da48d85ef7f1e662a1ae037214810601ebac8a55f168de719d0b741353e2cc60c0d88f2b2df20eb20af722cc03885f82f7d5d22f1b0bf3f5c8912ed47fa67b020000e2cd8fb90bd84c8e3db01f7fc7443688f19813af325e8604e04392f843d3ab55022c90b807f35697f8a163be7f9e1e04117008148e358884a866b05ffa147c274000f585ccc24302d7789c53b7d02b0303eebe5e681d231d52c9cbf9ecbd9359b1f90408c7df5d53f340f5f819836cd88d6045fc9e82feb8315b4b44444bd9ac4cfde00109d3610c2a822489940f235613ae73247bbd797a0b567d0bfc0692515c6dae2094369c48dbb5ae075e84af7796f1c44e0d6ed337342356077500920607d0fe5a00012d9887326e3e423c96785c4ea50215a89317cffa06b9e0c5515d1e16672e41d135a5fb723fffaa28a39d190d8d31201c80793ed9b12b7114e78e26a255796803100669047d000000000001ae101faedac5851e32b9b23b5f9411a8c2bac4aae3ed4dd7b811dd1a72ea4aa71000000000413a6b101415557560000000000091e27e70000271016b53643b3417e473504e1cdb8481f13f6015a9c01005500e62df6c8b4a85fe1a67db44dc12de5db330f7ac66b72dc658afedf0f4a415b430000053c1fa992cc000000011d96f83efffffff800000000669047d000000000669047cf0000053c6643dc6000000000f118cb0c0b273f87eae8262f745b5ff8a9e3c39a5ec1ad0e085934169b4ebefac01257ee9ac85aa7df284b6195306f43bc3131dcd7a0ea4bb4344211caccaadc928ea0afaabcf647ca4b624b86e66dd2a6474a5a4d10b25e85e51378c1545960b9763a524e90cfab75776c224fabe5917be933ed81ca6aff253a43aaf618a9eded2dbc90dd1849a613c98e5dd02955909e4b831b38e5a52344b39c61fbf59f7fe26b5ad173b1fe1beaef110ebf6d9bf1cbfe8cdbb7c277c3884ac7434b67d648d4d4409bb419084e68b6f91b087afaaaad9ccf31646fa54f9fd10f7d9765941bc8";
        // Price data from 1720731600 (Thu Jul 11 2024 21:00:00 GMT+0000)

        bytes32[] memory priceIds = new bytes32[](1);
        priceIds[0] = 0xe62df6c8b4a85fe1a67db44dc12de5db330f7ac66b72dc658afedf0f4a415b43;

        pyth.parsePriceFeedUpdates{value: pyth.getUpdateFee(updateData)}(
            updateData,
            priceIds,
            uint64(0), // minPublishTime
            uint64(block.timestamp + 1 days) // maxPublishTime
        ); // ❌ Does not revert even with prices as stale as 1 month

        pyth.parsePriceFeedUpdates{value: pyth.getUpdateFee(updateData)}(
            updateData,
            priceIds,
            uint64(block.timestamp - 1 days), // minPublishTime
            uint64(block.timestamp + 1 days) // maxPublishTime
        ); // ✅ Reverts for prices older than 1 day ago

        vm.stopPrank();
    }
}

interface Pyth {

    struct Price {
        int64 price;
        uint64 conf;
        int32 expo;
        uint256 publishTime;
    }

    struct PriceFeed {
        bytes32 id;
        Price price;
        Price emaPrice;
    }

    error PriceFeedNotFoundWithinRange();

    function getUpdateFee(bytes[] memory updateData) external view returns (uint256 feeAmount);

    function parsePriceFeedUpdates(
        bytes[] memory updateData,
        bytes32[] memory priceIds,
        uint64 minPublishTime,
        uint64 maxPublishTime
    ) external payable returns (PriceFeed[] memory priceFeeds);
}
```

Run the PoC via:

```sh
forge test --mt test_parsePriceFeedUpdates_with_minPublishTime_of_0_would_validate_any_price_since_pyth_inception -vvvvv
```
