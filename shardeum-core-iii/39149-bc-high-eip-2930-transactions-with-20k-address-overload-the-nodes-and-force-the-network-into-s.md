# #39149 \[BC-High] EIP-2930 transactions with 20k-address overload the nodes and force the network into "safety" mode

**Submitted on Jan 23rd 2025 at 13:20:53 UTC by @bountyhunter2048 for** [**Audit Comp | Shardeum: Core III**](https://immunefi.com/audit-competition/audit-comp-shardeum-core-iii)

* **Report ID:** #39149
* **Report Type:** Blockchain/DLT
* **Report severity:** High
* **Target:** <https://github.com/shardeum/shardeum/tree/bugbounty>
* **Impacts:**
  * Increasing network processing node resource consumption by at least 30% without brute force actions, compared to the preceding 24 hours
  * Network not being able to confirm new transactions (total network shutdown)

## Description

## Brief/Intro

Shardeum allows EIP 2930 txs in the code but there is no limit check on the number of addresses in the access list. This vulnerability allows attacker to inject huge EIP2030 transaction and cause the nodes's CPU usage to goes up a lot and eventually cause the network to go to safe mode and temporarily stop processing transactions.

## Vulnerability Details

Shardeum validators share accounts between them before applying the transaction in the EVM. When I attached 20k addresses in the EIP-2910 tx and send it to the network, the injected validator creates 20k accounts in the EVM and it causes lots of CPU resources. So, the tx will stuck in the queue for a long time until the emergency queue clearing mechanism kicks in because there is no way node can handle it and broadcast it to other nodes. During that period, I notice the injected validator's CPU usage goes up to 100+% (see the demo video).

Since the injected tx stuck in the queue and never get processed, it cost nothing to the attacker. He can inject the same tx to other nodes causing the same problem. After attacking the majority of the nodes, the network struggle to process new transactions because it is overwhelmed by handling 20k accounts.

The key idea behind this bug is that validator should limit the size of access list. Here is the current code causing this problem: <https://github.com/shardeum/shardeum/blob/bugbounty/src/index.ts#L5593>

## Impact Details

Nodes' CPU usages goes up 100+% and do not go down. Nodes becomes so occupied that they cannot process new transactions. After some time, nodes will crash due to hardware limit or being unresponsive and removed from the network. The network will go into "Safety" mode and stop processing transactions including staking and unstaking. Please see the demo: <https://youtu.be/QIF4-14jZgU>

## References

<https://gist.github.com/bountyhunter2048/b6c403717033737c36ff26cbceaa8dd2>

## Link to Proof of Concept

<https://gist.github.com/bountyhunter2048/b6c403717033737c36ff26cbceaa8dd2>

## Proof of Concept

## Proof of Concept

The concept to prove it is very simple. Attacker just need to create a few huge EIP-2930 txs and inject them to different nodes. The attack will be more dangerous if attacker do it from multiple machines in a short time. Here are the detail step and use the script provided in github gist.

* Start a network and add your address in genesis file so that you have funds to do the test
* Use this script in hardhat project. Put your private key in .env file
* Run the script. You can adjust number of wallets to use and number of transactions to inject
* The script will create EIP txs with access list having 20k addresses and inject those txs at 1 TPS
* The node that receives the transaction will use a lot of CPU resources to process it.
* The script will do this to each of the nodes in the network\
  In real world, attacker can use multiple machines to inject tx to different nodes in a short span of time
* The injected txs are not applied because nodes will never be able to handle 20k accounts in a transaction, making this attack basically free of charge.\
  After some time, due to continuous CPU load some nodes will crash and network will go into safe mode

Demo video: <https://youtu.be/QIF4-14jZgU>

Gist: <https://gist.github.com/bountyhunter2048/b6c403717033737c36ff26cbceaa8dd2>
