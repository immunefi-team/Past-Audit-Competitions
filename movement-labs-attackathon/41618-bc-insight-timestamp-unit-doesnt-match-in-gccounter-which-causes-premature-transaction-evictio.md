# #41618 \[BC-Insight] Timestamp unit doesn't match in GcCounter which causes premature transaction eviction

**Submitted on Mar 17th 2025 at 03:48:13 UTC by @Rhaydden for** [**Attackathon | Movement Labs**](https://immunefi.com/audit-competition/movement-labs-attackathon)

* **Report ID:** #41618
* **Report Type:** Blockchain/DLT
* **Report severity:** Insight
* **Target:** <https://github.com/immunefi-team/attackathon-movement/tree/main/util/collections>
* **Impacts:**
  * A bug in the respective layer 0/1/2 network code that results in unintended smart contract behavior with no concrete funds at direct risk

## Description

## Brief/Intro

The problem is in the GcCounter impl where timestamps stored in slot units are directly compared against cutoff times in raw time units. This mismatch causes premature garbage collection of entries, resulting in the protocol losing track of in-flight transactions.

## Vulnerability Details

In the increment method, timestamps are stored after division by the garbage collection slot duration:

```rust
let slot_timestamp = current_time / self.gc_slot_duration.get();
```

However, in gc, the comparison is done against raw time units:

```rust
pub fn gc(&self, current_time: u64) {
    let cutoff_time = current_time - self.value_ttl.get(); // Raw time units
    for (slot_timestamp, count) in self.value_lifetimes.iter() {
        if slot_timestamp.load(Ordering::Relaxed) <= cutoff_time { // BUG: Comparing slot units to raw units
            slot_timestamp.store(0, Ordering::SeqCst);
            count.store(0, Ordering::SeqCst);
        }
    }
}
```

This causes entries to be evicted much earlier than intended because slot unit timestamps (which are divided by `gc_slot_duration`) are being compared against much larger raw time values. Since slot timestamps are typically much smaller numbers (as they're divided by `gc_slot_duration`), they will almost always be less than the raw cutoff time making entries to be evicted prematurely.

## Impact Details

Transactions being tracked by the GcCounter will be prematurely evicted from the tracking system. This is particularly important as the GcCounter is used to track "transactions in flight" in the protocol's transaction pipeline. The protocol will lose track of transactions that are still being processed. This could result in duplicate transaction processing or transactions being dropped entirely.

## References

<https://github.com/immunefi-team/attackathon-movement//blob/a2790c6ac17b7cf02a69aea172c2b38d2be8ce00/util/collections/src/garbage/atomic/counted.rs#L57>

<https://github.com/immunefi-team/attackathon-movement//blob/a2790c6ac17b7cf02a69aea172c2b38d2be8ce00/util/collections/src/garbage/atomic/counted.rs#L96-L105>

## Proof of Concept

## Proof of Concept

Heres an easy illustration of how this works:

````


```plaintext
Scenario: gc_slot_duration = 10, value_ttl = 100

Timeline (Real Time Units)
--------------------------
0         100        200        300        400        500        600        700        800        900       1000
|---------|---------|---------|---------|---------|---------|---------|---------|---------|---------|
          ^                                                                               ^         ^
          |                                                                               |         |
    Entry stored here                                                               Should expire   Current time
                                                                                    (at t=900)

How it's STORED (Slot Units = Real Time / 10):
---------------------------------------------
0    10    20    30    40    50    60    70    80    90    100
|-----|-----|-----|-----|-----|-----|-----|-----|-----|-----|
      ^                                                      ^
      |                                                      |
Entry stored at slot 10                                Current slot 100
(real time 100 / slot_duration 10 = slot 10)

THE BUG:
--------
When GC runs at time 1000:
- Stored timestamp = 10 (because 100/10 = 10)
- Cutoff calculation = 1000 - 100 = 900 (in real time units)
- Comparison: 10 <= 900  // TRUE! But shouldn't be!
- Result: Entry gets incorrectly evicted

CORRECT BEHAVIOR:
----------------
When GC runs at time 1000:
- Stored timestamp = 10 (in slot units)
- Cutoff calculation = (1000 - 100)/10 = 90 (converted to slot units)
- Comparison: 10 <= 90  // TRUE, and correct!
- Result: Entry gets evicted at the right time

Impact Example:
--------------
Real time: 500
Entry timestamp in slots: 50 (500/10)
Cutoff time (wrong): 500 - 100 = 400
Compare: 50 <= 400 ✓ (INCORRECTLY EVICTS)

Real time: 500
Entry timestamp in slots: 50 (500/10)
Cutoff time (correct): (500 - 100)/10 = 40
Compare: 50 <= 40 ✗ (CORRECTLY KEEPS)
````

## Fix

Convert `cutoff_time` to slot units for proper comparison.

```diff
pub fn gc(&self, current_time: u64) {
-		let cutoff_time = current_time - self.value_ttl.get();
+       let cutoff_time = (current_time - self.value_ttl.get()) / self.gc_slot_duration.get();    

		for (slot_timestamp, count) in self.value_lifetimes.iter() {
			if slot_timestamp.load(Ordering::Relaxed) <= cutoff_time {
				slot_timestamp.store(0, Ordering::SeqCst);
				count.store(0, Ordering::SeqCst);
			}
```
