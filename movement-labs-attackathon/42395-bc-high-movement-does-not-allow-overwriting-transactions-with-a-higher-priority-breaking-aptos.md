# #42395 \[BC-High] Movement does not allow overwriting transactions with a higher priority, breaking Aptos mempool logic

**Submitted on Mar 23rd 2025 at 16:01:58 UTC by @HollaDieWaldfee for** [**Attackathon | Movement Labs**](https://immunefi.com/audit-competition/movement-labs-attackathon)

* **Report ID:** #42395
* **Report Type:** Blockchain/DLT
* **Report severity:** High
* **Target:** <https://github.com/immunefi-team/attackathon-movement/tree/main/protocol-units/execution/maptos/opt-executor>
* **Impacts:**
  * Causing network processing nodes to process transactions from the mempool beyond set parameters

## Description

## Brief/Intro

When a new transaction is submitted, it is ensured that the sequence number of that transaction is greater than the sequence number of all of the other transactions that have been submitted before. This ensures that a transaction cannot be submitted if it has a sequence number that has already been used. However, Aptos allows submitting a transaction with an already used sequence number if the transaction is not executed yet and if the gas unit price is higher to speed up that transaction.

## Vulnerability Details

Let's have a look at what exactly happens.

Submitting a transaction adds it to the `core_mempool`. When inserting the transaction in the mempool, it is strictly allowed to insert a transaction that's already in the mempool if the gas unit price is higher. This can be verified by taking a look at reference (1).

Aptos allows this so that users can speed up their transactions. For example, a user may submit a transaction but realize it’s not being included quickly enough. By resubmitting the same transaction with a higher gas price, the transaction will be executed faster by the DA light node, which takes transactions from the mempool according to their priority. Allowing resubmission lets the user “update” the gas price to reflect current conditions. With this design, it is ensured that the mempool only stores the "best" version.

However, Movement does not allow submitting transactions with an already used sequence number even if the priority is higher.

## Impact Details

The impact can be considered from two different angles.

Firstly, Movement does not comply with how Aptos handles transactions that are already stored in the mempool. As a result, users cannot overwrite a transaction to increase its priority which might lead to that transaction not being executed. Movement utilizes the Aptos core mempool but then breaks the Aptos functionality.

A second point of view is to consider the reasons why blockchains, including Aptos and Ethereum allow such re-submissions of transactions. As mentioned before, one reason is so that users can bump their transaction fees and get their transactions executed faster. This also applies to Aptos because the Aptos sequencer processes transactions based on their priority.\
But this is just the most straightforward reason. Consider if users can't bump their gas fees, then transactions can be censored by other actors that submit transactions with higher gas fees.\
Also, advanced trading and arbitrage strategies rely on being able to bump transactions fees.

So overall, the issue is submitted under the "Causing network processing nodes to process transactions from the mempool beyond set parameters" impact. We believe such impact is justified as the underlying Aptos core mempool can be considered to provide the "set parameters" and so the full node fails to process transactions that should be considered valid based on the validation rules in the core mempool (i.e. transactions with a higher gas fee).

## References

(1): <https://github.com/immunefi-team/attackathon-movement-aptos-core/blob/627b4f9e0b63c33746fa5dae6cd672cbee3d8631/mempool/src/core\\_mempool/transaction\\_store.rs#L192-L232>

(2): <https://github.com/immunefi-team/attackathon-movement/blob/a2790c6ac17b7cf02a69aea172c2b38d2be8ce00/protocol-units/execution/maptos/opt-executor/src/background/transaction\\_pipe.rs#L124>

(3): <https://github.com/immunefi-team/attackathon-movement/blob/a2790c6ac17b7cf02a69aea172c2b38d2be8ce00/protocol-units/execution/maptos/opt-executor/src/background/transaction\\_pipe.rs#L164-L218>

(4): <https://github.com/immunefi-team/attackathon-movement-aptos-core/blob/627b4f9e0b63c33746fa5dae6cd672cbee3d8631/mempool/src/core\\_mempool/transaction\\_store.rs#L187-L270>

## Proof of Concept

## Proof of Concept

1. A new transaction is submitted when `transaction_pipe::submit_transaction()` is called (reference (2)).
2. It is ensured that the sequence number of this transaction has not been used already and no transaction with the same sequence number is in the mempool by calling `has_invalid_sequence_number()`. This can be verified by taking a look at reference (3).
3. However, as can be seen in reference (4), the `core_mempool` allows the same transaction to be sent if the gas unit price is higher.
