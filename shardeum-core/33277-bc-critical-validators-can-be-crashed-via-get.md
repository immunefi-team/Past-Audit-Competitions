# 33277 - \[BC - Critical] Validators can be crashed via GET

Submitted on Jul 17th 2024 at 03:26:00 UTC by @usmannk for [Boost | Shardeum: Core](https://immunefi.com/bounty/shardeum-core-boost/)

Report ID: #33277

Report type: Blockchain/DLT

Report severity: Critical

Target: <https://github.com/shardeum/shardeum/tree/dev>

Impacts:

* Network not being able to confirm new transactions (total network shutdown)

## Description

## Brief/Intro

Simply calling the default endpoint `eth_getBlockByHash` **with no params** causes a node to crash and die permanently.

By looping this over the network, an attacker can halt all transactions.

## Proof of Concept

* Run a local cluster:
  * `$ shardus start 10`
* Pick a node, say, 9001.
* Crash the node by navigating to `http://localhost:9001/eth_getBlockByHash` and not providing parameters
* Go to `http://127.0.0.1:4000/nodelist` and observe that the node is offline
