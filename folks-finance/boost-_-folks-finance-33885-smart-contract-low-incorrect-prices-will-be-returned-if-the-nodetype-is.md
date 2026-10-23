# Boost \_ Folks Finance 33885 - \[Smart Contract - Low] Incorrect prices will be returned if the NodeTy

Submitted on Thu Aug 01 2024 01:57:42 GMT-0400 (Atlantic Standard Time) by @OxAnmol for [Boost | Folks Finance](https://immunefi.com/bounty/folksfinance-boost/)

Report ID: #33885

Report type: Smart Contract

Report severity: Low

Target: <https://testnet.snowtrace.io/address/0x46c425F4Ec43b25B6222bcc05De051e6D3845165>

Impacts:

* Griefing (e.g. no profit motive for an attacker, but damage to the users or the protocol)

## Description

## Brief/Intro

`PRICE_DEVIATION_CIRCUIT_BREAKER` should validate prices from `PriceDeviationCircuitBreakerNode`. However, due to an incorrect function call in if statement, it checks the prices using the same oracle circuit breaker and returns them.

## Vulnerability Details

The `NodeManager:_process` function directs the request to the nodeId specified by the caller, which in this case will be from the folks smart contract pool. In the if-else statement, it checks the provided node definition and returns prices accordingly after validation. However,

`if(nodeDefinition.nodeType == NodeDefinition.NodeType.PRICE_DEVIATION_CIRCUIT_BREAKER)` actually calls `PriceDeviationSameOracleCircuitBreakerNode` instead of `PriceDeviationCircuitBreaker`.

```jsx
if (nodeDefinition.nodeType == NodeDefinition.NodeType.PRICE_DEVIATION_CIRCUIT_BREAKER) {
    return PriceDeviationSameOracleCircuitBreakerNode.process(
        _processParentsNode(nodeDefinition),
        nodeDefinition.parameters
    );
}

```

This may cause insufficient price validation and result in incorrect prices. This can further lead to borrowing and liquidation at wrong prices, causing some positions to be incorrectly liquidated and borrowing to occur at low incorrect prices, leading to **`Griefing`**.

## Impact Details

Borrowing and liquidation can happen at the wrong price and lead to `Griefing (e.g. no profit motive for an attacker, but damage to the users or the protocol)`, because of that this issue should qualify for medium severity.

## References

<https://github.com/Folks-Finance/folks-finance-xchain-contracts/blob/fb92deccd27359ea4f0cf0bc41394c86448c7abb/contracts/oracle/modules/NodeManager.sol#L177C1-L180C1>

## Proof of concept

Here is a simple POC that shows how prices from `PriceDeviationSameOracleCircuitBreakerNode` are returned when we provide `PRICE_DEVIATION_CIRCUIT_BREAKER` node type which has a different primary price.

paste this in `nodeManagerUtils.ts`

```jsx

  public static encodePriceDeviationCircuitBreakerNodeDefinition(
    deviation: number,
    parentIds: string[]
  ): NodeDefinitionData {
    return [NodeType.PRICE_DEVIATION_CIRCUIT_BREAKER, abi.encode(["uint256"], [deviation.toString()]), parentIds];
  }
```

paste this in `PriceDeviationSameOracleCircuitBreakerNode.test.ts`

```jsx
it.only("Should process same oracle deviation only", async function () {
        deviationTolerance = 1e18;
        parentNodeIds = [external22NodeId, constant42NodeId, constant69NodeId];
        const encodedParams = NodeManagerUtil.encodePriceDeviationSameOracleCircuitBreakerNodeDefinition(
          deviationTolerance,
          parentNodeIds
        );
        nodeId = await NodeManagerUtil.registerNode(nodeManager, encodedParams);
        const encodedParamsDifferentOracle = NodeManagerUtil.encodePriceDeviationCircuitBreakerNodeDefinition(
          deviationTolerance,
          parentNodeIds
        );

        const constant68NodeId = await NodeManagerUtil.registerNode(
          nodeManager,
          NodeManagerUtil.encodeConstantNodeDefinition(68e18)
        );
        parentNodeIds = [constant68NodeId, external22NodeId, constant42NodeId];
        const nodeIdCircuitBreaker = await NodeManagerUtil.registerNode(nodeManager, encodedParamsDifferentOracle);

        const nodeOutput = await nodeManager.process(nodeIdCircuitBreaker);

        expect(nodeOutput.price).to.equal(externalNodePrice);
      });

```
