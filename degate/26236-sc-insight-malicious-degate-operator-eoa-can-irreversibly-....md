# 26236 - \[SC - Insight] Malicious DeGate Operator EOA can irreversibly ...

Submitted on Nov 29th 2023 at 00:53:43 UTC by @ongrid for [Boost | DeGate](https://immunefi.com/bounty/boosteddegatebugbounty/)

Report ID: #26236

Report type: Smart Contract

Report severity: Insight

Target: <https://etherscan.io/address/0x9C07A72177c5A05410cA338823e790876E79D73B#code>

Impacts:

* Irreversible stop of Exchange
* Prevent new users from registering (Zero Knowledge Proof Circuit)
* Permanent freezing of unclaimed rewards
* Griefing (e.g. no profit motive for an attacker, but damage to the users or the protocol)
* Contract fails to deliver promised returns, but doesn't lose value
* Force DeGate into Exodus Mode
* The account cannot be used (Zero Knowledge Proof Circuit)
* Temporary freezing of funds: Minimum freezing of 15 days (Zero Knowledge Proof Circuit)

## Description

## Bug Description

The current ExchangeProxy at `0x9C07A72177c5A05410cA338823e790876E79D73B` is managed by `LoopringIOExchangeOwner` `0x9b93e47b7F61ad1358Bd47Cd01206708E85AE5eD, which is in turn controlled by an Externally Owned Account (EOA)` 0xacD3A62F3eED1BfE4fF0eC8240d645c1F5477F82\`. This setup poses significant security risks due to centralized control, allowing the key holder to unilaterally modify critical contract parameters and potentially shut down the system irreversibly.

## Impact

### Irreversible System Shutdown

The DeGate Operator, using an EOA, can entirely shut down the system. This can be executed by invoking `Exchange.shutdown()` through `LoopringIOExchangeOwner.transact(abiEncodedCall),` as demonstrated in the `testExchangeShutdown` test case.

### Unilateral Modification of Critical Platform Configuration Parameters

In the `testSetMaxAgeDepositUntilWithdrawable` and `testExchangeSetDepositParams` tests, it's shown how the operator can change key Exchange settings via the same call chain initiated from their EOA.

## Risk Breakdown

Difficulty to Exploit: Easy

## Recommendation

To mitigate centralized control risks, Exchange's ownership (not just the proxy facade but also the business logic) should be restructured:

* **Transfer Ownership to Multisig**: Change the ownership of LoopringIOExchangeOwner to a Gnosis Multisig wallet, distributing control among multiple parties.
* **Consider Using Timelock**: Use Timelock contract as the owner to introduce a delay in critical operations, enhancing transparency.

## Proof of concept

I have created a forge test, functioning on an Ethereum mainnet fork using the addreses of the live contracts. The file demonstrating vulnerabilities is `MaliciousOperatorStopsExchangeContract.t.sol`

## Code (git)

```git
commit 625637a9e64fa298b3a80b3216382c3382656f97
Author: Kirill Varlamov <kirill@ongrid.pro>
Date:   Tue Nov 28 03:12:40 2023 +0400

    test: malicious DeGate operator stops Exchange (centralized control by EOA)

diff --git a/foundry.toml b/foundry.toml
index 25b918f9..aa662d8a 100644
--- a/foundry.toml
+++ b/foundry.toml
@@ -3,4 +3,5 @@ src = "src"
 out = "out"
 libs = ["lib"]
 
+solc = '0.8.23'
 # See more config options https://github.com/foundry-rs/foundry/blob/master/crates/config/README.md#all-options
diff --git a/src/Counter.sol b/src/Counter.sol
deleted file mode 100644
index aded7997..00000000
--- a/src/Counter.sol
+++ /dev/null
@@ -1,14 +0,0 @@
-// SPDX-License-Identifier: UNLICENSED
-pragma solidity ^0.8.13;
-
-contract Counter {
-    uint256 public number;
-
-    function setNumber(uint256 newNumber) public {
-        number = newNumber;
-    }
-
-    function increment() public {
-        number++;
-    }
-}
diff --git a/test/Counter.t.sol b/test/Counter.t.sol
deleted file mode 100644
index e9b9e6ac..00000000
--- a/test/Counter.t.sol
+++ /dev/null
@@ -1,24 +0,0 @@
-// SPDX-License-Identifier: UNLICENSED
-pragma solidity ^0.8.13;
-
-import {Test, console2} from "forge-std/Test.sol";
-import {Counter} from "../src/Counter.sol";
-
-contract CounterTest is Test {
-    Counter public counter;
-
-    function setUp() public {
-        counter = new Counter();
-        counter.setNumber(0);
-    }
-
-    function test_Increment() public {
-        counter.increment();
-        assertEq(counter.number(), 1);
-    }
-
-    function testFuzz_SetNumber(uint256 x) public {
-        counter.setNumber(x);
-        assertEq(counter.number(), x);
-    }
-}
diff --git a/test/MaliciousOperatorStopsExchangeContract.t.sol b/test/MaliciousOperatorStopsExchangeContract.t.sol
new file mode 100644
index 00000000..52685b34
--- /dev/null
+++ b/test/MaliciousOperatorStopsExchangeContract.t.sol
@@ -0,0 +1,65 @@
+// SPDX-License-Identifier: MIT
+pragma solidity ^0.8.10;
+
+import "forge-std/Test.sol";
+import "test/mock/IExchangeV3.sol";
+import "test/mock/ILoopringIOExchangeOwner.sol";
+
+contract ShutdownExchangeTest is Test {
+    event DepositParamsUpdate(
+        uint256 freeDepositMax, uint256 freeDepositRemained, uint256 freeSlotPerBlock, uint256 depositFee
+    );
+
+    IExchangeV3 exchange;
+    address exchangeOwner;
+    ILoopringIOExchangeOwner ioex;
+    address ioexOwner;
+
+    function setUp() public {
+        uint256 fork;
+        fork = vm.createFork("https://mainnet.infura.io/v3/bcb7b03333384d49957cc9b4b53daa1d");
+        vm.selectFork(fork);
+        exchange = IExchangeV3(0x9C07A72177c5A05410cA338823e790876E79D73B);
+        ioex = ILoopringIOExchangeOwner(0x9b93e47b7F61ad1358Bd47Cd01206708E85AE5eD);
+        ioexOwner = ioex.owner();
+    }
+
+    function testLoopringIOExchangeOwnerIsEOA() public {
+        // ioexOwner is EOA (i.e. centralized control)
+        assertFalse(isContract(ioexOwner));
+    }
+
+    function testExchangeShutdown() public {
+        // Exchange is operational before malicious admin shuts it down
+        assertFalse(exchange.isShutdown());
+        bytes memory data = abi.encodeWithSignature("shutdown()");
+        vm.prank(ioexOwner);
+        ioex.transact(data);
+        // Exchange stopped its operation as result
+        assertTrue(exchange.isShutdown());
+    }
+
+    function testSetMaxAgeDepositUntilWithdrawable() public {
+        assertEq(exchange.getMaxAgeDepositUntilWithdrawable(), 1296000);
+        bytes memory data = abi.encodeWithSignature("setMaxAgeDepositUntilWithdrawable(uint32)", 1);
+        vm.prank(ioexOwner);
+        ioex.transact(data);
+        assertEq(exchange.getMaxAgeDepositUntilWithdrawable(), 1);
+    }
+
+    function testExchangeSetDepositParams() public {
+        bytes memory data = abi.encodeWithSignature("setDepositParams(uint256,uint256,uint256,uint256)", 0, 0, 0, 0);
+        vm.expectEmit(true, true, true, true);
+        emit DepositParamsUpdate(0, 0, 0, 0);
+        vm.prank(ioexOwner);
+        ioex.transact(data);
+    }
+
+    function isContract(address _addr) public view returns (bool) {
+        uint32 size;
+        assembly {
+            size := extcodesize(_addr)
+        }
+        return (size > 0);
+    }
+}
diff --git a/test/mock/IExchangeV3.sol b/test/mock/IExchangeV3.sol
new file mode 100644
index 00000000..bdb9e1f8
--- /dev/null
+++ b/test/mock/IExchangeV3.sol
@@ -0,0 +1,121 @@
+// SPDX-License-Identifier: Apache-2.0
+pragma solidity ^0.8.0;
+
+import "./IOwnable.sol";
+
+interface IExchangeV3 is IOwnable {
+    // Event declarations
+    event DepositContractUpdate(address indexed depositContract);
+    event WithdrawExchangeFees(address indexed token, address indexed recipient);
+    event DepositParamsUpdate(
+        uint256 freeDepositMax, uint256 freeDepositRemained, uint256 freeSlotPerBlock, uint256 depositFee
+    );
+    event WithdrawalModeActivated(uint256 timestamp);
+    event WithdrawalRecipientUpdate(
+        address from, address to, address token, uint248 amount, uint32 storageID, address newRecipient
+    );
+    event TransactionApproved(address from, bytes32 transactionHash);
+    event TransactionsApproved(address[] owners, bytes32[] transactionHashes);
+    event Shutdown(uint256 timestamp);
+    event AllowOnchainTransferFrom(bool value);
+
+    struct Constants {
+        uint256 SNARK_SCALAR_FIELD;
+        uint256 MAX_OPEN_FORCED_REQUESTS;
+        uint256 MAX_AGE_FORCED_REQUEST_UNTIL_WITHDRAW_MODE;
+        uint256 TIMESTAMP_HALF_WINDOW_SIZE_IN_SECONDS;
+        uint256 MAX_NUM_ACCOUNTS;
+        uint256 MAX_NUM_TOKENS;
+        uint256 MIN_AGE_PROTOCOL_FEES_UNTIL_UPDATED;
+        uint256 MIN_TIME_IN_SHUTDOWN;
+        uint256 TX_DATA_AVAILABILITY_SIZE;
+        uint256 MAX_AGE_DEPOSIT_UNTIL_WITHDRAWABLE_UPPERBOUND;
+        uint256 MAX_FORCED_WITHDRAWAL_FEE;
+        uint256 DEFAULT_PROTOCOL_FEE_BIPS;
+    }
+
+    struct BlockInfo {
+        // The time the block was submitted on-chain.
+        uint32 timestamp;
+        // The public data hash of the block (the 28 most significant bytes).
+        bytes28 blockDataHash;
+    }
+
+    // Function declarations
+    function version() external pure returns (string memory);
+    function domainSeparator() external view returns (bytes32);
+    function initialize(address _loopring, address _owner, bytes32 _genesisMerkleRoot, bytes32 _genesisMerkleAssetRoot)
+        external;
+    function setDepositContract(address _depositContract) external;
+    function getDepositContract() external view returns (address);
+    function withdrawExchangeFees(address token, address recipient) external;
+    function setDepositParams(
+        uint256 freeDepositMax,
+        uint256 freeDepositRemained,
+        uint256 freeSlotPerBlock,
+        uint256 depositFee
+    ) external;
+    function isUserOrAgent(address from) external view returns (bool);
+    function getConstants() external pure returns (Constants memory);
+    function isInWithdrawalMode() external view returns (bool);
+    function isShutdown() external view returns (bool);
+    function registerToken(address tokenAddress) external returns (uint32);
+    function getTokenID(address tokenAddress) external view returns (uint32);
+    function getTokenAddress(uint32 tokenID) external view returns (address);
+    function getExchangeStake() external view returns (uint256);
+    function withdrawExchangeStake(address recipient) external returns (uint256);
+    function getProtocolFeeLastWithdrawnTime(address tokenAddress) external view returns (uint256);
+    function burnExchangeStake() external;
+    function getMerkleRoot() external view returns (bytes32);
+    function getMerkleAssetRoot() external view returns (bytes32);
+    function getBlockHeight() external view returns (uint256);
+    function getBlockInfo(uint256 blockIdx) external view returns (BlockInfo memory);
+    // function submitBlocks(ExchangeData.Block[] calldata blocks) external;
+    function getNumAvailableForcedSlots() external view returns (uint256);
+    function deposit(address from, address to, address tokenAddress, uint248 amount, bytes calldata extraData)
+        external
+        payable;
+    function getPendingDepositAmount(address from, address tokenAddress) external view returns (uint248);
+    function forceWithdraw(address from, address token, uint32 accountID) external payable;
+    function isForcedWithdrawalPending(uint32 accountID, address token) external view returns (bool);
+    // function withdrawFromMerkleTree(ExchangeData.MerkleProof calldata merkleProof) external;
+    function isWithdrawnInWithdrawalMode(uint32 accountID, address token) external view returns (bool);
+    function withdrawFromDepositRequest(address from, address token) external;
+    function withdrawFromApprovedWithdrawals(address[] calldata owners, address[] calldata tokens) external;
+    function getAmountWithdrawable(address from, address token) external view returns (uint256);
+    function notifyForcedRequestTooOld(uint32 accountID, address token) external;
+    function setWithdrawalRecipient(
+        address from,
+        address to,
+        address token,
+        uint248 amount,
+        uint32 storageID,
+        address newRecipient
+    ) external;
+    function getWithdrawalRecipient(address from, address to, address token, uint248 amount, uint32 storageID)
+        external
+        view
+        returns (address);
+    function onchainTransferFrom(address from, address to, address token, uint256 amount) external;
+    function approveTransaction(address from, bytes32 transactionHash) external;
+    function approveTransactions(address[] calldata owners, bytes32[] calldata transactionHashes) external;
+    function isTransactionApproved(address from, bytes32 transactionHash) external view returns (bool);
+    function getDomainSeparator() external view returns (bytes32);
+    function setMaxAgeDepositUntilWithdrawable(uint32 newValue) external returns (uint32);
+    function getMaxAgeDepositUntilWithdrawable() external view returns (uint32);
+    function shutdown() external returns (bool);
+    function getProtocolFeeValues()
+        external
+        view
+        returns (
+            uint32 syncedAt,
+            uint16 protocolFeeBips,
+            uint16 previousProtocolFeeBips,
+            uint32 executeTimeOfNextProtocolFeeBips,
+            uint16 nextProtocolFeeBips
+        );
+    function setAllowOnchainTransferFrom(bool value) external;
+    function getUnconfirmedBalance(address token) external view returns (uint256);
+    function getFreeDepositRemained() external view returns (uint256);
+    function getDepositBalance(address token) external view returns (uint248);
+}
diff --git a/test/mock/ILoopringIOExchangeOwner.sol b/test/mock/ILoopringIOExchangeOwner.sol
new file mode 100644
index 00000000..27f8fa63
--- /dev/null
+++ b/test/mock/ILoopringIOExchangeOwner.sol
@@ -0,0 +1,19 @@
+pragma solidity ^0.8.0;
+
+import "./IOwnable.sol";
+
+interface ILoopringIOExchangeOwner is IOwnable {
+    event SubmitBlocksAccessOpened(bool open);
+    event PermissionUpdate(address indexed user, bytes4 indexed selector, bool allowed);
+    event TargetCalled(address target, bytes data);
+    event Drained(address to, address token, uint256 amount);
+
+    function openAccessToSubmitBlocks(bool _open) external;
+    function submitBlocks(bool isDataCompressed, bytes calldata data) external;
+    function grantAccess(address user, bytes4 selector, bool granted) external;
+    function isValidSignature(bytes32 signHash, bytes memory signature) external view returns (bytes4);
+    function drain(address to, address token) external returns (uint256 amount);
+    function transferOwnership(address newOwner) external;
+    function claimOwnership() external;
+    function transact(bytes memory data) external;
+}
diff --git a/test/mock/IOwnable.sol b/test/mock/IOwnable.sol
new file mode 100644
index 00000000..d214376f
--- /dev/null
+++ b/test/mock/IOwnable.sol
@@ -0,0 +1,11 @@
+pragma solidity ^0.8.0;
+
+interface IOwnable {
+    // Event declarations
+    event OwnershipTransferred(address indexed previousOwner, address indexed newOwner);
+
+    // Function declarations
+    function transferOwnership(address newOwner) external;
+    function renounceOwnership() external;
+    function owner() external view returns (address);
+}
diff --git a/test/mock/IOwnedUpgradabilityProxy.sol b/test/mock/IOwnedUpgradabilityProxy.sol
new file mode 100644
index 00000000..2fefc629
--- /dev/null
+++ b/test/mock/IOwnedUpgradabilityProxy.sol
@@ -0,0 +1,38 @@
+// SPDX-License-Identifier: UNLICENSED
+pragma solidity ^0.8.0;
+
+interface IOwnedUpgradabilityProxy {
+    /**
+     * @dev Event to show ownership has been transferred
+     * @param previousOwner representing the address of the previous owner
+     * @param newOwner representing the address of the new owner
+     */
+    event ProxyOwnershipTransferred(address indexed previousOwner, address indexed newOwner);
+
+    /**
+     * @dev Tells the address of the owner
+     * @return The address of the owner
+     */
+    function proxyOwner() external view returns (address);
+
+    /**
+     * @dev Allows the current owner to transfer control of the contract to a newOwner.
+     * @param newOwner The address to transfer ownership to.
+     */
+    function transferProxyOwnership(address newOwner) external;
+
+    /**
+     * @dev Allows the proxy owner to upgrade the current version of the proxy.
+     * @param implementation representing the address of the new implementation to be set.
+     */
+    function upgradeTo(address implementation) external;
+
+    /**
+     * @dev Allows the proxy owner to upgrade the current version of the proxy and call the new implementation
+     * to initialize whatever is needed through a low level call.
+     * @param implementation representing the address of the new implementation to be set.
+     * @param data represents the msg.data to bet sent in the low level call. This parameter may include the function
+     * signature of the implementation to be called with the needed payload
+     */
+    function upgradeToAndCall(address implementation, bytes calldata data) external payable;
+}
```

I'm ready to share it as a private self-hosted GitLab repository by secret link.

## Run and traceable output

```
$ forge test -vvvv
[⠢] Compiling...
No files changed, compilation skipped

Running 4 tests for test/MaliciousOperatorStopsExchangeContract.t.sol:ShutdownExchangeTest
[PASS] testExchangeSetDepositParams() (gas: 55405)
Traces:
  [59618] ShutdownExchangeTest::testExchangeSetDepositParams() 
    ├─ [0] VM::expectEmit(true, true, true, true) 
    │   └─ ← ()
    ├─ emit DepositParamsUpdate(freeDepositMax: 0, freeDepositRemained: 0, freeSlotPerBlock: 0, depositFee: 0)
    ├─ [0] VM::prank(0xacD3A62F3eED1BfE4fF0eC8240d645c1F5477F82) 
    │   └─ ← ()
    ├─ [49084] 0x9b93e47b7F61ad1358Bd47Cd01206708E85AE5eD::transact(0xe79579920000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000) 
    │   ├─ [41422] 0x9C07A72177c5A05410cA338823e790876E79D73B::setDepositParams(0, 0, 0, 0) 
    │   │   ├─ [37394] 0xc56C1dfE64D21A345E3A3C715FFcA1c6450b964b::setDepositParams(0, 0, 0, 0) [delegatecall]
    │   │   │   ├─ emit DepositParamsUpdate(freeDepositMax: 0, freeDepositRemained: 0, freeSlotPerBlock: 0, depositFee: 0)
    │   │   │   └─ ← ()
    │   │   └─ ← ()
    │   ├─ emit TargetCalled(target: 0x9C07A72177c5A05410cA338823e790876E79D73B, data: 0xe79579920000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000)
    │   └─ ← ()
    └─ ← ()

[PASS] testExchangeShutdown() (gas: 59081)
Traces:
  [63185] ShutdownExchangeTest::testExchangeShutdown() 
    ├─ [7506] 0x9C07A72177c5A05410cA338823e790876E79D73B::isShutdown() [staticcall]
    │   ├─ [2492] 0xc56C1dfE64D21A345E3A3C715FFcA1c6450b964b::isShutdown() [delegatecall]
    │   │   └─ ← 0x0000000000000000000000000000000000000000000000000000000000000000
    │   └─ ← 0x0000000000000000000000000000000000000000000000000000000000000000
    ├─ [0] VM::prank(0xacD3A62F3eED1BfE4fF0eC8240d645c1F5477F82) 
    │   └─ ← ()
    ├─ [43372] 0x9b93e47b7F61ad1358Bd47Cd01206708E85AE5eD::transact(0xfc0e74d1) 
    │   ├─ [38927] 0x9C07A72177c5A05410cA338823e790876E79D73B::shutdown() 
    │   │   ├─ [38516] 0xc56C1dfE64D21A345E3A3C715FFcA1c6450b964b::shutdown() [delegatecall]
    │   │   │   ├─ emit Shutdown(timestamp: 1701194555 [1.701e9])
    │   │   │   └─ ← 0x0000000000000000000000000000000000000000000000000000000000000001
    │   │   └─ ← 0x0000000000000000000000000000000000000000000000000000000000000001
    │   ├─ emit TargetCalled(target: 0x9C07A72177c5A05410cA338823e790876E79D73B, data: 0xfc0e74d1)
    │   └─ ← ()
    ├─ [1006] 0x9C07A72177c5A05410cA338823e790876E79D73B::isShutdown() [staticcall]
    │   ├─ [492] 0xc56C1dfE64D21A345E3A3C715FFcA1c6450b964b::isShutdown() [delegatecall]
    │   │   └─ ← 0x0000000000000000000000000000000000000000000000000000000000000001
    │   └─ ← 0x0000000000000000000000000000000000000000000000000000000000000001
    └─ ← ()

[PASS] testLoopringIOExchangeOwnerIsEOA() (gas: 5011)
Traces:
  [5011] ShutdownExchangeTest::testLoopringIOExchangeOwnerIsEOA() 
    └─ ← ()

[PASS] testSetMaxAgeDepositUntilWithdrawable() (gas: 49087)
Traces:
  [53300] ShutdownExchangeTest::testSetMaxAgeDepositUntilWithdrawable() 
    ├─ [7473] 0x9C07A72177c5A05410cA338823e790876E79D73B::getMaxAgeDepositUntilWithdrawable() [staticcall]
    │   ├─ [2480] 0xc56C1dfE64D21A345E3A3C715FFcA1c6450b964b::getMaxAgeDepositUntilWithdrawable() [delegatecall]
    │   │   └─ ← 0x000000000000000000000000000000000000000000000000000000000013c680
    │   └─ ← 0x000000000000000000000000000000000000000000000000000000000013c680
    ├─ [0] VM::prank(0xacD3A62F3eED1BfE4fF0eC8240d645c1F5477F82) 
    │   └─ ← ()
    ├─ [33468] 0x9b93e47b7F61ad1358Bd47Cd01206708E85AE5eD::transact(0x960af2d90000000000000000000000000000000000000000000000000000000000000001) 
    │   ├─ [28703] 0x9C07A72177c5A05410cA338823e790876E79D73B::setMaxAgeDepositUntilWithdrawable(1) 
    │   │   ├─ [28287] 0xc56C1dfE64D21A345E3A3C715FFcA1c6450b964b::setMaxAgeDepositUntilWithdrawable(1) [delegatecall]
    │   │   │   ├─ [7404] 0x0a5d144ADF62e18eE222f2D05a2Bf2037ce8EeAe::1854b85c(00000000000000000000000000000000000000000000000000000000000000030000000000000000000000000000000000000000000000000000000000000001) [delegatecall]
    │   │   │   │   ├─ emit MaxAgeDepositUntilWithdrawableChanged(param0: 0x9C07A72177c5A05410cA338823e790876E79D73B, param1: 1296000 [1.296e6], param2: 1)
    │   │   │   │   └─ ← 0x000000000000000000000000000000000000000000000000000000000013c680
    │   │   │   └─ ← 0x000000000000000000000000000000000000000000000000000000000013c680
    │   │   └─ ← 0x000000000000000000000000000000000000000000000000000000000013c680
    │   ├─ emit TargetCalled(target: 0x9C07A72177c5A05410cA338823e790876E79D73B, data: 0x960af2d90000000000000000000000000000000000000000000000000000000000000001)
    │   └─ ← ()
    ├─ [973] 0x9C07A72177c5A05410cA338823e790876E79D73B::getMaxAgeDepositUntilWithdrawable() [staticcall]
    │   ├─ [480] 0xc56C1dfE64D21A345E3A3C715FFcA1c6450b964b::getMaxAgeDepositUntilWithdrawable() [delegatecall]
    │   │   └─ ← 0x0000000000000000000000000000000000000000000000000000000000000001
    │   └─ ← 0x0000000000000000000000000000000000000000000000000000000000000001
    └─ ← ()

Test result: ok. 4 passed; 0 failed; 0 skipped; finished in 5.77s
 
Ran 1 test suites: 4 tests passed, 0 failed, 0 skipped (4 total tests)
```
