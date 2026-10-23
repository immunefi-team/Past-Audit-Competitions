# #39791 \[BC-Critical] Filling the queue with "setCertTime" stop the network from processing new transactions

**Submitted on Feb 7th 2025 at 09:04:58 UTC by @fomohacker for** [**Audit Comp | Shardeum: Core III**](https://immunefi.com/audit-competition/audit-comp-shardeum-core-iii)

* **Report ID:** #39791
* **Report Type:** Blockchain/DLT
* **Report severity:** Critical
* **Target:** <https://github.com/shardeum/shardeum/tree/bugbounty>
* **Impacts:**
  * Network not being able to confirm new transactions (total network shutdown)

## Description

## Brief/Intro

By injecting fake `setCertTime` transactions, a malicious actor can fill the queue and reject other users' transaction. It effectively stops the network from processing new transactions.

## Vulnerability Details

Shardeum team, sorry for testing this on the bounty test network. I just wanted to prove the point.

Like most of the EVM-compatible blockchains Shardeum also has a speed limit in processing the transactions from the same party because of possible race conditions. The interesting thing is how Shardeum rejects the transaction based on the number of transaction in the queue unlike other blockchains. So, if an attacker can fill the queue quickly, the nodes will reject other incoming transactions. I was looking into different transaction types and notice `setCertTime` transaction can be used without having to fund the account with SHM or run a bad validator. It is not checked in the `txPreCrackData` function which blocks most of invalid transactions. Using this type of transaction, I can create lots of fake `nominee` and quickly fill the queue and hence blocking other users' legit transasctions.

### How will it behave in 1280 nodes network?

The attack script (pls see gist) creates fake nominees with account initials starting from `0` to `f`. This make sure all active validators are filled with fake transactions. A malicious actor does not need to increase the injection rate or use a powerful machine. A simple laptop can run the attack script.

## Impact Details

Network simply cannot process legit transactions anymore. Theoretically, it is not 100% rejection because Shardeum nodes try to flush the queue every few minutes. But **the attacker can easily maintain the rejection rate at \~99.9%**. So, every legit transactions including staking and unstaking will be rejected. I run the attack script against the test bug bounty network and it has stopped processing transactions.

## References

<https://github.com/shardeum/shardeum/blob/bugbounty/src/tx/setCertTime.ts#L94\\>
This code should check if the nominee account exists in the network.

## Link to Proof of Concept

<https://gist.github.com/fomohacker/60940fd82cadc0820ce29fb8a45a1d3a>

## Proof of Concept

## Proof of Concept

<https://gist.github.com/fomohacker/60940fd82cadc0820ce29fb8a45a1d3a>

The script to exploit this is simple. I created fake nominee key pairs with different initials. Each account is for each shard so that all nodes in the network will be filled with fake transactions.

Then, the script will loop through the active validators and inject at a TPS based on the rejection/accept rate. It smartly tries to maintain at the rejection rate of 99%.

To run the script from the gist:

1. put index.js and package.json file in a folder
2. run `npm install`
3. run `node index.js 34.133.41.202 4000 10 50 70` and see the outputs

The script format is `node index.js <archive_ip> <archive_port> <low_tps> <medium_tps> <high_tps>`
