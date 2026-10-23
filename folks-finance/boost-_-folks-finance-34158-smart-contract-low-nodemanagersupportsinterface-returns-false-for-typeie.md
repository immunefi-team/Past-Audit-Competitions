# Boost \_ Folks Finance 34158 - \[Smart Contract - Low] NodeManagersupportsInterface returns false for

Submitted on Tue Aug 06 2024 03:48:31 GMT-0400 (Atlantic Standard Time) by @Ironside\_Sec for [Boost | Folks Finance](https://immunefi.com/bounty/folksfinance-boost/)

Report ID: #34158

Report type: Smart Contract

Report severity: Low

Target: <https://testnet.snowtrace.io/address/0xA758c321DF6Cd949A8E074B22362a4366DB1b725>

Impacts:

* Contract fails to deliver promised returns, but doesn't lose value

## Description

## Brief/Intro

NodeManager supports only node manager interface but not erc 165.

```diff
NodeManager.sol

 function supportsInterface(bytes4 interfaceId) external pure override returns (bool) {
-       return interfaceId == type(INodeManager).interfaceId ;
+       return interfaceId == type(INodeManager).interfaceId || interfaceId == type(IERC165).interfaceId;
 }

```

## Vulnerability Details

`NodeManager.supportsInterface` returns true if it is queried supports node manager interface, but returns false if queried erc165\_Supported. All the support interface implemented contracts on Ethereum should support erc165 and if any extra interfaces like erc721, node manager available then they should be supported too.

Look at the examples in the GitHub search on every contract implementing `.supportsInterface()`, they all have erc165 allowed and they also implement super.supportsInterface in || way to allow other supported interfaces (ex: erc165 in this case). <https://github.com/search?q=path%3A\\*.sol+.supportsInterface\\&type=code>

<https://github.com/Folks-Finance/folks-finance-xchain-contracts/blob/fb92deccd27359ea4f0cf0bc41394c86448c7abb/contracts/oracle/modules/NodeManager.sol#L52>

```solidity
NodeManager.sol

58:     function supportsInterface(bytes4 interfaceId) external pure override returns (bool) {
59:   >>>   return interfaceId == type(INodeManager).interfaceId ;
60:     
61:     }

```

```bash
Ran 1 test for test/Foundry.t.sol:PoC
[PASes:
 [11352] PoC::testIssue()
 ├─ [324] 0xA758c321DF6Cd949A8E074B22362a4366DB1b725::supportsInterface(0x01ffc9a700000000000000000000000000000000000000000000000000000000) [staticcall]
 │   └─ ← [Return] false
 ├─ [324] 0xA758c321DF6Cd949A8E074B22362a4366DB1b725::supportsInterface(0xc06e446700000000000000000000000000000000000000000000000000000000) [staticcall]
 │   └─ ← [Return] true
 ├─ [0] VM::assertFalse(false) [staticcall]
 │   └─ ← [Return] 
 ├─ [0] VM::assertTrue(true) [staticcall]
 │   └─ ← [Return] 
 ├─ [0] console::log("erc165_Supported: ", false) [staticcall]
 │   └─ ← [Stop] 
 ├─ [0] console::log("nodeManager_Supported: ", true) [staticcall]
 │   └─ ← [Stop] 
 └─ ← [Stop] 
```

## Impact Details

No loss of funds, but anyone calling support interface might first call if it is erc165 compatabile, and it returns false in this case, so false might seem they will no further call if it supports node manager interface. It should return true for both erc165 and node manager interfaces

## References

<https://github.com/Folks-Finance/folks-finance-xchain-contracts/blob/fb92deccd27359ea4f0cf0bc41394c86448c7abb/contracts/oracle/modules/NodeManager.sol#L52>

<https://github.com/search?q=path%3A\\*.sol+.supportsInterface\\&type=code>

## Proof of concept

## Proof of Concept

for POC to work,

1. on `https://github.com/Folks-Finance/folks-finance-xchain-contracts` directory, do `forge i foundry-rs/forge-std --no-commit`,
2. then add `ds-test/=node_modules/ds-test/` to `remappings.txt`,
3. then create a file `Foundry.t.sol` on test/ dirctory.
4. Then run the poc with `forge t --mt testIssue -f https://rpc.ankr.com/avalanche_fuji -vvvv`

```solidity
// SPDX-License-Identifier: UNLICENSED
pragma solidity ^0.8.13;

import "forge-std/Test.sol";

import "../contracts/oracle/modules/NodeManager.sol";
import "../contracts/oracle/interfaces/INodeManager.sol";

contract PoC is Test {
    address from = 0xD5fba05dE4b2d303D03052e8aFbF31a767Bd908e;
    bytes32 accountId = 0xd32cc9b5264dc39d42622492da52b2b8100e6444367e20c9693ce28fe71286be;
    
    NodeManager constant nodeManager = NodeManager(0xA758c321DF6Cd949A8E074B22362a4366DB1b725);

    function setUp() public {}

    function testIssue() public {

        bool erc165_Supported =  nodeManager.supportsInterface(type(IERC165).interfaceId);
        bool nodeManager_Supported = nodeManager.supportsInterface(type(INodeManager).interfaceId);

        assertFalse(erc165_Supported);
        assertTrue(nodeManager_Supported);

        console.log('erc165_Supported: ', erc165_Supported);
        console.log('nodeManager_Supported: ', nodeManager_Supported);
        
    }  
}
```
