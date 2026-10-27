# #41794 \[BC-High] Not having any whitelisted account completely disables the prevalidator leading to transactions that cannot be deserialized

**Submitted on Mar 18th 2025 at 13:33:49 UTC by @HollaDieWaldfee for** [**Attackathon | Movement Labs**](https://immunefi.com/audit-competition/movement-labs-attackathon)

* **Report ID:** #41794
* **Report Type:** Blockchain/DLT
* **Report severity:** High
* **Target:** <https://github.com/immunefi-team/attackathon-movement/tree/main/protocol-units/da/movement/>
* **Impacts:**
  * Network not being able to confirm new transactions (total network shutdown)

## Description

## Brief/Intro

When a sequencer light node is set without any whitelisted account, there will be no prevalidator. But since the prevalidator is not only used for validating that the `transaction.sender()` is whitelisted, but also for ensuring that every transaction can be deserialized, this leads to blocks not being executable.

## Vulnerability Details

Let's take a look at how this happens.

There are two types of prevalidations inside of prevalidator/src/aptos: one for the whitelist and one for the transaction.

Both prevalidation functions are called inside of `prevalidate()` of the `prevalidator` that is set inside of `try_from_config()`.\
However, there is no prevalidator set if there are no whitelisted accounts. Hence, both the prevalidation for the whitelist and the prevalidation for the transaction depend on `config.whitelisted_accounts()` not being empty. This can be verified by taking a look at reference (1).

As a result, transactions will not be prevalidated to be deserializable when there are no whitelisted accounts. This introduces an issue since the execution logic heavily depends on transactions being deserializable. If this is not the case, even for only one transaction, the execution for the block that contains this transaction will not be possible. Additionally, executing any other block will be prevented as well since Celestia only allows execution in the correct order.

To mitigate this, the prevalidation for transaction should also be done when there are no whitelisted accounts.

## Impact Details

The execution of blocks will not be possible. Hence, the network will not be able to confirm new transactions.

## References

(1): <https://github.com/immunefi-team/attackathon-movement/blob/a2790c6ac17b7cf02a69aea172c2b38d2be8ce00/protocol-units/da/movement/protocol/light-node/src/sequencer.rs#L106-L112>

(2): <https://github.com/immunefi-team/attackathon-movement/blob/a2790c6ac17b7cf02a69aea172c2b38d2be8ce00/protocol-units/da/movement/protocol/light-node/src/sequencer.rs#L394-L418>

(3): <https://github.com/immunefi-team/attackathon-movement/blob/a2790c6ac17b7cf02a69aea172c2b38d2be8ce00/networks/movement/movement-full-node/src/node/tasks/execute\\_settle.rs#L236>

## Proof of Concept

1. There are no whitelisted accounts which means that `try_from_config()` won't set a prevalidator. Again, this can be verified by taking a look at reference (1).
2. Whenever `batch_write()` is called, there will be no prevalidation whatsoever. Instead, every transaction will just be pushed onto `transactions`. To confirm this, take a look at reference (2). This means that there is no validation for the transactions to be deserializable.
3. Now, when a block that contains any transaction that cannot be deserialized is processed before it is going to be executed, the deserialization showcased in reference (3) will fail and the execution of this block will not be possible. Ultimately, no other block will be executable either.
