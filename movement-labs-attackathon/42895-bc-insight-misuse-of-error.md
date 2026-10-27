# #42895 \[BC-Insight] Misuse of error

**Submitted on Mar 28th 2025 at 13:56:45 UTC by @okmxuse for** [**Attackathon | Movement Labs**](https://immunefi.com/audit-competition/movement-labs-attackathon)

* **Report ID:** #42895
* **Report Type:** Blockchain/DLT
* **Report severity:** Insight
* **Target:** <https://github.com/immunefi-team/attackathon-movement/tree/main/protocol-units/execution/maptos/opt-executor>
* **Impacts:**

## Description

### Description

The function `submit_transaction` is invoked during the transaction processing pipeline. Within this function, the following check is performed:

```rust
async fn submit_transaction(
    &mut self,
    transaction: SignedTransaction,
) -> Result<SubmissionStatus, Error> {
    // Check whether the account is whitelisted
    if !self.is_whitelisted(&transaction.sender())? {
        return Ok((MempoolStatus::new(MempoolStatusCode::TooManyTransactions), None));
    }
```

If `transaction.sender` is not whitelisted, the function returns an error. However, the error type used here is `MempoolStatusCode::TooManyTransactions`.

This error code, as its name suggests, is intended to indicate that an account has reached its maximum transaction capacity:

```rust
// Account reached max capacity per account
TooManyTransactions = 3,
```

This intended use is further reinforced by its correct application in `transaction_store.rs::insert`:

```rust
if txns.len() >= self.capacity_per_user {
    return MempoolStatus::new(MempoolStatusCode::TooManyTransactions).with_message(
        format!(
            "Mempool over capacity for account. Number of transactions from account: {} Capacity per account: {}",
            txns.len(),
            self.capacity_per_user,
        ),
    );
}
```

Here, the error is returned only when the number of transactions from an account exceeds the allowed limit.

### Impact

The primary issue with the current implementation is the confusion it may cause for both protocol users and developers. Misusing `TooManyTransactions` in a whitelist check introduces ambiguity in error handling and makes debugging more difficult.

### Recommendation

We recommend either:

* Introducing a new, more appropriate error code to indicate that a transaction sender is not whitelisted.
* Handling the whitelist rejection differently, ensuring error messages accurately reflect the underlying issue.

## Proof of Concept

### POC

* The `submit_transaction` function is called during transaction processing.
* If the sender is not whitelisted, the following check triggers an error:

```rust
if !self.is_whitelisted(&transaction.sender())? {
    return Ok((MempoolStatus::new(MempoolStatusCode::TooManyTransactions), None));
}
```

* However, `MempoolStatusCode::TooManyTransactions` is meant to indicate that the user has exceeded their transaction limit.
* In this case, the sender is simply not whitelisted, making the error misleading.
* This misrepresentation can cause confusion for both interacting protocols and users.
