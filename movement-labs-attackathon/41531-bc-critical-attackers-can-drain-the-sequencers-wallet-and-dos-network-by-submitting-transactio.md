# #41531 \[BC-Critical] Attackers can drain the sequencer’s wallet and DoS network by submitting transactions from unfunded accounts

**Submitted on Mar 16th 2025 at 10:42:40 UTC by @Cartel for** [**Attackathon | Movement Labs**](https://immunefi.com/audit-competition/movement-labs-attackathon)

* **Report ID:** #41531
* **Report Type:** Blockchain/DLT
* **Report severity:** Critical
* **Target:** <https://github.com/immunefi-team/attackathon-movement/tree/main/protocol-units/da/movement/protocol/light-node>
* **Impacts:**
  * Direct loss of funds
  * Increasing network processing node resource consumption by at least 30% without brute force actions, compared to the preceding 24 hours

## Description

## Brief/Intro

The Movement blockchain does not verify sender account balances before adding transactions to the mempool and publishing them in DA. As a result, attackers can submit transactions from accounts with insufficient funds, allowing them to pass through the entire processing pipeline—mempool, sequencing, and data availability (DA) storage—before ultimately failing at execution.\
This vulnerability enables attackers to repeatedly submit transactions from wallets with no funds, effectively draining the sequencer’s wallet, as it is responsible for covering the gas fees when publishing these transactions to DA. Furthermore, the sequencer may become unable to process valid transactions due to wallet depletion. Attackers' transactions pass all verification checks before being added to the mempool and stored in DA, and attackers can exploit this vulnerability at no cost.

## Vulnerability Details

When a user submits a transaction to the Movement Network, it undergoes a two-step validation process. Once validated, the transaction enters the mempool, where it waits for inclusion. The sequencer then selects a batch of transactions, arranges them in the proper order, and publishes the batch’s data to the Data Availability (DA), either L1 or an alternative. Finally, the executor processes the transactions, completing the workflow.

Specifically, the current transaction flow accepts transactions into the mempool based only on signature verification and whitelist checks (when configured). There is no verification that the sender's account has sufficient balance to cover even the minimum gas fees that will be required for execution.

**Let's show you how the above issue occurs**:

The only check that occurs when making transactions from blobs inside `batch_write` is the `prevalidate` check:

```rust
		// make transactions from the blobs
		let mut transactions = Vec::new();
		for blob in blobs_for_submission {
			let transaction: Transaction = serde_json::from_slice(&blob.data)
				.map_err(|e| tonic::Status::internal(e.to_string()))?;

			match &self.prevalidator {
				Some(prevalidator) => {
					// match the prevalidated status, if validation error discard if internal error raise internal error
=>            				match prevalidator.prevalidate(transaction).await {
						Ok(prevalidated) => {
							transactions.push(prevalidated.into_inner());
```

Inside the `prevalidate` function, we can see that it performs only two checks:

1. **Signature check**:

   ```rust
   	let aptos_transaction = AptosTransactionValidator.prevalidate(transaction).await?;
   ```

   ```rust
   	// Only allow properly signed user transactions that can be deserialized from the transaction.data()
   	let aptos_transaction: AptosTransaction =
   		bcs::from_bytes(&transaction.data()).map_err(|e| {
   			Error::Validation(format!("Failed to deserialize AptosTransaction: {}", e))
   		})?;

   	aptos_transaction
   		.verify_signature()
   		.map_err(|e| Error::Validation(format!("Failed to prevalidate signature: {}", e)))?;
   ```
2. **Whitelist check**:

   ```rust
   	let aptos_transaction = self
   		.whitelist_validator
   		.prevalidate(aptos_transaction.into_inner())
   		.await?
   		.into_inner();
   ```

   ```rust
   	// reject all non-user transactions, check sender in whitelist for user transactions
   	if self.whitelist.contains(&transaction.sender()) {
   		Ok(Prevalidated::new(transaction))
   	} else {
   		Err(Error::Validation("Transaction sender not in whitelist".to_string()))
   	}
   ```

In the implementation of Movement Network, account's balance checks are only performed at execution time at movement-aptos-core codebase. While this simplifies mempool management, it introduces a vulnerability where invalid transactions consume resources across the entire system before being rejected. Specifically, at the current implementation, the sequencer pays the gas fee for publishing transaction data in DA. Consequently, an attacker can submit transactions from accounts with insufficient fund in order to drain the sequencer's wallet. This attack incurs no cost to the attacker.

## Impact Details

Attackers can freely submit a lot of transactions from zero-balance accounts, forcing the network to process and store them.\
The sequencers must cover gas fees to store transaction data on Celestia, even for transactions that will never successfully execute. This vulnerability enables attackers to drain seqeuncer's wallet and fill the mempool with their own transactions.\
Once the sequencer’s funds are depleted, it can no longer publish transactions to the DA layer.\
Even after receiving new funding from the protocol, the sequencer resumes processing the attacker's transactions, leading to a permanent DoS of the Movement network.

## Recommendation

Require sender accounts to have at least enough balance to cover minimum gas fees before accepting their transactions.

## References

None

## Proof of Concept

## Proof of Concept

Although a detailed explanation is provided in **Vulnerability Details**, here’s an example scenario:

1. The attacker creates 100 unique wallet addresses on the Movement blockchain. Each wallet has a minimal balance (close to zero), insufficient for transaction gas fees.
2. The attacker submits 1,000 transactions from each wallet.
3. All 100,000 transactions are added to the mempool by the sequencer.
4. The sequencer pays Celestia gas fees to publish these transactions to the DA layer.
5. When executed, all transactions fail due to insufficient balance.
6. The attacker pays nothing, while the network bears the costs of processing and storing the transactions, potentially draining the sequencer’s funds and disrupting network operations.
