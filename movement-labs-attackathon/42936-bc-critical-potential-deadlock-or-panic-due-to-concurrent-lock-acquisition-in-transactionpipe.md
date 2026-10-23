# #42936 \[BC-Critical] Potential Deadlock or Panic Due to Concurrent Lock Acquisition in \`TransactionPipe\`

**Submitted on Mar 29th 2025 at 18:29:09 UTC by @savi0ur for** [**Attackathon | Movement Labs**](https://immunefi.com/audit-competition/movement-labs-attackathon)

* **Report ID:** #42936
* **Report Type:** Blockchain/DLT
* **Report severity:** Critical
* **Target:** <https://github.com/immunefi-team/attackathon-movement/tree/main/protocol-units/execution/maptos/opt-executor>
* **Impacts:**
  * Network not being able to confirm new transactions (total network shutdown)

## Description

## Bug Description

The `TransactionPipe` struct manages incoming transactions and uses an `Arc<RwLock<GcCounter>>` to track transactions in flight. Multiple methods acquire this lock, and the code may panic or deadlock if the lock is acquired recursively or concurrently in a way that poisons it. Specifically, the `receive_transaction_tick` and `submit_transaction` methods acquire the lock with `write().unwrap()` without ensuring it isn’t already held, which could lead to a panic, due to `unwrap()` (e.g., when tried to acquired the write lock again).

<https://doc.rust-lang.org/std/sync/struct.RwLock.html#panics-1>

> **Panics**\
> This function might panic when called if the lock is already held by the current thread.

Here are the relevant code blocks:

1. In `receive_transaction_tick` during garbage collection:\
   <https://github.com/immunefi-team/attackathon-movement/blob/a2790c6ac17b7cf02a69aea172c2b38d2be8ce00/protocol-units/execution/maptos/opt-executor/src/background/transaction\\_pipe.rs#L151>

```rust
pub(crate) async fn receive_transaction_tick(&mut self) -> Result<(), Error> {
	// ...
	if self.last_gc.elapsed() >= GC_INTERVAL {
	    let now = Instant::now();
	    let epoch_ms_now = chrono::Utc::now().timestamp_millis() as u64;
	    // ...
	    {
	        let mut transactions_in_flight = self.transactions_in_flight.write().unwrap(); //@audit-issue this will panic when call again after holding the lock.
	        transactions_in_flight.gc(epoch_ms_now);
	    }
	    // ...
	    self.last_gc = now;
	}
```

2. In `submit_transaction` when incrementing transactions in flight:\
   <https://github.com/immunefi-team/attackathon-movement/blob/a2790c6ac17b7cf02a69aea172c2b38d2be8ce00/protocol-units/execution/maptos/opt-executor/src/background/transaction\\_pipe.rs#L296>

```rust
async fn submit_transaction(
	&mut self,
	transaction: SignedTransaction,
) -> Result<SubmissionStatus, Error> {
	// ...
	match status.code {
	    MempoolStatusCode::Accepted => {
	        let now = chrono::Utc::now().timestamp_millis() as u64;
	        // ...
	        {
	            let mut transactions_in_flight = self.transactions_in_flight.write().unwrap(); //@audit-issue this will panic when call again after holding the lock.
	            transactions_in_flight.increment(now, 1);
	        }
	        // ...
	    }
	}
```

3. In `submit_transaction` when reading the current count:\
   <https://github.com/immunefi-team/attackathon-movement/blob/a2790c6ac17b7cf02a69aea172c2b38d2be8ce00/protocol-units/execution/maptos/opt-executor/src/background/transaction\\_pipe.rs#L231>

```rust
let in_flight = {
    let transactions_in_flight = self.transactions_in_flight.read().unwrap(); //@audit-issue this will panic when call again after holding the lock.
    transactions_in_flight.get_count()
};
```

The use of `unwrap()` on `RwLock` operations means that if the lock is poisoned (e.g., due to a panic while held), or if a reentrant call occurs within the same thread, the program will crash. This is particularly risky in an async context where `TransactionPipe::tick` method is called in a loop, potentially leading to recursive or concurrent lock access, potentially leading to **deadlocks or unexpected behavior** if not properly coordinated.

<https://github.com/immunefi-team/attackathon-movement/blob/a2790c6ac17b7cf02a69aea172c2b38d2be8ce00/protocol-units/execution/maptos/opt-executor/src/background/transaction\\_pipe.rs#L99-L108>

```rust
pub async fn run(mut self) -> Result<(), Error> {
	loop {
		self.tick().await?;
	}
}

/// Performs a transaction read, mempool batch formation, and garbage collection.
pub(crate) async fn tick(&mut self) -> Result<(), Error> {
	self.receive_transaction_tick().await
}
```

## Impact

In a multi-threaded or async context, simultaneous attempts to acquire the write lock (e.g., during garbage collection and transaction submission) could lead to contention or deadlocks, stalling the system.

If the `RwLock` tried to acquired again in multi-threaded or async context, it will cause a panic due to `unwrap()`. Any subsequent `unwrap()` call will crash the application, halting transaction processing and potentially disrupting the mempool services.

## References

* <https://github.com/immunefi-team/attackathon-movement/blob/a2790c6ac17b7cf02a69aea172c2b38d2be8ce00/protocol-units/execution/maptos/opt-executor/src/background/transaction\\_pipe.rs#L151>
* <https://github.com/immunefi-team/attackathon-movement/blob/a2790c6ac17b7cf02a69aea172c2b38d2be8ce00/protocol-units/execution/maptos/opt-executor/src/background/transaction\\_pipe.rs#L296>
* <https://github.com/immunefi-team/attackathon-movement/blob/a2790c6ac17b7cf02a69aea172c2b38d2be8ce00/protocol-units/execution/maptos/opt-executor/src/background/transaction\\_pipe.rs#L231>
* <https://github.com/immunefi-team/attackathon-movement/blob/a2790c6ac17b7cf02a69aea172c2b38d2be8ce00/protocol-units/execution/maptos/opt-executor/src/background/transaction\\_pipe.rs#L99-L108>
* <https://doc.rust-lang.org/std/sync/struct.RwLock.html#panics-1>

## Recommendation

Avoid unwrapping locks by replacing `unwrap()` with proper error handling using `match` to gracefully handle lock acquisition.

```rust
let mut transactions_in_flight = match self.transactions_in_flight.write() {
    Ok(guard) => guard,
    Err(e) => {
        warn!("Failed to acquire write lock on transactions_in_flight: {:?}", e);
        return Err(Error::InternalError("Lock poisoned".to_string()));
    }
};
```

For read operations, use `try_read()` to avoid blocking or panicking if the lock is unavailable:

```rust
let in_flight = match self.transactions_in_flight.try_read() {
    Ok(guard) => guard.get_count(),
    Err(_) => {
        warn!("Failed to acquire read lock on transactions_in_flight");
        return Err(Error::InternalError("Lock contention".to_string()));
    }
};
```

## Proof of Concept

## Proof of Concept

1. An attacker can submit transactions to the `TransactionPipe` via the mempool client.
2. The `submit_transaction` method processes the transaction and acquires the write lock:

```rust
let mut transactions_in_flight = self.transactions_in_flight.write().unwrap();
transactions_in_flight.increment(now, 1);
```

3. The next call to `receive_transaction_tick` (e.g., during garbage collection) attempts to acquire the lock (***Note: this is very likely to happens in a multi-threaded and async environment***):

```rust
let mut transactions_in_flight = self.transactions_in_flight.write().unwrap(); // Panics here
transactions_in_flight.get_count()
```

The `unwrap()` panic and crash the application.

4. The node becomes unavailable, disrupting transaction processing and potentially affecting network consensus if multiple nodes are affected.
