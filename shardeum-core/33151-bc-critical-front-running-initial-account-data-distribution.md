# 33151 - \[BC - Critical] Front running initial account data distribution

Submitted on Jul 12th 2024 at 21:44:47 UTC by @doxtopzhivago for [Boost | Shardeum: Core](https://immunefi.com/bounty/shardeum-core-boost/)

Report ID: #33151

Report type: Blockchain/DLT

Report severity: Critical

Target: <https://github.com/shardeum/shardus-core/tree/dev>

Impacts:

* Direct loss of funds

## Description

## Brief/Intro

Sending wrong initial account data information at the beginning of transaction processing can cause loss of account balance.

## Vulnerability Details

The message handler 'broadcast\_state\_complete\_data' is lacking incoming message verification, as a result it is possible to frontrun initial account data distribution in a way so it will be possible to have a full control over the amount of the final balance after transaction (before transaction execution validators are exchanging initial account data between each other). When transaction is received by malicious validator it will immediately send internal message 'broadcast\_state\_complete\_data' to the validators in transaction group with the wrong initial account balance of the crypto coin transfer source address. Execution group validators will use this wrong initial account data in transaction execution and provide result of coin transfer based on it. Transaction will be successfully executed and completed. This will work only if malicious validator is participating in transaction group.

## Impact Details

Full control of the initial account balance for transactions where malicious validator is participating in transaction group.

## References

Link to the internal message handler: <https://github.com/shardeum/shardus-core/blob/4d75f797a9d67af7a94dec8860220c4e0f9ade3c/src/state-manager/TransactionQueue.ts#L323>

## Proof of Concept

I’ve added callbacks before transaction gets in the processing queue and after processing is successfully completed. For POC I’ve used malicious validator which is removing some constant (1eth) amount from initial account balance for the source account of the coin transfer. Patch should be applied only for malicious validator shardeum-core repository.
