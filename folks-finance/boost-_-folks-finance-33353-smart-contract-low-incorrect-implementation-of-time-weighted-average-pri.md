# Boost \_ Folks Finance 33353 - \[Smart Contract - Low] Incorrect implementation of Time-Weighted Avera

Submitted on Thu Jul 18 2024 14:05:39 GMT-0400 (Atlantic Standard Time) by @Tripathi for [Boost | Folks Finance](https://immunefi.com/bounty/folksfinance-boost/)

Report ID: #33353

Report type: Smart Contract

Report severity: Low

Target: <https://testnet.snowtrace.io/address/0xA758c321DF6Cd949A8E074B22362a4366DB1b725>

Impacts:

* Protocol insolvency
* Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield

## Description

## Brief/Intro

protocol checks historical data to calculate Time-Weighted Average Price for a Chainlink feed from all the prices between the latest round and the latest round before the time interval. But checking the historical data is incorrect according to the chainlink docs which can damage some serious logic with in the protocol. Since liquidation amount, borrow amount is calculated from fetching price of asset(i.e processPriceFeed)

## Vulnerability Details

```solidity
unction getTwapPrice(
        AggregatorV3Interface chainlink,
        uint80 latestRoundId,
        uint256 latestPrice,
        uint256 twapTimeInterval
    ) internal view returns (uint256 price) {
        uint256 priceSum = latestPrice;
        uint256 priceCount = 1;

        uint256 startTime = block.timestamp - twapTimeInterval;

        /// @dev Iterate over the previous rounds until reaching a round that was updated before the start time
        while (latestRoundId > 0) {
            try chainlink.getRoundData(--latestRoundId) returns (
                uint80,
                int256 answer,
                uint256,
                uint256 updatedAt,
                uint80
            ) {
                if (updatedAt < startTime) {
                    break;
                }
                priceSum += answer.toUint256();
                priceCount++;
            } catch {
                //@audit-issue roundId doesn't decrement one one 
                break;
            }
        }

        return priceSum / priceCount;
    }

```

<https://github.com/Folks-Finance/folks-finance-xchain-contracts/blob/main/contracts/oracle/nodes/ChainlinkNode.sol#L47>

But this is incorrect way of fetching historical data. chainlink docs say:

> Oracles provide periodic data updates to the aggregators. Data feeds are updated in rounds. Rounds are identified by their roundId, which increases with each new round. This increase may not be monotonic. Knowing the roundId of a previous round allows contracts to consume historical data

so it is not mendatory that there will be valid data for currentRoundID-1. if there is not data for currentRooundId-1 then it will just return the weighted average till that time.

weighted average was mean to be from `currentTimestamp` to `currentTimestamp-twapTimeInterval` but it will end of returning spot price which is too close to `currentTimestamp` failing all the logic of using TWAP price.

check this - <https://docs.chain.link/data-feeds/historical-data#solidity>

> roundId is NOT incremental. Not all roundIds are valid. You must know a valid roundId before consuming historical data.

## Impact Details

`HubPool::updatePoolWithDeposit()`

`HubPool::preparePoolForBorrow()`

`Liquidation::calcLiquidationAmounts()`

`LoanManager::executeDepositFToken()`

etc. All these crucial function fetch balance from Chainlink oracles and due to above issue instead of using TWAP till twapTimeInterval they will end up consuming more spot price breaking logic of TWAP.

## Recommendations

As chainlink docs says.

> Increase in roundId may not be monotonic so loop through the previous roundID and fetch the previoous roundId data

```
 iterate (from roundId-1 to untill we get previous first data corressponding to roundID){
    if(data present for roundID){
        fetch the data and return
    }else{
        again iterate to get the data
    }
 }



```

## Proof of concept

## Proof of Concept

1. Replace `MockChainlinkAggregator::getRoundData()` with the below `getRoundData()` function. It reverts with a `RoundId doesn't exist error`, which is the root cause of the issue. Since Chainlink doesn't provide data for every `roundID`, there will be `roundIDs` for which `getRoundData()` reverts

```solidity
    function getRoundData(
        uint80 _roundId
    )
        external
        view
        override
        returns (uint80 roundId, int256 answer, uint256 startedAt, uint256 updatedAt, uint80 answeredInRound)
    {
        require(_prices[_roundId - 1] != 0, "RoundId doesn't exist");
        return (_roundId, _prices[_roundId - 1], 0, _updatedAt[_roundId - 1], roundId);
    }

```

2. Paste the below test in `ChainlinkNode.test.ts`, run the test, and it can be seen that the test shows that TWAP is the same as the spot price due to an issue in `ChainlinkNode::getTwapPrice()`, which validates the issue that `Chainlink getTwapPrice()` is returning the spot price instead of the TWAP price.

```js
import { loadFixture } from "@nomicfoundation/hardhat-toolbox/network-helpers";
import { expect } from "chai";
import { ethers } from "hardhat";
import { MockChainlinkAggregator, NodeManager } from "../../../typechain-types";
import NodeType from "../assets/NodeType";
import { deployNodeManagerFixture } from "../bootstrap";
import { NodeManagerUtil } from "../utils/nodeManagerUtils";
import { PRECISION, abi, deployMockChainlinkAggregator, getOracleNodeId } from "../utils/utils";

describe("ChainlinkNode", async function () {
  let nodeManager: NodeManager;
  let mockChainlinkAggregator: MockChainlinkAggregator;
  let mockChainlinkAggregatorAddr: string;
  let deployBlockTimestamp: number;
  let decimals: number;
  let prices: [number, number, number, number, number];
  let timestampDeltas: [number, number, number, number, number];

  beforeEach("Deploy NodeManager and MockChainlinkAggregator contracts", async function () {
    ({ nodeManager } = await loadFixture(deployNodeManagerFixture));

    decimals = 6;
    prices = [420e5, 426e5, 429e5, 431e5, 432e5];
    timestampDeltas = [60 * 25, 60 * 20, 60 * 15, 60 * 10, 0];
    ({ mockChainlinkAggregator, deployBlockTimestamp } = await deployMockChainlinkAggregator(
      decimals,
      prices,
      timestampDeltas
    ));
    mockChainlinkAggregatorAddr = await mockChainlinkAggregator.getAddress();
  });

  describe("Register node", async function () {
    it("Should register a chainlink node", async function () {
      const twapInterval = 0;
      const decimals = 6;
      const encodedParams = abi.encode(
        ["address", "uint256", "uint8"],
        [mockChainlinkAggregatorAddr, twapInterval, decimals]
      );
      const registerTxn = await nodeManager.registerNode(NodeType.CHAINLINK, encodedParams, []);
      await registerTxn.wait();

      const nodeId = getOracleNodeId(NodeType.CHAINLINK, encodedParams, []);
      const node = await nodeManager.getNode(nodeId);

      expect(node.nodeType).to.equal(NodeType.CHAINLINK);
      expect(node.parameters).to.equal(encodedParams);
      expect(node.parents).to.deep.equal([]);
    });

    it("Should emit InvalidNodeDefinition cause parameters length is not 32*3", async function () {
      const encodedNodeAddress = abi.encode(["address"], [mockChainlinkAggregatorAddr]);

      const registerTxn = nodeManager.registerNode(NodeType.CHAINLINK, encodedNodeAddress, []);

      await expect(registerTxn).to.revertedWithCustomError(nodeManager, "InvalidNodeDefinition");
    });

    it("Should emit InvalidNodeDefinition cause param decimals is not correct", async function () {
      const twapInterval = 0;
      const decimals = 0;
      const encodedParams = abi.encode(
        ["address", "uint256", "uint8"],
        [mockChainlinkAggregatorAddr, twapInterval, decimals]
      );

      const registerTxn = nodeManager.registerNode(NodeType.CHAINLINK, encodedParams, []);

      await expect(registerTxn).to.revertedWithCustomError(nodeManager, "InvalidNodeDefinition");
    });

    it("Should emit InvalidNodeDefinition cause has parent node", async function () {
      const twapInterval = 0;
      const decimals = 0;
      const encodedParams = abi.encode(
        ["address", "uint256", "uint8"],
        [mockChainlinkAggregatorAddr, twapInterval, decimals]
      );
      const fakeParent = ethers.encodeBytes32String("FakeParent");

      const registerTxn = nodeManager.registerNode(NodeType.CHAINLINK, encodedParams, [fakeParent]);

      await expect(registerTxn).to.revertedWithCustomError(nodeManager, "InvalidNodeDefinition");
    });
  });

  describe("Contract methods without TWAP", async function () {
    let twapInterval: number;
    let decimals: number;
    let nodeId: string;

    beforeEach("Register Chainlink node", async function () {
      twapInterval = 0;
      decimals = 6;
      prices = [420e5, 426e5, 429e5, 431e5, 432e5];
      timestampDeltas = [60 * 25, 60 * 20, 60 * 15, 60 * 10, 0];

      const encNodeNoTwapParams = NodeManagerUtil.encodeChainlinkNodeDefinition(
        mockChainlinkAggregatorAddr,
        twapInterval,
        decimals
      );
      nodeId = await NodeManagerUtil.registerNode(nodeManager, encNodeNoTwapParams);
    });

    it("Should process correctly without twap", async function () {
      const nodeOutput = await nodeManager.process(nodeId);

      expect(nodeOutput.price).to.equal(ethers.parseUnits(prices[prices.length - 1].toString(), PRECISION - decimals));
      expect(nodeOutput.timestamp).to.equal(deployBlockTimestamp - timestampDeltas[timestampDeltas.length - 1]);
      expect(nodeOutput.additionalParam1).to.equal(0);
      expect(nodeOutput.additionalParam2).to.equal(0);
    });

    it("Should process price correctly with precision bigger than decimals", async function () {});
    it("Should process twap price correctly with twap interval newer than first update", async function () {});
  });

  describe("Contract methods with TWAP", async function () {
    let mockChainlinkAggregator26Decimals;
    let mockChainlinkAggregator26DecimalsTimestamp: number;
    let twapInterval: number;
    let decimalsTwapNode: number;
    let nodeIdTwap: string;

    beforeEach("Register MockChainlinkAggregator with 20 decimals and register Chainlink node", async function () {
      twapInterval = 60 * 30;
      decimalsTwapNode = 20;

      const MockChainlinkAggregator26Decimals = await ethers.getContractFactory("MockChainlinkAggregator");
      prices = [420e5, 426e5, 429e5, 0, 432e5];
      timestampDeltas = [60 * 25, 60 * 20, 60 * 15, 60 * 10, 0];
      ({
        mockChainlinkAggregator: mockChainlinkAggregator26Decimals,
        deployBlockTimestamp: mockChainlinkAggregator26DecimalsTimestamp,
      } = await deployMockChainlinkAggregator(decimalsTwapNode, prices, timestampDeltas));

      const encNodeTwapParams = NodeManagerUtil.encodeChainlinkNodeDefinition(
        await mockChainlinkAggregator26Decimals.getAddress(),
        twapInterval,
        decimalsTwapNode
      );

      nodeIdTwap = await NodeManagerUtil.registerNode(nodeManager, encNodeTwapParams);
    });

    // it("Should process correctly with twap", async function () {
    //   const twap = prices.reduce((a, b) => a + b, 0) / prices.length;

    //   const nodeOutput = await nodeManager.process(nodeIdTwap);

    //   expect(nodeOutput.price).to.equal(twap * 10 ** (PRECISION - decimalsTwapNode));
    //   expect(nodeOutput.timestamp).to.equal(
    //     mockChainlinkAggregator26DecimalsTimestamp - timestampDeltas[timestampDeltas.length - 1]
    //   );
    //   expect(nodeOutput.additionalParam1).to.equal(0);
    //   expect(nodeOutput.additionalParam2).to.equal(0);
    // });
    it("test which shows that twap is same as spot price due to issue in ChainlinkNode::getTwapPrice()", async function () {
      //@audit-info setting one price to zero so that roundId could revert at that index
      // This will create same environment as chainlink reverting on some roundID during getRoundData()
      const twap = prices.reduce((a, b) => a + b, 0) / (prices.length - 1); //@audit-info since we are skipping one index

      const nodeOutput = await nodeManager.process(nodeIdTwap);
      //@audit-issue nodeOutput.price should be twap price but it returns spot price
      expect(nodeOutput.price).to.equal(432000); //432000 is spot price, price at current timestamp
      expect(twap * 10 ** (PRECISION - decimalsTwapNode)).to.equal(426750); //426750
    });

    it("Should process price correctly with precision bigger than decimals", async function () {});
    it("Should process twap price correctly with twap interval newer than first update", async function () {});
  });
});

```

Linking an [issue](https://solodit.xyz/issues/positions-may-be-liquidated-due-to-incorrect-implementation-of-oracle-logic-codehawks-steadefi-git) where a similar implementation was used. The linked issue is also submitted by me. They accepted it as Medium severity since it was not used in critical functions.

But Here in Folks Finance it is used in calculation of liquidation amounts, borrow positions etc. makes it high severity
