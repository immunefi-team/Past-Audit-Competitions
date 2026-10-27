# 33925 - \[BC - Critical] Improper input validation in fixDeserializedWra...

## Improper input validation in fixDeserializedWrappedEVMAccount leads to DOS and total network shutdown

Submitted on Aug 2nd 2024 at 04:31:46 UTC by @neplox for [Boost | Shardeum: Core](https://immunefi.com/bounty/shardeum-core-boost/)

Report ID: #33925

Report type: Blockchain/DLT

Report severity: Critical

Target: <https://github.com/shardeum/shardus-core/tree/dev>

Impacts:

* Direct loss of funds
* Network not being able to confirm new transactions (total network shutdown)
* Increasing network processing node resource consumption by at least 30% without brute force actions, compared to the preceding 24 hours

### Description

### Brief/Intro

Shardeum validator nodes implemented in the <https://github.com/shardeum/shardeum> repository are vulnerable to complete DOS due to lack of input validation in `fixDeserializedWrappedEVMAccount`, which is used in internal endpoints. Exploitation leads to a complete stall of all of the validator node's processes, and due to the simplicity of the exploit, it is possible to execute it on all active nodes. A more sophisticated attack would involve shutting down only a large part of the nodes, but not all, for attacker-controlled nodes to be the only ones available, which can then be used to overtake the whole network.

### Vulnerability Details

The `fixDeserializedWrappedEVMAccount` [function](https://github.com/shardeum/shardeum/blob/c7b10c2370028f7c7cbd2a01839e50eb50faa904/src/shardeum/wrappedEVMAccountFunctions.ts#L85), which is commonly used throughout `shardeum` repositorie, does not perform validation on the passed data. It is possible to pass an "Array-like" object as the `storageRoot`,`codeHash`,`codeByte`,`value`, fields, which is accepted by `Uint8Array.from` used by the [`fixWrappedEVMAccountBuffers`](https://github.com/shardeum/shardeum/blob/c7b10c2370028f7c7cbd2a01839e50eb50faa904/src/shardeum/wrappedEVMAccountFunctions.ts#L98)/ [`fixAccountFields`](https://github.com/shardeum/shardeum/blob/c7b10c2370028f7c7cbd2a01839e50eb50faa904/src/state/transactionState.ts#L154) functions called from `fixDeserializedWrappedEVMAccount`.

Since "Array-like" objects only require a `length` property to be set, it is easy to pass such an object from an incoming request, in which case `Uint8Array.from({length: x})` would attempt to sequentially copy `x` values from the "Array-like" object, each of which will be 0 due to no actual array values being set, making for an easy way to greatly increase the memory usage of the validator node without sending a large request, and cause a DOS of the validator node because the copying done by `Buffer.from` linearly depends in CPU time on the value of `x`. This can easily be checked in a `node` shell, using the same version that is used by `shardeum`, v18.16.1:

```js
node
Welcome to Node.js v18.16.1.
Type ".help" for more information.
> function time(f) {console.time('f');f();console.timeEnd('f')}
undefined
> time(() => Uint8Array.from({length:1_000_000}))
f: 37.399ms
undefined
> time(() => Uint8Array.from({length:10_000_000}))
f: 231.761ms
undefined
> time(() => Uint8Array.from({length:100_000_000}))
f: 2.098s
```

The most effective way this could be exploited is through the internal protocol `repair_oos_accounts` [handler](https://github.com/shardeum/shardus-core/blob/4d75f797a9d67af7a94dec8860220c4e0f9ade3c/src/state-manager/AccountPatcher.ts#L336)

This handler calls `calculateAccountHash`, which in turn calls `fixDeserializedWrappedEVMAccount` with data controlled by a potential attacker

### Impact Details

The most basic outcome would be a total network shutdown caused by exploiting the DOS vulnerability on each available active validator node. Sending large `length` values will cause nodes to either crash due to OOM, or spend long times in the `Uint8Array.from` call, which, when utilized across all nodes in the network, will halt all transaction processing and cause consensus mechanisms to crash later on. Because there are no prerequisites to the attack, the attacker can just keep on spamming DOS-causing requests to any endpoint which calls `fixDeserializedWrappedEVMAccount`, so that actually starting the network back up would require a fix to be deployed to all nodes, making it even more difficult to mitigate.

A more complex attack scenario, as said in the intro, would be the shutdown of nearly all, but not all, validator nodes. After the attackers nodes are picked as active and join the validators, the remaining honest validators can be shut down using the DOS vulnerability, which would leave the whole network to be controlled by the attacker. This means the network will continue functioning, but the attacker will be able to execute any transaction they want, and drain all the funds to themselves.

### References

<https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global\\_Objects/TypedArray/from>

## Proof of concept

This POC was written in order to demonstrate total network shutdown as the main impact, as fund loss would be an impact that would follow due to all validator nodes except the attacker's being crippled. In order to simplify the POC only the main impact of validator node DOS is implemented.

`poc.js` contained in the attached [gist](https://gist.github.com/Slonser/68fe321192852f2dde2924623e0256e7) contains the main exploit code for automatically disabling all the active of the network after retrieving their addresses from the archiver. This writeup is present just to showcase exactly how it was tested and how it works.

**WARNING!** the exploit causes **each** validator node to slowly accumulate up to a 4 gigabytes (the `4294967294` constant in the exploit) worth of memory, so make sure to run it in an unimportant environment and be ready for possible crashes and OOM errors. The value `4294967294` can also be decreased for running the setup and exploit locally, it was chosen in order to clearly demonstrate the impact of node DOS via memory and CPU resource consumption.

### Local Shardeum network setup

This vulnerability is equally exploitable with any number of nodes as exploitation of a single validator node requires sending just a single message via the internal protocol. For demonstration purposes, however, a network with only 32 validator nodes is created.

Any Shardeum network using the current validator node code will be vulnerable, so it is not necessary to follow these exact steps. They are here just to showcase how the POC was tested by me.

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
3. Apply the `32-nodes.patch` file from the attached [gist](https://gist.github.com/Slonser/68fe321192852f2dde2924623e0256e7) for network setup. Note that it DOES NOT enable debug mode, demonstrating the vulnerability in a semi-realistic release setup.

   ```bash
   git apply 32-nodes.patch
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

After this step, 15-20 minutes are required as usual for at least some validator nodes to go into being active, at which point the exploit itself can be ran. I used the <http://localhost:3000/> monitor to wait for nodes to start activating.

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

### DOS exploitation for network shutdown

As said in the introduction, `poc.js` from the attached [gist](https://gist.github.com/Slonser/68fe321192852f2dde2924623e0256e7) contains the exploit code and can be ran on any network as long as a valid archiver URL and Attacker validator node info is suplied It can be ran using `NodeJS`, requires a few of the Shardus packages (`@shardus/crypto-utils`, `@shardus/net`, `@shardus/types`) as well as `ether` and `axios` libraries to be installed, as specified in the attached [`package.json`](https://gist.github.com/Slonser/68fe321192852f2dde2924623e0256e7).

**NOTE**: modify the keypair and NODE\_ID values with the values of one of the nodes in the network, for the POC to simulate requests from an attacker-controlled validator node. The keypair should be set to the contents of a node's secrets.json, and NODE\_ID can be set to the value of id retrieved from a node's /nodeinfo endpoint.

Following is a detailed writeup of how it works:

1. Multiple requests to the specified archiver's `full-nodelist` endpoint are made in order to retrive the full list of validator active nodes in the network. Running the DOS on these nodes means that no more nodes will be available, leading to a total shutdown of the network.
2. Each node's `/nodeinfo` endpoint is queried to retrieve the `internalIp`, `internalPort` and `id` values
3. Using getTxId, we create a new transaction (it doesn't matter if it succeeds or not). This is done to obtain a txId that will be used in the future.
4. A message with payload of the form

   ```
      repairInstructions:[{
      "accountID": "0000000000000000000000000000000000000000",
      "txId": txId,
      "hash":"1231231231231231231231231",
      "accountData":{
        "data":{
            "accountType":0,
            "account":{
                "storageRoot":{
                    "data":{
                        "length": 4294967294
                    }
                },
                "codeHash":{
                    "data":{
                        "length": 4294967294
                    }
                }
            }
        }
     },
      "targetNodeId": nodeAddr.id,
      "receipt2":{
        "appliedVote":sign({
            "node_id":NODE_ID,
            "transaction_result":"result"
        })
      }
    }]
   ```

   is sent using the internal protocol to `repair_oos_accounts` handler triggering the vulnerability. `Uint8Array.from` will then block NodeJS' event loop, making each validator node unresponsive and practically stopping the network.

Shardeum's monitor dashboard, launched by default on <http://localhost:3000/> can be used to check that all active nodes will be now marked as red as they go offline and stop reporting to the monitor. Making any request to the validator nodes will hang, as the network is completely stopped at this point.
