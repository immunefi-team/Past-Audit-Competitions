# #43190 \[BC-Critical] Deadlock in \`submit\_transaction()\`

**Submitted on Apr 3rd 2025 at 14:45:52 UTC by @br0nz3p1ck4x3 for** [**Attackathon | Movement Labs**](https://immunefi.com/audit-competition/movement-labs-attackathon)

* **Report ID:** #43190
* **Report Type:** Blockchain/DLT
* **Report severity:** Critical
* **Target:** <https://github.com/immunefi-team/attackathon-movement/tree/main/protocol-units/execution/maptos/opt-executor>
* **Impacts:**
  * Temporary freezing of network transactions by delaying one block by 500% or more of the average block time of the preceding 24 hours beyond standard difficulty adjustments

## Description

## Description

Inside `submit_transactions()`, the variable `transactions_in_flight` will be written and read to. In the following code-block, a `read()` is done:

```rust
let in_flight = {
			let transactions_in_flight = self.transactions_in_flight.read().unwrap();
			transactions_in_flight.get_count()
		};
```

A few lines below the `read()`, we can find `write()`

```rust
				{
					let mut transactions_in_flight = self.transactions_in_flight.write().unwrap();
					transactions_in_flight.increment(now, 1);
				}
```

The problem here is the dangerous usage of the `read()` and the `write()` methods. In Rust, a lock is placed during the `read()` and the `write()` operation. If the current thread attempts an access on a lock held by a current thread, this will cause a panic. We can see the following from the standard library

<https://github.com/rust-lang/rust/blob/e0883a2a6c8e4afcb6e26a182885f687ba63a7f5/library/std/src/sync/poison/rwlock.rs#L441>

```rust
	/// # Panics
    ///
    /// This function might panic when called if the lock is already held by the current thread.
```

As such, the `submit_transaction()` function is susceptible to a deadlock when it's under heavy load, which will lead to a panic and break the sequencer.

## Impact

The deadlock will stall the sequencer, degrading the network performance and/or stalling the chain, depending on which sequencer is stalled.

## Recommended Mitigation Steps

Use common patterns that eliminate the chance of deadlock when using read/write locks.

## Proof of Concept

## Proof of Concept

This proof of concept is straight-forward. This will happen under heavy workload because this will increase the chances of a thread trying to write/read at the same time.

Step one: Network is digested, `sequencer_x` is under heavy workload

Step two: `thread_12` tries to `read()` the variable `transactions_in_flight()` while it still has a lock from a previous `write()` call.

Step three: A panic will happen, leading to a deadlock and stalling `sequencer_x`
