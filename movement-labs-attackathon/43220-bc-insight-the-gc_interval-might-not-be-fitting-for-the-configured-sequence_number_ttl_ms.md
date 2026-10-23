# #43220 \[BC-Insight] The GC\_INTERVAL might not be fitting for the configured sequence\_number\_ttl\_ms

**Submitted on Apr 3rd 2025 at 19:45:13 UTC by @KlosMitSoss for** [**Attackathon | Movement Labs**](https://immunefi.com/audit-competition/movement-labs-attackathon)

* **Report ID:** #43220
* **Report Type:** Blockchain/DLT
* **Report severity:** Insight
* **Target:** <https://github.com/immunefi-team/attackathon-movement/tree/main/protocol-units/execution/maptos/opt-executor>
* **Impacts:**

## Description

## Brief/Intro

While GC\_INTERVAL is a constant that equals 30 seconds, the `sequence_number_ttl_ms` can be configured. As a result, sequence numbers that expire after 1 second can still only be garbage collected every 30 seconds.

## Vulnerability Details

The `used_sequence_number_pool`, `transactions_in_flight` and `core_mempool` can only be garbage collected every 30 seconds.

```rust
	pub(crate) async fn receive_transaction_tick(&mut self) -> Result<(), Error> {
        ... ...
>>		if self.last_gc.elapsed() >= GC_INTERVAL {
			// todo: these will be slightly off, but gc does not need to be exact
			let now = Instant::now();
			let epoch_ms_now = chrono::Utc::now().timestamp_millis() as u64;

			// garbage collect the used sequence number pool
			self.used_sequence_number_pool.gc(epoch_ms_now);

			// garbage collect the transactions in flight
			{
				// unwrap because failure indicates poisoned lock
				let mut transactions_in_flight = self.transactions_in_flight.write().unwrap();
				transactions_in_flight.gc(epoch_ms_now);
			}

			// garbage collect the core mempool
			self.core_mempool.gc();

			self.last_gc = now;
		}
    ... ...
	}
```

<https://github.com/immunefi-team/attackathon-movement/blob/a2790c6ac17b7cf02a69aea172c2b38d2be8ce00/protocol-units/execution/maptos/opt-executor/src/background/transaction\\_pipe.rs#L112-L162>

However, since `sequence_number_ttl_ms` can be configured, a sequence number could already be expired after for example 1 second.

```rust
	pub(crate) fn gc(&mut self, current_time_ms: u64) {
		let gc_slot = current_time_ms / self.gc_slot_duration_ms;

		// remove all slots that are too old
>>		let slot_cutoff = gc_slot - self.sequence_number_ttl_ms / self.gc_slot_duration_ms;
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

<https://github.com/immunefi-team/attackathon-movement/blob/a2790c6ac17b7cf02a69aea172c2b38d2be8ce00/protocol-units/execution/maptos/opt-executor/src/gc\\_account\\_sequence\\_number.rs#L74-L94>

As a result, a sequence number could for example still be in the `used_sequence_number_pool` for 29 seconds after expiring.

To mitigate this, it should also be possible to configure the GC\_INTERVAL.

## Impact Details

A sequence number will remain in the `used_sequence_number_pool` even when they have been expired for a long time since GC\_INTERVAL is a constant that might not fit the configured `sequence_number_ttl_ms`.

## References

Code references are provided throughout the report.

## Proof of Concept

## Proof of Concept

1. `used_sequence_number_pool.set_sequence_number()` is called for a specifc sequence number.
2. This sequence number expires after 1 second.
3. The `used_sequence_number_pool` can only be garbage collected after 29 more seconds.
