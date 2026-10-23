# 34456 - \[BC - Critical] Lack of consensus validation in repair\_oos\_acco...

## Lack of consensus validation in repair\_oos\_accounts leads to total fund loss and tokenomics crash

Submitted on Aug 13th 2024 at 10:56:29 UTC by @neplox for [Boost | Shardeum: Core](https://immunefi.com/bounty/shardeum-core-boost/)

Report ID: #34456

Report type: Blockchain/DLT

Report severity: Critical

Target: <https://github.com/shardeum/shardus-core/tree/dev>

Impacts:

* Direct loss of funds
* Permanent freezing of funds (fix requires hardfork)

### Description

### Brief/Intro

Shardeum validator nodes utilizing the `shardus-core` package (<https://github.com/shardeum/shardus-core>) are vulnerable to arbitrary account data manipulation in the network through the sharded account hash trie consensus mechanism implemented in `src/state-manager/AccountPatcher.ts` due to insufficient validation of voting data received in the binary and non-binary `repair_oos_accounts` endpoints. Notably, this vulnerability allows attackers to set the balance of any account in the network to any value, either leading to the complete loss of funds for all accounts in the network, exploitation of the Proof-of-Stake (PoS) consensus mechanism, or possibly resulting in a complete crash of the tokenomics of Shardeum.

### Vulnerability Details

Shardeum nodes expose the `repair_oos_accounts` and `binary/repair_oos_accounts` endpoints through the `shardus-core` package as part of the `AccountPatcher` module (<https://github.com/shardeum/shardus-core/blob/72fba67d3a551f21368c8b0fe94f951c8f5cc4f8/src/state-manager/AccountPatcher.ts#L337>). These are internal protocol endpoints that are legitimately used by Shardeum validator nodes to restore broken accounts as part of the transaction and account consensus mechanisms.

However, the receipt for the vote (of type `AppliedReceipt2`) along with "repaired" account information, is simply passed as an argument in the payload of the function and its signatures are not verified, making it possible to forge and send a request to "restore" any account to any value, breaking the intended `AccountPatcher` trie hash consensus mechanism.

Pretty much the only validation which occurs is that of the attached `txId`, which is meant to be the ID of the transaction which caused the account to become out-of-sync in a node:

```ts
const archivedQueueEntry =
  this.stateManager.transactionQueue.getQueueEntryArchived(
    txId,
    "repair_oos_accounts"
  );

if (archivedQueueEntry == null) {
  nestedCountersInstance.countEvent(
    "accountPatcher",
    `repair_oos_accounts: no archivedQueueEntry for txId: ${txId}`
  );
  this.mainLogger.debug(
    `repair_oos_accounts: no archivedQueueEntry for txId: ${txId}`
  );
  continue;
}

// check the vote and confirmation status of the tx
const bestMessage = receipt2.confirmOrChallenge
const receivedBestVote = receipt2.appliedVote

if (receivedBestVote != null) {
  // Check if vote is from eligible list of voters for this TX
  if(this.stateManager.transactionQueue.useNewPOQ && !archivedQueueEntry.eligibleNodeIdsToVote.has(receivedBestVote.node_id)) {
    nestedCountersInstance.countEvent('accountPatcher', `repair_oos_accounts: vote from ineligible node for txId: ${txId}`)
    continue
  }

...
```

This means that everything an attacker would need is a valid txId of any transaction in the network that has been moved to the archive of the validator node, which is something that is normally done for pretty much all Shardeum transactions, both internal and normal ones. Since, as mentioned previously, the receipt is part of the request payload and is not verified to be consensed on by the validator nodes properly, there are no additional requirements for the attacker's `node_id` and anything else.

Once the request passes these basic checks, the modified account data is used to directly overwrite the present account data for the specified address using the `stateManager.checkAndSetAccountData` method (<https://github.com/shardeum/shardus-core/blob/72fba67d3a551f21368c8b0fe94f951c8f5cc4f8/src/state-manager/AccountPatcher.ts#L467>).

Due to how this consensus mechanism is designed and implemented, exploitation would also be pretty much impossible to detect and rollback, since it happens outside of any normal transaction processing logic, directly affecting the account data stored for an address. Despite only one node being affected by a single `repair_oos_accounts` request at a time, since there are no node-specific pre-conditions for the request, it is possible to overwrite the account data of any address in the network on all validator nodes at once, effectively changing the state of the account in the whole network.

### Impact Details

As shown in the POC, exploitation of this vulnerability is trivial once the attacker has control over an active validator node, which is needed because the `repair_oos_accounts` endpoint is served over the internal network protocol. Combined with the economic incentive to exploit this vulnerability, since it directly allows the attacker to set an arbitrarily large balance on their own account, this vulnerability becomes a very serious one.

In the most basic scenario, an attacker who has exploited the vulnerability for their own gain, could then drain any pool and bridge trading SHM coins, which would be fatal for the tokenomics of Shardeum when repeated multiple times. With a less self-centered goal, an attacker might exploit this vulnerability to zero-out the balance of all accounts in the network, halting it as a result. I have not tested this scenario, but it might also be possible to overwrite the account data of global accounts, handled by the `GlobalAccounts` module, which would allow the attacker to globally modify the validator node and Shardeum network configuration.

### References

1. `repair_oos_accounts` endpoint - <https://github.com/shardeum/shardus-core/blob/72fba67d3a551f21368c8b0fe94f951c8f5cc4f8/src/state-manager/AccountPatcher.ts#L337>
2. `StateManager.checkAndSetAccountData` method implementation: <https://github.com/shardeum/shardus-core/blob/72fba67d3a551f21368c8b0fe94f951c8f5cc4f8/src/state-manager/index.ts#L1114>

## Proof of concept

This PoC (poc.js in gist <https://gist.github.com/renbou/07a7a411f7046f6c6b874801672dc0b3>) demonstrates the vulnerability in AccountPatcher's `repair_oos_accounts` endpoint by setting a target account's balance to an arbitrarily large value, which would practically allow an attacker take control of the whole tokenomics model in a real network, and take over the Shardeum network itself by having the ability to launch as many validator nodes with staking as they want.

### Local Shardeum network setup

This vulnerability is equally exploitable with any number of nodes as exploitation of a single validator node requires sending just a single message via the internal protocol once a valid txId is acquired. For demonstration purposes, however, a network with only 32 validator nodes is created.

Any Shardeum network using the current validator node code will be vulnerable, so it is not necessary to follow these exact steps. They are here just to showcase how the POC was tested by me.

1. Clone the Shardeum repo and switch to the last commit on the `dev` branch, which is `d7dddf01002846b77f83ebea3557e949d8c9c90f` as of this POC's submission.

   ```bash
   git clone https://github.com/shardeum/shardeum.git
   cd shardeum
   git switch --detach d7dddf01002846b77f83ebea3557e949d8c9c90f
   ```
2. Switch to NodeJS 18.16.1, which is the version used by Shardeum in `dev.Dockerfile` and its various library requirements. For example, using asdf (<https://asdf-vm.com/>):

   ```bash
   asdf install nodejs 18.16.1
   asdf local nodejs 18.16.1
   ```
3. Apply the `shardeum-32.patch` file from the attached [gist](https://gist.github.com/renbou/07a7a411f7046f6c6b874801672dc0b3) for network setup. It additionally adds some genesis coins to the attacker account used in the PoC in order to create transactions, needed to retrieve a txId for exploitation. Note that it DOES NOT enable debug mode, demonstrating the vulnerability in a semi-realistic release setup.

   ```bash
   git apply shardeum-32.patch
   ```
4. Install dependencies and build the project.

   ```bash
   npm ci
   npm run prepare
   ```
5. Launch the network with 32 nodes as specified in the patch using the `shardus` tool.

   ```bash
   npx shardus create 32
   ```

After this step, 15-20 minutes are required as usual for the network to become active, which is needed for the exploit to be able to create transactions. I used the <http://localhost:3000/> monitor to wait for nodes to become fully active.

### JSON-RPC API setup

To simplify the POC, Shardeum's `json-rpc-server` is used to interact with the network, specifically, to send a single stake transaction using the attacker's account, to use its ID later with the `repair_oos_accounts` endpoint.

1. Clone the `json-rpc-server` repo and switch to the last commit on the `dev` branch, which is `d799a64c1ab4a7cffdf472a8be689fe7afb993e9` as of this POC's submission.

   ```bash
   git clone https://github.com/shardeum/json-rpc-server.git
   cd json-rpc-server
   git switch --detach d799a64c1ab4a7cffdf472a8be689fe7afb993e9
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

### Account balance manipulation using repair\_oos\_aaccounts voting exploitation

As said in the introduction, `poc.js` from the attached [gist](https://gist.github.com/renbou/07a7a411f7046f6c6b874801672dc0b3) contains the exploit code and can be ran on any network as long as a valid archiver URL and Attacker validator node info is suplied. It can be ran using `NodeJS`, requires a few of the Shardus packages (`@shardus/crypto-utils`, `@shardus/net`, `@shardus/types`) as well as `ether` and `axios` libraries to be installed, as specified in the attached [`package.json`](https://gist.github.com/renbou/07a7a411f7046f6c6b874801672dc0b3). Following is a detailed writeup of how it works:

**NOTE**: modify the `keypair` and `NODE_ID` variables with the values of one of the nodes in the network, for the POC to simulate requests from an attacker-controlled validator node. The keypair should be set to the contents of a node's secrets.json, and NODE\_ID can be set to the value of id retrieved from a node's /nodeinfo endpoint.

1. A request to the specified archiver's `full-nodelist` endpoint is made in order to retrive the full list of validator active nodes in the network.
2. Each node's `/nodeinfo` endpoint is queried to retrieve the `internalIp`, `internalPort` and `id` values.
3. Using getTxId, we create a new transaction (it doesn't matter if it succeeds or not). This is done to obtain a txId that will be used in the future.
4. A message with the malicious account data is sent using the internal protocol to the `repair_oos_accounts` handler triggering the vulnerability.
5. A balance request is made via JSON-RPC, which will display our increased balance in console.

To manually verify the POC, a basic curl request to the /account endpoint of any node can be made BEFORE and AFTER running the POC:

```bash
curl 'http://localhost:9001/account/0xe6e789891Aad9E4ea1e0E37214Bd7067598BAdEc'
{"account":{"balance":"100000000000000000000","codeHash":"0xc5d2460186f7233c927e7db2dcc703c0e500b653ca82273b7bfad8045d85a470","nonce":"0","operatorAccountInfo":null,"storageRoot":"0x56e81f171bcc55a6ff8345e692c0f86e5b48e01b996cadc001622fb5e363b421"}}
```

After running the POC, and waiting for it to confirm that the exploit has worked, the same request should return the attacker's account with its balance set to `9999999999999999999999999.999999999999999999 SHM`. Since the genesis balance of the attacker-controlled account specified in the POC is only `100 SHM`, there is no way this amount of coins could have ended up in the attacker's account by any other means. The request can be repeated on all the nodes (which is what is done in the POC as well), to validate that all validator nodes have reached consensus on the new balance of the attacker's account.

Since the POC does not use any functionality such as developer keys, and does not rely on certain configuration parameters being enabled, it is enough to showcase the vulnerability and how it can be used to exploit the sharded account hash trie consensus mechanism to manipulate account data in the network.
