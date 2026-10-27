# #35710 \[BC-Insight] addressToPartition input is unsanitized, allowing to take whole network down

**Submitted on Oct 4th 2024 at 09:34:38 UTC by @Merkle\_Bonsai for** [**Audit Comp | Shardeum: Core II**](https://immunefi.com/audit-competition/shardeum-core-ii-boost)

* **Report ID:** #35710
* **Report Type:** Blockchain/DLT
* **Report severity:** Insight
* **Target:** <https://github.com/shardeum/shardus-core/tree/dev>
* **Impacts:**
  * Network not being able to confirm new transactions (total network shutdown)
  * Causing network processing nodes to process transactions from the transaction queue beyond set parameters
  * Shutdown of greater than or equal to 30% of network processing nodes without brute force actions, but does not shut down the network
  * Shutdown of greater than 10% or equal to but less than 30% of network processing nodes without brute force actions, but does not shut down the network
  * informational: potential risk

## Description

This is extra attack case referring to reports #35696 ("Specifically crafted penalty TX may cause total network shutdown") and #35694 ("Consensus can be bypassed by single validator node from transaction execution group"), abusing same root causes, yet potentially posessing extra risks due to additional code issue, so I'm reporting it as low. I'm OK if this will be reclassified as informational, since it refers to same vectors and only provides additional data.

## Vulnerability Details

\`addressToPartition\` function in \`shardus-core/.../shard-functions.ts\` handles unsanitized input from multiple sources. Code logic is assuming that for any string input that contains hexadecimal numeric value, it will guarantee to output correct partition within range. Yet, it is possible to pass array value instead of string by numeric ways, including, but not limited to (it's only what I've found)

* \`Shardus.put\`: \`getTxSenderAddress(tx)\` is used, where for internal and debug TXs it is possible to pass value from JSON field, not extract from eth tx with some guarantees
* \`binary/poqo\_send\_receipt\`: pass as \`source\` field

Due to array-string implicit type casting, it is possible to use \`string\`, \`number\`, \`\[string]\` and \`\[number]\` in many cases vice-versa. This means that manipulations like those will have same effect: \`\`\` > parseInt('100') 100 > parseInt(\['100']) 100 > parseInt(\['100', '200']) 100 > BigInt(100) 100n > BigInt(\[100]) 100n > '0x' + \['100'] '0x100' \`\`\`

However, other functions behavior may vary - e.g. \`Array#slice\` vs \`String#slice\`.

For example, when \`set-global\` will be sent like that: \`\`\` { signs: \[], tx: { address: <...>, value: <...>, when: <...>, source: \['0xdeadbeef000000000'], }, consensusGroup: \[], } \`\`\`

it will call \`addressToPartition\` function in following way: \`\`\` static addressToPartition( shardGlobals: ShardGlobals, address: \['0xdeadbeef000000000'] ): { homePartition: number; addressNum: number } { const numPartitions = shardGlobals.numPartitions // let's say 128 const addressNum = parseInt(address.slice(0, 8), 16) // parseInt(\['0xdeadbeef000000000'], 16) => 256731055697558500000

```
// 2^32  4294967296 or 0xFFFFFFFF + 1
const size &#x3D; Math.round((0xffffffff + 1) / numPartitions) // 33554432
let homePartition &#x3D; Math.floor(addressNum / size) // 7651181688832
if (homePartition &#x3D;&#x3D;&#x3D; numPartitions) {
  homePartition &#x3D; homePartition - 1
}

return { homePartition, addressNum } // {homePartition: 7651181688832, addressNum: 256731055697558500000}
```

} \`\`\`

## Impact Details

The most potentially impactful vector I've been able to find is reusing #35694 ("Consensus can be bypassed by single validator node from transaction execution group") to deliver malicious payload.

\`set-global\` gossip route is executing \`Context.shardus.put\` without \`await\` in \`processReceipt\` function (that is only available after transaction validation by multiple nodes that can be bypassed with #35694 report), causing every node that received message to shutdown due to uncaught exception (like in #35696) because of following logic in \`Shardus.put\`: \`\`\` const senderAddress = this.app.getTxSenderAddress(tx); ... const consensusGroup = this.getConsenusGroupForAccount(senderAddress) \`\`\` It can be e.g. \`InitNetwork\` global internal transaction that is using \`tx.network\` passed (yet it is not used anywhere except \`getTxSenderAddress\` function). Any other path that leads to \`addressToPartition\` I've found is also simply crashing, yet it is handling the error.

## Extra note on \`InitNetwork\` internal TX

Please also note that any user can impersonate this internal call by passing \`tx.network\` value modified - it is not used anywhere except signature check, that allows anyone to pass this tx. Yet, I do not see any potential impacts that can be caused by that tx.

## Proof of Concept

## Proof of Concept

This is simplest PoC, demonstrating only safe error handling.

I'm using API method added to node to pass internal gossips: \`\`\` shardus.registerExternalPost('binary\_gossip', async (req, res) => { function serializeGossipReq(stream: VectorBufferStream, obj: GossipReqBinary, root = false): void { if (root) { stream.writeUInt16(57) // TypeIdentifierEnum.cGossipReq } stream.writeUInt8(1) stream.writeString(obj.type) stream.writeString(obj.data as string) }

```
await tellBinary&lt;GossipReqBinary&gt;(
    p2pNodeList.byIdOrder,
    InternalRouteEnum.binary_gossip,
    {
      type: req.body.type,
      data: req.body.data
    },
    serializeGossipReq,
    {
      tracker_id: &#x27;&#x27;,
    },
    true,
    &#x27;&#x27;
)
```

}) \`\`\`

Following call will cause transaction to be dropped with error "cannot read property consensusNodeForOurNodeFull of undefined" (in \`getConsensusGroupIds\`), demonstrating that \`addressToPartition\` got out of range.
