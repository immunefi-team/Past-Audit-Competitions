# #35601 \[BC-Critical] Consensus algorithm doesn't deduplicate votes, allowing a malicious validator t

**Submitted on Sep 30th 2024 at 13:59:00 UTC by @throwing5tone7 for** [**Audit Comp | Shardeum: Core II**](https://immunefi.com/audit-competition/shardeum-core-ii-boost)

* **Report ID:** #35601
* **Report Type:** Blockchain/DLT
* **Report severity:** Critical
* **Target:** <https://github.com/shardeum/shardus-core/tree/dev>
* **Impacts:**
  * Direct loss of funds
  * Permanent freezing of funds (fix requires hardfork)

## Description

## Brief/Intro

When validating a receipt, the new consensus algorithm allows duplicate votes and counts them as though they are distinct. This allows a maliciously modified validator node to submit a receipt that contains only votes from itself, and trick the other nodes into thinking that this is a valid receipt that a majority of validators have voted for. The transaction end state linked to the bogus vote will then be assumed to be agreed upon by a quorum of validators, whereas in fact it is under the attacker's control and no quorum was achieved. The attacking node can fake whatever transaction end state it likes for its vote - so for example, it can create SHM out of thin air or just burn funds from another account. This allows a single malicious validator to falsify transaction states on the blockchain for accounts that are in its shard, leading to a wide range of possible bad outcomes for the network.

## Vulnerability Details

The main issue lies in <https://github.com/shardeum/shardus-core/blob/23e06ded6744d8521cff9d749c1f1dd482c5fcb6/src/state-manager/TransactionConsensus.ts#L1742> and onwards, we can see that the code here (part of the \`verfiyAppliedReceipt\` function) is responsible for counting up the votes in the receipt to see if a quorum was reached:

\`\`\` for (let i = 0; i < receipt.signaturePack.length; i++) { const sign = receipt.signaturePack\[i] if (!executionGroupNodes.has(sign.owner)) continue appliedVoteHash.voteTime = receipt.voteOffsets\[i] const signedObject = { ...appliedVoteHash, sign }; if (this.crypto.verify(signedObject, sign.owner)) { validSignatures++; } }

```
const totalNodes &#x3D; executionGroupNodes.size;
const requiredMajority &#x3D; Math.ceil(totalNodes * this.config.p2p.requiredVotesPercentage)
return validSignatures &gt;&#x3D; requiredMajority;
```

\`\`\`

You can see that there is no check on whether a single node has voted twice, so the attacker node can just repeat a single signed vote enough times to represent a majority and trick other nodes into thinking that is a legitimate receipt.

### Exploit steps

Based on the bug, validators can can be tricked into accepting a receipt that is invalid. In order to exploit this to malicious effect, an attacker needs to be running a node in the network, and to be able to make that node act maliciously. The threat model is that the attacker modifies the validator code subtly, so that it normally behaves as a legitimate validator, but can be made to inject an attacking transaction into the network based on some trigger (in my PoC it is a post endpoint that receives an EVM transaction).

When the attacker's node is performing an attack it does essentially the following:

* Create or capture some legitimate transaction to submit to the network (e.g. an EVM transaction to transfer an SHM from one of their accounts to another)
* Spread that transaction data to the rest of the nodes in it's shard in the normal way, so that they know that consensus will be happening on the transaction
* Immediately create a vote for a new state of the Shardeum accounts that reflects the outcome of the attack (e.g. in my PoC I just add a large amount of SHM to both account balances before applying the legitimate EVM transaction)
* Create a receipt by vote-stuffing of this single vote enough times to reach a quorum of the shard, and send it to all of the other validators in the shard
* The other validators will typically accept this as a legit receipt for a legit transaction, but their end states for the transaction will not match the voted state, which will mean that they attempt to repair the transaction
  * At this point, they will ask the attacker's node (the sender of the receipt in question) for the "correct" state data, at which point the attacker's node can reply with it's own bogus state
* The other validators accept the bogus state, and whatever effect the attacker wanted to achieve has been committed to the blockchain

### Exploitability on a larger network

I have only demonstrated this on a smaller network for now, since I thought that reporting such a critical bug quickly would be more productive than tuning the PoC exploit for larger network sizes. However, I believe the attack will work against larger network sizes based on these assumptions:

* The attacker can only affect the accounts that belong to the shard that their validator node processes - however, this should not restrict them, as once they know what shard they process they should be able to create an account for themselves that belongs to the shard (e.g. by creating random accounts repeatedly until one matches their requirements)
* The attacker can broadcast the relevant messages to all of the shard participants, ensuring that their bogus receipt is received before any other validators in the shard start gossiping possible results to each other
  * I assume this because the attacker can send the relevant messages in any order they like and because each of the other nodes delays before doing much processing on a transaction - there are several checks in the \`processTransactions\` function of \`TransactionQueue\` which check that the queue entry is a certain age before continuing processing of it. Hence there should be enough time for the attacker's receipt message to arrive before another node gets the chance to create a different receipt.
  * Given that the attack model for this is a hacked validator node, the attacker's code clearly has all of the data available to know who needs to receive the bogus messages (i.e. who is in their shard)

## Impact Details

Using this exploit, the attacker can force the network to accept an invalid state transition for any accounts that are in the shard of their malicious validator node. This means they could:

* Create coins out of nothing - this is the effect demo'd in the POC
* Empty out balances of victims, i.e. set them to zero without transferring the funds elsewhere
* Deploy completely broken / malicious EVM contract code to do whatever they want
* Destroy existing contracts (since they can change the state of the contract account) potentially locking / destroying tokens or other contract-managed assets
* Change contract storage directly - e.g. to freeze / steal ERC-20 tokens or similar

In essence, they can perform any EVM-level effects on accounts in their shard. The above are examples. There may also be some Shardus chain internal transactions they can fake in a similar way, although I haven't confirmed it.

## References

<https://youtu.be/8-\\_fydHRUbU> - the video of my PoC

## Link to Proof of Concept

<https://youtu.be/8-_fydHRUbU>

## Proof of Concept

## Proof of Concept

See the demo video in <https://youtu.be/8-\\_fydHRUbU> for a direct demonstration

The PoC uses a hacked validator node to connect to a network containing legitimate nodes. Most of the PoC functionality is built into this maliciously modified validator codebase. Although a range of effects are possible using this bug, in this PoC I demonstrate the ability to change account balances in illegitimate ways.

### PoC demo instructions

In order to recreate the PoC demo, you need to set up a legitimate network that the hacked validator is allowed to join, and modify a validator by using the patches in my GIST.

#### Configure a legitimate network

NOTE - I assume that the developers can do this another way - what is required is a network running non-modified code that the attacker's node can join (it therefore needs to at least allow connections from localhost IP addresses, which is covered by my config patch). The other requirement is that the attacker has an account on the shard that they validate which has some SHM balance.

To follow the steps I use in the demo video:

* Get source code for real shardeum down (<https://github.com/shardeum/shardeum>) to a folder of your choice, e.g. \`LEGIT\_Shardeum\`
* Install dependencies using \`npm ci\`
* Apply the config to use a smaller net for testing, and to allow localhost network addresses for nodes & archivers - \`git apply config-small-net-local-nodes.patch\`
* Apply the patch to genesis.json to grant the attacker some funds - \`git apply net-genesis.patch\`

#### Patch a validator codebase to run the attack

Maliciously modify shardus-core code:

* Download a fresh copy of shardus-core repo (<https://github.com/shardeum/shardus-core>) to a folder where you modify it, e.g. \`HACKED\_shardus-core\`
  * Apply the patch to this \`HACKED\_shardus-core\` folder - \`git apply shardus-core.patch\`
  * Run \`npm install\` to ensure it's all up-to-date with the patches and to fetch all dependencies

Maliciously modify shardeum app code:

* Download a fresh copy of shardeum repo (<https://github.com/shardeum/shardeum>) to a folder where you modify it, e.g. \`HACKED\_shardeum\`
  * Run \`npm ci\` to get the initial dependencies installed
  * Apply the **code patch** to this \`HACKED\_shardeum-folder\` - \`git apply shardeum-src-changes.patch\`
  * Ensure that this hacked shardeum code, uses the hacked shardus-core code rather than the legit dependency - \`git apply shardeum-package-changes.patch\`
    * **IMPORTANT**: this patch assumes that the hacked shardus-core folder can be found at path \`../HACKED\_shardus-core\` - you will need to manually edit package.json to set a different path if it's not in that location
  * \`npm install\` to get everything built & up to date

Configure a validator CLI to connect to the legit network but run the maliciously modified code:

* Configure validator-cli to talk to local nodes and to run over hacked source (assuming the HACKED\_shardeum folder has the relative path shown)
  * \`ln -s "$(cd ../HACKED\_shardeum && pwd)" ../validator\`
  * Apply the patch to configure validator to talk to local networks \`git apply validator-cli-config.patch\`
  * Run \`npm run compile\` to get everything up to date and ready to launch

#### Start up the legitimate nodes in a network

Devs might do this step differently, but to follow along with the demo video, in the \`LEGIT\_Shardeum\` folder launch a network that still has space for one node to join (the attacker's node)

* \`shardus create-net 10 --no-start && shardus start --dir instances 9\`

#### Start up attacker node - maliciously modified validator

In the validator-cli folder:

* Run \`operator-cli start\`

#### Wait for network to reach an appropriate state

I wait until all of the nodes have reached active state, and wait until 15 cycles after the network mode is first reported as \`processing\` according to the monitoring logs

#### Start up JSON RPC server

* Run \`npm run start\` in JSON-RPC folder

#### Setup and run attack

Setup a folder for the scripts and infra for attack

* Download \`poc.js\`, \`checkBalance.js\` and \`package.json\` from the GIST and put into an attack-scripts folder
* Ensure you use node 18.16.1 for this folder
* Run \`npm install\` to fetch all dependencies

Ensure network is in a good state before proceeding, and then:

* Run \`node checkBalance.js\` to see the account initial balance - should be \`10000000000000000000000000\`
* Run \`node poc.js 127.0.0.1:9050\` - assuming that your hacked validator node has been started at this address & port
  * This sends a legitimate EVM transaction to the attacker node that transfers 1 wei from attacker to a random address
  * You will see the transaction in the output of the script, take a note of address in the \`to\` field of transaction
* You should see a log that says "ATTACK working it's way through the network - poll for results."
* Wait a minute or so and then run \`node checkBalance.js\`
  * If the attack succeeds, attacker's balance will have increased
* Using the recipient address captured from the attack output, run \`node checkBalance.js RECIPIENT\_ADDRESS\`
  * The recipient's balance has also increased
* So SHM has been created out of thin air - since post\_balance\_sender + post\_balance\_recipient > pre\_balance\_sender + pre\_balance\_recipient!

### PoC deep dive

**NOTE**: this is very much a PoC, so the code is pretty rough and mostly copy-pasted and then modified versions of the original code. There is a lot of mechanics needed to make the validator act as a legitimate validator when not performing the attack, so I haven't attempted to extract a huge amount of code out of shardeum / shardus to provide a completely independent script that would perform the attack by sending messages itself.

The attack uses a legitimate EVM transaction (transfer 1 wei from attacker account to a random wallet) but pretends that this leads to an EVM state where both accounts end up with an extra \`100000000000000000000000000\` wei SHM. This is achieved by the line

\`\`\` wrappedEVMAccount.account.balance = (wrappedEVMAccount.account.balance || BigInt(0)) + BigInt(100000000000000000000000000) \`\`\`

Which is in my \`ATTACK\_apply\` function in the patch applied to the Shardeum app codebase (see shardeum-src-changes.patch).

In order to run the attack the PoC code patches to Shardeum and Shardus core achieve the following:

* Expose a new web endpoint on the validator node - line starting \`shardus.registerExternalPost('run-attack', externalApiMiddleware, async (req, res) => {\` in my Shardeum app patch (see shardeum-src-changes.patch)
* This endpoint receives a normal EVM transaction, and then calls into shardus core via a newly created function \`shardus.run\_ATTACK\`
* Within the implementation of \`run\_ATTACK\` inside the patch to the shardus-core codebase (see shardus-core.patch in the GIST) the code does the following:
  * Construct a new QueueEntry object for further processing \`TransactionQueue.ATTACKER\_createQueueEntry\` and return it - does not add this to any processing queues
  * Save this queue entry to \`TransactionQueue.ATTACKERS\_transaction\` so that if any node asks for it we can return the details
  * Preprocess it to add sufficient data to be able to create a vote - \`TransactionQueue.ATTACKER\_preprocessTransaction\`
    * Within this processing, it calls back into the Shardeum app to generate a bogus result by calling \`app.ATTACK\_apply\` - this function does everything as normal but also adds 100000000000000000000000000 wei to each account it processes
  * Pass this into another routine to construct a legitimate vote (but based on this bogus EVM end state) - \`ATTACKER\_createVote\`
  * Does not send the vote to other participants, instead creates a bogus receipt that is a vote stuffing (i.e. repetition) of this one single legitimate vote - \`ATTACKER\_tryProduceReceipt\`
  * Gossips this to all of the other participants
* If any other nodes ask for details on the bogus transaction, return the copy saved into \`TransactionQueue.ATTACKERS\_transaction\`
