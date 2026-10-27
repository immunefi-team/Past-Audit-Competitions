# 33946 - \[BC - Critical] Lack of voter deduplication in sync\_trie\_hashes...

Submitted on Aug 2nd 2024 at 15:17:16 UTC by @GuplerSaxanoid for [Boost | Shardeum: Core](https://immunefi.com/bounty/shardeum-core-boost/)

Report ID: #33946

Report type: Blockchain/DLT

Report severity: Critical

Target: <https://github.com/shardeum/shardus-core/tree/dev>

Impacts:

* Direct loss of funds

## Description

## Brief/Intro

A malicious node is able to send falsified trie hashes through `sync_trie_hashes` internal call multiple times, and secure multiple votes for the falsified hash because there is no deduplication of voters happening in the process. Following this method, the malicious actor is able to mathematically secure majority votes for the given trie. This will trigger a repair process for accounts subject to falsification, and the nodes that voted on the falisified hashes are requested for the account data to repair. This provides an opportunity for the malicious actor to repair(or forge) account data.

## Vulnerability Details

The foundation for this exploit is built on the fact there is no deduplication of voters happening during the `sync_trie_hashes` call: <https://github.com/shardeum/shardus-core/blob/4d75f797a9d67af7a94dec8860220c4e0f9ade3c/src/state-manager/AccountPatcher.ts#L830>

This allows a malicious node to cast multiple votes and secure majority for a falsified trie hash.

The call to `sync_trie_hashes` endpoint is made every cycle to test and repair accounts against OOS. A malicious node can make use of the exploit to make the nodes in the active set falsely infer that their data is out of sync: <https://github.com/shardeum/shardus-core/blob/4d75f797a9d67af7a94dec8860220c4e0f9ade3c/src/state-manager/index.ts#L3892>

When a node infers that it's data is out of sync, it makes to call to nodes that voted on the falsified hash to get account data and patch it's state in reference to the newly obtained data. The node that voted on the falsified data can forge the account data and send them to node whose state is being repaired: <https://github.com/shardeum/shardus-core/blob/4d75f797a9d67af7a94dec8860220c4e0f9ade3c/src/state-manager/AccountPatcher.ts#L3888>

However, the attacker must control atleast two nodes in the active set because of the additional layer of security implemented here: <https://github.com/shardeum/shardus-core/blob/4d75f797a9d67af7a94dec8860220c4e0f9ade3c/src/state-manager/AccountPatcher.ts#L3928>

## Impact Details

By executing this exploit successfully, the attacker is able to forge account data as they wish. This leads to direct loss of funds.

## Proof of Concept

In this PoC, nodes at port 9001 and 9002 are considered as malicious nodes. The patches made to @shardus/core will depict the same

1. Apply this patch to @shardus/core: <https://gist.github.com/guplersaxanoid/d45daec81d7e0ecf7ee9eec0b5a09f3b#file-account-forge-patch>

This patch will depict malicious behaviour for nodes at port 9001 and 9002. Make sure to link the patched module to shardeum repo

2. Apply a debug patch to shardeum (20 nodes patch with verbose logging was used in our testing), and spin up a local network.
3. Wait for all nodes (or atleast the two malicious nodes) to go active. Notice that fatal logs of all active nodes will contain the details of all the accounts repaired with the falsified data.

Example logs: <https://gist.github.com/guplersaxanoid/d45daec81d7e0ecf7ee9eec0b5a09f3b#file-fatal-log>
