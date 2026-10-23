# Boost \_ Folks Finance 34188 - \[Smart Contract - Insight] BridgeRouterHub can add address adapter

Submitted on Tue Aug 06 2024 07:36:54 GMT-0400 (Atlantic Standard Time) by @Obin for [Boost | Folks Finance](https://immunefi.com/bounty/folksfinance-boost/)

Report ID: #34188

Report type: Smart Contract

Report severity: Insight

Target: <https://testnet.snowtrace.io/address/0xa9491a1f4f058832e5742b76eE3f1F1fD7bb6837>

Impacts:

* Bridgerouter `MANAGER_ROLE` can "irreversibly" add address(0) Adaapter

## Description

## Brief/Intro

`MANAGER_ROLE` in bridgeRouterHub can add address(0) as Adapter. This is not supposed to be allowed by the contract.

## Impact Details

When this happens, it is impossible to reverse its effect as the opposite function `removeAdapter()` will revert.

* Permanently wasting an adapterId.
* Potential adapterId synchronicities across chains be permanently disrupted.

## Mitigation

```diff
+   error adapterIndexMustNotBeZero();
+   error adapterCantBeZeroAddress();
    function addAdapter(uint16 adapterId, IBridgeAdapter adapter) external onlyRole(MANAGER_ROLE) {
        
+       if(adapterId == uint16(0)) revert adapterIndexMustNotBeZero(); //@audit The index for `adapterId` must start from 1 not 0.
        if(address(adapter) == address(0)) revert adapterCantBeZeroAddress();
        // check if no existing adapter
        if (isAdapterInitialized(adapterId)) revert AdapterInitialized(adapterId); 
        
        // add adapter
        idToAdapter[adapterId] = adapter;
        adapterToId[adapter] = adapterId;
    }
```

## References

Add any relevant links to documentation or code

## Proof of concept

## Proof of Concept

### Changes made to BridgeRouter.sol file for simplicity in illustration using foundry

```diff
diff --git a/Diff.sol b/Diff.sol
index a8ea541..862431b 100644
--- a/Diff.sol
+++ b/Diff.sol
@@ -1,33 +1,28 @@
-// SPDX-License-Identifier: BUSL-1.1
-pragma solidity 0.8.23;
+// SPDX-License-Identifier: UNLICENSED
+pragma solidity ^0.8.13;
+import {AccessControlDefaultAdminRules} from "@openzeppelin/contracts/access/extensions/AccessControlDefaultAdminRules.sol";

-import "@openzeppelin/contracts/access/extensions/AccessControlDefaultAdminRules.sol";
-import "./BridgeMessenger.sol";
-import "./interfaces/IBridgeAdapter.sol";
-import "./interfaces/IBridgeRouter.sol";
 import "./libraries/Messages.sol";

-abstract contract BridgeRouter is IBridgeRouter, AccessControlDefaultAdminRules {
-    bytes32 public constant override MANAGER_ROLE = keccak256("MANAGER");
-    bytes32 public constant override MESSAGE_SENDER_ROLE = keccak256("MESSAGE_SENDER");
-
-    event MessageSucceeded(uint16 adapterId, bytes32 indexed messageId);
-    event MessageFailed(uint16 adapterId, bytes32 indexed messageId, bytes reason);
-    event MessageRetrySucceeded(uint16 adapterId, bytes32 indexed messageId);
-    event MessageRetryFailed(uint16 adapterId, bytes32 indexed messageId, bytes reason);
-    event MessageReverseSucceeded(uint16 adapterId, bytes32 indexed messageId);
-    event MessageReverseFailed(uint16 adapterId, bytes32 indexed messageId, bytes reason);
-    event Withdraw(bytes32 userId, address receiver, uint256 amount);
-
-    error NotEnoughFunds(bytes32 user);
-    error FailedToWithdrawFunds(address recipient, uint256 amount);
-    error ChainUnavailable(uint16 folksChainId);
-    error SenderDoesNotMatch(address messager, address caller);
-    error AdapterInitialized(uint16 adapterId);
-    error AdapterNotInitialized(uint16 adapterId);
-    error AdapterUnknown(IBridgeAdapter adapter);
+// an examplary interface
+interface IBridgeAdapter {
+    function dog() external view returns(bool);
+    function cat() external view returns(bool);
+
+}
+
+contract Counter is AccessControlDefaultAdminRules {
+    bytes32 public constant  MANAGER_ROLE = keccak256("MANAGER");
+    bytes32 public constant  MESSAGE_SENDER_ROLE = keccak256("MESSAGE_SENDER");
+
+    error AdapterInitialized(uint);
+    error AdapterNotInitialized(uint);
     error MessageAlreadySeen(bytes32 messageId);
-    error MessageUnknown(bytes32 messageId);
+    error AdapterUnknown(IBridgeAdapter adapter);
+
+    event successfullyBypassedAllrevertChecksAbove();
+
+    uint256 public number = 13; //testing


```

### Actual POC on above File

```
// SPDX-License-Identifier: UNLICENSED
pragma solidity ^0.8.13;

import {Test, console} from "forge-std/Test.sol";
import { IBridgeAdapter,Counter, Messages} from "../src/Counter.sol";

contract CounterTest is Test {
    Counter public counter;
    address admin = makeAddr("admin");
    //address manager = makeAddr('manager');

    

    function setUp() public {
        //vm.startPrank(admin);
        counter = new Counter(admin);
        // counter.grantRole(counter.MANAGER_ROLE(),manager); // Already set
    }

    
    // adapter Id cannot be removed
    function testImpactOfAddressZeroInputCannotBeRemoved() public {
        vm.startPrank(admin);
        // adds 5 instances of adapter (For instance)
        for(uint160 i =0; i < 5; i++){ //makes 5 address (address(0) as first) 
            
            IBridgeAdapter adapter = IBridgeAdapter(address(i)); //address(0) will be first
            counter.addAdapter(uint16(i),adapter);
            console.log('addapterInstances',address(adapter));
        }
        
        uint16 adapterId = 0; //adapterId of addressZero adapter
        vm.expectRevert(abi.encodeWithSelector(Counter.AdapterNotInitialized.selector, 0)); 
        //attepmpt to reverse this error is impossible.
        counter.removeAdapter(adapterId);

      
    }

    /**
    =========Signatures
    function isAdapterInitialized(uint16 adapterId) public view returns (bool) 
    function getAdapter(uint16 adapterId) public view returns (IBridgeAdapter)
    function addAdapter(uint16 adapterId, IBridgeAdapter adapter) external
     */
}


```
