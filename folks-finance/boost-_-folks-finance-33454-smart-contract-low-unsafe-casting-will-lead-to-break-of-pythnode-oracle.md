# Boost \_ Folks Finance 33454 - \[Smart Contract - Low] unsafe casting will lead to break of PythNode O

Submitted on Sat Jul 20 2024 20:46:30 GMT-0400 (Atlantic Standard Time) by @Tripathi for [Boost | Folks Finance](https://immunefi.com/bounty/folksfinance-boost/)

Report ID: #33454

Report type: Smart Contract

Report severity: Low

Target: <https://testnet.snowtrace.io/address/0xA758c321DF6Cd949A8E074B22362a4366DB1b725>

Impacts:

* Protocol insolvency
* Temporary freezing of funds of at least 24h

## Description

## Brief/Intro

`PythNode` tries to cast a negative number to uint256 . Which is not possible

## Vulnerability Details

```solidity
    function process(bytes memory parameters) internal view returns (NodeOutput.Data memory nodeOutput) {
        (address pythAddress, bytes32 priceFeedId, bool useEma) = abi.decode(parameters, (address, bytes32, bool));

        /// @dev using unsafe methods to avoid reverting, so this accepts old data
        IPyth pyth = IPyth(pythAddress);
        PythStructs.Price memory pythData = useEma
            ? pyth.getEmaPriceUnsafe(priceFeedId)
            : pyth.getPriceUnsafe(priceFeedId);

        /// @dev adjust the price to 18 d.p., exponent is a int32 so it could be negative or positive
        int256 factor = PRECISION + pythData.expo;
        uint256 price = factor > 0
            ? pythData.price.toUint256() * (10 ** factor.toUint256())
            : pythData.price.toUint256() / (10 ** factor.toUint256());

        return NodeOutput.Data(price, pythData.publishTime, NodeDefinition.NodeType.PYTH, 0, 0);
    }
```

<https://github.com/Folks-Finance/folks-finance-xchain-contracts/blob/main/contracts/oracle/nodes/PythNode.sol#L23>

factor is calculated as `PRECISION + pythData.expo`. Since pythData.expo can be both positive and negative. WHenever `factor = PRECISION + pythData.expo < 0` in second line it calls `factor.toUint256()` {using SafeCast for int256} which will revert with `SafeCastOverflowedIntToUint()` error

## Impact Details

Price mechanism breaks if factor<0. which renders most of protocol function useless

## References

## Proof of concept

## Proof of Concept

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.19;

import "@openzeppelin/contracts/utils/math/SafeCast.sol";
import "@openzeppelin/contracts/utils/math/Math.sol";



contract ImpementedPythNode {
    using Math for uint256;
    using SafeCast for int64;
    using SafeCast for int256;
  
   int256 public constant PRECISION = 18;
  

// This is the way Folks Finance consume the price
   function process(int32 expo) external pure returns( uint256){
    int256 factor = PRECISION + expo;

     factor.toUint256();
     return factor.toUint256();
   }
}

```

copy and paste above code in remix . deploy and call `process()` function with a expo param which makes `factor = PRECISION + expo<0`

eg process(-19) or process(-20) etc
