# #43346 \[BC-Insight] Transactions arriving at the node out of sequence order will be rejected due to the has\_invalid\_sequence\_number function

**Submitted on Apr 4th 2025 at 17:54:10 UTC by @niroh for** [**Attackathon | Movement Labs**](https://immunefi.com/audit-competition/movement-labs-attackathon)

* **Report ID:** #43346
* **Report Type:** Blockchain/DLT
* **Report severity:** Insight
* **Target:** <https://github.com/immunefi-team/attackathon-movement/tree/main/protocol-units/execution/maptos/opt-executor>
* **Impacts:**
  * Temporary freezing of network transactions by delaying one block by 500% or more of the average block time of the preceding 24 hours beyond standard difficulty adjustments

## Description

## Vulnerability Details

When transactions arrive to the TransactionPipe, their sequence number is checked for validity in the has\_invalid\_sequence\_number function. The logic in the function is only accept transaction sequence numbers that are between the Min = max of the committed sn and the last used sn and Max = committed sn + tolerance (32) as can be seen in this code:

```rust
let used_sequence_number = self
        .used_sequence_number_pool
        .get_sequence_number(&transaction.sender())
        .unwrap_or(0);

    // validate against the state view
    let state_view = self.db_reader.latest_state_checkpoint_view().map_err(|e| {
        Error::InternalError(format!("Failed to get latest state view: {:?}", e))
    })?;

    // this checks that the sequence number is too old or too new
    let committed_sequence_number =
        vm_validator::get_account_sequence_number(&state_view, transaction.sender())?;

    debug!(
        "Used sequence number: {:?} Committed sequence number: {:?}",
        used_sequence_number, committed_sequence_number
    );
    let min_used_sequence_number =
        if used_sequence_number > 0 { used_sequence_number + 1 } else { 0 };

    let min_sequence_number = (min_used_sequence_number).max(committed_sequence_number);

    let max_sequence_number = committed_sequence_number + TOO_NEW_TOLERANCE;
```

The issue is that if an account sends multiple transactions with consecutive sns, and they arrive out of order, all the transactions the account sends will be dropped and not executed for the duration of the sequence number ttl (180 seconds).\
To see why, consider the following scenario:

1. Account A has committed sequence number 10. The account sends multiple transactions with sequence numbers 10, 11, 12, 13 ....
2. The first transaction to arrive at the node is 11. based on the has\_invalid\_sequence\_number function logic, the sn is valid because it is between 10 and 10+32
3. The transaction is also accepted by the core\_mempool because it is of a higher value than the committed sn and it is sent to the da node for processing. 11 is set as the last used sn in the used\_sequence\_number\_pool.
4. From this point on teh account is stuck because transaction with sn 10 will not get accepted by the has\_invalid\_sequence\_number logic (since now the last used sn is 11) and all transactions will fail during block execution because all transactions will be considered as sequence number too new.
5. The problem can only be resolved after the used\_sequence\_number\_pool ttl (180 seconds) elapse and transaction with sequence number 10 can be accepted by the node and sent to processing.

## Impact Details

Freezing of account for the duration of the ttl, as well as causing any transaction sent by the account is that time frame to fail

## Proof of Concept

## Proof of Concept

vulnerability scenario:

1. Account A has committed sequence number 10. The account sends multiple transactions with sequence numbers 10, 11, 12, 13 ....
2. The first transaction to arrive at the node is 11. based on the has\_invalid\_sequence\_number function logic, the sn is valid because it is between 10 and 10+32
3. The transaction is also accepted by the core\_mempool because it is of a higher value than the committed sn and it is sent to the da node for processing. 11 is set as the last used sn in the used\_sequence\_number\_pool.
4. From this point on teh account is stuck because transaction with sn 10 will not get accepted by the has\_invalid\_sequence\_number logic (since now the last used sn is 11) and all transactions will fail during block execution because all transactions will be considered as sequence number too new.
5. The problem can only be resolved after the used\_sequence\_number\_pool ttl (180 seconds) elapse and transaction with sequence number 10 can be accepted by the node and sent to processing.
