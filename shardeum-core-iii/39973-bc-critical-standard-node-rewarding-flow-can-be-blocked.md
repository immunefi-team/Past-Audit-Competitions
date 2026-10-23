# #39973 \[BC-Critical] Standard node rewarding flow can be blocked

**Submitted on Feb 11th 2025 at 23:57:05 UTC by @periniondon630 for** [**Audit Comp | Shardeum: Core III**](https://immunefi.com/audit-competition/audit-comp-shardeum-core-iii)

* **Report ID:** #39973
* **Report Type:** Blockchain/DLT
* **Report severity:** Critical
* **Target:** <https://github.com/shardeum/shardeum/tree/bugbounty>
* **Impacts:**
  * Direct loss of funds

## Description

## Brief/Intro

An attacker can frontrun validator reward transactions across the entire network and disrupt the reward distribution process.

## Vulnerability Details

When the network deactivates or removes a node, it issues a nodereward transaction. An attacker can censor this transaction across the entire network, effectively disrupting the reward distribution process.

### Root Cause

The issue lies in the hash field of network transactions, as defined in the AddNetworkTx interface:

```
export interface AddNetworkTx<T = any> {
  hash: string
  type: string
  txData: T
  cycle: number // cycle the tx was added
  priority: number // priority in the txList (0 is lowest), second only to cycle
  subQueueKey?: string
}
```

The hash field is not properly validated each time a node receives a gossip message containing a new network transaction. Instead, it is directly used to determine whether the node has already received the transaction:

```
 if (txAdd.some((entry) => entry.hash === payload.hash)) {
      return
 }
```

### Attack Mechanism

Since all nodes generate the same network transactions based on cycle records, an attacker can precompute the hash of a new transaction and inject a modified transaction with a new hash but an old payload.\
On target nodes, the new hash will occupy a key in the txAdd hashmap, effectively preventing the valid transaction from being propagated and processed. The attacker must craft the old payload in a way that allows it to bypass initial filtering so that it gets added to the txAdd hashmap:

```
    const verifyFunction = beforeAddVerifier.get(addTx.type)
    if (!verifyFunction) {
      /* prettier-ignore */ if (logFlags.p2pNonFatal) error('Verify function is undefined')
      return false
    }

    if (!(await verifyFunction(addTx))) {
      /* prettier-ignore */ if (logFlags.p2pNonFatal) error(
        `Failed add network tx verification of type ${addTx.type} \n tx: ${stringifyReduce(addTx.txData)}`
      )
      return false
    }
    /* prettier-ignore */ if (logFlags.p2pNonFatal) console.log('add network tx', addTx.type, addTx.txData.publicKey, addTx)
    return true
```

Once the transaction passes validation in beforeAddVerifier, it is added to addProposals and subsequently included in txAdd through the sendRequests function:

```
export function sendRequests(): void {
  for (const add of addProposals) {
    if (!txAdd.some((entry) => entry.hash === add.hash)) {
      const { sign: sign1, ...unsignedAddNetworkTx } = add
      const addTxCopy = structuredClone(unsignedAddNetworkTx)
      const { sign: sign2, ...txDataWithoutSign } = addTxCopy.txData
      addTxCopy.txData = txDataWithoutSign
      txAdd.push(addTxCopy)
    }
```

By injecting a new hash with an old payload, the attacker prevents the legitimate transaction from propagating across the network. This censorship can also be performed at the node level, as the gossip protocol and internal transaction flow both rely on the same hashmap:

```
} else if (eventType === 'node-deactivated') {
	    // not relevant code
            shardus.serviceQueue.addNetworkTx('nodeReward', shardus.signAsNode(txData), data.publicKey)
          }
        }
```

## Impact Details

Targeted (specific node) or full network node reward process disruption, allowing an attacker to manipulate or delay validator payouts.

## References

<https://github.com/shardeum/lib-types/blob/ebf34c1538e8ece8a6e022cf4774ee27fd4ddcc7/src/p2p/ServiceQueueTypes.ts#L9\\>
<https://github.com/shardeum/core/blob/5515dedc5f67d621c9e179f22a32e54e66e8682d/src/p2p/ServiceQueue.ts#L77\\>
<https://github.com/shardeum/core/blob/5515dedc5f67d621c9e179f22a32e54e66e8682d/src/p2p/ServiceQueue.ts#L628C5-L641C16\\>
<https://github.com/shardeum/core/blob/5515dedc5f67d621c9e179f22a32e54e66e8682d/src/p2p/ServiceQueue.ts#L455>

## Link to Proof of Concept

<https://gist.github.com/periniondon630/9f86f0fae7e782aab93ca256cb7c60dd>

## Proof of Concept

## Proof of Concept

1. Apply patch for core & shardeum repos from gist
2. Build, start network, add extra nodes for rotation, stake.
3. Activate attack mode on attacker's node:

```
curl -X http://<ip>:<port>/hack
```

4. See logs on the nodes how nodereward internal transactions are rejected.

Log samples:

Attacker:

```
HACK changing nodeReward transaction for 8917a99551c7385fbbb30b9961f2e0cb731b46cc4d4c0113fa5fb8ac8a8be49b->84409a1ea84c3f3308cad043dc6a0badf70cc8661697ecada3a4774eb34c3156 new hash:88523a72ced6dc71fb36d3de3563aa80e1863827e1ef42904586eb0a4fc5fce8
```

Victim:

```
eventNotify node-deactivated 8917a99551c7385fbbb30b9961f2e0cb731b46cc4d4c0113fa5fb8ac8a8be49b
// gossip with hacked tx
HACK received nodeReward gossip tx for 84409a1ea84c3f3308cad043dc6a0badf70cc8661697ecada3a4774eb34c3156 with hash:88523a72ced6dc71fb36d3de3563aa80e1863827e1ef42904586eb0a4fc5fce8
HACK received nodeReward gossip 2 tx for 84409a1ea84c3f3308cad043dc6a0badf70cc8661697ecada3a4774eb34c3156 with hash:88523a72ced6dc71fb36d3de3563aa80e1863827e1ef42904586eb0a4fc5fce8
// was added to txAdd
HACK adding hash:88523a72ced6dc71fb36d3de3563aa80e1863827e1ef42904586eb0a4fc5fce8 from nodeReward gossip for 84409a1ea84c3f3308cad043dc6a0badf70cc8661697ecada3a4774eb34c3156 to txAdd
// gossip with original tx same hash
HACK received nodeReward gossip tx for 8917a99551c7385fbbb30b9961f2e0cb731b46cc4d4c0113fa5fb8ac8a8be49b with hash:88523a72ced6dc71fb36d3de3563aa80e1863827e1ef42904586eb0a4fc5fce8
HACK received nodeReward gossip 2 tx for 8917a99551c7385fbbb30b9961f2e0cb731b46cc4d4c0113fa5fb8ac8a8be49b with hash:88523a72ced6dc71fb36d3de3563aa80e1863827e1ef42904586eb0a4fc5fce8
// it didn’t go through hash condition
HACK received nodeReward gossip tx for 8917a99551c7385fbbb30b9961f2e0cb731b46cc4d4c0113fa5fb8ac8a8be49b with hash:88523a72ced6dc71fb36d3de3563aa80e1863827e1ef42904586eb0a4fc5fce8
HACK received nodeReward gossip 2 tx for 8917a99551c7385fbbb30b9961f2e0cb731b46cc4d4c0113fa5fb8ac8a8be49b with hash:88523a72ced6dc71fb36d3de3563aa80e1863827e1ef42904586eb0a4fc5fce8
```
