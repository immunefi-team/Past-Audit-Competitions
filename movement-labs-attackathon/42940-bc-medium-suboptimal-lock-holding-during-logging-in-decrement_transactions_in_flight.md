# #42940 \[BC-Medium] Suboptimal Lock Holding During Logging in \`decrement\_transactions\_in\_flight\`

**Submitted on Mar 29th 2025 at 20:03:42 UTC by @savi0ur for** [**Attackathon | Movement Labs**](https://immunefi.com/audit-competition/movement-labs-attackathon)

* **Report ID:** #42940
* **Report Type:** Blockchain/DLT
* **Report severity:** Medium
* **Target:** <https://github.com/immunefi-team/attackathon-movement/tree/main/protocol-units/execution/maptos/opt-executor>
* **Impacts:**
  * Increasing network processing node resource consumption by at least 30% without brute force actions, compared to the preceding 24 hours

## Description

## Bug Description

The `decrement_transactions_in_flight` method acquires a write lock on `self.transactions_in_flight` using `write().unwrap()`, performs a read operation (`get_count()`), logs information with `info!`, and then modifies the state with `decrement(count)`. The logging statement i.e., `info!`, could be a relatively slow operation (due to sending the logs over network or writing it in a file on disk), which is executed while holding the lock, unnecessarily extending the lock's critical section.

Here’s the relevant code block:\
<https://github.com/immunefi-team/attackathon-movement/blob/a2790c6ac17b7cf02a69aea172c2b38d2be8ce00/protocol-units/execution/maptos/opt-executor/src/executor/mod.rs#L43-L54>

```rust
pub fn decrement_transactions_in_flight(&self, count: u64) {
    // unwrap because lock is poisoned
    let mut transactions_in_flight = self.transactions_in_flight.write().unwrap(); //@audit optimization for lock by moving info! after releasing the lock.
    let current = transactions_in_flight.get_count();
    info!(
        target: "movement_timing",
        count,
        current,
        "decrementing_transactions_in_flight",
    );
    transactions_in_flight.decrement(count);
}
```

* The `RwLock` write guard (`transactions_in_flight`) is held for the entire duration of the method, including during the `info!` macro execution.
* Logging operations, especially with `tracing::info!`, may involve I/O (e.g., writing to a file or network sink), which can be slow compared to the simple counter operations (`get_count` and `decrement`).
* This unnecessarily prolongs the time the lock is held, increasing contention in a concurrent environment where other threads or tasks might need to access `transactions_in_flight`.

## Impact

Holding the write lock longer than necessary prevents other threads or async tasks from reading or writing to `transactions_in_flight`, potentially causing delays in transaction processing or state updates.

## Recommendation

Move the logging operation outside the critical section by releasing the lock before calling `info!`. This can be achieved by scoping the lock usage to only the operations that require it (`get_count` and `decrement`), then logging afterward. Here’s a refactored version:

```rust
pub fn decrement_transactions_in_flight(&self, count: u64) {
    let current = {
        let mut transactions_in_flight = self.transactions_in_flight.write().unwrap();
        let current = transactions_in_flight.get_count();
        transactions_in_flight.decrement(count);
        current // Return current for logging
    }; // write lock release here
    info!(
        target: "movement_timing",
        count,
        current,
        "decrementing_transactions_in_flight",
    );
}
```

This approach ensures the lock is held for the shortest possible time and adds robustness against poisoned locks.

## Proof of Concept

## Proof of Concept

The "attack" scenario demonstrates how an attacker could amplify the inefficiency to degrade system performance.

1. Assume a blockchain node running the `Executor` processes a high volume of transactions, frequently calling `decrement_transactions_in_flight`. The system uses a logging configuration that writes to a slow sink (e.g., a file on a busy disk or a remote server).
2. An attacker floods the system with transactions that trigger frequent calls to `decrement_transactions_in_flight`. For example, submit numerous small transactions that quickly enter and exit the in-flight state:

   ```rust
   for _ in 0..10000 {
       executor.decrement_transactions_in_flight(1);
   }
   ```
3. In the original code:
   * Each call locks `transactions_in_flight`, retrieves `current`, logs via `info!` (potentially taking milliseconds if the sink is slow), and then decrements.
   * With 10,000 calls and a slow log operation (e.g., 1ms each), the lock is held for \~10 seconds cumulatively.
   * Other threads or tasks needing to access `transactions_in_flight` (e.g., to increment or read the count) are blocked during this time.
4. If the logging sink is very slow (e.g., a remote server with high latency or a contended disk), then:
   * Logging to a network sink with 10ms latency per write.
   * Each `info!` call takes 10ms, extending the lock duration to \~100 seconds for 10,000 calls.
