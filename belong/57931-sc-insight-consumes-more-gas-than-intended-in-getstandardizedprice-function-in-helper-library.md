# 57931 sc insight consumes more gas than intended in getstandardizedprice function in helper library

**Submitted on Oct 29th 2025 at 14:03:36 UTC by @iehnnkta for** [**Audit Comp | Belong**](https://immunefi.com/audit-competition/audit-comp-belong)

* Report ID: #57931
* Report Type: Smart Contract
* Report severity: Insight
* Target: <https://github.com/immunefi-team/audit-comp-belong/blob/main/contracts/v2/utils/Helper.sol>
* Impacts: (not specified in the original report)

## Description

`getStandardPrice` uses `standardize`, which internally calls `_standardise` (or `_standardize`) where the actual operation happens. The extra indirection through the public `standardize` mediator function causes slightly higher gas usage. While this is a small overhead, the protocol is intended to be deployed on Ethereum mainnet where gas costs are significant.

A gas composition comparison is provided in the Proof of Concept section.

## Vulnerability Details

* The gas overhead arises because `getStandardizedPrice` is calling a public function `standardize` which in turn calls the internal/private `_standardize` function, instead of calling the internal function directly.
* This indirection results in a small but measurable increase in gas consumption for the operation.

Reference location in the target repository: <https://github.com/immunefi-team/audit-comp-belong/blob/a17f775dcc4c125704ce85d4e18b744daece65af/contracts/v2/utils/Helper.sol#L106>

## Proof of Concept

The following minimal contract reproduces the observed difference:

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

import {MetadataReaderLib} from "solady/src/utils/MetadataReaderLib.sol";
import {FixedPointMathLib} from "solady/src/utils/FixedPointMathLib.sol";

contract getter {
    /// @dev Used for precise calculations.
    using FixedPointMathLib for uint256;
    /// @dev Used for metadata reading (e.g., token decimals).
    using MetadataReaderLib for address;

    uint256 public constant BPS = 10 ** 27;

    function getStandardizedPrice(address token, address tokenPriceFeed, uint256 amount, uint256 maxPriceFeedDelay)
        external
        pure
        returns (uint256 priceAmount)
    {
        (uint256 tokenPriceInUsd, uint8 pfDecimals) = getPrice(tokenPriceFeed, maxPriceFeedDelay);
        // (amount * price) / 10^priceFeedDecimals
        uint256 usdValue = amount.fullMulDiv(tokenPriceInUsd, 10 ** pfDecimals);
        // Standardize the USD value to 27 decimals
        priceAmount = _standardize(18, usdValue); //@audit-issue [Insight] use internal function
    }

    function standardize(address, uint256 amount) public pure returns (uint256) {
        return _standardize(18, amount);
    }

    function _standardize(uint8 decimals, uint256 amount) private pure returns (uint256) {
        return amount.fullMulDiv(BPS, 10 ** decimals);
    }

    function getPrice(address, uint256)
        public
        pure
        returns (uint256 price, uint8 decimals)
    {
        return (100000000, 8); // indicates $1 (for token/USD - the price is in 8 decimals), 18 decimals
    }
}
```

Observed results when running in RemixIDE:

* With `standardize` (indirect call) — gas consumption: 2690\
  Execution logs: <https://ibb.co/tMn2YChg>
* With direct call to `_standardize` — gas consumption: 2642\
  Execution logs: <https://ibb.co/dwYdfbth>

<details>

<summary>Execution logs</summary>

* With `standardize`: <https://ibb.co/tMn2YChg>
* With `_standardize`: <https://ibb.co/dwYdfbth>

</details>

## References

* Contract location referenced:\
  <https://github.com/immunefi-team/audit-comp-belong/blob/a17f775dcc4c125704ce85d4e18b744daece65af/contracts/v2/utils/Helper.sol#L106>
