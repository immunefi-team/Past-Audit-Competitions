# Boost \_ Folks Finance 34153 - \[Smart Contract - Low] TWAP query by chainlink is wrong according to c

Submitted on Tue Aug 06 2024 02:44:31 GMT-0400 (Atlantic Standard Time) by @Ironside\_Sec for [Boost | Folks Finance](https://immunefi.com/bounty/folksfinance-boost/)

Report ID: #34153

Report type: Smart Contract

Report severity: Low

Target: <https://testnet.snowtrace.io/address/0xA758c321DF6Cd949A8E074B22362a4366DB1b725>

Impacts:

* Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield

## Description

## Brief/Intro

Imagine if there's 10 rounds in 30 min TWAP period, if 3rd round query fails (due to unavailability of price update on that round id), then catch triggeres breaking the while loop, and 3 rounds price sum / 3 will be used as TWAP price. Instead, it should skip that round in catch and move to the next round to query so in the end sum of 9 rounds / 9 will be the TWAP price.

Since the Chainlink node is a library used inside node manager, the asset in scope chosen is Node manager

## Vulnerability Details

<https://docs.chain.link/data-feeds/historical-data#roundid-in-aggregator-aggregatorroundid>

`Data feeds are updated in rounds. Rounds are identified by their roundId, which increases with each new round. This increase may not be monotonic. Knowing the roundId of a previous round allows contracts to consume historical data.`

Chainlink says that round ids can be consumed to get previous prices history, but round ids are not increased monotonically. But in chainlnk node library, the price is queried monotonically in line 70 below `--latestRoundId`. The issue here is, the try will fail and trigger the catch if there's no price data in that round. If catch triggers then the loop will break making the TWAP price legit. Currently pools with chainlink node deesn't have parent, but other nodes have chainlink as parent and give chainlink for first priority. In this case, if TWAP price is considered stronger than all and used as fallback, the the attacker will be able to liquidate / repay / deposit at a price away from current legit price.

<https://github.com/Folks-Finance/folks-finance-xchain-contracts/blob/fb92deccd27359ea4f0cf0bc41394c86448c7abb/contracts/oracle/nodes/ChainlinkNode.sol#L60-L73>

```solidity
ChainlinkNode.sol

57:     function getTWAPPrice(
58:         AggregatorV3Interface chainlink,
59:         uint80 latestRoundId,
60:         uint256 latestPrice,
61:         uint256 TWAPTimeInterval
62:     ) internal view returns (uint256 price) {
63:         uint256 priceSum = latestPrice;
64:         uint256 priceCount = 1;
65: 
66:         uint256 startTime = block.timestamp - TWAPTimeInterval;
67: 
69:         while (latestRoundId > 0) {
70:    >>>      try chainlink.getRoundData(--latestRoundId) returns (
71:                 uint80,
72:                 int256 answer,
73:                 uint256,
74:                 uint256 updatedAt,
75:                 uint80
76:             ) {
77:                 if (updatedAt < startTime) {
78:                     break;
79:                 }
80:                 priceSum += answer.toUint256();
81:                 priceCount++;
82:             } catch {
83:    >>>          break;
84:             }
85:         }
86: 
87:         return priceSum / priceCount;
88:     }

```

## Impact Details

TWAP query stops the loop for the whole TWAP duration and still considers it as legit price and this price will be used by the hub pools in determining at what price to deposit/borrow/repay/liquidate.

## References

<https://docs.chain.link/data-feeds/historical-data#roundid-in-aggregator-aggregatorroundid>

<https://github.com/Folks-Finance/folks-finance-xchain-contracts/blob/fb92deccd27359ea4f0cf0bc41394c86448c7abb/contracts/oracle/nodes/ChainlinkNode.sol#L60-L73>

## Proof of concept

## Proof of Concept

Since the Chainlink node is a library used inside node manager, the asset in scope choosen is Node manager

The POC shows the oracle node querying the TWAP on multiple rounds, but to find the round which was not monotonically rised is somewhere in the past and could not be found, so my poc just shows the querying past and if the round is not there then, it will revert with error `No data present`. Also this POC doesn't show exactly how to repay at less price, because current chainlink node has 0 twap duration in LINK pool. So, its a showcase what happens in this reverting case. Also check the attached images

for POC to work,

1. on `https://github.com/Folks-Finance/folks-finance-xchain-contracts` directory, do `forge i foundry-rs/forge-std --no-commit`,
2. then add `ds-test/=node_modules/ds-test/` to `remappings.txt`,
3. then create a file `Foundry.t.sol` on test/ dirctory.
4. Then run the poc with `forge t --mt testIssue -f https://rpc.ankr.com/avalanche_fuji -vvvv`

// SPDX-License-Identifier: UNLICENSED pragma solidity ^0.8.13;

import "forge-std/Test.sol";

import "../contracts/oracle/modules/NodeManager.sol"; import "../contracts/oracle/interfaces/INodeManager.sol"; import "../contracts/oracle/storage/NodeDefinition.sol"; import "../contracts/oracle/storage/NodeOutput.sol"; import "@chainlink/contracts/src/v0.8/interfaces/AggregatorV3Interface.sol";

contract PoC is Test { address from = 0xD5fba05dE4b2d303D03052e8aFbF31a767Bd908e; bytes32 accountId = 0xd32cc9b5264dc39d42622492da52b2b8100e6444367e20c9693ce28fe71286be;

```
NodeManager constant nodeManager = NodeManager(0xA758c321DF6Cd949A8E074B22362a4366DB1b725);

function setUp() public {}

function testIssue() public {
    // https://testnet.snowtrace.io/tx/0x452af7f5cb38378ff97a20811a89596f381935060d49686f96de33911fc30b76?chainid=43113
    bytes memory parameters = abi.encode(address(0x34C4c526902d88a3Aa98DB8a9b802603EB1E3470), 2 hours, 8);
    bytes32[] memory parents;

    bytes32 nodeId = nodeManager.registerNode(NodeDefinition.NodeType.CHAINLINK, parameters, parents);

    NodeOutput.Data memory node = nodeManager.process(nodeId);
    

    AggregatorV3Interface chainlinkAggregator = AggregatorV3Interface(0x34C4c526902d88a3Aa98DB8a9b802603EB1E3470);
    vm.expectRevert();
    (uint80 roundId, int256 answer, , uint256 updatedAt, ) = chainlinkAggregator.getRoundData(19446744073709595053);
}  
```

}
