# #35531 \[BC-Critical] Absence of signature deduplication for receipt in the binary\_repair\_oos\_account

**Submitted on Sep 26th 2024 at 15:03:22 UTC by @periniondon630 for** [**Audit Comp | Shardeum: Core II**](https://immunefi.com/audit-competition/shardeum-core-ii-boost)

* **Report ID:** #35531
* **Report Type:** Blockchain/DLT
* **Report severity:** Critical
* **Target:** <https://github.com/shardeum/shardus-core/tree/dev>
* **Impacts:**
  * Direct loss of funds

## Description

## Brief/Intro

The lack of signature deduplication for receipt in the binary\_repair\_oos\_accounts handler allows an attacker to craft any necessary receipt and update the balance of any account, as long as the attacker's node is part of the execution group.

## Vulnerability Details

The binary\_repair\_oos\_accounts P2P handler calls the verifyAppliedReceipt function to validate the signatures in the receipt. However, it fails to check for duplicate signatures. This oversight allows an attacker with just one active node and a single signature to bypass the minimum signature requirement. As a result, a malicious receipt can be verified, and the account update will be successfully applied across the network."

## Impact Details

An attacker can set the balance to any value for any account stored on their active node. The updated balance will be accepted as valid across the entire network.

## References

<https://github.com/shardeum/shardus-core/blob/23e06ded6744d8521cff9d749c1f1dd482c5fcb6/src/state-manager/AccountPatcher.ts#L490> <https://github.com/shardeum/shardus-core/blob/23e06ded6744d8521cff9d749c1f1dd482c5fcb6/src/state-manager/TransactionConsensus.ts#L1714>

## Link to Proof of Concept

<https://gist.github.com/periniondon630/e7f4488062f558b8909ed7d528446e37>

## Proof of Concept

## Proof of Concept

I’ve added a callback for the event when a transaction is archived, following the method outlined in the documentation. Apply the patch from the gist to the attacker's node, wait for it to become active, and then send a transaction. Check the attacker's node output log—it will display a message when the transaction is archived, indicate which account will be targeted (the first one is selected). You can also observe that all the signatures are identical, yet the request is still processed successfully, and the account balance is updated.
