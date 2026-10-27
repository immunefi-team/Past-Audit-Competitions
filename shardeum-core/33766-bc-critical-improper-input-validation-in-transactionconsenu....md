# 33766 - \[BC - Critical] Improper input validation in TransactionConsenu...

Submitted on Jul 29th 2024 at 02:00:06 UTC by @neplox for [Boost | Shardeum: Core](https://immunefi.com/bounty/shardeum-core-boost/)

Report ID: #33766

Report type: Blockchain/DLT

Report severity: Critical

Target: <https://github.com/shardeum/shardus-core/tree/dev>

Impacts:

* Network not being able to confirm new transactions (total network shutdown)
* Direct loss of funds

## Description

## Brief/Intro

Shardeum validator nodes are vulnerable to complete DOS due to lack of input validation in <https://github.com/shardeum/shardus-core/blob/4d75f797a9d67af7a94dec8860220c4e0f9ade3c/src/state-manager/TransactionConsensus.ts#L265>

Any active node can exploit this vulnerability. Exploitation leads to a complete stall of all of the validator node's processes, and due to the simplicity of the exploit, it is possible to execute it on all active nodes, simultaneously, shutting down the entire network. A more sophisticated attack would involve shutting down only a large part of the nodes, but not all, for attacker-controlled nodes to be the only ones available, which can then be used to overtake the whole network.

## Vulnerability Details

The handlers `remove_timestamp_cache`(<https://github.com/shardeum/shardus-core/blob/4d75f797a9d67af7a94dec8860220c4e0f9ade3c/src/state-manager/TransactionConsensus.ts#L264>) and `get_tx_timestamp`(<https://github.com/shardeum/shardus-core/blob/4d75f797a9d67af7a94dec8860220c4e0f9ade3c/src/state-manager/TransactionConsensus.ts#L242>) use assignments like:

```
this.txTimestampCache[cycleCounter][txId] = null
...
this.txTimestampCache[signedTsReceipt.cycleCounter][txId] = signedTsReceipt
```

Meanwhile, txTimestampCache is an object with a global prototype: <https://github.com/shardeum/shardus-core/blob/4d75f797a9d67af7a94dec8860220c4e0f9ade3c/src/state-manager/TransactionConsensus.ts#L137>

There is also no check to ensure that cycleCounter is indeed a hex string, which allows us to assign it to `__proto__` and overwrite anything in the object's prototype.

This opens up possibilities for the class of Attack Prototype Pollution, which allows for drastic changes in the behavior of the application's logic as well as its dependencies.

The simplest demonstration is the removal of the toString method from the global prototype, which inevitably leads to a crash of the node validator. This is because this method is used in a vast number of dependencies and in the validator's own code

## Impact Details

The most basic outcome would be a total network shutdown caused by exploiting the DOS vulnerability on each active validator node.

A more complex attack scenario, as said in the intro, would be the shutdown of nearly all, but not all, validator nodes, so that attackers nodes would be the only ones left available in the standby state. After the attackers nodes are picked as active and join the validators, the remaining honest validators can be shut down using the DOS vulnerability, which would leave the whole network to be controlled by the attacker. This means the network will continue functioning, but the attacker will be able to execute any transaction they want, and drain all the funds to themselves.

## References

**proto** property: <https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global\\_Objects/Object/proto> Prototype Pollution: <https://cheatsheetseries.owasp.org/cheatsheets/Prototype\\_Pollution\\_Prevention\\_Cheat\\_Sheet.html#:\\~:text=Prevention%20Cheat%20Sheet-,Explanation,and%20even%20remote%20code%20execution>.

## Proof of Concept

This POC was written in order to demonstrate total network shutdown as the main impact, as fund loss would be an impact that would follow due to all validator nodes except the attacker's being crippled. In order to simplify the POC only the main impact of validator node DOS is implemented.

`exploit.js` contained in the attached [gist](https://gist.github.com/Slonser/e8f1d4c0fdf3b51a45e34fb6de924318) contains the main exploit code for automatically disabling all the active nodes of the network after retrieving their addresses from the archiver. This writeup is present just to showcase exactly how it was tested and how it works.

## Local Shardeum network setup

This vulnerability is equally exploitable with any number of nodes as exploitation of a single validator node requires sending just a single message via the internal protocol to `remove_timestamp_cache`/`get_tx_timestamp` handlers. For demonstration purposes, however, a network with only 32 validator nodes is created.

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
3. Apply the `32-nodes.patch` file from the attached [gist](https://gist.github.com/Slonser/e8f1d4c0fdf3b51a45e34fb6de924318) for network setup. Note that it DOES NOT enable debug mode, demonstrating the vulnerability in a semi-realistic release setup.

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

## DOS exploitation for network shutdown

As said in the introduction, `exploit.js` from the attached [gist](https://gist.github.com/Slonser/e8f1d4c0fdf3b51a45e34fb6de924318) contains the exploit code and can be ran on any network as long as a valid archiver URL is suplied via the `ARCHIVER_URL` variable at the top of the script.

**REQUIREMENTS!!!** To carry out this attack, you need to replace the variables `NODE_ID` and `keypair` with the ID of the node you control and its key pair used for signing messages via the internal protocol.

It can be ran using `NodeJS`, and only requires the `@shardus/net`, `axios`, `@shardus/types`, `@shardus/crypto-utils` libraries to be installed, as specified in the attached [`package.json`](https://gist.github.com/Slonser/e8f1d4c0fdf3b51a45e34fb6de924318). Following is a detailed writeup of how it works:

1. Multiple requests to the specified archiver's `full-nodelist` endpoint are made in order to retrive the full list of validator nodes in the network, including standby, syncing, and active nodes. Running the DOS on these nodes means that no more nodes will be available, leading to a total shutdown of the network.
2. Each node's `/nodeinfo` endpoint is queried to retrieve the `internalIp` and `internalPort` values, which host the internal protocol of the node, used in this exploit of the vulnerability.
3. A message of the form `{"txId":"toString","cycleCounter":"__proto__","cycleMarker": "test"}` is sent using the internal protocol to `remove_timestamp_cache` handler. This will cause the toString function to be removed from the object's prototype, after which it will be assigned to null.

Shardeum's monitor dashboard, launched by default on <http://localhost:3000/> can be used to check that all active nodes will be now marked as red as they go offline and stop reporting to the monitor. Making any request to the validator nodes will hang, as the network is completely stopped at this point. Additionally, the node logs will show that they are unsuccessfully trying to exit. An example of such logs is the file `log.txt` attached in the [gist](https://gist.github.com/Slonser/e8f1d4c0fdf3b51a45e34fb6de924318).
