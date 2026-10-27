# #41023 \[BC-Insight] Incomplete transaction decrementing leading to undesired behaviour

**Submitted on Mar 9th 2025 at 16:56:43 UTC by @okmxuse for** [**Attackathon | Movement Labs**](https://immunefi.com/audit-competition/movement-labs-attackathon)

* **Report ID:** #41023
* **Report Type:** Blockchain/DLT
* **Report severity:** Insight
* **Target:** <https://github.com/immunefi-team/attackathon-movement/tree/main/networks/movement/movement-full-node>
* **Impacts:**
  * A bug in the respective layer 0/1/2 network code that results in unintended smart contract behavior with no concrete funds at direct risk

## Description

### Description

Inside `process_block_from_da`, transactions are retrieved and decremented by calling `decrement_transactions_in_flight`:

```javascript
let transactions_count = block.transactions().len();
let span = info_span!(target: "movement_timing", "execute_block", id = ?block_id);
let commitment =
    self.execute_block_with_retries(block, block_timestamp).instrument(span).await?;

// Decrement the number of transactions in flight on the executor
self.executor.decrement_transactions_in_flight(transactions_count as u64);
```

This function then calls `decrement`:

```javascript
pub fn decrement(&mut self, mut value: u64) {
    // Iterate over each slot
    for lifetime in self.value_lifetimes.values_mut() {
        if *lifetime > 0 {
            // Determine how much to decrement, ensuring it doesn't go below zero
            let decrement_amount = value.min(*lifetime);
            *lifetime -= decrement_amount;
            // Reduce the remaining value by what was actually decremented
            value -= decrement_amount;

            // If no remaining value to decrement, exit early
            if value == 0 {
                break;
            }
        }
    }
}
```

Here, transactions are decremented only when `*lifetime` is greater than 0. If no transactions are left, the function exits early. However, there is an edge case where no non-empty slots remain, yet some transactions remain unprocessed.

### Impact

The remaining 2 transactions are neither decremented nor accounted for anywhere else, they are simply forgotten while the function moves on. This can lead to inconsistencies in transaction handling, potentially causing mismatches in block processing. The block might include these transactions while they remain untracked due to improper decrementing.

For example, the logging statement indicates that 10 transactions are being decremented, but in reality, only 8 have been processed.

```javascript
		info!(
			target: "movement_timing",
			count,
			current,
			"decrementing_transactions_in_flight",
		);
	
```

### Recommendation

We would recommend handling such edge cases that could lead to mismatch of processed transactions when decrementing.

## Proof of Concept

### POC

Consider the following scenario:

1. `decrement_transactions_in_flight` is called with `transactions_count = 10`.
2. `decrement` is called with `value = 10`.
3. There are two lifetimes: one with a value of 8 and another with 0.
4. The first loop executes:

```javascript
for lifetime in self.value_lifetimes.values_mut() {
    // Entering the loop since lifetime is 8
    if *lifetime > 0 {
        // decrement_amount is 8
        let decrement_amount = value.min(*lifetime);
        // *lifetime becomes 0
        *lifetime -= decrement_amount;
        // value becomes 2
        value -= decrement_amount;

        // value != 0, so continue looping
        if value == 0 {
            break;
        }
    }
}
```

5. The second loop iteration begins, but since the second `*lifetime` is 0, the loop is skipped, and the function returns.

```javascript
// this will not execute this lifetime is 0
if *lifetime > 0 {
```
