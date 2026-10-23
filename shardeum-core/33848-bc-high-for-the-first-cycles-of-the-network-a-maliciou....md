# 33848 - \[BC - High] For the first cycles of the network a maliciou...

Submitted on Jul 31st 2024 at 08:40:24 UTC by @infosec\_us\_team for [Boost | Shardeum: Core](https://immunefi.com/bounty/shardeum-core-boost/)

Report ID: #33848

Report type: Blockchain/DLT

Report severity: High

Target: <https://github.com/shardeum/shardeum/tree/dev>

Impacts:

* Direct loss of funds

## Description

## Brief/Intro

Shardeum requires nodes to stake funds before joining the network.

The staked amount can be slashed if malicious behavior is detected and prevents the mass creation of malicious nodes.

This report demonstrates how to bypass that staking process, allowing malicious users to join with an unlimited amount of malicious nodes to increase their chances of taking over an active set in a future cycle, hence, controlling the consensus mechanism that defines the outcome of any transaction in the blockchain.

## Vulnerability Details

When joining the network as a node, checks are made to the staking TX.

All staking checks for requests to join the network are disabled when the amount of active nodes is under `ShardeumFlags.minActiveNodesForStaking`

```
const stakingEnabled = ShardeumFlags.StakingEnabled && numActiveNodes >= ShardeumFlags.minActiveNodesForStaking
```

> Code snippet from: <https://github.com/shardeum/shardeum/blob/dev/src/index.ts#L6153-L6156>

When the network is launched for the very first time it starts with 1 active node, and others will send a request to join.

It takes about 10 cycles before some nodes can finally join.

In our tests, getting from cycle number 0 to cycle number 10 takes \~11 minutes.

During that time, the value of the variable `numActiveNodes` is **1**, therefore `stakingEnabled` becomes **false**, skipping all checks to the stake of a node when requesting to join the network.

As a result, from **Cycle 0** to **Cycle 10** (and sometimes a higher cycle number), any malicious user can request to join the network from an unlimited amount of nodes, bypassing all checks for his stake.

In our proof of concept, we will demonstrate how these malicious nodes ultimately join the network and become active validators, after bypassing all checks to the **stake certificate** in the **join** request.

## Impact Details

Exponentially increase an attacker's chances of controlling all active nodes in a cycle to a point where is almost certain he will control all active nodes in a cycle.

Controlling all nodes in a cycle leads to a loss of funds by manipulating the outcome of any transaction.

## Proof of Concept

Download locally Shardus Core repo and Shardeum's repo locally.

Apply the following patch to the Shardeum repo:

```
git apply debug-10-nodes.patch
```

Now point Shardeum to use the local copy of Shardu's Core as instructed in the README.md file of Shardeum's codebase.

Go to the file `/src/p2p/Join/index.ts` in Shardus Core (<https://github.com/shardeum/shardus-core/blob/dev/src/p2p/Join/index.ts#L780>) line number 780, and replace the following code snippet:

```
if (appJoinData) {
  joinReq['appJoinData'] = appJoinData
}
```

With this one:

```
if (appJoinData) {
  console.log(`INFOSEC: Lets break the stake params in the appJoinData to prove anyone can bypass stake requirements from cycle 0 to cycle +10`);
  appJoinData['stakeCert'] = {};
  joinReq['appJoinData'] = appJoinData
}
```

In the code above, we have modified all nodes in the network to send an empty object as their `Stake Cert` - recap: the stake cert is the prove of having staked funds in the network, to join as a validator.

Now build both codebases, and start the network with 10 nodes using shardus's CLI (`shardus start 10`)

Wait until cycle number 15.

All nodes will have successfully joined the network with an empty `Stake Cert` and they will all become active validators.

\*\*With this test, we have proved that anyone can join the network with any amount of validators and 0 staked funds if they join during the first 10 (very often more) cycles of the network.
