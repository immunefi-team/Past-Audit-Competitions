# #41978 \[BC-Insight] Values of the current gc\_slot can be garbage collected in edge case

**Submitted on Mar 19th 2025 at 18:30:52 UTC by @HollaDieWaldfee for** [**Attackathon | Movement Labs**](https://immunefi.com/audit-competition/movement-labs-attackathon)

* **Report ID:** #41978
* **Report Type:** Blockchain/DLT
* **Report severity:** Insight
* **Target:** <https://github.com/immunefi-team/attackathon-movement/tree/main/util/collections>
* **Impacts:**
  * Causing network processing nodes to process transactions from the mempool beyond set parameters

## Description

## Brief/Intro

When the transactions in flight are garbage collected, values that have expired should be removed. But in certain cases, this even removes values of the current gc\_slot that have not expired yet and have just been added.

## Vulnerability Details

Let's take a look at how this happens.

Every time a transaction is received and added to the mempool, the transactions in flight are garbage collected, if enough time has passed after the last time they have been garbage collected. This can be verified by taking a look at reference (1).

Inside of `gc()`, all slots that are too old are removed. However, if `self.value_ttl.get() / self.gc_slot_duration.get()` rounds down to zero, the current slot will be removed as well even though it has not expired yet.

As a result, values that have not expired will be garbage collected.

In contrast, the garbage collector for the `used_sequence_number_pool` correctly only removes slots until the current one. To verify this, please take a look at reference (4). Given this difference to reference (4), the garbage collection for in-flight transactions is clearly flawed and does not meet the security guarantees - that the current slot can't be removed - that the sequence number garbage collector meets.

## Impact Details

Values that have not expired will be garbage collected. Hence, the amount of transactions in flight will be lower than it should and more transactions are allowed to be submitted.

## References

(1): <https://github.com/immunefi-team/attackathon-movement/blob/a2790c6ac17b7cf02a69aea172c2b38d2be8ce00/protocol-units/execution/maptos/opt-executor/src/background/transaction\\_pipe.rs#L140-L153\\>
(2): <https://github.com/immunefi-team/attackathon-movement/blob/a2790c6ac17b7cf02a69aea172c2b38d2be8ce00/util/collections/src/garbage/counted.rs#L70-L71\\>
(3): <https://github.com/immunefi-team/attackathon-movement/blob/a2790c6ac17b7cf02a69aea172c2b38d2be8ce00/util/collections/src/garbage/counted.rs#L72-L76\\>
(4): <https://github.com/immunefi-team/attackathon-movement/blob/a2790c6ac17b7cf02a69aea172c2b38d2be8ce00/protocol-units/execution/maptos/opt-executor/src/gc\\_account\\_sequence\\_number.rs#L79-L84>

## Proof of Concept

## Proof of Concept

1. A new transaction is received and successfully submitted. `transactions_in_flight.gc()` is called since `self.last_gc.elapsed() >= GC_INTERVAL`. Again, this can be verified by taking a look at reference (1).
2. As showcased in reference (2), this function calculates the `slot_cutoff` in the following way: `gc_slot.saturating_sub(self.value_ttl.get() / self.gc_slot_duration.get())`. If `self.value_ttl.get() / self.gc_slot_duration.get()` rounds down to zero, the `slot_cutoff` will be the current `gc_slot`. This means that only the slots after the cutoff will be kept and all of the other slots, including the current slot, wll be removed (reference (3)).
3. Now, the current slot will be removed even though it has not expired yet. As a result, the amount of transactions in flight will be decreased by too much and allows more transactions to be submitted than it should.
