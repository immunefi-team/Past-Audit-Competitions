# #38012 \[SC-Insight] Unused Function in CLAdapter Contract

**Submitted on Dec 21st 2024 at 17:11:08 UTC by @IlIlHunterlIlI for** [**Audit Comp | Lombard**](https://immunefi.com/audit-competition/audit-comp-lombard)

* **Report ID:** #38012
* **Report Type:** Smart Contract
* **Report severity:** Insight
* **Target:** <https://github.com/lombard-finance/evm-smart-contracts/blob/main/contracts/bridge/adapters/CLAdapter.sol>
* **Impacts:**
  * Contract fails to deliver promised returns, but doesn't lose value

## Description

## Brief/Intro

The `initWithdrawalNoSignatures` function in CLAdapter.sol has the `onlyTokenPool` modifier but is never called from the `TokenPool` contract or any other location in the codebase. This creates dead code that increases contract size and gas costs unnecessarily while potentially creating confusion about the security model around signature-less withdrawals.

## Vulnerability Details

The function is defined in CLAdapter.sol:

```solidity
function initWithdrawalNoSignatures(
    uint64 remoteSelector,
    bytes calldata onChainData
) external onlyTokenPool returns (uint64) {
    _receive(getChain[remoteSelector], onChainData);
    return bridge.withdraw(onChainData);
}
```

Key observations:

1. The function has `onlyTokenPool` modifier meaning only the TokenPool contract can call it
2. Examining TokenPool.sol shows no calls to this function
3. The function allows withdrawals without signature verification unlike `initiateWithdrawal`

## Impact Details

While this is not an exploitable vulnerability since the function is protected by `onlyTokenPool`, it has several negative impacts:

1. Increased deployment costs due to unnecessary bytecode
2. Potential future security risks if the function is accidentally used instead of the signature-verified version

## References

* CLAdapter.sol: Line 192-198
* TokenPool.sol implementation

## Proof of Concept

## Proof of Concept

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

import {Test} from "forge-std/Test.sol";
import {CLAdapter} from "../contracts/bridge/adapters/CLAdapter.sol";
import {LombardTokenPool} from "../contracts/bridge/adapters/TokenPool.sol";
import {IBridge} from "../contracts/bridge/IBridge.sol";
import {IERC20} from "@openzeppelin/contracts/token/ERC20/IERC20.sol";

contract CLAdapterTest is Test {
    CLAdapter public adapter;
    LombardTokenPool public tokenPool;
    address public constant MOCK_ROUTER = address(0x1);
    address public constant MOCK_BRIDGE = address(0x2);
    address public constant MOCK_LBTC = address(0x3);
    address public constant MOCK_RMN = address(0x4);
    uint128 public constant EXECUTION_GAS_LIMIT = 200000;

    function setUp() public {
        // Deploy mock contracts
        vm.mockCall(
            MOCK_BRIDGE,
            abi.encodeWithSelector(IBridge.lbtc.selector),
            abi.encode(MOCK_LBTC)
        );

        // Deploy CLAdapter
        adapter = new CLAdapter(
            IBridge(MOCK_BRIDGE),
            EXECUTION_GAS_LIMIT,
            MOCK_ROUTER,
            new address[](0),
            MOCK_RMN
        );

        // Get TokenPool reference
        tokenPool = adapter.tokenPool();
    }

    function testNoCallsToInitWithdrawalNoSignatures() public {
        // Get TokenPool bytecode
        bytes memory bytecode = address(tokenPool).code;
        
        // Search for initWithdrawalNoSignatures selector in bytecode
        bytes4 selector = adapter.initWithdrawalNoSignatures.selector;
        
        bool selectorFound = false;
        for(uint i = 0; i < bytecode.length - 4; i++) {
            bytes4 candidate;
            assembly {
                candidate := mload(add(add(bytecode, 0x20), i))
            }
            if(candidate == selector) {
                selectorFound = true;
                break;
            }
        }
        
        assertFalse(selectorFound, "initWithdrawalNoSignatures selector found in TokenPool bytecode");
    }

    function testDirectCallReverts() public {
        vm.expectRevert("CLUnauthorizedTokenPool");
        adapter.initWithdrawalNoSignatures(1, "0x");
    }
}
```

my foundry.toml

```
[profile.default]
src = 'contracts'
out = 'out'
libs = ['node_modules', 'lib']
test = 'test'
remappings = [
    'ds-test/=lib/forge-std/lib/ds-test/src/',
    'forge-std/=lib/forge-std/src/'
]
```

and this is how i got the environment to be in foundry

```
npm install --save-dev @nomicfoundation/hardhat-foundry
import "@nomicfoundation/hardhat-foundry";
npx hardhat init-foundry
```
