# 34019 - \[BC - Critical] Lack of vote validation in sync\_trie\_hashes lea...

## Lack of vote validation in sync\_trie\_hashes leads to total fund loss and tokenomics crash

Submitted on Aug 4th 2024 at 09:46:29 UTC by @neplox for [Boost | Shardeum: Core](https://immunefi.com/bounty/shardeum-core-boost/)

Report ID: #34019

Report type: Blockchain/DLT

Report severity: Critical

Target: <https://github.com/shardeum/shardus-core/tree/dev>

Impacts:

* Direct loss of funds

### Description

### Brief/Intro

Shardeum validator nodes utilizing the `shardus-core` package (<https://github.com/shardeum/shardus-core>) are vulnerable to arbitrary account data manipulation in the network through the sharded account hash trie consensus mechanism implemented in `src/state-manager/AccountPatcher.ts` due to insufficient validation of voting data received in the binary and non-binary `sync_trie_hashes` endpoints. Notably, this vulnerability allows attackers to set the balance of any account in the network to any value, either leading to the complete loss of funds for all accounts in the network, exploitation of the Proof-of-Stake (PoS) consensus mechanism, or possibly resulting in a complete crash of the tokenomics of Shardeum. Exploitation of this vulnerability requires control over only two active validator nodes without any additional requirements. This report showcases exploitation of this vulnerability in order to manipulate the balance of any account in the network, setting it to any desired value.

### Vulnerability Details

Shardeum nodes expose the `sync_trie_hashes` and `binary/sync_trie_hashes` endpoints through the `shardus-core` package as part of the `AccountPatcher` module (<https://github.com/shardeum/shardus-core/blob/4d75f797a9d67af7a94dec8860220c4e0f9ade3c/src/state-manager/AccountPatcher.ts#L696>). These are internal protocol endpoints which are validly used by Shardeum validator nodes themselves in order to share the state of the account hash trie across the network, done in the `broadcastSyncHashes` method (<https://github.com/shardeum/shardus-core/blob/4d75f797a9d67af7a94dec8860220c4e0f9ade3c/src/state-manager/AccountPatcher.ts#L3085>), which is itself called during the `processPreviousCycleSummaries` procedure of `StateManager`'s `cycle_q1_start` event handler.

These endpoints, however, do not check that the message sender has already performed a vote for a cycle or for a specific radix of the hash trie in the context of a cycle, allowing nodes to send multiple votes for the same cycle or radix, breaking the consensus mechanism. Furthermore, despite the account hash trie being sharded just like all other state in Shardeum, the `sync_trie_hashes` endpoint does not check that the message sender is part of the consensus group which is responsible for the receiving node's shard.

By having malicious nodes send multiple votes for the same cycle and radix, they can over-vote the genuine validator nodes of the network, and force an invalid hash to be counted as the correct state, since no validation is done in the vote counting process as well:

```js
if (voteCount > hashVote.bestVotes) {
  hashVote.bestVotes = voteCount;
  hashVote.bestHash = nodeHashes.hash;
}
```

Also part of the same `StateManager` `cycle_q1_start` event handling procedure, `AccountPatcher.testAndPatchAccounts` will mark the local hash trie as incorrect through the `isInSync` and `findBadAccounts` methods, which use the votes gathered previously during the `sync_trie_hashes` endpoint calls. These accounts will then be retrieved from the nodes which voted for the hashes with the highest number of votes in the `getAccountRepairData` method, which requires the winning hash to be voted for by at least **2** nodes (<https://github.com/shardeum/shardus-core/blob/4d75f797a9d67af7a94dec8860220c4e0f9ade3c/src/state-manager/AccountPatcher.ts#L3929>). As specified in the comment, however, this is only a minor layer of security, since getting 2 nodes to join the network is pretty easily achievable with any network size, especially considering the financial gain this would bring to the attacker. If the accounts are successfully retrieved from the nodes which voted for the winning hash, they are used to directly update the local account state by calling `stateManager.checkAndSetAccountData` (<https://github.com/shardeum/shardus-core/blob/4d75f797a9d67af7a94dec8860220c4e0f9ade3c/src/state-manager/AccountPatcher.ts#L3719>).

Due to how this consensus mechanism is designed and implemented, exploitation would also be pretty much impossible to detect and rollback, since it happens outside of any normal transaction processing logic, directly affecting the account data stored for an address. After one node being exploited, the manipulated account data will be transferred by it to other nodes in the network, too, since the "infected" node will gossip the updated hash to other appropriate nodes.

### Impact Details

As shown in the POC, exploitation of this vulnerability is trivial once the attacker has control over two active validator nodes. Combined with the economic incentive to exploit this vulnerability, since it directly allows the attacker to set an arbitrarily large balance on their own account, the only requirement of having control over two active nodes is feasible.

In the most basic scenario, an attacker who has exploited the vulnerability for their own gain, could then drain any pool and bridge trading SHM coins, which would be fatal for the tokenomics of Shardeum when repeated multiple times. With a less self-centered goal, an attacker might exploit this vulnerability to zero-out the balance of all accounts in the network, halting it as a result. I have not tested this scenario, but it might also be possible to overwrite the account data of global accounts, handled by the `GlobalAccounts` module, which would allow the attacker to globally modify the validator node and Shardeum network configuration.

### References

* `sync_trie_hashes` method implementation: <https://github.com/shardeum/shardus-core/blob/4d75f797a9d67af7a94dec8860220c4e0f9ade3c/src/state-manager/AccountPatcher.ts#L696>
* `StateManager.checkAndSetAccountData` method implementation: <https://github.com/shardeum/shardus-core/blob/4d75f797a9d67af7a94dec8860220c4e0f9ade3c/src/state-manager/index.ts#L1114>

## Proof of Concept

This POC demonstrates the voting vulnerability in the `AccountPatcher`'s `sync_trie_hashes` endpoint by setting a target account's balance to an arbitrarily large value, which would practically allow an attacker take control of the whole tokenomics model in a real network, and take over the Shardeum network itself by having the ability to launch as many validator nodes with staking as they want.

`fund-manipulation.js` contained in the attached gist (<https://gist.github.com/renbou/6d00c42198b7e542703b8411e6e29039>) contains the exploit code utilizing two attacker endpoints of the two malicious validator nodes needed to perform the attack. The malicious endpoints themselves are implemented using a modified `shardus-core` package, the diff for which is provided in the attached gist as well.

Please note that the POC relies on having access to an attacker-controlled account with some SHM, which is used to perform initial initialization of the chosen target account in the case it does not exist in the Shardeum network yet. External endpoint URLs of the two attacker-controlled malicious validator nodes are needed as well. Since only two malicious nodes are needed for the attack, and the number of nodes required does not depend on the total number of active validator nodes in the network, this attack is feasible in any real network scenario.

### Local Shardeum network setup

This vulnerability is equally exploitable with any number of nodes. For demonstration purposes, however, a network with only 32 validator nodes is created. Any Shardeum network using the current validator node code will be vulnerable, so it is not necessary to follow these exact steps.

1. Clone the Shardeum repo and switch to the last commit on the `dev` branch, which is `c7b10c2370028f7c7cbd2a01839e50eb50faa904` as of this POC's submission.

   ```bash
   git clone https://github.com/shardeum/shardeum.git
   cd shardeum
   git switch --detach c7b10c2370028f7c7cbd2a01839e50eb50faa904
   ```
2. Switch to NodeJS 18.16.1, which is the version used by Shardeum in `dev.Dockerfile` and its various library requirements. For example, using asdf (<https://asdf-vm.com/>):

   ```bash
   asdf install nodejs 18.16.1
   asdf local nodejs 18.16.1
   ```
3. Apply the `shardeum-32.patch` file from the attached gist (<https://gist.github.com/renbou/6d00c42198b7e542703b8411e6e29039>) for network setup. It adds 100 SHM to the attacker's account through `genesis.json` and sets `minActiveNodesForStaking` to 32 for easier launch without the need of staking. Lack of staking does not affect the POC in any way, it just allows nodes to start up without having a stake, which is needed when launching some nodes with a slight delay. Exploitation requires the attacker to have only 2 nodes in the whole network anyway, so no unrealistic assumptions are made by the POC in this regard. Note that it DOES NOT enable debug mode, demonstrating the vulnerability in a semi-realistic release setup.

   ```bash
   git apply shardeum-32.patch
   ```
4. Install dependencies and build the project.

   ```bash
   npm ci
   npm run prepare
   ```
5. Launch the network with **30** nodes (**NOT 32**) using the `shardus` tool. The remaining 2 nodes will be the attacker's nodes, and will be launched separately with a modified malicious Shardeum validator node.

   ```bash
   npx shardus create 30
   ```

Once this is completed, it isn't necessary to wait for the network to start, since the 2 malicious nodes need to start up as well for the network to go into `processing` mode. After completing the *next* section, however, you should wait for the network to go fully active, so that the POC can properly demonstrate the impact.

### Malicious nodes setup

This part showcases how an attacker could setup their own two validator nodes, which are needed for running this POC. These malicious nodes do not rely on any Shardeum network requirements, and represent the same Shardeum nodes, but with modified `AccountPatcher` logic in the `shardus-core` part of the node, which allows them to "double-vote" on the `sync_trie_hashes` endpoint of other active validator nodes with modified network account information, and, as such, modified hashes.

For the POC to work as expected, the malicious nodes must be configured with specific `config.json` files and use the modified malicious `shardus-core` package. All patches necessary are provided in the attached gist, and detailed setup instructions are provided below. **NOTE** that **BOTH** malicious validator nodes are needed, since `AccountPatcher`'s `getAccountRepairData` requires at least two nodes to respond with the same hashes, which is utilized by us in order to manipulate the balance of an account by having two malicious nodes send a bunch of votes for the modified hash.

1. Repeat steps 1-3 of the "Local Shardeum network setup" section, but in two extra directories, lets say `malicious-1` and `malicious-2`. The directories should pretty much resemble just two more validator nodes with the same configuration at this point.
2. Apply the `malicious-node-1.patch` and `malicious-node-2.patch` files from the attached gist (<https://gist.github.com/renbou/6d00c42198b7e542703b8411e6e29039>) in each of the malicious nodes' directories, respectively. They will configure the nodes' external ports, which are needed to be known in the POC for it to correctly interact with the malicious shardus-core endpoints of the attacker nodes.

   ```bash
   # In the malicious-1 directory
   git apply malicious-node-1.patch
   ...
   # In the malicious-2 directory
   git apply malicious-node-2.patch
   ```
3. Install dependencies and build the malicious validator nodes projects. This should be done after applying the `shardeum-32.patch` and `malicious-node-{n}.patch` files, **BUT BEFORE** linking the `malicious-shardus-core` package which is set up in the next steps, since otherwise `npm ci` will overwrite the attacker's node's link to the malicious modified `shardus-core` package. Both the `malicious-1` and `malicious-2` directories need to be built and prepared for next steps.

   ```bash
   npm ci
   npm run prepare
   ```
4. Clone the base `shardus-core` repository and switch to the last commit on the `dev` branch, which is `4d75f797a9d67af7a94dec8860220c4e0f9ade3c` as of this POC's submission. `shardus-core` needs to be modified for the malicious nodes in this POC, in order to allow direct account modification in them and multiple `sync_trie_hashes` voting requests on behalf of one node. Switch to NodeJS 18.16.1 in the `malicious-shardus-core` package like it was done in the validator node setup.

   ```bash
   git clone https://github.com/shardeum/shardus-core.git malicious-shardus-core
   cd malicious-shardus-core
   git switch --detach 4d75f797a9d67af7a94dec8860220c4e0f9ade3c
   asdf local nodejs 18.16.1
   ```
5. Apply the `malicious-core.patch` file from the attached gist (<https://gist.github.com/renbou/6d00c42198b7e542703b8411e6e29039>) to the `malicious-shardus-core` repository clone. This patch adds the `attacker-multiple-votes` and `attacker-set-balance` methods, which allow setting the number of `sync_trie_hashes` votes duplications, and modifying the balance of an arbitrary account, respectively. They will be used by the POC in the malicious nodes in order to manipulate the target account's balance from 0 to 1\_337\_000 SHM.

   ```bash
   # In the malicious-shardus-core directory
   git apply malicious-core.patch
   ```
6. Build the `malicious-shardus-core` and link it in `malicious-1` and `malicious-2` directories instead of the original `@shardus/core` dependency.

   ```bash
   # In the malicious-shardus-core directory
   npm ci
   npm run prepare
   cd ../malicious-1
   ...
   # In the malicious-1 directory
   npm link ../malicious-shardus-core
   cd ../malicious-2
   ...
   # In the malicious-2 directory
   npm link ../malicious-shardus-core
   ```
7. Now that the nodes have been built, and linked to the malicious `shardus-core`, manually start the malicious nodes. They should show up on the <http://localhost:3000/> monitor, and start joining in a few cycles. No stake should be needed, as `minActiveNodesForStaking` has been set to 32 and only 30 nodes have been launched at this point.

   ```bash
   # In both malicious-1 and malicious-2 directories
   node ./dist/src/index.js
   ```

At this point it's best to wait for the network consisting of 30 safe validator nodes and 2 malicious attacker nodes to go fully active, for example, by using the <http://localhost:3000> monitor. Waiting for the network to be active will allow the POC to showcase that a completely valid production-like network is exploitable using the vulnerabilities described in this report.

### JSON-RPC API setup

To simplify the POC, Shardeum's `json-rpc-server` is used to interact with the network, specifically, to send small transfer transactions using the attacker's account. This is needed to initialize the specified target account when it does not exist in the network yet.

1. Clone the `json-rpc-server` repo and switch to the last commit on the `dev` branch, which is `c3c462a4b18bc7517e086ff70f08ae6afede3b31` as of this POC's submission.

   ```bash
   git clone https://github.com/shardeum/json-rpc-server.git
   cd json-rpc-server
   git switch --detach c3c462a4b18bc7517e086ff70f08ae6afede3b31
   ```
2. Switch to NodeJS 18.16.1, which is the version used by `json-rpc-server` in `Dockerfile` and its various library requirements. For example, using asdf (<https://asdf-vm.com/>):

   ```bash
   asdf install nodejs 18.16.1
   asdf local nodejs 18.16.1
   ```
3. Install dependencies.

   ```bash
   npm install
   ```
4. Launch the JSON RPC server. This must be done once the Shardeum network is at least partially active, for the server to receive and be able to interact with valid archiver and validator nodes.

   ```bash
   npm run start
   ```

### Account balance manipulation using sync\_trie\_hashes voting exploitation

As said in the introduction, `fund-manipulation.js` from the attached gist (<https://gist.github.com/renbou/6d00c42198b7e542703b8411e6e29039>) contains the exploit code and can be configured using the various parameters at the top of the script. It can be ran using `NodeJS`, and requires only the `ether` and `axios` libraries to be installed, as specified in the attached `package.json` (<https://gist.github.com/renbou/6d00c42198b7e542703b8411e6e29039>).

The `main` function of the script contains the POC's logic, with preceding functions declared just to simplify the main POC. Following is a detailed writeup of how it works:

1. The POC checks that both the attacker's nodes are in the network and are active, which is needed for the exploit to work, since `sync_trie_hashes` endpoints, just like any other internal protocol endpoint, will accept requests only from active validator nodes.
2. The POC retrieves the initial balance of the `TARGET_ADDRESS` account, which is expected to be consensed by the network at this point. If such account does not exist in the network (`/account/{TARGET_ADDRESS}` returns `null`), the POC will transfer a small sum of SHM from the attacker's genesis account to it in order to initialize it. At this point all initial checks are done.
3. The `/attacker-multiple-votes` endpoint implemented in the malicious `shardus-core` modification is called on the two attacker nodes, telling each of them to send as many votes as there are active nodes in the network. This means that even a single malicious node will outweight all the other nodes. However, because `AccountPatcher` later retrieves "correct" account data from a second node which voted for the same hash (in the `getAccountRepairData` method it calls `getNodeForQuery` with `nextNode = true`), two malicious nodes are needed here.
4. Having done all necessary preparations, the POC calls the malicious `/attacker-set-balance` endpoint, making sure to set the same latest timestamp of the account for both attacker's validator nodes, needed for their hashes to match up. At this point the malicious nodes will already have the updated accounts, since this endpoint directly calls the `stateManager.checkAndSetAccountData` method. On this or the next cycle, the malicious nodes will update their `AccountPatcher` trie, and begin distributing large amounts of votes for the manipulated trie hashes.
5. The POC just waits for the Shardeum network nodes to reach consensus on the new balance of the target account.

To manually verify the POC, a basic curl request to the /account endpoint of any node can be made BEFORE and AFTER running the POC:

```bash
curl 'http://localhost:9001/account/0x0000000000000000000000000000000000001337'
{"account":null}
```

After running the POC, and waiting for it to confirm that the exploit has worked, the same request should return the target account with its balanced increased by `1_337_000 SHM`. Since the genesis balance of the attacker-controlled account specified in the POC is only `100 SHM`, there is no way this amount of coins could have been transferred to the target account by any other means. The request can be repeated on all the nodes (which is what is done in the POC as well), to validate that all validator nodes have reached consensus on the new balance of the target account.

Since the POC does not use any functionality such as developer keys, and does not rely on certain configuration parameters being enabled, it is enough to showcase the vulnerability and how it can be used to exploit the sharded account hash trie consensus mechanism to manipulate account data in the network.

out.log attached to the gist (<https://gist.github.com/renbou/6d00c42198b7e542703b8411e6e29039>) contains an example of the output of the POC's execution.
