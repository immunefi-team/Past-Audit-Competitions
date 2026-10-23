# Boost \_ Folks Finance 33540 - \[Smart Contract - Low] ChainlinkNode uses cached decimals in the calcu

Submitted on Mon Jul 22 2024 21:35:42 GMT-0400 (Atlantic Standard Time) by @Tripathi for [Boost | Folks Finance](https://immunefi.com/bounty/folksfinance-boost/)

Report ID: #33540

Report type: Smart Contract

Report severity: Low

Target: <https://testnet.snowtrace.io/address/0xA758c321DF6Cd949A8E074B22362a4366DB1b725>

Impacts:

* Contract fails to deliver promised returns, but doesn't lose value

## Description

## Brief/Intro

ChainlinkNode uses cached decimals in the calculation of 18 decimal precision, which can lead to inflated or deflated prices. The ChainlinkNode fetches the Chainlink price feed's decimals once during the registration of the node in `NodeManager::registerNode()` and same decimal is used in every price calculation

As Chainlink price feeds fetch the decimals from the current aggregator and the aggregator can be changed, `the decimals could also change`. This would lead to incorrect price conversion if the price with new decimals is used with old cached decimals.

## Vulnerability Details

Let's look at the `NodeManager::registerNode()`

```solidity
    function registerNode(
        NodeDefinition.NodeType nodeType,
        bytes calldata parameters,
        bytes32[] calldata parents
    ) external override returns (bytes32 nodeId) {
        NodeDefinition.Data memory nodeDefinition = NodeDefinition.Data({
            nodeType: nodeType,
            parameters: parameters,
            parents: parents
        });
        return _registerNode(nodeDefinition);
    }

```

<https://github.com/Folks-Finance/folks-finance-xchain-contracts/blob/main/contracts/oracle/modules/NodeManager.sol#L20>

Assuming the node is a Chainlink Node, it calls the internal `_registerNode()`:

```solidity
    function _registerNode(NodeDefinition.Data memory nodeDefinition) internal returns (bytes32 nodeId) {
    ..............................
        /// @dev Check if the node definition is valid
        if (!_isValidNodeDefinition(nodeDefinition)) {
            revert InvalidNodeDefinition(nodeDefinition);
        }
    ..............................

    }

```

<https://github.com/Folks-Finance/folks-finance-xchain-contracts/blob/main/contracts/oracle/modules/NodeManager.sol#L85> it ensures that the NodeDefinition is valid by checking the decimal parameter with the corresponding Chainlink Aggregator decimals in `_isValidNodeDefinition()`.

`_isValidNodeDefinition()` calls `ChainlinkNode::isValid()` to check if the node definition is valid:

```solidity
    function isValid(NodeDefinition.Data memory nodeDefinition) internal view returns (bool) {
        /// @dev Must have no parents and three parameters: contract address, twapInterval, decimals
        if (nodeDefinition.parents.length > 0 || nodeDefinition.parameters.length != 32 * 3) {
            return false;
        }

        (address chainlinkAggregatorAddr, , uint8 decimals) = abi.decode(
            nodeDefinition.parameters,
            (address, uint256, uint8)
        );
        AggregatorV3Interface chainlinkAggregator = AggregatorV3Interface(chainlinkAggregatorAddr);

        /// @dev Check call Chainlink without error
        chainlinkAggregator.latestRoundData();

        return decimals == chainlinkAggregator.decimals();
    }

```

<https://github.com/Folks-Finance/folks-finance-xchain-contracts/blob/main/contracts/oracle/nodes/ChainlinkNode.sol#L83>

`decimals == chainlinkAggregator.decimals()` ensures that the registered decimals are the same as the corresponding Chainlink Aggregator decimals. However, the issue is that the same cached decimal is used in the calculation of the price every time, which is incorrect. Chainlink can change the aggregator, and if the new aggregator changes the decimal value of the asset, it will fetch an incorrect price.

## Impact Details

Incorrect prices due to the use of cached decimals.

## Recommendation

Ideally, the decimals should be fetched every time latestRoundData() is called:

```solidity
    function process(bytes memory parameters) internal view returns (NodeOutput.Data memory nodeOutput) {
        (address chainlinkAggregatorAddr, uint256 twapTimeInterval, uint8 decimals) = abi.decode(
            parameters,
            (address, uint256, uint8)
        );

        AggregatorV3Interface chainlinkAggregator = AggregatorV3Interface(chainlinkAggregatorAddr);
+        decimals = chainlinkAggregator.decimals(); //@audit-ok // Fetch decimals each time
        (uint80 roundId, int256 answer, , uint256 updatedAt, ) = chainlinkAggregator.latestRoundData();

        /// @dev Calculate the price. If the TWAP time interval is 0, use the latest price. Otherwise, calculate the TWAP price.
        uint256 price = twapTimeInterval == 0
            ? answer.toUint256()
            : getTwapPrice(chainlinkAggregator, roundId, answer.toUint256(), twapTimeInterval);

        /// @dev Adjust the price to 18 d.p.
        price = decimals > PRECISION ? price / (10 ** (decimals - PRECISION)) : price * (10 ** (PRECISION - decimals));

        return NodeOutput.Data(price, updatedAt, NodeDefinition.NodeType.CHAINLINK, 0, 0);
    }
```

<https://github.com/Folks-Finance/folks-finance-xchain-contracts/blob/main/contracts/oracle/nodes/ChainlinkNode.sol#L27>

## Proof of concept

## Proof of Concept

> To accommodate the dynamic nature of offchain environments, Chainlink Data Feeds are updated from time to time to add new features and capabilities as well as respond to externalities such as token migrations, protocol rebrands, extreme market events, and upstream issues with data or node operations. These updates include changes to the aggregator configuration or a complete replacement of the aggregator that the proxy uses. If you consume data feeds through the proxy, your applications can continue to operate during these changes

<https://docs.chain.link/data-feeds#updates-to-proxy-and-aggregator-contracts>
