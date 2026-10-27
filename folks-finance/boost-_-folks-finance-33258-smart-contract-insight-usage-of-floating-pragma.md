# Boost \_ Folks Finance 33258 - \[Smart Contract - Insight] Usage of floating pragma

Submitted on Tue Jul 16 2024 10:36:19 GMT-0400 (Atlantic Standard Time) by @chista0x for [Boost | Folks Finance](https://immunefi.com/bounty/folksfinance-boost/)

Report ID: #33258

Report type: Smart Contract

Report severity: Insight

Target: <https://github.com/Folks-Finance/folks-finance-xchain-contracts/blob/main/contracts/oracle/storage/NodeDefinition.sol>

Impacts:

* Primacy of Impact

## Description

## Brief/Intro

The problem lies in the inconsistent usage of Solidity compiler versions in the project’s contracts and libraries. While the main contracts are compiled with Solidity version 0.8.23, two libraries use a floating version pragma (>=0.8.11 <0.9.0). This inconsistency could lead to unexpected behavior and potential security vulnerabilities if the floating versions compile to a version with known issues.

## Vulnerability Details

In the project, the main contracts use a fixed version pragma:

```solidity
pragma solidity 0.8.23;
```

However, two libraries use a floating version pragma:

```solidity
pragma solidity >=0.8.11 <0.9.0;
```

This discrepancy can lead to compilation with different versions of the Solidity compiler, which might introduce subtle bugs or security issues. For example, using a version above 0.8.23 might introduce changes not supported by all EVM-compatible chains, potentially causing runtime errors or unexpected behavior. Given that the rest of the libraries adhere to the fixed version 0.8.23, it’s likely that the floating pragma in these two libraries was unintended.

## Impact Details

The primary risk is that using different compiler versions could introduce incompatibilities and potential security vulnerabilities. If a floating version resolves to a newer compiler that includes breaking changes or deprecated features, it could result in runtime errors or unexpected behavior on EVM-compatible chains that do not support those changes. This inconsistency may lead to loss of funds, failed transactions, or exploit opportunities for attackers targeting these discrepancies.

The reason I did submit this low issue is because all contracts and libraries besides two had a fixed pragma. So I thought that it was an oversight on your part, since if it was a coding style choice there would have been consistency. Anyways it is true that the impact is non existent and is considered a best practice, but in most audit contest it is still a valid low.

## References

<https://github.com/Folks-Finance/folks-finance-xchain-contracts/blob/fb92deccd27359ea4f0cf0bc41394c86448c7abb/contracts/oracle/storage/NodeDefinition.sol#L2>

<https://github.com/Folks-Finance/folks-finance-xchain-contracts/blob/fb92deccd27359ea4f0cf0bc41394c86448c7abb/contracts/oracle/storage/NodeOutput.sol#L2>

## Proof of concept

## Proof of Concept

If we have this codes:

NodeDefinition.sol

```solidity
// SPDX-License-Identifier: MIT
pragma solidity >=0.8.11 <0.9.0;

library NodeDefinition {
    enum NodeType {
        NONE,
        CHAINLINK,
        PYTH,
        PRICE_DEVIATION_CIRCUIT_BREAKER,
        PRICE_DEVIATION_SAME_ORACLE_CIRCUIT_BREAKER,
        STALENESS_CIRCUIT_BREAKER,
        SAME_ORACLE_BREAKER,
        CONSTANT,
        REDUCER,
        EXTERNAL
    }

    struct Data {
        /**
         * @dev Oracle node type enum
         */
        NodeType nodeType;
        /**
         * @dev Node parameters, specific to each node type
         */
        bytes parameters;
        /**
         * @dev Parent node IDs, if any
         */
        bytes32[] parents;
    }

    /**
     * @dev Returns the node stored at the specified node ID.
     */
    function load(bytes32 id) internal pure returns (Data storage node) {
        bytes32 s = keccak256(abi.encode("folks.finance.xlending.oracle.Node", id));
        assembly {
            node.slot := s
        }
    }

    /**
     * @dev Register a new node for a given node definition. The resulting node is a function of the definition.
     */
    function create(Data memory nodeDefinition) internal returns (NodeDefinition.Data storage node, bytes32 id) {
        id = getId(nodeDefinition);

        node = load(id);

        node.nodeType = nodeDefinition.nodeType;
        node.parameters = nodeDefinition.parameters;
        node.parents = nodeDefinition.parents;
    }

    /**
     * @dev Returns a node ID based on its definition
     */
    function getId(Data memory nodeDefinition) internal pure returns (bytes32 id) {
        return keccak256(abi.encode(nodeDefinition.nodeType, nodeDefinition.parameters, nodeDefinition.parents));
    }
}
```

OpCodeZero.sol

```solidity
// SPDX-License-Identifier: MIT
pragma solidity 0.8.23;

import {NodeDefinition} from "contracts/NodeDefinition.sol";

contract OpCodeZero {}
```

the bytecode of `OpCodeZero.sol` is:

```
6080604052348015600e575f80fd5b50603e80601a5f395ff3fe60806040525f80fdfea2646970667358221220c2f878d366d5b43354f0a9e0afb38847e3b70a5bfb838c4b834db6478ed2952064736f6c63430008170033
```

if you test this bytecode in `https://www.evm.codes/playground` with `cancun` the code run without any error. but if you run with `paris` evm version you will sea the error. because we have `INVALID` opcodes.
