# #43150 \[BC-High] Excessive transaction processing caused by a faulty garbage collector in transaction\_pipe.rs

**Submitted on Apr 2nd 2025 at 20:39:14 UTC by @avoloder for** [**Attackathon | Movement Labs**](https://immunefi.com/audit-competition/movement-labs-attackathon)

* **Report ID:** #43150
* **Report Type:** Blockchain/DLT
* **Report severity:** High
* **Target:** <https://github.com/immunefi-team/attackathon-movement/tree/main/protocol-units/execution/maptos/opt-executor>
* **Impacts:**
  * Causing network processing nodes to process transactions from the mempool beyond set parameters

## Description

## Brief/Intro

It is possible to submit more transactions to the mempool than initially allowed (beyond set parameter, i.e., beyond set limit) due to the miscalculation of transactions in flight (falty garbage collection logic)

## Vulnerability Details

`transaction_pipe.rs` is responsible for preprocessing and validating transactions before submitting them to the mempool. The `receive_transaction_tick()` function receives and adds all transactions to the mempool while also periodically garbage-collecting transactions in flight. A transaction is considered in flight until it exits the mempool and is sent to the DA. In `submit_transactions()`, there is a check to determine if the transaction-in-flight limit has been reached, indicating that the mempool is full and no further transactions should be processed at that moment.

```
// For now, we are going to consider a transaction in flight until it exits the mempool and is sent to the DA as is indicated by WriteBatch.
		let in_flight = {
			let transactions_in_flight = self.transactions_in_flight.read().unwrap();
			transactions_in_flight.get_count()
		};
		info!(
			target: "movement_timing",
			in_flight = %in_flight,
			"transactions_in_flight"
		);
		if let Some(inflight_limit) = self.in_flight_limit {
			if in_flight >= inflight_limit {
				info!(
					target: "movement_timing",
					"shedding_load"
				);
				let status = MempoolStatus::new(MempoolStatusCode::MempoolIsFull);
				return Ok((status, None));
			}
		}
```

We can see that the `transactions_in_flight` is of type `GcCounter`

```
// Shared reference on the counter of transactions in flight.
	transactions_in_flight: Arc<RwLock<GcCounter>>
```

Furthermore we can see that the GcCounter is a struct defined like this:

```
/// A garbage collected count, represented as a u64 sum.
/// As slots exit the active window and `gc` is called, they will no longer be included in the count.
/// Note: in the current implementation, the total sum returned by `get_count` is allowed to overflow.
pub struct GcCounter {
	/// The number of some unit time a value is valid for.
	value_ttl: Duration,
	/// The duration of a garbage collection slot in some unit time.
	/// This is used to bin values into slots for O(value_ttl/gc_slot_duration * log value_ttl/gc_slot_duration) garbage collection.
	gc_slot_duration: Duration,
	/// The value lifetimes, indexed by slot.
	value_lifetimes: BTreeMap<u64, u64>,
}
```

The `value_lifetimes` BTreeMap is used to count the number of transactions within a given time slot and sum them for the total count when checking the `inflight_limit`. This allows for tracking the overall number of transactions while also pruning expired ones.

The issue arises in the `gc()` function due to how it removes expired transactions

```
/// Garbage collects values that have expired.
	/// This should be called periodically.
	pub fn gc(&mut self, current_time: u64) {
		let gc_slot = current_time / self.gc_slot_duration.get();

		// remove all slots that are too old
		let slot_cutoff =
			gc_slot.saturating_sub(self.value_ttl.get() / self.gc_slot_duration.get());
		let to_keep = self.value_lifetimes.split_off(&(slot_cutoff + 1));

		// Now, `self.value_lifetimes` contains only entries with keys < `slot_cutoff`.
		// Reassign `self.value_lifetimes` to `to_keep` to keep only entries >= `slot_cutoff`.
		self.value_lifetimes = to_keep;
	}
```

The function calculates the `slot_cutoff`, that is the slot below which all transactions are considered old and will be removed. Afterwards it conduct `split_off` of BTreeMap `value_lifetimes` at key`slot_cutoff + 1`. The comments below indicate that after `split_off` the `value_lifetimes` contains entries with keys < `slot_cutoff` and then afterwards reassigns `value_lifetimes` to `to_keep`, where the expected behavior is to keep only entries >= `slot_cutoff`. This is however not true, considering how `split_off` function of a `BTreeMap` works. From the official documentation:

"Splits the collection into two at the given key. Returns everything after the given key, including the key".

The function calls `split_off` with the key `slot_cutoff + 1` which means that `to_keep` does not contain values at key `slot_cutoff` but rather all values after and equal to `slot_cutoff + 1`. After reassigning the `value_lifetimes` it will be `= to_keep` and the values equal and below `slot_cutoff` will be pruned, which contradicts the comment "Reassign `self.value_lifetimes` to `to_keep` to keep only entries >= `slot_cutoff`" and therefore does not fulfill a desired behavior

## Impact Details

The `gc()` function removes one slot more than needed affecting the total count of transactions in flight and therefore wrongly assume that fewer transactions are in flight than there actually are. This can lead to over-submitting new transactions beyond the intended limit (inflight\_limit), potentially overloading the mempool, since the system specifically relies on the `inflight_limit` to manage transactions load.

## References

<https://github.com/immunefi-team/attackathon-movement/blob/a2790c6ac17b7cf02a69aea172c2b38d2be8ce00/util/collections/src/garbage/counted.rs#L64-L78>

<https://github.com/immunefi-team/attackathon-movement/blob/a2790c6ac17b7cf02a69aea172c2b38d2be8ce00/protocol-units/execution/maptos/opt-executor/src/background/transaction\\_pipe.rs#L148-L153>

<https://github.com/immunefi-team/attackathon-movement/blob/a2790c6ac17b7cf02a69aea172c2b38d2be8ce00/protocol-units/execution/maptos/opt-executor/src/background/transaction\\_pipe.rs#L229-L248>

## Proof of Concept

## Proof of Concept

Let's assume the following:

`value_ttl: Duration::from_secs(10)` // Values valid for 10s\
`gc_slot_duration: Duration::from_secs(2)` // Each slot covers 2s

so we have the following `GcCounter`:

```
let mut counter = GcCounter {
    value_ttl: Duration::from_secs(10)
    gc_slot_duration: Duration::from_secs(2)
    value_lifetimes: BTreeMap::new()  // Initially empty
};
```

For the sake of simplicity let's assume we added values at seconds

```
counter.increment(1, 5);  // Time = 1s, value = 5
counter.increment(3, 10); // Time = 3s, value = 10
counter.increment(4, 20); // Time = 4s, value = 20
counter.increment(6, 3);  // Time = 6s, value = 3
counter.increment(10, 7);  // Time = 8s, value = 7
```

After doing slot calculation, given `gc_slot_duration = 2s` we have following slots and values in the BTreeMap:

```
{
    0 => 5,
    1  => 10, 
    2  => 20, 
    3 => 3,
    5 => 7
}
```

Calling a `gc()` at time `12s` results in the following values:

`gc_slot = 12s / 2s = 6s`\
`slot_cutoff = 6 - (10 / 2) = 1`

Slot cutoff is at the key 1, which means that only transactions count at slot 0 should be removed, since the end value of `value_lifetimes` should contain all entries `>= slot_cutoff`.

However, we call `split_off(1 + 1)`, where the `to_keep` will be the new BTreeMap with following values (according to the `split_off` definition, since we do split\_off at the key 2):

```
  {
    2  => 20, 
    3 => 3,
    5 => 7
  }
```

Now the `value_lifetimes` is reassigned to `to_keep` which does not contain transaction counts at slot `=slot_cutoff`, therefore removing one slot more than intended.

To simplify it further, if the `inflight_limit` is 500 and we have 5 equal slots each containing 100 transactions count, in this case, the `gc()` should remove only one slot, allowing another 100 transactions to be submitted before filling up the mempool. However, due to the above described, wrong expectations of the `split_off()` behavior, 200 transaction counts (2 slots) will be removed, allowing another 200 to enter the mempool (before reaching the limit of the mempool) which should not be the case.
