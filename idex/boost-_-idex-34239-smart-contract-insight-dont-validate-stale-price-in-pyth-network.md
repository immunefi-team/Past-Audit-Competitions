# Boost \_ IDEX 34239 - \[Smart Contract - Insight] Dont validate stale price in Pyth Network

Submitted on Wed Aug 07 2024 10:21:25 GMT-0400 (Atlantic Standard Time) by @Hoverfly9132 for [Boost | IDEX](https://immunefi.com/bounty/boost-idex/)

Report ID: #34239

Report type: Smart Contract

Report severity: Insight

Target: <https://github.com/idexio/idex-contracts-ikon/blob/main/contracts/oracle-price-adapters/PythOraclePriceAdapter.sol>

Impacts:

* Protocol insolvency

## Description

## Bug Description

The [`PythOraclePriceAdapter#loadPriceForBaseAssetSymbol`](https://github.com/idexio/idex-contracts-ikon/blob/a4bfee2cb80daec8ba22ee926a13884807d0a94a/contracts/oracle-price-adapters/PythOraclePriceAdapter.sol#L103) get price by pyth oracle `getPriceUnsafe` function:

```solidity
function loadPriceForBaseAssetSymbol(string memory baseAssetSymbol) public view returns (uint64 price) {
    PythMarket memory market = marketsByBaseAssetSymbol[baseAssetSymbol];
    require(market.exists, "Unknown base asset symbol");

    // @audit-issue - may get stale price
    PythStructs.Price memory pythPrice = pyth.getPriceUnsafe(market.priceId);

    uint64 priceInPips = _priceToPips(pythPrice.price, pythPrice.expo, market.priceMultiplier);
    require(priceInPips > 0, "Unexpected zero price");

    return priceInPips;
  }
```

However, the `getPriceUnsafe` function may return stale price as the official [describe](https://github.com/pyth-network/pyth-sdk-solidity/blob/c24b3e0173a5715c875ae035c20e063cb900f481/IPyth.sol#L30-L37):

`/// @notice Returns the price of a price feed without any sanity checks. /// @dev This function returns the most recent price update in this contract without any recency checks. /// This function is unsafe as the returned price update may be arbitrarily far in the past. /// /// Users of this function should check the` publishTime`in the price to ensure that the returned price is /// sufficiently recent for their application. If you are considering using this function, it may be /// safer / easier to use either`getPrice`or`getPriceNoOlderThan`. /// @return price - please read the documentation of PythStructs.Price to understand how to use this safely.`

So it may return stale price but the protocol don't validate it.

## Impact

The protocol may use stale pyth price may cause users asset account error.

## Recommendation

Using `pyth.updatePriceFeeds` for updating prices, followed by `pyth.getPrice` for retrieval. Following the example in: <https://github.com/pyth-network/pyth-sdk-solidity/blob/main/README.md#example-usage>

Impact: High Likelihood: Low

So i evaluate this issue is medium.

## Proof of concept

## PoC

The finding is easy to understand but as boost rule we need provide PoC, so we mock the `loadPriceForBaseAssetSymbol` function with `getPriceUnsafe` function, compare the price before and after.

```solidity
contract MockPriceConsumer {
    PythStructs.Price public price;

    constructor() {
    }

    function loadPriceForBaseAssetSymbol(bytes32 priceId) public view returns (uint64 price) {
        uint64 eth_usdc = 2000e6;
        return eth_usdc;
    }

    function updatePriceFeeds(uint publishTime) public {
        price = PythStructs.Price({price: 2000e6, conf: 1, expo: -8, publishTime: publishTime});
    }

    function getPublishTime() public view returns (uint) {
        return price.publishTime;
    }
}


contract PythUnsafeTest is Test {

    function testUnsafePrice() public {
        MockPriceConsumer mockPriceConsumer = new MockPriceConsumer();
        mockPriceConsumer.updatePriceFeeds(block.timestamp);
        uint pubTime1 = mockPriceConsumer.getPublishTime();
        uint64 p1 = mockPriceConsumer.loadPriceForBaseAssetSymbol(0x00);
        
        vm.warp(block.timestamp + 3600*24);
        uint64 p2 = mockPriceConsumer.loadPriceForBaseAssetSymbol(0x00);
        uint pubTime2 = mockPriceConsumer.getPublishTime();
        assertEq(p1, p2);
        assertEq(pubTime1, pubTime2);
    }
}
```
