# 29068 - \[SC - Medium] AaveOracle contract does not verify price stale...

Submitted on Mar 6th 2024 at 14:02:31 UTC by @Paludo0x for [Boost | ZeroLend](https://immunefi.com/bounty/zerolend-boost/)

Report ID: #29068

Report type: Smart Contract

Report severity: Medium

Target: <https://explorer.zksync.io/address/0x785765De3E9ac3D8eEb42B4724A7FEA8990142B8>

Impacts:

* Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield

## Description

## Brief/Intro

**AaveOracle** contract relies on Pyth oracles to retrieve assets price feeds and the actual implementation doesn't check whether prices are fresh or staled. Assets prices are retrieved by means of function `AaveOracle::getAssetPrice()`. For instance this function is called by `LiquidationLogic::_calculateAvailableCollateralToLiquidate` which *"calculates how much of a specific collateral can be liquidated, given a certain amount of debt asset"*. Therefore a wrong calculation can lead to liquidate q wrong amount of base assets.

## Vulnerability Details

This is the code snippet of function `AaveOracle::getAssetPrice()`.

```
  function getAssetPrice(address asset) public view override returns (uint256) {
    AggregatorInterface source = assetsSources[asset];

    if (asset == BASE_CURRENCY) {
      return BASE_CURRENCY_UNIT;
    } else if (address(source) == address(0)) {
      return _fallbackOracle.getAssetPrice(asset);
    } else {
      int256 price = source.latestAnswer();
      if (price > 0) {
        return uint256(price);
      } else {
        return _fallbackOracle.getAssetPrice(asset);
      }
    }
  }
```

The price oracle address is stored in mapping `assetsSources[asset]`. The price is retrieved by means of `int256 price = source.latestAnswer();` and only if `price = 0`the **\_fallbackOracle** is called. Therefore even if price is far from the actual market price it's deemed acceptable.

Let's take as an example **<https://explorer.zksync.io/address/0xf531672C92Ad4658c54B4fBE855029Df43c57390#contract>** which is the oracle for asset **0x4B9eb6c0b6ea15176BBF62841C6B2A8a398cb656** (Dai Stablecoin).

You can see that `PythAggregatorV3::latestAnswer()` calls `pyth.getPriceUnsafe(priceId);`

```
    function latestAnswer() public view virtual returns (int256) {
        PythStructs.Price memory price = pyth.getPriceUnsafe(priceId);
        return int256(price.price);
    }
```

`pyth` address correspond to 0xf087c864AEccFb6A2Bf1Af6A0382B0d0f6c5D834 which is a proxy contract. While the implementation can be found here: <https://explorer.zksync.io/address/0xb1b239054fa2e37da736e43102d175f21c5f7450#contract>

You can see that `getPriceUnsafe` just query the price feed without checking if it is up to date

```
function getPriceUnsafe(
    bytes32 id
) public view virtual override returns (PythStructs.Price memory price) {
    PythStructs.PriceFeed memory priceFeed = queryPriceFeed(id);
    return priceFeed.price;
}
```

## Impact Details

This are examples of where `AaveOracle::getAssetPrice()` is used:

* `LiquidationLogic::_calculateAvailableCollateralToLiquidate` which is called by `LiquidationLogic::executeLiquidationCall` which i turns is called by Pool::liquidationCall\`
* `GenericLogic::calculateUserAccountData` retrieves assets price to verify account health factor, and it's called in turn by `ValidationLogic::validateHealthFactor` or `ValidationLogic.validateBorrow`

That means that if price is staled, especially in a turbolent market condition, can lead to incorrect liquidations or worse, lack of liquidation, and health factor wrong calculation

## Fix suggestion

The advice is to implement a verification if the feed last update is within a thresold by means of `PythAggregatorV3::latestTimestamp()`

```
function latestTimestamp() public view returns (uint256) {
        PythStructs.Price memory price = pyth.getPriceUnsafe(priceId);
        return price.publishTime;
    }

```

## Proof of concept

The requirement of price staleness verification is suggested by Pyth protocol in NATSPEC of function `pyth.getPriceUnsafe(priceId);`

```
 /// @notice Returns the price of a price feed without any sanity checks.
    /// @dev This function returns the most recent price update in this contract without any recency checks.
    /// This function is unsafe as the returned price update may be arbitrarily far in the past.
    ///
    /// Users of this function should check the `publishTime` in the price to ensure that the returned price is
    /// sufficiently recent for their application. If you are considering using this function, it may be
    /// safer / easier to use either `getPrice` or `getPriceNoOlderThan`.
    /// @return price - please read the documentation of PythStructs.Price to understand how to use this safely.
```

Next the following sentence can be found at function docs: <https://docs.pyth.network/price-feeds/api-reference/evm/get-price-unsafe>

*"This function may return a price from arbitrarily far in the past. It is the caller's responsibility to check the returned publishTime to ensure that the update is recent enough for their use case."*
