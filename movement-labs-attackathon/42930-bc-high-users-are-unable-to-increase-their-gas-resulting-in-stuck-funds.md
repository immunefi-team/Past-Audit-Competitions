# #42930 \[BC-High] Users are unable to increase their gas resulting in stuck funds

**Submitted on Mar 29th 2025 at 14:32:03 UTC by @okmxuse for** [**Attackathon | Movement Labs**](https://immunefi.com/audit-competition/movement-labs-attackathon)

* **Report ID:** #42930
* **Report Type:** Blockchain/DLT
* **Report severity:** High
* **Target:** <https://github.com/immunefi-team/attackathon-movement/tree/main/protocol-units/execution/maptos/opt-executor>
* **Impacts:**
  * Permanent freezing of funds (fix requires hardfork)
  * Direct loss of funds
  * A bug in the respective layer 0/1/2 network code that results in unintended smart contract behavior with no concrete funds at direct risk

## Description

### Description

The Aptos mempool, like other networks, allows users to increase the gas price of a previously submitted transaction.

For this to happen, the newly submitted transaction must have the same sequence number as the original transaction the user intends to speed up. This requirement is enforced in:

**`transaction_store.rs::insert`**

```rust
if let Some(txns) = self.transactions.get_mut(&address) {
    if let Some(current_version) = txns.get_mut(&txn_seq_num) {
        // ..code
```

If these conditions are met, the user's transaction with the increased gas price is accepted:

```rust
if let Some(txns) = self.transactions.get_mut(&address) {
    if let Some(current_version) = txns.get_mut(&txn_seq_num) {
        // ..code
    else if current_version.get_gas_price() < txn.get_gas_price() {
    // Update transaction if gas unit price is higher than before
        if let Some(txn) = txns.remove(&txn_seq_num) {
           self.index_remove(&txn);
       }
    　　//..code 
    MempoolStatus::new(MempoolStatusCode::Accepted)
}
```

Despite this intended behavior, a conflicting check in `transaction_pipe.rs::has_invalid_sequence_number` will always prevent gas-increase transactions from succeeding:

```rust
if transaction.sequence_number() < min_sequence_number {
    info!("Transaction sequence number too old: {:?}", transaction.sequence_number());
    return Ok(SequenceNumberValidity::Invalid((
        MempoolStatus::new(MempoolStatusCode::InvalidSeqNumber),
        Some(DiscardedVMStatus::SEQUENCE_NUMBER_TOO_OLD),
    )));
}
```

This check will never allow the same sequence number to be submitted.

Note that this check is different than the one made in `mempool.rs::add_txn` and this is not the reason why the transactions fails.

```javascript
 // don't accept old transactions (e.g. seq is less than account's current seq_number)
        if txn.sequence_number() < db_sequence_number {
            return MempoolStatus::new(MempoolStatusCode::InvalidSeqNumber).with_message(format!(
                "transaction sequence number is {}, current sequence number is  {}",
                txn.sequence_number(),
                db_sequence_number,
            ));
        }
```

We will further prove all this in the POC.

### Impact

This issue inevitably results in a user's funds becoming stuck. At worst, it can cause a user to have a direct loss of funds since the transaction might become stuck permanently since a user is never able to speed it up.

### Recommendation

We would recommend mitigating this issue properly.

## Proof of Concept

## Proof of Concept

1. Add the following test function\*\* inside `transaction_pipe.rs`:

```rust
async fn test_user_cannot_increase_gas() -> Result<(), anyhow::Error> {
    // Set up environment
    let maptos_config = Config::default();
    let (_context, mut transaction_pipe, _tx_receiver, _tempdir) = setup().await;

    // Submit a transaction with a valid sequence number
    let user_transaction = create_signed_transaction(1, &maptos_config);
    let (mempool_status, _) = transaction_pipe.submit_transaction(user_transaction).await?;
    assert_eq!(mempool_status.code, MempoolStatusCode::Accepted);

    // Attempt to submit another transaction with the same sequence number but increased gas
    let user_transaction = create_signed_transaction(1, &maptos_config);
    let (mempool_status, _) = transaction_pipe.submit_transaction(user_transaction).await?;
    assert_eq!(mempool_status.code, MempoolStatusCode::InvalidSeqNumber);

    Ok(())
}
```

2. Run the test:

   ```sh
   nix develop -c cargo test test_user_cannot_increase_gas
   ```
3. Logs

   ```
   running 1 test
   test background::transaction_pipe::tests::test_user_cannot_increase_gas ... ok

   test result: ok. 1 passed; 0 failed; 0 ignored; 0 measured; 15 filtered out; finished in 3.46s
   ```

To confirm that the issue is not related to the `mempool.rs::add_txn` check, modify `transaction_pipe.rs::has_invalid_sequence_number` by removing the root issue check:

```diff
+	// if transaction.sequence_number() < min_sequence_number {
+	// 	info!("Transaction sequence number too old: {:?}", transaction.sequence_number());
+	// 	return Ok(SequenceNumberValidity::Invalid((
+	// 		MempoolStatus::new(MempoolStatusCode::InvalidSeqNumber),
+	// 		Some(DiscardedVMStatus::SEQUENCE_NUMBER_TOO_OLD),
+	// 	)));
+	// }
```

4. Re-run the test:

   ```sh
   nix develop -c cargo test test_user_cannot_increase_gas
   ```
5. Failure Logs:

   ```
   assertion `left == right` failed
     left: Accepted
    right: InvalidSeqNumber
   note: run with `RUST_BACKTRACE=1` environment variable to display a backtrace

   failures:
       background::transaction_pipe::tests::test_user_cannot_increase_gas

   test result: FAILED. 0 passed; 1 failed; 0 ignored; 0 measured; 15 filtered out; finished in 3.53s

   error: test failed, to rerun pass `--lib`
   ```

Since the test now fails with `Accepted` instead of `InvalidSeqNumber`, it confirms that the original rejection was due to the `has_invalid_sequence_number` check and not the one from `mempool.rs::add_txn`:

```javascript
 // don't accept old transactions (e.g. seq is less than account's current seq_number)
        if txn.sequence_number() < db_sequence_number {
            return MempoolStatus::new(MempoolStatusCode::InvalidSeqNumber).with_message(format!(
                "transaction sequence number is {}, current sequence number is  {}",
                txn.sequence_number(),
                db_sequence_number,
            ));
        }
```
