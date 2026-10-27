# #35839 \[BC-Critical] Slash avoidance: Ineffective controls on unstaking allow unstaking before takin

**Submitted on Oct 10th 2024 at 12:42:48 UTC by @throwing5tone7 for** [**Audit Comp | Shardeum: Core II**](https://immunefi.com/audit-competition/shardeum-core-ii-boost)

* **Report ID:** #35839
* **Report Type:** Blockchain/DLT
* **Report severity:** Critical
* **Target:** <https://github.com/shardeum/shardeum/tree/dev>
* **Impacts:**
  * Bypassing Slashing

## Description

## Brief/Intro

The slashing behaviour of the network relies on the validator's stake being available to be slashed. In order to avoid a malicious validator withdrawing their stake before they can be slashed, the validators prevent an unstaking transaction if the staked validator is in the \`active\` state. However, this is ineffective because the malicious validator can force themselves into a non active state and then unslash. They can then take an action that would incur a penalty, like leaving the network, without their stake being slashed at all. Hence they can avoid slashing.

## Vulnerability Details

As described, the vulnerability occurs because the network only checks whether a validator is in the \`active\` state when checking whether it is possible to unstake the validator's stake. If the node is active, it is assumed that there is still a possibility the validator might take actions that should be penalised, and so the unstake is not allowed. You can see this check in <https://github.com/shardeum/shardeum/blob/dfdce8fb9a7a9e07a4a4f54d4bfc7ea920289cff/src/tx/staking/verifyStake.ts#L159> - which looks like:

\`\`\` } else if (shardus.isNodeActiveByPubKey(nomineeAccount.id) === true) { success = false reason = \`This node is still active in the network. You can unstake only after the node leaves the network!\` } \`\`\`

(Identical code also appears in <https://github.com/shardeum/shardeum/blob/dfdce8fb9a7a9e07a4a4f54d4bfc7ea920289cff/src/setup/validateTxnFields.ts#L456> - this latter one is used when the transaction is first received by a node, the code shown is used when the node attempts to apply the transaction to Shardeum's state)

However, the check shown is ineffective because it does not account for the fact that the node might have recently been active, and the fact that the node can take actions that are penalisable (like leaving the network) from non-active states. If an attacker can get their node to move to a non-active state, and then leave the network, then they can unstake their stake while in the non-active state, and avoid the penalty for leaving the network.

In my investigation, I found a way that the attacker can force their node back into the \`ready\` state, in order to achieve the attack. However, there may be other possible non-active states that the attacker can cause themselves to reach that would allow them to unstake. Whilst this bug is enabled by this ability to transition to other non-active states, I believe the main bug to be the reliance on the \`active\` state to denote "those who should not be allowed to unstake" and more checks should be required around this business logic.

### Exploit details

In order to achieve the attack the attacker needs to:

1. Decide a time when they want to leave the network and hence would normally be slashed
2. Just before leaving the network, have their node send a \`sync-finished\` message which puts them into the \`ready\` state
3. As soon as they detect they are in the \`ready\` state, unstake their coins
4. Shut down their node - they already have their full stake back

### Exploitability on a larger network

I have only demonstrated this on a smaller network for now. However, I believe the attack will work against larger network sizes based on these assumptions:

* The main requirement an attacker has is to get the message saying they have finished syncing processed by nodes in the same cycle (or at least to those nodes in the shard that hold's the nominator wallet account, since they process the unstake transaction)
  * Given that the attacker is running it's own validator, they can easily add mechanisms to know which nodes are required to be gossiped to, and they can tune the network messaging and timing to ensure this constraint is met
  * The network generally needs to be capable of sharing a single state update amongst all nodes in one shard within one cycle in order to be effective at confirming transactions, hence I assume this would generally be feasible

Please let me know if you would like me to do more work on tuning the PoC to larger networks.

## Impact Details

I have only demonstrated this bug in terms of avoiding the "leaving network early" slashing penalty, however, I believe a similar attack will be generalisable to other penalties, since the bug in general allows an attacker to unstake when they should not be allowed to, and if they can unstake then they can avoid penalties.

In order to exploit the bug, the attacker would need to be a validator in the network, and typically they would have modified the code of their validator node to perform the attack. This is easier to do than make a standalone script, as the code modifications needed to the validator are relatively simple. In order to make the PoC easier for the project to replicate, I have extracted logic from the validator / shardus core code base into the PoC script to deliver a standalone script.

## References

Demo video of PoC: <https://youtu.be/rkp3IG9YPPQ>

GIST with PoC code: <https://youtu.be/rkp3IG9YPPQ>

## Link to Proof of Concept

<https://youtu.be/rkp3IG9YPPQ>

## Proof of Concept

## Proof of Concept

See the demo video at <https://youtu.be/rkp3IG9YPPQ> for a direct demonstration.

The PoC is a script that can *impersonate* the attacker's validator node - it does this by using the \`bugbounty\` endpoint recommended in Shardeum's PoC GIST to fetch the keypair and other details from an arbitrary node. In this PoC I focus on avoiding the slashing that should occur when a node leaves the network early.

The PoC script replicates a fair amount of low-level Shardus networking mechanics, which means that I do not require the validator node itself to be modified, except for switching on slashing and having the \`bugbounty\` endpoint.

A real-world attacker would be far more likely to just modify the code in the validator directly, as this is a much faster way to develop the exploit and gives them easier access to useful data from the node (e.g. shard details, cycle timings). My initial version of the exploit used a maliciously modified validator, but I extracted the attack logic to a standalone PoC script in an attempt to follow the recommended pattern to make confirmation of the bug easier.

## PoC demo instructions

In order to recreate the PoC demo, you need to set up a legitimate network with slashing enabled and where at least one validator node has the \`bugbounty\` endpoint available - this will be the "attacker's" node. You also need to designate one node that can be reached to enable fetching data around the attack, the "query node" - I keep this separate from the attacker's node so we can determine what a legitimate node sees.

### Configure a network

NOTE: I assume that the developers can do this another way - what is required is a network running code that is not modified, except to add the \`bugbounty\` endpoint to at least one node (the attacker) so that we can read the keys and impersonate the attacker's node from the PoC script, and for the network configuration to have slashing enabled. For the ease of explanation / demo I have just applied this patch to all of the nodes. The other requirement is that the attacker has a wallet account on the shard that they validate which has some SHM balance.

To follow the steps I use in the demo video:

* Use Node 18.16.1 everywhere
* Patch a version of shardus-core to allow the endpoint:
  * Download a fresh copy of shardus-core repo (<https://github.com/shardeum/shardus-core>) to a folder where you modify it, e.g. \`LEGIT-shardus-core\`
    * NOTE: my patches were applied starting from commit \`23e06ded6744d8521cff9d749c1f1dd482c5fcb6\`
    * Apply the patch to this \`LEGIT\_shardus-core\` folder \`git apply shardus-core-bugbounty-endpoint.patch\`
      * This just creates the \`bugbounty\` endpoint, and should be equivalent to the project's patch in the Shardeum PoC GIST
    * Run \`npm install\` to ensure it's all up-to-date with the patches and to fetch all dependencies
* Patch a Shardeum version that will point to this version of shardus-core:
  * Download a fresh copy of shardeum repo (<https://github.com/shardeum/shardeum>) to a folder where you modify it, e.g. \`LEGIT-shardeum\`
    * Run \`npm ci\` to get the initial dependencies installed
    * NOTE: my patches were applied starting from commit \`dfdce8fb9a7a9e07a4a4f54d4bfc7ea920289cff\`
    * Apply the config to use a smaller net for testing, and to allow localhost network addresses for nodes & archivers \`git apply config-small-net-local-nodes.patch\`
    * Apply the patch to genesis.json to grant the attacker some funds \`git apply net-genesis.patch\`
    * Apply the patch to switch on the slashing parameters \`git apply use-slashing.patch\`
    * Ensure that this hacked shardeum code, uses the hacked shardus-core code rather than the legit dependency: \`git apply shardeum-package-changes.patch\`
      * **IMPORTANT**: this patch assumes that the hacked shardus-core folder can be found at path \`../LEGIT-shardus-core\` - you will need to manually edit package.json to set a different path if it's not in that location
    * \`npm install\` to get everything built & up to date

### Start up the legit nodes in a network

Devs might do this step differently, but to follow along with the demo video, in the \`LEGIT-shardeum\` folder launch a network that is large enough to meet the configured requirements

* \`npm run prepare\`
* \`shardus start instances 10\`

### Wait for network to reach an appropriate state

I wait until all of the nodes have reached active state, and wait until 15 cycles after the network mode is first reported as \`processing\` according to the monitoring logs

### Start up JSON RPC server

* Run \`npm run start\` in JSON-RPC folder

### Setup and run attack

#### Required nodes

In this attack, the "query" node is just any other vanilla node that can be relied upon to be active for the duration of the attack (I could have taken the time to make the attack only work with JSON-rpc server & archiver connectivity, but I have not, so using a query node is convenient for the PoC rather than strictly essential).

Let's arbitrarily select the 10th node, i.e. the node running at \`127.0.0.1:9010\` as the "attacker's node" for this demo, and the 2nd node, i.e. the node running at \`127.0.0.1:9002\` as the "query node".

Setup a folder for the scripts and infra for attack

* Download \`poc-sync-finished.js\`, \`stake-for-attacker.js\`, \`checkBalance.js\` and \`package.json\` from the GIST and put into an attack-scripts folder
* Ensure you use node 18.16.1 for this folder
* Run \`npm install\` to fetch all dependencies

Ensure network is in a good state before proceeding, and then:

* Run \`node checkBalance.js\` to see the wallet account initial balance - it should be \`10000000.0 SHM\` if you have the same genesis.json as me
* Run \`node stake-for-attacker.js 127.0.0.1:9010\` (or replacing the arg with the attacker node address & port)
  * At the end you should see the new wallet balance is \`9999989.99 SHM\` - original balance minus \`10 SHM\` stake, minus \`0.01 SHM\` fee
* Potentially wait some cycles to ensure staking has been processed
* Run attack PoC - \`node poc-sync-finished.js 127.0.0.1:9010 127.0.0.1:9002\` - or replacing the attacker & query addresses and ports as required
  * This impersonates the attacker's node (normally you would just modify the validator code directly to do this, I've kept it in a PoC script for ease of reproduction) and gossips a message saying that they've finished syncing to the other nodes
    * Because the nodes will only accept this kind of gossip within Q1 of a network cycle, the attacker sends an equivalent message every 15s to try to guarantee hitting the correct quarter at least once
    * **IMPORTANT** - if your network's cycle duration differs from 60 seconds, you will need to change the \`delaySecs\` constant in the script to match the approximate quarter-cycle duration (15 seconds for a cycle of 60 seconds)
  * When the nodes accept this message they will mark the attacker's node as "finished syncing" and update it for the next cycle
  * When the scripts detects that the attacker's node has moved from \`active\` state to \`ready\` state, it immediately unstakes
* If this script succeeds, you should see the line \`Injection successful\` at the end of the script output (if not, it is probably a timing issue and you should just retry it)
* Now you can immediately kill the attacker's node - e.g. by running \`kill PID\`
  * It is this action that should cause the stake to be slashed after a few cycles
* Wait a minute or so for the transaction to process and then check the attacker's wallet balance again \`node checkBalance.js\`
  * You should see that the attacker's wallet balance is now \`9999999.98 SHM\`, i.e. only \`0.02 SHM\` less than before they staked - they have withdrawn their stake without any slashing (but have paid 2 lots of \`0.01 SHM\` in fees)
* Wait a few cycles for the other nodes to notice the attacker's node is down and apply a penalty and then run \`node checkBalance.js\`
  * No penalty has been applied, even though the node has left the network - the attacker's wallet should be \`9999997.98 SHM\` due to slashing, but they've avoided the slashing!

## PoC deep dive

The main things the attacker needs to achieve are:

* Have the other nodes accept a message saying the attacker's node has finished syncing
* **AND** withdraw their stake while their node is in a non-active state (\`ready\` in this case)

The only major difficulty in achieving these steps is timing:

* The way the gossip handling works, the other nodes will not accept the gossip message unless the message is received during the first quarter (Q1) of a cycle
  * In order to work around this, a maliciously modified validator could just schedule the message send based on internal mechanics where it knows which quarter it is (e.g. the processing in CycleCreator)
  * However, to keep the PoC standalone, I just keep repeating the message every 15s (roughly the length of a quarter cycle) until it is accepted or until I have tried it across multiple cycles
* The way the targetted node state works, the node will move from \`ready\` to \`active\` after the message has been received correctly (at the end of the cycle where the message is received), and then will automatically move back to \`active\` one cycle later
  * Hence the attacker only has one cycle to get their unstake transaction accepted
  * For a maliciously modified validator, it would be relatively straightforward to inject the transaction immediately when the appropriate cycle begins
  * For the PoC script, I just do it immediately when I can see that the node has gone to a state that is not \`active\`

### PoC limitations

The PoC just gossips to all nodes in the network that are not itself, however this is inefficient and would not scale well to a bigger network. In order to improve this, I would need to port some more of the shardus-core code into the PoC script to select an appropriate subset of nodes to start the gossip off on (who then gossip to others on my behalf). I can suggest some changes to achieve this if you require it.

Additionally, I could make the script more robust to different environments. For example, I could make it query the duration of the last cycle, rather than assuming it is 60 seconds.
