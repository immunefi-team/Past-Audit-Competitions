# 33278 - \[BC - Critical] Improper input validation leads to DOS and tota...

## Improper input validation leads to DOS and total network shutdown

Submitted on Jul 17th 2024 at 03:35:07 UTC by @neplox for [Boost | Shardeum: Core](https://immunefi.com/bounty/shardeum-core-boost/)

Report ID: #33278

Report type: Blockchain/DLT

Report severity: Critical

Target: <https://github.com/shardeum/shardeum/tree/dev>

Impacts:

* Network not being able to confirm new transactions (total network shutdown)
* Direct loss of funds

### Description

### Brief/Intro

Shardeum validator nodes implemented in the <https://github.com/shardeum/shardeum> repository are vulnerable to complete DOS through the `eth_getBlockHashes` HTTP endpoint, which is publically available and does not require any specific access rights to be called. Exploitation leads to a complete stall of all of the validator node's processes, and due to the simplicity of the exploit, it is possible to execute it on all nodes, including standby, simultaneously, shutting down the entire network. A more sophisticated attack would involve shutting down only a large part of the nodes, but not all, for attacker-controlled nodes to be the only ones available, which can then be used to overtake the whole network.

### Vulnerability Details

The `eth_getBlockHashes` endpoint accepts two parameters, `fromBlock` and `toBlock`, specifying the blocks which hashes should be returned. `fromBlock` and `toBlock` are expected to be in `ShardeumFlags.maxNumberOfOldBlocks` (256 at the time of writing) from each other and `latestBlock`, but due to a parsing bug it is actually possible to set `fromBlock` to a very small number. This bug can be used to set `fromBlock` to the value `parseInt("-9007199254740992333")`, which becomes `-9007199254740992000` after parsing, which is so small that incrementing it is a NO-OP due to JavaScript's nature of using floating point numbers.

The bug is contained in this line of the `eth_getBlockHashes` handler implementation in `src/index.ts` (<https://github.com/shardeum/shardeum/blob/c7b10c2370028f7c7cbd2a01839e50eb50faa904/src/index.ts#L1286>):

```javascript
if (typeof toBlock === "string") fromBlock = parseInt(toBlock);
```

Instead of setting `toBlock`, the result of `parseInt(toBlock)` is assigned to `fromBlock`. And, since this happens after the `ShardeumFlags.maxNumberOfOldBlocks` check, execution will reach the `for` loop which gathers hashes with the values of `fromBlock` and `toBlock` equal to `-9007199254740992000` and `latestBlock`, respectively.

```javascript
for (let i = fromBlock; i <= toBlock; i++) {
  const block = readableBlocks[i];
  if (block) blockHashes.push(block.hash);
}
```

The variable `i` will also be set to `-9007199254740992000`, and `i++` will then be a NO-OP on each iteration, causing the loop to iterate infinitely. Due to how NodeJS works, this will block the process' event loop thread, and no other logic will be able to run, practically shutting the validator node down and making it simply infinitely waste CPU cycles.

This code snippet can be run in `node` or the web browser JS console to demonstrate that the value of `i` in such a loop would never change. It will output `-9007199254740992000` 10 times despite the `i++` increment.

```javascript
let i = parseInt("-9007199254740992333");
for (let j = 0; j < 10; j++) {
  console.log(i++);
}
```

### Impact Details

The most basic outcome would be a total network shutdown caused by exploiting the DOS vulnerability on each available validator node, including the ones in standby. Since exploitation does not cause the nodes' processes to actually shut down or crash, the whole network will just stop processing any requests and user transactions. Because there are no prerequisites to the attack, the attacker can just keep on spamming DOS-causing requests to the `eth_getBlockHashes` endpoint, so that actually starting the network back up would require a fix to be deployed to all nodes, making it more difficult to mitigate.

A more complex attack scenario, as said in the intro, would be the shutdown of nearly all, but not all, validator nodes, so that attackers nodes would be the only ones left available in the standby state. After the attackers nodes are picked as active and join the validators, the remaining honest validators can be shut down using the DOS vulnerability, which would leave the whole network to be controlled by the attacker. This means the network will continue functioning, but the attacker will be able to execute any transaction they want, and drain all the funds to themselves.

### References

* `eth_getBlockHashes` implementation in Shardeum: <https://github.com/shardeum/shardeum/blob/c7b10c2370028f7c7cbd2a01839e50eb50faa904/src/index.ts#L1275>
* JavaScript max and min safe-to-use number values: <https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global\\_Objects/Number/MAX\\_SAFE\\_INTEGER>, <https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global\\_Objects/Number/MIN\\_SAFE\\_INTEGER>

## Proof of concept

This POC was written in order to demonstrate total network shutdown as the main impact, as fund loss would be an impact that would follow due to all validator nodes except the attacker's being crippled. In order to simplify the POC only the main impact of validator node shutdown is implemented.

`dos.py` contained in the attached gist (<https://gist.github.com/renbou/1064628ec6573b2e8ea30cbb28f54666>) contains the main exploit code for automatically shutting down all the active and standby nodes of the network after retrieving their addresses from the archiver. This writeup is present just to showcase exactly how it was tested and how it works.

### Local Shardeum network setup

This vulnerability is equally exploitable with any number of nodes as exploitation of a single validator node requires sending just a single HTTP request to the `eth_getBlockHashes` endpoint. For demonstration purposes, however, a network with only 32 validator nodes is created.

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
3. Apply the `32-nodes.patch` file from the attached gist (<https://gist.github.com/renbou/1064628ec6573b2e8ea30cbb28f54666>) for network setup. Note that it DOES NOT enable debug mode, demonstrating the vulnerability in a semi-realistic release setup.

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

### DOS exploitation for network shutdown

As said in the introduction, `dos.py` from the attached gist (<https://gist.github.com/renbou/1064628ec6573b2e8ea30cbb28f54666>) contains the exploit code and can be ran on any network as long as a valid archiver URL is suplied via the `ARCHIVER_URL` variable at the top of the script. It can be ran using Python 3, and only requires the `requests` library to be installed. Following is a detailed writeup of how it works:

1. Multiple requests to the specified archiver's `full-nodelist` endpoint are made in order to retrive the full list of validator nodes in the network, including standby, syncing, and active nodes. Running the DOS on these nodes means that no more nodes will be available, leading to a total shutdown of the network.
2. The publically available endpoint `eth_getBlockHashes` of each validator node is called with the parameters `fromBlock=1` and `toBlock=-9007199254740992333`. As specified in the **Vulnerability Details** section of the report **Description**, this causes the node to go into an infinite loop, practically blocking NodeJS' event loop and making the whole node unresponsive.

Shardeum's monitor dashboard, launched by default on <http://localhost:3000/> can be used to check that all active nodes will be now marked as red as they go offline and stop reporting to the monitor. Making any request to the validator nodes will hang, as the network is completely stopped at this point.
