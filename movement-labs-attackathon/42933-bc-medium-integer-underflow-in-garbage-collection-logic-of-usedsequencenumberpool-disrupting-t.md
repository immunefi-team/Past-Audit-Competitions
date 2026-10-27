# #42933 \[BC-Medium] Integer Underflow in Garbage Collection Logic of UsedSequenceNumberPool disrupting transaction processing

**Submitted on Mar 29th 2025 at 18:04:53 UTC by @savi0ur for** [**Attackathon | Movement Labs**](https://immunefi.com/audit-competition/movement-labs-attackathon)

* **Report ID:** #42933
* **Report Type:** Blockchain/DLT
* **Report severity:** Medium
* **Target:** <https://github.com/immunefi-team/attackathon-movement/tree/main/protocol-units/execution/maptos/opt-executor>
* **Impacts:**
  * Temporary freezing of network transactions by delaying one block by 500% or more of the average block time of the preceding 24 hours beyond standard difficulty adjustments

## Description

## Bug Description

The `gc` method in the `UsedSequenceNumberPool` struct contains a vulnerability that arises in the calculation of `slot_cutoff`, which determines the threshold for removing expired sequence number slots. When `current_time_ms` is less than `gc_slot_duration_ms`, the division and subtraction operations can cause an integer underflow, leading to incorrect garbage collection behavior.

Here’s the problematic code block in Rust:\
<https://github.com/immunefi-team/attackathon-movement/blob/a2790c6ac17b7cf02a69aea172c2b38d2be8ce00/protocol-units/execution/maptos/opt-executor/src/gc\\_account\\_sequence\\_number.rs#L78>

```rust
pub(crate) fn gc(&mut self, current_time_ms: u64) {
    let gc_slot = current_time_ms / self.gc_slot_duration_ms;
    let slot_cutoff = gc_slot - self.sequence_number_ttl_ms / self.gc_slot_duration_ms; //@audit-issue
    let slots_to_remove: Vec<u64> = self
        .sequence_number_lifetimes
        .keys()
        .take_while(|slot| **slot < slot_cutoff)
        .cloned()
        .collect();
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

The expression `gc_slot - self.sequence_number_ttl_ms / self.gc_slot_duration_ms` assumes that `gc_slot` (computed as `current_time_ms / self.gc_slot_duration_ms`) will always be large enough to avoid underflow. However, if `current_time_ms < self.gc_slot_duration_ms`, `gc_slot` becomes 0, and subtracting a positive value (`self.sequence_number_ttl_ms / self.gc_slot_duration_ms`) results in an underflow. In Rust, this wraps around to a large positive u64 value (e.g., `u64::MAX - n`), causing `slot < slot_cutoff` to always evaluate to true. Consequently, all sequence number slots are incorrectly marked for removal as it thinks all slots are expired.

The vulnerability is particularly severe in early system states (e.g., shortly after startup) or if time values are misconfigured, as these scenarios increase the likelihood of `current_time_ms` being small.

## Impact

This bug can lead to:

* All sequence numbers in the pool are erroneously garbage collected, even if they are still within their valid TTL (`sequence_number_ttl_ms`), disrupting transaction tracking for accounts.
* Loss of sequence numbers may cause transaction replay issues or inconsistencies in the blockchain state, as sequence numbers are critical for ensuring transaction order and uniqueness.

## References

<https://github.com/immunefi-team/attackathon-movement/blob/a2790c6ac17b7cf02a69aea172c2b38d2be8ce00/protocol-units/execution/maptos/opt-executor/src/gc\\_account\\_sequence\\_number.rs#L78>

## Recommendation

* Use saturating subtraction or explicit checks to ensure `slot_cutoff` does not underflow. For example:

```rust
let slot_cutoff = gc_slot.saturating_sub(self.sequence_number_ttl_ms / self.gc_slot_duration_ms);
```

This ensures `slot_cutoff` never goes below 0.

* Check if `current_time_ms` is too small to perform garbage collection:

```rust
if current_time_ms < self.gc_slot_duration_ms {
    debug!("Skipping GC: current_time_ms too small for garbage collection");
    return;
}
let gc_slot = current_time_ms / self.gc_slot_duration_ms;
let slot_cutoff = gc_slot - self.sequence_number_ttl_ms / self.gc_slot_duration_ms;
```

## Proof of Concept

## Proof of Concept

1. For this PoC, Assume a `UsedSequenceNumberPool` instance is initialized with typical parameters, such as `sequence_number_ttl_ms = 1000` (1 second) and `gc_slot_duration_ms = 100` (100 milliseconds).

```rust
let mut pool = UsedSequenceNumberPool::new(1000, 100);
```

2. Add sequence numbers for two accounts at a valid timestamp (e.g., 500 ms).

```rust
let account1 = AccountAddress::random();
let account2 = AccountAddress::random();
pool.set_sequence_number(&account1, 1, 500);
pool.set_sequence_number(&account2, 2, 500);
assert_eq!(pool.get_sequence_number(&account1), Some(1));
assert_eq!(pool.get_sequence_number(&account2), Some(2));
```

3. Call `gc` with `current_time_ms = 50`, which is less than `gc_slot_duration_ms = 100`.

```rust
pool.gc(50);
```

4. Underflow Occurs
   * `gc_slot = 50 / 100 = 0`
   * `self.sequence_number_ttl_ms / self.gc_slot_duration_ms = 1000 / 100 = 10`
   * `slot_cutoff = 0 - 10`, which underflows to `u64::MAX - 10` (a very large number, approximately `18446744073709551605`)
   * For any slot in `sequence_number_lifetimes` e.g., `slot < 18446744073709551605` is always true.
5. The `take_while` condition collects all slots and they are removed:

```rust
assert_eq!(pool.get_sequence_number(&account1), None);
assert_eq!(pool.get_sequence_number(&account2), None);
```

6. Despite the sequence numbers being set at `500 ms` and still within the `1000 ms` TTL relative to a normal timeline, they are incorrectly garbage collected due to the underflow. An attacker could repeat this by ensuring `gc` is called with a low `current_time_ms`, effectively wiping the pool and disrupting transaction processing.
