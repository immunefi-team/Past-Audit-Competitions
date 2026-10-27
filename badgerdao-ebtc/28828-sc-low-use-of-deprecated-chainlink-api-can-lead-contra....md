# 28828 - \[SC - Low] Use of deprecated Chainlink API can lead contra...

Submitted on Feb 28th 2024 at 04:14:22 UTC by @holydevoti0n for [Boost | eBTC](https://immunefi.com/bounty/ebtc-boost/)

Report ID: #28828

Report type: Smart Contract

Report severity: Low

Target: <https://github.com/ebtc-protocol/ebtc/blob/release-0.7/packages/contracts/contracts/ChainlinkAdapter.sol>

Impacts:

* Block stuffing
* Contract fails to deliver promised returns, but doesn't lose value

## Description

## Brief/Intro

The `ChainlinkAdapter` is used to aggregate the prices of BTC/USD and ETH/USD. Also it serves as one of the sources of truth for the price on the `PriceFeed`. The problem is that the the `ChainlinkAdapter` uses a deprecated function from Chainlink `latestRound()`: <https://github.com/ebtc-protocol/ebtc/blob/a96bd000c23425f04c3223a441a625bfb21f6686/packages/contracts/contracts/ChainlinkAdapter.sol#L67>

As stated by Chainlink this function is deprecated and it should not be used: <https://docs.chain.link/data-feeds/api-reference#latestround>

Also in the live contracts, there is even a doc stating the deprecation: <https://etherscan.io/address/0x986b5E1e1755e3C2440e960477f25201B0a8bbD4#code#L226\\&gt>

```
* @dev #[deprecated] Use latestRoundData instead. This does not error if no
   * answer has been reached, it will simply return 0. Either wait to point to
   * an already answered Aggregator or use the recommended latestRoundData
   * instead which includes better verification information.
```

## Vulnerability Details

Once in future upgrades of the Chainlink price feeds the `latestRounded` is no longer supported, this will always return zero. By returning zero, this will lead the function `_chainlinkIsBroken()` on `PriceFeed` to always return true as the `prevChainlinkResponse` will either revert or return 0 because `latestRoundId` fetched from `uint80 latestRoundId = _feed.latestRound();` will be always zero:

```solidity
  (feedRoundId, answer, , updatedAt, ) = _feed.getRoundData(
            _roundId == CURRENT_ROUND ? latestRoundId : latestRoundId - 1
        );
```

`getRoundData` from `ChainlinkAdapter` is called here: <https://github.com/ebtc-protocol/ebtc/blob/a96bd000c23425f04c3223a441a625bfb21f6686/packages/contracts/contracts/PriceFeed.sol#L759>

## Impact Details

This will prevent the ChainlinkAdapter from working as expected and the system will have to rely on the fallback oracle or the last good price seen by BTC.

## Recommendations

Use `latestRoundData` to fetch the latest round id as recommended by Chainlink.

## References

Add any relevant links to documentation or code <https://docs.chain.link/data-feeds/api-reference#latestround> <https://etherscan.io/address/0x986b5E1e1755e3C2440e960477f25201B0a8bbD4#code#L226\\&gt> <https://github.com/ebtc-protocol/ebtc/blob/a96bd000c23425f04c3223a441a625bfb21f6686/packages/contracts/contracts/PriceFeed.sol#L759>

## Proof of Concept

Add the PoC below on `ChainlinkAdapter.t.sol` and run: `forge test --match-test testLatestRoundWhenDeprecatedWillRevertDueToUnderflow`

```solidity
function testLatestRoundWhenDeprecatedWillRevertDueToUnderflow() public {
        usdBtcAggregator.setLatestRoundId(0);
        usdBtcAggregator.setPrevRoundId(110680464442257320246);
        usdBtcAggregator.setPrice(3983705362408);
        usdBtcAggregator.setPrevPrice(3983705362407);
        usdBtcAggregator.setUpdateTime(1706208947);

        ethUsdAggregator.setLatestRoundId(0);
        ethUsdAggregator.setPrevRoundId(110680464442257320664);
        ethUsdAggregator.setPrice(221026137517);
        ethUsdAggregator.setPrevPrice(221026137516);
        ethUsdAggregator.setUpdateTime(1706208947);

        // uint80 latestRoundId = _feed.latestRound(); = 0
        // then
        // (feedRoundId, answer, , updatedAt, ) = _feed.getRoundData(
        //    _roundId == CURRENT_ROUND ? latestRoundId : latestRoundId - 1 (0-1 from uint - underflow)
        //);

        chainlinkAdapter.getRoundData(chainlinkAdapter.PREVIOUS_ROUND());
    }
```

Result:

```
Encountered 1 failing test in foundry_test/ChainlinkAdapter.t.sol:ChainlinkAdapterTest
[FAIL. Reason: panic: arithmetic underflow or overflow (0x11)] testLatestRoundWhenDeprecatedWillRevertDueToUnderflow() (gas: 204663)
```
