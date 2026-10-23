# #36029 \[BC-Insight] Node.js crash on counterMap overflow

**Submitted on Oct 16th 2024 at 10:24:53 UTC by @dldLambda for** [**Audit Comp | Shardeum: Core II**](https://immunefi.com/audit-competition/shardeum-core-ii-boost)

* **Report ID:** #36029
* **Report Type:** Blockchain/DLT
* **Report severity:** Insight
* **Target:** <https://github.com/shardeum/shardus-core/tree/dev>
* **Impacts:**
  * Network not being able to confirm new transactions (total network shutdown)
  * Shutdown of greater than or equal to 30% of network processing nodes without brute force actions, but does not shut down the network

## Description

## Brief/Intro

Sending a large number of requests will cause the "counterMap" overflow and Node.js crash.

## Vulnerability Details

In "sendRequests" function, the "countEvent" function is called with the second parameter, which can controlled from the request.

If the "counterMap" does not yet have such a key, then a new key-value pair is created and added to "counterMap". «counterMap» is a global Map that is filled but not cleared.

It’s overflow will lead to Node.js crash.

<https://github.com/shardeum/shardus-core/blob/dev/src/p2p/ServiceQueue.ts#L387> — link to "sendRequests" function.

Next, a gossip request is formed and the second parameter is a string "gossip send - ${add.hash}" , that includes the hash of the original request parameter (<https://github.com/shardeum/shardus-core/blob/dev/src/p2p/ServiceQueue.ts#L397>).

And this is how counterMap is created and "countEvent" is declared (<https://github.com/shardeum/shardus-core/blob/dev/src/utils/nestedCounters.ts#L96>):

\`\`\` class NestedCounters { constructor() { this.eventCounters = new Map() // <--- this.rareEventCounters = new Map() this.crypto = null this.infLoopDebug = false } //some lines... countEvent(category1: string, category2: string, count = 1): void { let counterMap: CounterMap = this.eventCounters // <---

```
let nextNode: CounterNode &#x3D; null
if (counterMap.has(category1) &#x3D;&#x3D;&#x3D; false) {
  nextNode &#x3D; { count: 0, subCounters: new Map() }
  counterMap.set(category1, nextNode)
} else {
  nextNode &#x3D; counterMap.get(category1)
}
nextNode.count +&#x3D; count
counterMap &#x3D; nextNode.subCounters

//unrolled loop to avoid memory alloc
category1 &#x3D; category2
if (counterMap.has(category1) &#x3D;&#x3D;&#x3D; false) {
  nextNode &#x3D; { count: 0, subCounters: new Map() }
  counterMap.set(category1, nextNode)
} else {
  nextNode &#x3D; counterMap.get(category1)
}
nextNode.count +&#x3D; count
counterMap &#x3D; nextNode.subCounters
```

} //some lines... } \`\`\` It is noticeable that if, when checking the uniqueness of the received second parameter, this key is not found, then a new key-value pair is added, but counterMap is not cleared, which can lead to overflow.

## Impact Details

An attacker will be able to send a huge number of requests, overflow the counterMap and therefore Node.js will fail.

## Proof of Concept

## Proof of Concept

1. Let’s check that we can get there using a request. In your local copy of Shardeum, open the file ./src/index.ts and add the following function at line #1139 ( <https://gist.github.com/dldLambda/335024ad4787abcced5f5550aed0da31> — in gistfile1.txt)

Is ./src/index.ts you need to add "import \* as Comms from '@shardus/core/dist/p2p/Comms' " and "import { nodeListFromStates } from './Join' ".

Then rebuild the project using "npm run prepare".

Now, you can send a request with the parameters you selected and make sure that your data is received:

\`\`\` fetch(\`<http://193.108.115.45:9001/dl\\_func\\`>, { method: 'POST', headers: { 'Content-Type': 'application/json' }, body: JSON.stringify({"cycle":13,"hash":"1","priority":0,"sign":{"owner":"2","sig":"3"},"subQueueKey":"4","txData":{"nodeId":"5","publicKey":"6","sign":{"owner":"7","sig":"8"},"startTime":1729004605},"type":"nodeInitReward" })}).then(response => { if (!response.ok) { throw new Error('Network response was not ok'); } return response.json(); }).then(data => { console.log('Success:', data); }).catch(error => { console.error('Error:', error); })

\`\`\` The data in the request can be quite arbitrary. Then in the ./Shardeum do the following command "grep -r "hoho55" . "

You will see in the out.log after the line "hoho55" the body of your request.

2. Example

First, set a memory limit using the command "ulimit -Sv 1000000". Next, run the following script with "npx tsc "insert filename" " (run "npm install -g npx" if you need).

<https://gist.github.com/dldLambda/e60f585401f01e77b8d7338bb0431edd> — link to example script.

Firstly, you will see that the size of the counterMap grows indefinitely, and secondly, the Node.js soon fail. Thus, an attacker can disable nodes.
