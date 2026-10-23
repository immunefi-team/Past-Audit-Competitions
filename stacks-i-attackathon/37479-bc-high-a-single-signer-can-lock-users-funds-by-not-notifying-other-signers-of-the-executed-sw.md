# #37479 \[BC-High] A single signer can lock users' funds by not notifying other signers of the execute

**Submitted on Dec 5th 2024 at 20:53:10 UTC by @n4nika for** [**Attackathon | Stacks**](https://immunefi.com/audit-competition/stacks-attackathon-1)

* **Report ID:** #37479
* **Report Type:** Blockchain/DLT
* **Report severity:** High
* **Target:** <https://github.com/stacks-network/sbtc/tree/immunefi\\_attackaton\\_0.9/signer>
* **Impacts:**
  * Direct loss of funds
  * Permanent freezing of funds (fix requires hardfork)

## Description

## Summary

The handling of sweep-signatures allows the transaction coordinator to execute `sweep` transactions without other signers being aware of it. This allows him to sweep users' transactions without sBTC being minted on Stacks, causing those funds to be permanently locked.

## Finding Description

When the coordinator handles a deposit request, the flow is the following:

1. get requests from `emily`
2. deploy smart contracts if needed
3. handle key rotation
4. get pre-signature (just pre-validation)
5. handle `sweep` transactions
   1. create them
   2. coordinate a signing round to sign them
   3. submit them to bitcoin
   4. notify all signers of them
6. ...

Now the problem is that the other signers only register the `sweep` transactions at step `5.4`! This means that the coordinator can:

1. get the signatures for the `sweep` txs
2. submit them
3. NOT notify the signers

If they do that, noone except the coordinator will have a record of the `sweep` transactions. This means that the users' UTXOs will be spent but there will be NO sBTC minted on Stacks.

## Impact

This allows a single signer (the coordinator is chosen randomly at a certain time from the signer set) to permanently freeze users' funds.

On Discord was mentioned:

```
Signers are permissioned and whitelisted operators. Any attack that requires a majority of signers to be malicious should be out of scope. Attacks that require a minority of signers to be malicious would still be in scope but with reduced severity.
```

Since we require one signer to be malicious, I deem this as high instead of critical.

## In-depth analysis

This can be shown if we look at `transaction_coordinator.rs::sign_and_broadcast` (which handles steps `5.1 - 5.4`):

```rs
async fn sign_and_broadcast(
    &mut self,
    bitcoin_chain_tip: &model::BitcoinBlockHash,
    aggregate_key: &PublicKey,
    signer_public_keys: &BTreeSet<PublicKey>,
    transaction: &mut utxo::UnsignedTransaction<'_>,
) -> Result<(), Error> {
    // [...]
    let signature = self
        .coordinate_signing_round( // [1] <-----
            bitcoin_chain_tip,
            &mut coordinator_state_machine,
            txid,
            &msg,
            SignatureType::Taproot(None),
        )
        .await?;
    
    // [...]
    
    // Broadcast the transaction to the Bitcoin network.
    self.context
        .get_bitcoin_client()
        .broadcast_transaction(&transaction.tx) // [2] <-----
        .await?;

    // Publish the transaction to the P2P network so that peers get advance
    // knowledge of the sweep.
    self.send_message( // @audit-issue the coordinator can just NOT send this, enabling him to cause funds loss to users!!
        SweepTransactionInfo::from_unsigned_at_block(bitcoin_chain_tip, transaction), // [3] <-----
        bitcoin_chain_tip,
    )
    .await?;
    // [...]
}
```

Here we coordinate the signing round `[1]`, then submit the tx to bitcoin `[2]` and then notify the other signers `[3]`.

We see that we get the signature by calling `coordinate_signing_round`. This coordination is done by broadcasting a `wsts` message to the signers.

If we look at the [`wsts`](https://github.com/stacks-network/sbtc/blob/53cc756c0ddecff7518534a69bef59fadb5ab1d4/signer/src/transaction_signer.rs#L242) handling code of the signers, we can see that we do NOT add anything to the database at this point.

The only place where signers add a `sweep` transaction to their database, is when they receive a `SweepTransactionInfo` (sent at `[3]`). This can be done with the `write_sweep_transaction` function, which is ONLY used here:

`transaction_signer.rs::handle_signer_message`

```rs
(
    message::Payload::SweepTransactionInfo(sweep_tx),
    is_coordinator,
    ChainTipStatus::Canonical,
) => {
    // [...]
    self.context
        .get_storage_mut()
        .write_sweep_transaction(&sweep_tx.into())
        .await?;
}
```

## Mitigation

To fix this, I would suggest enforcing that signers get context about what they sign on each signing round so they can add context about it to their database WITHOUT relying on the coordinator to provide that information.

## Proof of Concept

## PoC

In order to show that when the coordinator omits this call, the signers will have no record of the `sweep` transaction, please apply the following diff, and execute the test with `cargo test --package signer --test integration -- transaction_coordinator::sign_bitcoin_transaction --exact --show-output --ignored --nocapture`. This will print `RESULT: None` but everything else succeeds. If you comment-in the changes in `transaction_coordinator` and execute the test again, it will print an actual transaction.

```diff
diff --git a/signer/src/transaction_coordinator.rs b/signer/src/transaction_coordinator.rs
index 26afe19c..2eebf251 100644
--- a/signer/src/transaction_coordinator.rs
+++ b/signer/src/transaction_coordinator.rs
@@ -926,11 +926,11 @@ where
 
         // Publish the transaction to the P2P network so that peers get advance
         // knowledge of the sweep.
-        self.send_message(
-            SweepTransactionInfo::from_unsigned_at_block(bitcoin_chain_tip, transaction),
-            bitcoin_chain_tip,
-        )
-        .await?;
+        // self.send_message(
+        //     SweepTransactionInfo::from_unsigned_at_block(bitcoin_chain_tip, transaction),
+        //     bitcoin_chain_tip,
+        // )
+        // .await?;
 
         tracing::info!("bitcoin transaction accepted by bitcoin-core");
 
diff --git a/signer/tests/integration/transaction_coordinator.rs b/signer/tests/integration/transaction_coordinator.rs
index bdf10d7e..df5239ea 100644
--- a/signer/tests/integration/transaction_coordinator.rs
+++ b/signer/tests/integration/transaction_coordinator.rs
@@ -1505,7 +1505,7 @@ async fn sign_bitcoin_transaction() {
     // - The coordinator should submit a sweep transaction. We check the
     //   mempool for its existance.
     // =========================================================================
-    faucet.generate_blocks(1);
+    let chain_tip_: BitcoinBlockHash = faucet.generate_blocks(1).pop().unwrap().into();
 
     wait_for_signers(&signers).await;
 
@@ -1541,13 +1541,13 @@ async fn sign_bitcoin_transaction() {
     more_asserts::assert_ge!(broadcast_stacks_txs.len(), 2);
     // Check that the first N - 1 are all rotate keys contract calls.
     let rotate_keys_count = broadcast_stacks_txs.len() - 1;
-    for tx in broadcast_stacks_txs.iter().take(rotate_keys_count) {
-        assert_stacks_transaction_kind::<RotateKeysV1>(tx);
-    }
-    // Check that the Nth transaction is the complete-deposit contract
-    // call.
-    let tx = broadcast_stacks_txs.last().unwrap();
-    assert_stacks_transaction_kind::<CompleteDepositV1>(tx);
+    // for tx in broadcast_stacks_txs.iter().take(rotate_keys_count) {
+    //     assert_stacks_transaction_kind::<RotateKeysV1>(tx);
+    // }
+    // // Check that the Nth transaction is the complete-deposit contract
+    // // call.
+    // let tx = broadcast_stacks_txs.last().unwrap();
+    // assert_stacks_transaction_kind::<CompleteDepositV1>(tx);
 
     // Now lets check the bitcoin transaction, first we get it.
     let txid = txids.pop().unwrap();
@@ -1575,6 +1575,10 @@ async fn sign_bitcoin_transaction() {
     .await
     .unwrap();
 
+    let result = signers[0].0.storage.get_latest_sweep_transaction(&chain_tip_, 10000).await.unwrap();
+
+    println!("RESULT: {:?}", result);
+
     let script = tx.output[0].script_pubkey.clone().into();
     for (_, db, _, _) in signers {
         assert!(db.is_signer_script_pub_key(&script).await.unwrap());
```
