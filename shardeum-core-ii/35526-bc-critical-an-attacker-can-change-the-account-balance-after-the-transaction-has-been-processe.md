# #35526 \[BC-Critical] An attacker can change the account balance after the transaction has been proce

**Submitted on Sep 26th 2024 at 13:11:33 UTC by @periniondon630 for** [**Audit Comp | Shardeum: Core II**](https://immunefi.com/audit-competition/shardeum-core-ii-boost)

* **Report ID:** #35526
* **Report Type:** Blockchain/DLT
* **Report severity:** Critical
* **Target:** <https://github.com/shardeum/shardus-core/tree/dev>
* **Impacts:**
  * Direct loss of funds

## Description

## Brief/Intro

An attacker's active node from the execution group can change the account balance after a transaction involving this account has been processed by the network.

## Vulnerability Details

The P2P handler binary\_repair\_oos\_accounts does not recalculate the proposalHash during the signature verification process from receipt.signaturePack. This vulnerability allows an attacker to modify account data and the corresponding afterStateHash in the receipt proposal, enabling the update of the account state on the validator node. For the attack to be successful, the attacker's node must be part of the execution group for the transaction.

## Impact Details

An attacker can set the balance to any value for any account involved in transactions processed within the execution group in which the attacker's node is participating.

## References

<https://github.com/shardeum/shardus-core/blob/23e06ded6744d8521cff9d749c1f1dd482c5fcb6/src/state-manager/AccountPatcher.ts#L490>

## Link to Proof of Concept

<https://gist.github.com/periniondon630/ee50d02c09c87fb97d31d53b1f1bd9cf>

## Proof of Concept

## Proof of Concept

I’ve added a callback for the event when a transaction is archived, following the method outlined in the documentation. Apply the patch from the gist to the attacker's node, wait for it to become active, and then send a transaction. Check the attacker's node output log—it will display a message when the transaction is archived, indicate which account will be targeted (the first one is selected), and show the nodes to which requests will be sent. The final balance will be set to 100 coins.
