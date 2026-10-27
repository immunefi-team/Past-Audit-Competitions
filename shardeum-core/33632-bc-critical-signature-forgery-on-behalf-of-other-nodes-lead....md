# 33632 - \[BC - Critical] Signature forgery on behalf of other nodes lead...

## Signature forgery on behalf of other nodes leads to network takeover

Submitted on Jul 25th 2024 at 06:03:45 UTC by @neplox for [Boost | Shardeum: Core](https://immunefi.com/bounty/shardeum-core-boost/)

Report ID: #33632

Report type: Blockchain/DLT

Report severity: Critical

Target: <https://github.com/shardeum/shardeum/tree/dev>

Impacts:

* Network not being able to confirm new transactions (total network shutdown)
* Direct loss of funds

### Description

### Brief/Intro

Shardeum validator nodes implemented in the <https://github.com/shardeum/shardeum> allow signing near-arbitrary data via the `signAppData` method exposed using the `sign-app-data` internal endpoint implemented in shardus-core. This allows any validator node part of the currently active nodes to send signed data and requests on behalf of other nodes, which can be exploited to gain control of all the consensus protocols implemented in the Shardeum network (transactions, gossip, etc). This report showcases exploitation of this vulnerability in order to run internal transactions of the type `InternalTXType.ApplyChangeConfig`, which change the validator nodes configuration and can be used to DOS the nodes and gain complete control over the node parameters.

### Vulnerability Details

Shardeum nodes expose a `sign-app-data` internal protocol route through `shardus-core` (<https://github.com/shardeum/shardus-core/blob/4d75f797a9d67af7a94dec8860220c4e0f9ade3c/src/shardus/index.ts#L2762>), which calls the app's (in this case Shardeum validator nodes being the app) `signAppData` method (<https://github.com/shardeum/shardeum/blob/c7b10c2370028f7c7cbd2a01839e50eb50faa904/src/index.ts#L5822>) to actually handle the request. `signAppData` accepts signing requests of types `sign-stake-cert` and `sign-remove-node-cert` on behalf of the validator node, both of which are part of the validator node staking mechanism and allow join requests to have signed proofs of stake from multiple nodes.

Lack of input validation in `signAppData`, however, enables signature generation for practically arbitrary objects, since the method validates the `appData` field of the request by checking for required fields, namely `nominator`, `nominee`, `stake`, `certExp`, but does misses checks for the lack of other unexpected fields. For example, passing an object which has additional fields named `value`, `when`, and `source`, as long as the required fields are present, will cause the validator node to sign the whole object on its behalf and return the signature to the requestor:

```js
{
  nominator: wallet.address,
  nominee: ATTACKER_NODE_SECRETS.publicKey,
  stake: stakeLock,
  certExp: Date.now() + 24 * 60 * 60 * 1000,

  // Injected SetGlobalTx fields, not validated by signAppData.
  value: {
    isInternalTx: true,
    internalTXType: 4, // InternalTXType.ApplyChangeConfig
    change: {
      cycle: 1,
      change: {
        mode: "debug",
      },
    },
  },
  when: Date.now(),
  source: "1000000000000000000000000000000000000000000000000000000000000001",
}
```

Since the only requirement for `signAppData` to correctly execute, is to have just **one** validator node with staked coins in the whole network, this makes exploitation trivial for any active participating validator node. Because all internal node communications use signatures to verify the authenticity of the sender, this vulnerability makes it possible to send any internal requests and gossips on behalf of all nodes in the network from a single validator node, completely breaking any consensus mechanisms.

### Impact Details

Practically all internal Shardeum network communications are affected, since little to none endpoints validate that no unexpected fields (`nominator`, `nominee`, `stake`, `certExp` when exploiting the vulnerability using the `sign-stake-cert` appData type) are present. `TransactionConsensus` voting mechanisms are affected, `Lost` node verification mechanisms are affected, `GlobalAccounts` transaction handling is affected, and so on. Not only is it possible to take over the internal communications and consensus mechanisms of the network, but, since `GlobalAccounts` transactions, such as `ApplyChangeConfig`, `ApplyNetworkParam`, can be executed without proper consensus, the attacker can modify the validator node configuration and network parameters, changing the `mode` to `debug` or removing present `devPublicKeys`, for example. Debug mode handlers can then be used to DOS the nodes through some of the available debug handlers, and perhaps escalate the impact on specific validator nodes even further.

### References

* `signAppData` method implementation: <https://github.com/shardeum/shardeum/blob/c7b10c2370028f7c7cbd2a01839e50eb50faa904/src/index.ts#L5822>
* `'set-global'` gossip implementation: <https://github.com/shardeum/shardus-core/blob/4d75f797a9d67af7a94dec8860220c4e0f9ade3c/src/p2p/GlobalAccounts.ts#L75>

## Proof of Concept

This POC demonstrates the vulnerability by manipulating Shardeum node configuration, bypassing the `GlobalAccounts` transaction consensus mechanism and executing transactions via the `set-global` gossip route using signatures retrieved from all network nodes via `sign-app-data` calls with the `sign-stake-cert` type.

`modify-config-poc.js` contained in the attached gist (<https://gist.github.com/renbou/775c1555f45c2154b202a289530eeb0b>) contains the main exploit code to automatically perform a stake for the attacker's node, wait for it to become active, and then send internal protocol requests on behalf of it, specifically `sign-app-data` and `set-global`. Overall, the POC code can be adapted to test other internal endpoints which also require validator node signatures.

Please note that the POC code relies on having a separate attacker validator node with the same secrets and contained in the code, as well as enough balance for a stake, which is done using the `genesis.json` configuration of Shardeum (`src/config/genesis.json`). All of this is taken into account and is contained in the detailed steps below.

### Local Shardeum network setup

This vulnerability is equally exploitable with any number of nodes. For demonstration purposes, however, a network with only 32 validator nodes is created.

Any Shardeum network using the current validator node code will be vulnerable, so it is not necessary to follow these exact steps. It is necessary, however, to provide enough coins for a stake to the POC's attacker account, `0xe6e789891Aad9E4ea1e0E37214Bd7067598BAdEc`, for example using the `src/config/genesis.json` configuration file.

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
3. Apply the `network-32.patch` file from the attached gist (<https://gist.github.com/renbou/775c1555f45c2154b202a289530eeb0b>) for network setup. It also configures the `genesis.json` file to deposit 100 SHM coins to the POC's attacker account. Note that it DOES NOT enable debug mode, demonstrating the vulnerability in a semi-realistic release setup.

   ```bash
   git apply network-32.patch
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

After this step, 15-20 minutes are required as usual for the network to become active and go into `processing` mode, which is required for the stake transaction to be handled. I used the <http://localhost:3000/> monitor to wait for all the nodes to fully activate.

### JSON-RPC API setup

To simplify the POC, Shardeum's `json-rpc-server` is used to interact with the network, specifically, to send the stake transaction using the attacker's account. This is needed because by default the initial network is setup without any staking validator nodes, since this mechanism is enabled only once `minNodes` number of validator nodes (32 in this POC) is reached.

1. Clone the `json-rpc-server` repo and switch to the last commit on the `dev` branch, which is `5dc56e5f4312529d4262cab618ec618d288de5dd` as of this POC's submission.

   ```bash
   git clone https://github.com/shardeum/json-rpc-server.git
   cd json-rpc-server
   git switch --detach 5dc56e5f4312529d4262cab618ec618d288de5dd
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

### Malicious node setup

Before running the POC itself, an additional validator node, the supposed attacker's node, must be setup and launched as well. This should happen after the network is fully active, so that the 32 `minNodes` requirement is met, in which case the node will require an active stake to be made, which is done in the POC. For the POC to work as expected, the malicious node must be configured with a specific `config.json` and `secrets.json` files.

1. Repeat steps 1-3 of the "Local Shardeum network setup" section, but in a separate directory. The directory should pretty much resemble just another validator with the same configuration at this point.
2. Apply the `malicious-node.patch` file from the attached gist (<https://gist.github.com/renbou/775c1555f45c2154b202a289530eeb0b>) in the malicious node's directory. It will configure the node's ports and public/private keys, which is needed for the POC to interact with the node and make requests on its behalf correctly.

   ```bash
   git apply malicious-node.patch
   ```
3. Install dependencies and build the project.

   ```bash
   npm ci
   npm run prepare
   ```
4. Manually start the malicious node. It should show up on the <http://localhost:3000/> monitor, but it will not make join requests until the POC is run for the node to receive a stake.

   ```bash
   node ./dist/src/index.js
   ```

### Global transaction forgery using signAppData exploitation

As said in the introduction, `modify-config-poc.js` from the attached gist (<https://gist.github.com/renbou/775c1555f45c2154b202a289530eeb0b>) contains the exploit code and can be configured using the various parameters at the top of the script. It can be ran using `NodeJS`, and requires a few of the Shardus packages (`@shardus/crypto-utils`, `@shardus/net`, `@shardus/types`) as well as `ether` and `axios` libraries to be installed, as specified in the attached `package.json` (<https://gist.github.com/renbou/775c1555f45c2154b202a289530eeb0b>).

The `main` function of the script contains the POC's logic, with preceding functions declared just to simplify the main POC. Following is a detailed writeup of how it works:

1. A random active validator node is retrieved from the archiver and queried for information about the attacker's node account. The presence of an existing stake for the account is checked, to allow running the exploit more than once without reconfiguring the setup. If the node doesn't have staked coins yet, the JSON RPC server is called using the `ether` library with a stake transaction using the attacker's account, which has 100 SHM from the genesis coins.
2. The script waits for the attacker's node to become active by querying its `nodeInfo` endpoint each 30 seconds for 20 minutes, which should be enough for the node to detect that it has received a stake, and go through the Shardeum joining process all the way to becoming an active node.
3. The vulnerability described in this report is exploited: a `sign-stake-cert` request to the `sign-app-data` endpoint of each active validator node on the network is made, asking to sign a valid stake cert for the attacker's node, which genuinely has a stake at this point, additionally with the fields needed to later call the `set-global` gossip endpoint with this data, pretending it is a global transaction receipt. The `value` field specifies the global transaction which will be passed to the network as a valid transaction through the `processReceipt` `GlobalAccounts` function (<https://github.com/shardeum/shardus-core/blob/4d75f797a9d67af7a94dec8860220c4e0f9ade3c/src/p2p/GlobalAccounts.ts#L269>).

   NOTE: this step sometimes hang without receiving a response, and the requests time out in 10 seconds. This has probably something to do with `lib-net` not being ready yet. It is okay to just re-run the POC in this case.
4. The signatures from the last step are used to send out a `set-global` gossip to a single active node of the network. Since signatures for all active nodes were retrieved, this is guaranteed to pass the consensus verification and be applied by the node. The node will then pass the forged receipt to all other nodes in the network through the gossip protocol.

   Since the POC is configured to change the validator node's configuration `mode` value from `release` to `debug`, a request to `/config` of the nodes would fail prior to the exploit being run, but succeed after the forged global transaction is applied, since the `debug` mode will be active on each and every node in the network.

To manually verify the POC, a basic `curl` request to the `/config` endpoint of any node can be made BEFORE and AFTER running the POC:

```bash
curl http://localhost:9002/config
{"message":"Unauthorized!","status":401}
```

After running the POC, and waiting for it to confirm that the exploit has worked, the same request should return the full config of the node, since debug mode has been enabled, allowing public access to any debug endpoint.

Since the POC does not use any functionality such as developer keys, and does not rely on certain configuration parameters being enabled, it is enough to showcase the vulnerability and how it can be used to exploit consensus-based mechanisms such as global transactions in the network.

`out.log` attached to the gist (<https://gist.github.com/renbou/775c1555f45c2154b202a289530eeb0b>) contains an example of the output of the POC's execution.
