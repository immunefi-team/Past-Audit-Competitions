# #42430 \[BC-Insight] \`add\_mempool\_transaction()\` does not check if the transaction already exist in the mempool

**Submitted on Mar 23rd 2025 at 21:23:34 UTC by @avoloder for** [**Attackathon | Movement Labs**](https://immunefi.com/audit-competition/movement-labs-attackathon)

* **Report ID:** #42430
* **Report Type:** Blockchain/DLT
* **Report severity:** Insight
* **Target:** <https://github.com/immunefi-team/attackathon-movement/tree/main/protocol-units/mempool/move-rocks>
* **Impacts:**

## Description

## Brief/Intro

`add_mempool_transaction` in the move-rocks does not check for duplicates but instead writes into the database regardless

## Vulnerability Details

Inside mempool there are two functions for adding transactions, one for batch processing (`add_mempool_transactions()`) and another one for single processing `add_mempool_transaction()`. Inside the batch processing function there is a check against duplicate transactions to prevent unnecessary writes to the database, as it can be seen from the snipet below:

```
async fn add_mempool_transactions(
		&self,
		transactions: Vec<MempoolTransaction>,
	) -> Result<(), anyhow::Error> {

   --- SNIPET ---
   for transaction in transactions {
				if Self::internal_has_mempool_transaction(&db, 
                    transaction.transaction.id())? {
					continue;
				}
   --- SNIPET ---
```

However, the function for single processing fails to check if the transaction already exists in the mempool, and does redundant updates.

## Impact Details

Since the `add_mempool_transaction` function does not check for duplicates, an attacker could repeatedly submit the same transaction to the mempool, causing RocksDB to continually overwrite the same transaction

## References

<https://github.com/immunefi-team/attackathon-movement/blob/a2790c6ac17b7cf02a69aea172c2b38d2be8ce00/protocol-units/mempool/move-rocks/src/lib.rs#L161-L194>

## Proof of Concept

## Proof of Concept

1. User A submits transaction A that gets added to the mempool with `add_mempool_transaction()`, i.e., saved to the rocksDB.
2. User A submits the same transaction again (using the same data) and the mempool tries to add the transaction again.
3. The transaction is overwritten in the rocksDB as the key is already present.

Currently there is no direct harm, as there will be no duplicate transactions created, however everytime, the redundant overwrites (updates) will happen. By doing this, a malicious user could flood the system with repeated write operations, potentially blocking or slowing down other legitimate transactions. The system would still perform I/O operations, database locking, and updates even though the data may be identical.

By checking for duplicates, the redundant writes could be avoided, like in the `add_mempool_transactions()` (batch processing)
