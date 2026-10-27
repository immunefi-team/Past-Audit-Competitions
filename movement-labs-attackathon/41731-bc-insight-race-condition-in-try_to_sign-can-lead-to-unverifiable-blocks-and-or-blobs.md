# #41731 \[BC-Insight] Race Condition in try\_to\_sign can lead to unverifiable blocks and/or blobs

**Submitted on Mar 17th 2025 at 21:09:26 UTC by @jovi for** [**Attackathon | Movement Labs**](https://immunefi.com/audit-competition/movement-labs-attackathon)

* **Report ID:** #41731
* **Report Type:** Blockchain/DLT
* **Report severity:** Insight
* **Target:** <https://github.com/immunefi-team/attackathon-movement/tree/main/protocol-units/da/movement/protocol/util>
* **Impacts:**
  * A bug in the respective layer 0/1/2 network code that results in unintended smart contract behavior with no concrete funds at direct risk
  * Unintended chain split (network partition)
  * Some blocks and/or blobs can be signed but not ever be able to be verified

## Description

## Summary

In the `try_to_sign` function, the code fetches a signature and the corresponding public key in separate asynchronous steps without locking. This can cause the public key to change in the middle of signing, so you end up with a signature created by one key but published under a different key. As a result, any node that tries to verify this signaturelater will fail.

When this issue occurs during an admin key rotation, it can also cause blocks or blobs to be dropped if their signatures do not validate. In turn, this might trigger partial network outages, fork-like scenarios, or data unavailability.

***

## Vulnerability Details

### Location

* **File**: `protocol-units/da/movement/protocol/util/src/blob/ir/data.rs`
* **Function**: `try_to_sign`

### Description

The `try_to_sign` function does the following in sequence:

1. Computes the message hash (`id`).
2. Calls `signer.inner().sign(...)` to produce a signature.
3. Calls `signer.inner().public_key()` to retrieve the current public key.

These steps happen asynchronously without a mutex or other concurrency guard. Consequently, if an admin key rotation completes between steps 2 and 3, the returned public key will be different from the key that created the signature—making verification impossible.

#### Code Snippet

```rust
pub async fn try_to_sign<O>(
    self,
    signer: &Signer<O, C>,
) -> Result<InnerSignedBlobV1<C>, anyhow::Error>
where
    O: Signing<C>,
    C: Curve + Digester<C>,
{
    let id = self.compute_id()?;
    info!("Signing blob with id {:?}", id);
    // Potential race: signature and public key retrieved without a lock
    let signature = signer.inner().sign(&id.as_slice()).await?.to_bytes();
    let signer = signer.inner().public_key().await?.to_bytes();

    Ok(InnerSignedBlobV1::new(self, signature, signer, id))
}
```

### Impact

* **Verification Failures**: Any consumer (e.g., consensus nodes, block verifiers) sees a mismatch between the signer and public key of a published block or blob, causing verification to fail.
* **Data Loss / Forks**: During admin key rotation, blocks or blobs with mismatched signatures may be generated, leading to forks as those cannot be verified by anyone externally.

***

## Recommended Fix

Use a lock or other concurrency mechanism to guarantee the signature and public key come from the same key version. For example:

```rust
pub async fn try_to_sign<O>(
    self,
    signer: &Signer<O, C>,
) -> Result<InnerSignedBlobV1<C>, anyhow::Error>
where
    O: Signing<C>,
    C: Curve + Digester<C>,
{
    let id = self.compute_id()?;
    info!("Signing blob with id {:?}", id);

    // Acquire a lock for both signing and key retrieval
    let mut guard = signer.lock().await;

    let signature = guard.sign(&id.as_slice()).await?.to_bytes();
    let signer_pk = guard.public_key().await?.to_bytes();

    drop(guard); // Release the lock

    Ok(InnerSignedBlobV1::new(self, signature, signer_pk, id))
}
```

Any approach that atomically binds the signature and public key—ensuring they originate from the same key instance—eliminates the race condition and prevents verification failures, even in the case of admin key rotations.

## Proof of Concept

1. **Setup**
   * A sequencer node signs blocks before broadcasting them.
   * A Data Availability (DA) layer signs “blobs” (transaction data bundles, witnesses) to prove data possession.
   * Both systems call the same `try_to_sign` function.
2. **Concurrent Operations**
   * Typically, many sign operations happen in parallel for blocks and blobs.
   * Meanwhile, an admin key rotation begins, changing the signer’s public/private key pair.
3. **Race Condition Trigger**
   * The sequencer calls `try_to_sign` (step 2: `signer.inner().sign(...)`) to sign a new block/blob.
   * Immediately after signing, the admin rotation updates the public key.
   * When `try_to_sign` then retrieves the public key, it’s now the new one—thus returning a signature from the old key but labeled with the new key.
4. **Verification Failure**
   * Other nodes check the signature against the included public key.
   * Verification fails, so those nodes reject the block or blob as invalid.
