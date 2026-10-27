# #42938 \[BC-Insight] Inefficient Garbage Collection Implementation in \`UsedSequenceNumberPool\`

**Submitted on Mar 29th 2025 at 19:23:51 UTC by @savi0ur for** [**Attackathon | Movement Labs**](https://immunefi.com/audit-competition/movement-labs-attackathon)

* **Report ID:** #42938
* **Report Type:** Blockchain/DLT
* **Report severity:** Insight
* **Target:** <https://github.com/immunefi-team/attackathon-movement/tree/main/protocol-units/execution/maptos/opt-executor>
* **Impacts:**

## Description

## Bug Description

The `gc` method in the `UsedSequenceNumberPool` struct uses an inefficient approach to collect expired sequence number slots. The current implementation employs `take_while` followed by `cloned` and a separate loop to remove slots, resulting in unnecessary cloning of data and two iterations over the slot collection.

Here’s the relevant code block:\
<https://github.com/immunefi-team/attackathon-movement/blob/a2790c6ac17b7cf02a69aea172c2b38d2be8ce00/protocol-units/execution/maptos/opt-executor/src/gc\\_account\\_sequence\\_number.rs#L74-L94>

```rust
pub(crate) fn gc(&mut self, current_time_ms: u64) {
    let gc_slot = current_time_ms / self.gc_slot_duration_ms;
    let slot_cutoff = gc_slot - self.sequence_number_ttl_ms / self.gc_slot_duration_ms;
    let slots_to_remove: Vec<u64> = self
        .sequence_number_lifetimes
        .keys()
        .take_while(|slot| **slot < slot_cutoff)
        .cloned()
        .collect(); //@audit same thing can be done using retain, helps in avoiding clone and two loop cycles
    for slot in slots_to_remove {
        debug!(
            "Garbage collecting sequence number slot {} with duration {} timestamp {}",
            slot,
            self.gc_slot_duration_ms,
            slot * self.gc_slot_duration_ms
        );
        self.sequence_number_lifetimes.remove(&slot);
    }
}
```

* `take_while` iterates over the keys until the condition fails, collecting all slots less than `slot_cutoff`.
* `cloned()` creates a copy of each key, which is unnecessary since the keys are only needed for removal.
* `collect()` builds a `Vec`, followed by a second loop (for slot in `slots_to_remove`) to perform the removals.

This approach is less efficient than using `retain`, which could iterate once and directly identify slots for removal without cloning.

## Impact

* The unnecessary cloning and dual iteration increase CPU and memory usage, especially when `sequence_number_lifetimes` contains many slots. This could slow down garbage collection, particularly in high-throughput systems processing numerous transactions.
* As the number of slots grows, the inefficiency becomes more pronounced, potentially causing delays in transaction processing or increased resource consumption on the node.

While this is not a security vulnerability, it represents a performance bug that could degrade system efficiency under load.

## Recommendation

Consider using `retain` to modify the `sequence_number_lifetimes` in place:

```rust
pub(crate) fn gc(&mut self, current_time_ms: u64) {
    let gc_slot = current_time_ms / self.gc_slot_duration_ms;
    let slot_cutoff = gc_slot - self.sequence_number_ttl_ms / self.gc_slot_duration_ms;
    self.sequence_number_lifetimes.retain(|slot, _| {
        if *slot < slot_cutoff {
            debug!(
                "Garbage collecting sequence number slot {} with duration {} timestamp {}",
                slot,
                self.gc_slot_duration_ms,
                slot * self.gc_slot_duration_ms
            );
            false // Remove this slot
        } else {
            true // Keep this slot
        }
    });
}
```

This avoids creating an intermediate `Vec` entirely, performing the removal directly.

## Proof of Concept

## Proof Of Concept

* In the original `gc` method implementation, `sequence_number_lifetimes` is iterated using `take_while` first, which is cloning values and then all the filtered slots are collected in a newly memory allocated vector i.e., `slots_to_remove`.
* `slots_to_remove` is then again iterated to remove each elements from the `sequence_number_lifetimes`.
* This is wasting memory as well as compute, as its allocating separate vector to hold the slots that needs to be remove (i.e., `slots_to_remove`) and also its performing two cycles to complete the operation.
* Whereas in improved `gc` implementation, everything is done `in-place`, i.e., directly updating `sequence_number_lifetimes` without allocating any addition memory using `clone` or for vector.
