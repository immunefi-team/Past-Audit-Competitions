# 75398 bc insight p2p block validation performs expensive operations before cheap signature check

Submitted on Apr 28th 2026 at 23:27:21 UTC by @DeltaXV for [Audit Comp | Base Azul](https://immunefi.com/audit-competition/audit-comp-base-azul)

* **Report ID:** #75398
* **Report Type:** Blockchain/DLT
* **Report severity:** Insight
* **Target:** <https://github.com/base/base/tree/v0.8.0-rc.28>
* **Impacts:**

## Description

The `BlockHandler::validate_block_internal()` in `base/crates/consensus/gossip/src/block_validity.rs` should reject unauthenticated messages as early as possible using the cheap ECDSA signature check (\~80μs, which only depends on `payload_hash` already computed during decode), but instead performs expensive operations first — deep cloning the payload, RLP-decoding all transactions via `try_into_block()`, and recomputing the block header hash via `hash_slow()` — before checking whether the message was actually signed by the sequencer.

```rust
fn validate_block_internal(&mut self, envelope: &NetworkPayloadEnvelope) -> Result<(), BlockInvalidError> {
    // 1. Timestamp check (cheap) ✓

    // 2. Expensive block hash verification — runs before signature check
    let expected = envelope.payload.block_hash();
    let mut block: Block<BaseTxEnvelope> = envelope.payload.clone().try_into_block()?;
    let received = block.header.hash_slow();
    if received != expected { return Err(...); }

    // 3. Signature check — could have run right after timestamp
    let msg = envelope.payload_hash.signature_message(self.rollup_config.l2_chain_id.id());
    let Ok(msg_signer) = envelope.signature.recover_address_from_prehash(&msg) else { return Err(...); };
    if msg_signer != block_signer { return Err(...); }
}
```

## Impact

Every gossip message from a non-sequencer peer causes avoidable ressource consumption waste by performing expensive clone, deserialization, and hashing operations on unauthenticated data allowing potential absuse before the cheap signature check that would have rejected the message immediately.

## Recommended Patch

Move the signature check (step 4) to run immediately after the timestamp check (step 1), before any expensive operations.

## References

<https://github.com/base/base/blob/4d184b8e8a641db66497a8cfce167262fe97c5d4/crates/consensus/gossip/src/block\\_validity.rs#L162-L226>

## Link to Proof of Concept

<https://gist.github.com/DeltaXV/7ab6ab9b18487ec060f39f3bcb4458cb>

## Proof of Concept

Only the sequencer holds the signing key, so every gossip message from any other peer fails the signature check. With the current order, each such message executes `payload.clone()`, `try_into_block()` (RLP-decodes all transactions), and `hash_slow()` (keccak256 over the header) before reaching that check. Moving the signature verification first — which only needs `envelope.payload_hash`, already computed during decode — skips all of that work for every rejected message.

```diff
diff --git a/crates/consensus/gossip/src/block_validity.rs b/crates/consensus/gossip/src/block_validity.rs
index bef378717..fb931a02c 100644
--- a/crates/consensus/gossip/src/block_validity.rs
+++ b/crates/consensus/gossip/src/block_validity.rs
@@ -180,6 +180,19 @@ impl BlockHandler {
             });
         }
 
+        // CHECK: The signature is valid.
+
+        let msg = envelope.payload_hash.signature_message(self.rollup_config.l2_chain_id.id());
+        let block_signer = *self.signer_recv.borrow();
+
+        let Ok(msg_signer) = envelope.signature.recover_address_from_prehash(&msg) else {
+            return Err(BlockInvalidError::Signature);
+        };
+
+        if msg_signer != block_signer {
+            return Err(BlockInvalidError::Signer { expected: block_signer, received: msg_signer });
+        }
+
         // CHECK: Ensure the block hash is valid.
         let expected = envelope.payload.block_hash();
         let mut block: Block<BaseTxEnvelope> = envelope.payload.clone().try_into_block()?;
@@ -215,20 +228,6 @@ impl BlockHandler {
             }
         }
 
-        // CHECK: The signature is valid.
-        let msg = envelope.payload_hash.signature_message(self.rollup_config.l2_chain_id.id());
-        let block_signer = *self.signer_recv.borrow();
-
-        // The block has a valid signature.
-        let Ok(msg_signer) = envelope.signature.recover_address_from_prehash(&msg) else {
-            return Err(BlockInvalidError::Signature);
-        };
-
-        // The block is signed by the expected signer (the unsafe block signer).
-        if msg_signer != block_signer {
-            return Err(BlockInvalidError::Signer { expected: block_signer, received: msg_signer });
-        }
-
         self.seen_hashes
             .entry(envelope.payload.block_number())
             .or_default()
```
