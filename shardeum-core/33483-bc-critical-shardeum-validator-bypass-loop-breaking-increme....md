# 33483 - \[BC - Critical] shardeum validator bypass loop breaking increme...

Submitted on Jul 22nd 2024 at 01:49:42 UTC by @ZhouWu for [Boost | Shardeum: Core](https://immunefi.com/bounty/shardeum-core-boost/)

Report ID: #33483

Report type: Blockchain/DLT

Report severity: Critical

Target: <https://github.com/shardeum/shardeum/tree/dev>

Impacts:

* Shutdown of greater than or equal to 30% of network processing nodes without brute force actions, but does not shut down the network
* Network not being able to confirm new transactions (total network shutdown)

## Description

## Description

In shardeum endpoint `eth_getBlockHashes` handler, it takes in fromBlock and toBlock as query params. later down the line it would parse into an integer using `parseInt()`. The number is then use to control the for loop. The number is not checked if it is within safe integer range of v8 engine, which would cause the for loop to run indefinitely.

## Proof of Concept

```
https://[target_node_ip:port]/eth_getBlockHashes?fromBlock=1000&toBlock=-9999999999999999
```

see impacted area [here](https://github.com/shardeum/shardeum/blob/c7b10c2370028f7c7cbd2a01839e50eb50faa904/src/index.ts#L1286-L1293)

```
parseInt(-9999999999999999)
// will be
-10000000000000000
```

With this set of input the loop will be like this `for (let i = -10000000000000000; i <= "-9999999999999999"; i++)`, Please use debugger to examine more.

Since `-10000000000000000` value is over the safe ranges of v8 integer loop `i++` incremental will always yeild the same value. Thus, loop breaking condion will never satsify and hand the node indefinitely.

## Impact

This'll cause the node to hang for at least a cycle or more eventually being kicked out of network, practically DoS-ing the node.
