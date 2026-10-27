# 33655 - \[BC - Critical] Complete shutdown of the transaction processing...

Submitted on Jul 25th 2024 at 22:31:08 UTC by @doxtopzhivago for [Boost | Shardeum: Core](https://immunefi.com/bounty/shardeum-core-boost/)

Report ID: #33655

Report type: Blockchain/DLT

Report severity: Critical

Target: <https://github.com/shardeum/shardus-core/tree/dev>

Impacts:

* Network not being able to confirm new transactions (total network shutdown)

## Description

## Brief/Intro

The handler of the internal message 'binary/get\_account\_data\_with\_queue\_hints' can cause transaction processing queue blocking because of the sharing the same locking mechanism.

## Vulnerability Details

The handler of the internal message 'binary/get\_account\_data\_with\_queue\_hints' is not setting correct bounds for length of the accounts list (Actually it is 2^32). For each account the handler is executing app.getAccountDataByList which is calling AccountStorage.getAccount -> getAccountEntry -> creates and runs SQL query for each account ID. Also the handler is creating fifoLock('accountModification') which is used in the following functions: preApplyTransaction and commitConsensedTransaction. Which are two critical functions for transaction processing. If this fifoLock is occupied by this handler, transaction processing will be slowed down or even stopped. So by sending a big list (2^32) of random accounts it can create a significant delay in transaction processing on the single node.

It also can be scaled so that whole network performance will fall down:

For incomming messages there is no check that remote IP address belongs to the node, only verification by signature. So private key from single standby node can be used by multiple senders.

In case of mass sending from multiple senders to all validators (list is open) attacker can completely stop transaction processing or significantly slow it down because the fifoLock is shared between transaction processing flow and the handler.

In my experiment it was causing onProcesssingQueueStuck and node became offline.

## Impact Details

I want to emphasize that it is not just a simple DOS, it is overloading bottleneck (fifoLock) which is also used in some critical functions in a system. By increasing number of accounts in a message you can impact transaction processing on target nodes or the whole network itself.

## References

<https://github.com/shardeum/shardus-core/blob/4d75f797a9d67af7a94dec8860220c4e0f9ade3c/src/state-manager/index.ts#L2015>

## Proof of Concept

Please apply the following gist to Shardeum repository. In order to activate this attack call the http GET /attack method on any active validator in the network.
