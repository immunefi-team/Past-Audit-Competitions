# Boost \_ Folks Finance 33713 - \[Smart Contract - Insight] Some transactions can revert when nodetype

Submitted on Sat Jul 27 2024 05:40:53 GMT-0400 (Atlantic Standard Time) by @arno for [Boost | Folks Finance](https://immunefi.com/bounty/folksfinance-boost/)

Report ID: #33713

Report type: Smart Contract

Report severity: Insight

Target: <https://testnet.snowtrace.io/address/0xA758c321DF6Cd949A8E074B22362a4366DB1b725>

Impacts:

* Protocol insolvency

## Description

## Brief/Intro

The `PriceDeviationSameOracleCircuitBreakerNode::process()` method will revert for the price feeds when `parentNodeOutputs[0].price` is 0. This is because this price is then used to calculate the percentage difference, which involves dividing the difference by `parentNodeOutputs[0].price`. This will cause some transactions to revert while others do not, making it harder to detect the point of failure. Crucial transactions, such as repayment to avoid liquidation, will revert, whereas transactions to liquidate may not revert, thus causing harm to the user.

## Vulnerability Details

The function `process` is designed to check the price deviation between two parent nodes and ensure it is within a specified tolerance. If the deviation exceeds the tolerance and the node types of the parent nodes are the same, the function reverts. Additionally, if the deviation is within tolerance, it returns the first parent node output, or if the deviation exceeds the tolerance and a fallback node output is provided, it returns the fallback output.

However, the function does not account for the scenario where `parentNodeOutputs[0].price` is 0. This omission leads to a critical vulnerability.

### Code Snippet

The relevant part of the code is as follows:

```solidity
uint256 price = parentNodeOutputs[0].price;
uint256 comparisonPrice = parentNodeOutputs[1].price;

uint256 difference = price > comparisonPrice ? price - comparisonPrice : comparisonPrice - price;
uint256 percentageDifference = difference.mulDiv(WAD, price);
```

In the above code:

1. `price` is assigned the value of `parentNodeOutputs[0].price`.
2. `comparisonPrice` is assigned the value of `parentNodeOutputs[1].price`.
3. The difference between `price` and `comparisonPrice` is calculated.
4. The percentage difference is then calculated using `difference.mulDiv(WAD, price)`.

The calculation of `percentageDifference` involves a division by `price`. If `price` is 0, this operation will revert due to division by zero, leading to the transaction being reverted instead of falling back to third node.

### Additional Point

If `parentNodeOutputs[0].price` is 0, this effectively means that the deviation tolerance value is exceeded. In such cases, the function should fall back to the third node output, as the current logic intends to do when the deviation tolerance is exceeded.

The relevant part of the code that handles the fallback is as follows:

```solidity
if (percentageDifference > deviationTolerance) {
    if (parentNodeOutputs.length == 2) revert DeviationToleranceExceeded(percentageDifference);
    return parentNodeOutputs[2];
}
```

However, due to the division by zero, this fallback logic is not reached.

## Impact Details

The impact of this vulnerability is significant:

* **Repayment Transactions:** Transactions intended to repay loans and avoid liquidation can revert if `parentNodeOutputs[0].price` is 0. This can result in users unintentionally losing their collateral.
* **Liquidation Transactions:** Transactions intended to liquidate positions may revert under the same conditions, allowing liquidations not to proceed even when they should.
* **Borrow Transactions:** Borrow transactions will also be affected because the utilization ratio will increase for some borrowers whose transactions to borrow reverted, while for others it did not. This discrepancy can cause higher interest rates for the first borrower. This can make borrowing more expensive and less predictable for users.
* **Pool Protection:** Many pools may use this node type to protect themselves from high market fluctuations between different oracles. Admins changing the node for a particular pool will not solve the issue since most trusted oracles being used, such as Chainlink and Pyth, are prone to deviations from the actual market price. Relying on a single oracle as the source of the price feed increases the risk of manipulation.

## References

<https://github.com/Folks-Finance/folks-finance-xchain-contracts/blob/fb92deccd27359ea4f0cf0bc41394c86448c7abb/contracts/oracle/nodes/PriceDeviationSameOracleCircuitBreakerNode.sol#L29>

## Proof of concept

## Proof of Concept

```
// SPDX-License-Identifier: MIT
pragma solidity 0.8.23;

import "forge-std/Test.sol";
import "../PriceDeviationSameOracleCircuitBreakerNode.sol";
import "../storage/NodeDefinition.sol";
import "../storage/NodeOutput.sol";
import "@openzeppelin/contracts/utils/math/Math.sol";

contract PriceDeviationSameOracleCircuitBreakerNodeTest is Test {
    using Math for uint256;

    NodeOutput.Data[] parentNodeOutputs;
    bytes parameters;

    function setUp() public {
        // Set up parentNodeOutputs with zero price
        NodeOutput.Data memory parent1 = NodeOutput.Data({
            price: 0,
            timestamp: block.timestamp,
            nodeType: NodeDefinition.NodeType.EXTERNAL,
            additionalParam1: 0,
            additionalParam2: 0
        });
        NodeOutput.Data memory parent2 = NodeOutput.Data({
            price: 100 * 1e18, 
            timestamp: block.timestamp,
            nodeType: NodeDefinition.NodeType.PYTH,
            additionalParam1: 0,
            additionalParam2: 0
        });
        NodeOutput.Data memory fallback = NodeOutput.Data({
            price: 50 * 1e18, 
            timestamp: block.timestamp,
            nodeType: NodeDefinition.NodeType.CHAINLINK,
            additionalParam1: 0,
            additionalParam2: 0
        });

        parentNodeOutputs.push(parent1);
        parentNodeOutputs.push(parent2);
        parentNodeOutputs.push(fallback);

        // Set the deviation tolerance
        uint256 deviationTolerance = 5 * 1e17; // 50% tolerance
        parameters = abi.encode(deviationTolerance);
    }

    function testProcessWithZeroPrice() public {
        vm.expectRevert();
        NodeOutput.Data memory result = PriceDeviationSameOracleCircuitBreakerNode.process(
            parentNodeOutputs,
            parameters
        );
        
    }
}

```
