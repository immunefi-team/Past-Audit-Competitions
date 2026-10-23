# #42535 \[BC-High] Garbage collecting in flight transactions can lead to spiraling network delays

**Submitted on Mar 24th 2025 at 14:54:02 UTC by @HollaDieWaldfee for** [**Attackathon | Movement Labs**](https://immunefi.com/audit-competition/movement-labs-attackathon)

* **Report ID:** #42535
* **Report Type:** Blockchain/DLT
* **Report severity:** High
* **Target:** <https://github.com/immunefi-team/attackathon-movement/tree/main/protocol-units/execution/maptos/opt-executor>
* **Impacts:**
  * Temporary freezing of network transactions by delaying one block by 500% or more of the average block time of the preceding 24 hours beyond standard difficulty adjustments

## Description

## Brief/Intro

When a new transaction is submitted, it is ensured that the mempool is not full. The `inflight_limit` represents the maximum amount of transactions that can be in the mempool at the same time. The `transactions_in_flight` variable represents the amount of transactions currently in the mempool. This value is incremented when a new transaction is submitted and decremented when a block is executed. However, there is also a garbage collector that removes expired transactions. The purpose of garbage collection is to reduce the number of in flight transactions if the transactions get dropped and not executed. The problem is that the mechanism of garbage collecting allows submitting more transactions, increasing network delays, causing more transactions to be dropped. This creates a loop where the nework cannot recover from transaction overload.

## Vulnerability Details

Let's take a closer look at how this occurs.

The `transactions_in_flight` value changes at three points:

1. Submitting a transaction increments `transactions_in_flight` by one.
2. Executing a block decrements the amount of `transactions_in_flight` by the number of transactions included in that block.
3. Garbage collection decreases `transactions_in_flight` by the number of expired transactions.

Essentially, there is one way to increase `transactions_in_flight`, but two ways to decrease it. However, these two decrementing mechanisms do not exclude each other, meaning that the same transaction could cause `transactions_in_flight` to be decremented twice.

For example, a transactions could be included in a block that is yet to be executed. That transaction is stored in a slot which expires which means that it is going to be garbage collected and the `transactions_in_flight` is decreased. However, once the block is executed, the transaction will cause the `transactions_in_flight` to decrease again. As a result, the amount of `transactions_in_flight` will be lower than it should be and more transactions will be allowed in the mempool than the `inflight_limit`.

The additional transactions then cause further load on the network, causing further delays and garbage collection of in-progress transactions. This creates the loop where `transactions_in_flight` provides no protection against network overload because the increasing load gives rise to yet more transactions.

## Impact Details

Consider a scenario where a block is not executed due to some network delay. When the slots containing these transactions expire and are garbage collected, new transactions can be submitted. This causes the network delay to increase and `transactions_in_flight` to be decreased. Due to this loop, the transaction load on the network increases, undermining the purpose of `transactions_in_flight`, which is to limit the network load.

A possible direction to solve the problem is to let the sequencer node report when transactions get dropped and to not garbage collect without knowing whether the transaction really got dropped. This prevents double-decrementing of `transactions_in_flight` and also reduces the mempool load until the full node can really be sure that the transaction got dropped.

## References

(1): <https://github.com/immunefi-team/attackathon-movement/blob/a2790c6ac17b7cf02a69aea172c2b38d2be8ce00/protocol-units/execution/maptos/opt-executor/src/background/transaction\\_pipe.rs#L295-L298>

(2): <https://github.com/immunefi-team/attackathon-movement/blob/a2790c6ac17b7cf02a69aea172c2b38d2be8ce00/protocol-units/sequencing/memseq/sequencer/src/lib.rs#L101-L131>

(3): <https://github.com/immunefi-team/attackathon-movement/blob/a2790c6ac17b7cf02a69aea172c2b38d2be8ce00/util/collections/src/garbage/counted.rs#L66-L77>

(4): <https://github.com/immunefi-team/attackathon-movement/blob/a2790c6ac17b7cf02a69aea172c2b38d2be8ce00/networks/movement/movement-full-node/src/node/tasks/execute\\_settle.rs#L153-L154>

## Proof of Concept

## Proof of Concept

1. A new transaction is submitted when `transaction_pipe::submit_transaction()` is called. This increments the `transactions_in_flight` (reference (1)).
2. A block that includes this transaction is created (reference (2)).
3. The block will not be executed due to some network delay and the slot containing this transaction expires. This slot is garbage collected which decreases the amount of `transactions_in_flight`. To verify this, you can take a look at reference (3). Now, new transactions can be submitted.
4. The block is now executed which decrements the `transactions_in_flight` again (reference (4)). As a result, for every transaction, two new transactions can be submitted which further increases the network delay.
5. This creates a loop of increasing network delays and network load.
