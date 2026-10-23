# #43214 \[BC-Critical] Unchecked transaction size allows malicious users to DOS honest users transactions

**Submitted on Apr 3rd 2025 at 18:46:51 UTC by @okmxuse for** [**Attackathon | Movement Labs**](https://immunefi.com/audit-competition/movement-labs-attackathon)

* **Report ID:** #43214
* **Report Type:** Blockchain/DLT
* **Report severity:** Critical
* **Target:** <https://github.com/immunefi-team/attackathon-movement/tree/main/protocol-units/execution/maptos/opt-executor>
* **Impacts:**
  * Network not being able to confirm new transactions (total network shutdown)
  * A bug in the respective layer 0/1/2 network code that results in unintended smart contract behavior with no concrete funds at direct risk

## Description

### Description

Inside `submit_transaction`, the `transaction` is validated in the following manner:

```javascript
let tx_result = vm_validator.validate_transaction(transaction.clone())?;
```

Note that up until now, **no size check** has been performed for the transaction data. This refers **only to the size** of the data, not its contents.

The `transaction` is then sent in the following manner:

```javascript
//..code
self.transaction_sender.send((application_priority, transaction)).await
    .map_err(|e| anyhow::anyhow!("Error sending transaction: {:?}", e))?;
//..code
```

After this, it is further handled inside `spawn_write_next_transaction_batch`:

```javascript
Some((application_priority, transaction)) => {
    info!(
    //..code
```

Here, the `transaction` is **received, serialized, and pushed into `transactions`**:

```javascript
transactions.push(BlobWrite { data: serialized_transaction });
```

Again, **no data size check has been performed up to this point**.

After serialization, the `transactions` are **bundled into blobs** and passed into `batch_write`:

```javascript
let batch_write = BatchWriteRequest { blobs: transactions };
//..code
match da_light_node_client.batch_write(batch_write.clone()).await {
```

This is where the system **makes contact with the Celestia node**, which enforces a **strict data size limit**:

> The maximum total blob size in a transaction is just under **2 MiB (1,973,786 bytes)**, based on a 64x64 share grid (4096 shares). With one share for the PFB transaction, 4095 shares remain: 1 at 478 bytes and 4094 at 482 bytes each.

If `batch_write` fails, **the entire batch—including all bundled transactions—fails**:

```javascript
match da_light_node_client.batch_write(batch_write.clone()).await {
    Ok(_) => {
        info!(
            target: "movement_timing",
            batch_id = %batch_id,
            "batch_write_success"
        );
        return;
    }
    Err(e) => {
        warn!("failed to write batch to DA: {:?} {:?}", e, batch_id);
    }
}
```

A malicious user can intentionally submit a transaction that exceeds Celestia's limits, causing the entire batch to fail. This happens because transaction size is not checked until all transactions are batched together and sent to Celestia.

### Impact

This will trigger `Err`, resulting in all other transactions in the batch failing, regardless of their validity.

### Recommendation

Do not allow users to single-handedly cause honest users' transactions to fail. Implement a transaction size check before batching.

## Proof of Concept

### POC:

A user submits a transaction with a higher data size than accepted by Celestia, it is first recognized inside `receive_transaction_tick`:

```javascript
MempoolClientRequest::SubmitTransaction(transaction, callback) => {
```

It is then used inside `submit_transaction`:

```javascript
let status = self.submit_transaction(transaction).instrument(span).await?;
```

Inside `submit_transaction` the transaction goes through the following process:

* Checked for whitelist

```javascript
if !self.is_whitelisted(&transaction.sender())? {
    return Ok((MempoolStatus::new(MempoolStatusCode::TooManyTransactions), None));
}
```

* Checked for validation

```javascript
let tx_result = vm_validator.validate_transaction(transaction.clone())?;
```

Inside `validate_transaction` there are multiple checks, but no check for the data size.

It is further checked inside `add_txn`'s flow, once again no data size check.

```javascript
let status = self.core_mempool.add_txn(
```

Now finally, if the status is accepted, the transaction is sent:

```javascript
self.transaction_sender.send((application_priority, transaction))
```

Now the next time this transaction is received is inside `spawn_write_next_transaction_batch`:

```javascript
Some((application_priority, transaction)) => {
```

It is then finally batched with other honest valid transactions and then failed since Celestia will not accept data blobs that are higher than the limit, which is not checked up until they are batched and tried to process.

The malicious user now successfully DOS'ed multiple transactions with very little effort.
