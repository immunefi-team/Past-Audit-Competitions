# Boost \_ Folks Finance 33441 - \[Smart Contract - Insight] Protocol uses Pyth to fetch price which is

Submitted on Sat Jul 20 2024 14:05:39 GMT-0400 (Atlantic Standard Time) by @Tripathi for [Boost | Folks Finance](https://immunefi.com/bounty/folksfinance-boost/)

Report ID: #33441

Report type: Smart Contract

Report severity: Insight

Target: <https://testnet.snowtrace.io/address/0xA758c321DF6Cd949A8E074B22362a4366DB1b725>

Impacts:

* Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield

## Description

## Brief/Intro

Protocol uses Pyth to fetch price which is a pull based oracle and requires price updates to be pushed by the user which is not taken care off. There used to be 2 kind of Oracles Pull based on push based.

Push based - <https://docs.pyth.network/price-feeds/pull-updates#push-oracles>

Pull based - <https://docs.pyth.network/price-feeds/pull-updates#pull-oracles>

NOTE - `pyth doesn't push the prices updates onchain contracts ever it is user responsibility to update the price before using it`.

Since Pyth is pull based which requires price updates to be pushed by the user before calling `PythNode::process()` to fetch balances.

## Vulnerability Details

Implementation of Pyth pull based oracle have 2 steps

1. Implement a onchain conract which integrates pyth onchain network this will enable protocol to accept pyth price, update messages and validate that they are authentic by doing sanity checks. This is done in current implementation

```solidity
    function process(bytes memory parameters) internal view returns (NodeOutput.Data memory nodeOutput) {
        (address pythAddress, bytes32 priceFeedId, bool useEma) = abi.decode(parameters, (address, bytes32, bool));

        /// @dev using unsafe methods to avoid reverting, so this accepts old data
        IPyth pyth = IPyth(pythAddress);
        PythStructs.Price memory pythData = useEma
            ? pyth.getEmaPriceUnsafe(priceFeedId)
            : pyth.getPriceUnsafe(priceFeedId);

        /// @dev adjust the price to 18 d.p., exponent is a int32 so it could be negative or positive
        int256 factor = PRECISION + pythData.expo; //@audit-info do we need to have min/max exponent
        uint256 price = factor > 0
            ? pythData.price.toUint256() * (10 ** factor.toUint256())
            : pythData.price.toUint256() / (10 ** factor.toUint256()); 

        return NodeOutput.Data(price, pythData.publishTime, NodeDefinition.NodeType.PYTH, 0, 0);
    }
```

2. Users of the protocol need to update the price onchain before consuming it.

This can be done before calling `PythNode::process()` dispatch a call updatePriceFeeds on the Pyth oracle proxy to refresh the price. Or provide a interface for users for updating price before consuming it.

## Impact Details

Pyth Oracle doesn' update price on its own. User need to update the price before consuming it. Since folks finance fail to integrate Pyth correctly user will end of consuming stale price which is not up to date

## Reference

1. <https://www.youtube.com/watch?v=qdwrs23Qc9g> this could help to understand the integration issues betterr
2. <https://docs.pyth.network/price-feeds/pull-updates#pull-oracles>

or Euler or any other protocol which already sets an example for best integration

## POC

1. Currently `PythNode::process()` uses `pyth.getPriceUnsafe()` and `pyth.getEmaPriceUnsafe()` before updating the price, SO USer will consume the stale price which is not upto date.

Showing a POC on onchain condintions

## Proof of concept

## Proof of Concept

Setting environment from original repo seems difficult so created onchain similar environment to show POC

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.19;

import "@openzeppelin/contracts/utils/math/SafeCast.sol";
import "@openzeppelin/contracts/utils/math/Math.sol";
import "@pythnetwork/pyth-sdk-solidity/IPyth.sol";
import "@pythnetwork/pyth-sdk-solidity/PythStructs.sol";

// Fetching ETH/USDC price

contract ImpementedPythNode {
    using Math for uint256;
    using SafeCast for int64;
    using SafeCast for int256;
     IPyth pyth;

    constructor(address pythContract) {
    pyth = IPyth(pythContract);
    }


// This is the way Folks Finance consume the price
   function ImplementedfetchPrice() external view returns(uint256){
    bytes32 priceFeedId = 0x7f5cc8d963fc5b3d2ae41fe5685ada89fd4f14b435f8050f28c7fd409f40c2d8; // ETH/USD
    PythStructs.Price memory p = pyth.getPrice(priceFeedId);
    int256 factor= 18 + p.expo;
    uint256 price = factor > 0
            ? p.price.toUint256() * (10 ** factor.toUint256())
            : p.price.toUint256() / (10 ** factor.toUint256()); 

            return price; 
   }

//Fetching price priceUpdate will be different than implemented price
   function CorrectFetchPrice(bytes[] calldata priceUpdate) external payable returns(uint256){
// First user need to update the price
     uint fee = pyth.getUpdateFee(priceUpdate);
    pyth.updatePriceFeeds{ value: fee }(priceUpdate);
    
    bytes32 priceFeedId = 0x7f5cc8d963fc5b3d2ae41fe5685ada89fd4f14b435f8050f28c7fd409f40c2d8; // ETH/USD
    PythStructs.Price memory p = pyth.getPrice(priceFeedId);
    int256 factor= 18 + p.expo;
    uint256 price = factor > 0
            ? p.price.toUint256() * (10 ** factor.toUint256())
            : p.price.toUint256() / (10 ** factor.toUint256()); 

            return price; 
   }


}



```

Copy above contract and deploy in the remix.

1. `ImplementedfetchPrice()` represents the [PythNode::process()](https://github.com/Folks-Finance/folks-finance-xchain-contracts/blob/main/contracts/oracle/nodes/PythNode.sol#L23) which consume price from `pyth.getEmaPriceUnsafe()` and `pyth.getPriceUnsafe()` methods.
2. Call `ImplementedfetchPrice()` on any chain or testnet(also change corresponding priceFeedId ) and note the returned price
3. Now call `CorrectFetchPrice()` by updating the price. For `priceUpdate` calldata fetch this from Hermes. And note the price. <https://docs.pyth.network/price-feeds/api-reference/evm/update-price-feeds> this could help

At 1:30UTC eth/usd price went from 3498 to 3490 in some miliseconds. FOr me first call returned \~3494 and second call returned \~3490

The first method is going to return 3498 and correct method will return 3490 USDC. This is what i'm referring. Pull based oracle doesn't update this price updates while user are expected to fetch the price update from offchain `Hermes` in this case and update the price before using it.

In this case

```solidity
    function calcLiquidationAmounts(
        DataTypes.LiquidationLoansParams memory loansParams,
        mapping(bytes32 => LoanManagerState.UserLoan) storage userLoans,
        mapping(uint16 => LoanManagerState.LoanType) storage loanTypes,
        IHubPool collPool,
        IOracleManager oracleManager,
        uint256 maxRepayBorrowValue,
        uint256 maxAmountToRepay
    ) external view returns (DataTypes.LiquidationAmountParams memory liquidationAmounts) {
        ..................................................

        DataTypes.PriceFeed memory borrPriceFeed = oracleManager.processPriceFeed(borrPoolId);
        DataTypes.PriceFeed memory collPriceFeed = oracleManager.processPriceFeed(collPoolId);
   
       ....................................................

}         
```

<https://github.com/Folks-Finance/folks-finance-xchain-contracts/blob/main/contracts/hub/logic/LiquidationLogic.sol#L185>

In such cases processPriceFeed will give inflated or stale values which will lead to loss of funds for user and protocol.

<https://in.tradingview.com/symbols/ETHUSD/> whenever there is a sudden change call both function and check the difference.

Check [Euler](https://github.com/euler-xyz/euler-price-oracle/blob/master/src/adapter/pyth/PythOracle.sol#L13) how manages dispatch a call `updatePriceFeeds` on the Pyth oracle proxy to refresh the price before any consumption of price
