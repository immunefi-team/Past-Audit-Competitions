# 88951 bc high batch level fee rounding permanently strands fees locked by wormhole leaf proofs

**Submitted on Aug 15th 2026 at 21:39:05 UTC by @Porcupine17126 for** [**Audit Comp | Quantus**](https://immunefi.com/audit-competition/audit-comp-quantus)

* **Report ID:** #88951
* **Report Type:** Blockchain/DLT
* **Report severity:** High
* **Target:** <https://github.com/immunefi-team/audit-comp-quantus-chain>
* **Impacts:**
  * Permanent freezing of unclaimed yield

## Description

## Brief/Intro

Each Wormhole leaf proof independently constrains its quantized input and outputs:

```
(output_1 + output_2) * 10000 <= input * (10000 - fee_bps)
```

At the production fee rate of 4 bps, a one-quantum exit therefore requires a two-quantum input. The leaf consumes one full fee quantum (0.01 QUAN).

The private-batch circuit verifies and aggregates up to seven leaf proofs, but its public inputs do not retain each leaf's private `input_amount` or fee gap. After verifying the batch, `process_exit_bundle()` sums all successful outputs and calculates the fee only once:

```
ceil(sum(outputs) * fee_bps / (10000 - fee_bps))
```

This is not generally equal to the fee that the independently constrained leaves had to lock:

```
ceil(sum(outputs) * rate) < sum(ceil(each_output * rate))
```

The omitted amount is not minted to an exit account, burned, paid to the miner, or rebated to an aggregator. Because the valid leaf nullifiers are permanently consumed, that value remains in the spent Wormhole deposits and cannot be recovered.

## Root cause

The leaf circuit in `qp-wormhole-circuit` enforces the fee relation separately for every leaf. For any leaf output `o`, the minimum input is `ceil(o * 10000 / (10000 - bps))`, so the minimum fee gap is:

```
ceil(o * bps / (10000 - bps))
```

The scoped runtime correctly mints each aggregated output, but then reduces the bundle to one `minted_exit_amount` and calls `volume_fee_for_exit()` once:

```rust
let total_fee_u128 =
    Self::volume_fee_for_exit(total_exit_u128, T::VolumeFeeRateBps::get());
```

`volume_fee_for_exit()` performs a single `div_ceil()` on the summed output. The current comment says this is “exactly the minimum fee gap the proofs lock,” but that is true only for a single leaf. A sum of ceilings is generally larger than a ceiling of the sum.

The production private-batch size is seven leaves (`QP_NUM_LEAF_PROOFS=7`). Each valid leaf has already been constrained independently before aggregation, so padding or proof validity does not close the gap.

## Minimal reachable example

{% stepper %}
{% step %}

### Leaf A

Leaf A proves input 2 quanta and output 1 quantum. It necessarily locks 1 fee quantum.
{% endstep %}

{% step %}

### Leaf B

Leaf B independently proves input 2 quanta and output 1 quantum. It necessarily locks 1 fee quantum.
{% endstep %}

{% step %}

### Aggregate proofs

The production private-batch prover aggregates the two valid leaf proofs.
{% endstep %}

{% step %}

### Mint exits

The runtime mints both one-quantum exits.
{% endstep %}

{% step %}

### Settle fee

Runtime fee settlement calculates `ceil(2 * 4 / 9996) = 1` quantum.
{% endstep %}

{% step %}

### Consume nullifiers

Both real nullifiers are consumed.
{% endstep %}
{% endstepper %}

Two fee quanta were unavoidably left behind by the leaf constraints, but only one is distributed. The other 0.01 QUAN is permanently omitted.

For a full seven-leaf private batch of minimum exits, the leaf constraints lock seven fee quanta while the runtime settles one, omitting six quanta (0.06 QUAN). The loss is repeatable and cumulative across ordinary fragmented exit batches; it does not require malformed proofs, invalid public inputs, a privileged actor, or network probing.

## Impact

The omitted value was locked by valid users' spent deposits specifically because of the protocol's per-leaf fee constraint. After the nullifiers are accepted, nobody can create another exit for those deposits. Yet the runtime allocates only the smaller bundle-level amount to the fee recipients.

The strict rounding difference is permanently inaccessible after the underlying nullifiers are consumed. The portion allocated by protocol policy to the miner or aggregator is permanently frozen unclaimed yield; the remaining portion is fee value that should have been burned but instead remains stranded in the consumed Wormhole inputs. The closest listed impact is High, “Permanent freezing of unclaimed yield.”

This report does not claim attacker profit, direct theft, protocol insolvency, or a consensus failure.

## Relationship to the public fee-rounding fix

PR #656 / commit `7d7c4bd9` publicly fixed a different rounding error: base-unit truncation caused even a single minimum exit to settle less than one quantum. The change introduced the current quantized `div_ceil()` calculation.

This report is the unhandled aggregation case in that remediation. The PR correctly states `ceil(sum) <= sum(ceil)`, but treats that inequality only as proof that the bundle calculation cannot overcharge. It does not account for the strict-inequality case, where the difference is value that every underlying leaf was forced to leave behind but the runtime never distributes. The current source still calls the bundle result “exactly the minimum fee gap the proofs lock,” which the PoC disproves for two valid leaves.

The live competition rules state that a bypass of a public fix is a new valid bug. The listed known issues and bundled audit reports do not disclose cross-leaf fee loss after aggregation.

## Recommended remediation

{% stepper %}
{% step %}

### Expose proof-locked fees

Expose each leaf's constrained fee gap, or an authenticated sum of those gaps, in the leaf proof's public inputs.
{% endstep %}

{% step %}

### Aggregate fees recursively

Recursively sum that value in private- and public-batch circuits.
{% endstep %}

{% step %}

### Settle the authenticated aggregate

Have `process_exit_bundle()` burn/distribute the authenticated aggregate fee sum for the valid processed segments.
{% endstep %}
{% endstepper %}

Alternatively, redesign the circuit so the fee is constrained once at the same aggregation level at which the runtime settles it. Recomputing a single ceiling from final aggregate outputs cannot recover per-leaf rounding already imposed by the leaf constraints.

## Proof of Concept

Apply the following patch to scoped commit `3b243b870a5442a6e5f443056f67bbba87d472f5` and run:

```bash
git checkout 3b243b870a5442a6e5f443056f67bbba87d472f5
git apply poc.patch
cargo test -p pallet-wormhole --lib \  poc_real_private_batch_strands_per_leaf_fee_rounding -- \  --ignored --nocapture
cargo test -p pallet-wormhole --lib
```

The PoC inserts canonical leaves directly only to avoid unrelated deposit setup. The leaf structure, hash, Merkle path, circuit verification, aggregation, runtime verification, settlement, and nullifier consumption are all production implementations; replacing insertion with the ordinary recorder produces the same leaf consumed by the proof.

### poc.patch

```diff
diff --git a/pallets/wormhole/src/tests.rs b/pallets/wormhole/src/tests.rs
index 4473b70..23d4f62 100644
--- a/pallets/wormhole/src/tests.rs
+++ b/pallets/wormhole/src/tests.rs
@@ -971,6 +971,203 @@ mod private_batch_proof_tests {
 		});
 	}
 
+	/// End-to-end Candidate 8 PoC. Two independently valid leaf proofs each exit the
+	/// minimum one quantum from a two-quantum input, so the leaf circuit forces and locks
+	/// one fee quantum per proof. The production private-batch circuit aggregates both,
+	/// but the runtime settles `ceil(sum(outputs) * bps / denominator)` once and allocates
+	/// only one quantum. The two real nullifiers are then permanently consumed.
+	#[test]
+	#[ignore]
+	fn poc_real_private_batch_strands_per_leaf_fee_rounding() {
+		use frame_support::traits::fungible::{Inspect, Mutate};
+		use qp_wormhole_aggregator::private_batch::prover::PrivateBatchProver;
+		use qp_wormhole_circuit::{
+			block_header::header::HeaderInputs,
+			inputs::{CircuitInputs, PrivateCircuitInputs},
+			nullifier::Nullifier,
+			unspendable_account::UnspendableAccount,
+		};
+		use qp_wormhole_inputs::{BytesDigest, PublicCircuitInputs};
+		use qp_zk_circuits_common::utils::digest_to_bytes;
+		use sp_runtime::traits::ValidateUnsigned;
+
+		new_test_ext().execute_with(|| {
+			System::set_block_number(1);
+
+			let secrets = [BytesDigest::new_unchecked([42u8; 32]), BytesDigest::new_unchecked([43u8; 32])];
+			let unspendable_accounts: Vec<BytesDigest> = secrets
+				.iter()
+				.map(|secret| digest_to_bytes(UnspendableAccount::from_secret(*secret).account_id))
+				.collect();
+
+			// Create the two real on-chain leaves at the exact two-quantum input used by
+			// the proofs. The tree supplies production Merkle paths for a shared root.
+			for account_digest in &unspendable_accounts {
+				let account_bytes: [u8; 32] = account_digest.as_ref().try_into().unwrap();
+				ZkTree::insert_leaf(
+					sp_core::crypto::AccountId32::new(account_bytes),
+					0,
+					0,
+					2 * crate::SCALE_DOWN_FACTOR,
+				);
+			}
+			let zk_tree_root = ZkTree::root();
+
+			let parent_hash = [0x11u8; 32];
+			let state_root = [0x22u8; 32];
+			let extrinsics_root = [0x33u8; 32];
+			let digest: [u8; 110] = [
+				8, 6, 112, 111, 119, 95, 128, 233, 182, 183, 107, 158, 1, 115, 19, 219,
+				126, 253, 86, 30, 208, 176, 70, 21, 45, 180, 229, 9, 62, 91, 4, 6, 53,
+				245, 52, 48, 38, 123, 225, 5, 112, 111, 119, 95, 1, 1, 0, 0, 0, 0, 0,
+				0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0,
+				0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0,
+				0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 18, 79, 226,
+			];
+			let block_number = 7u32;
+			let header = HeaderInputs::new(
+				BytesDigest::new_unchecked(parent_hash),
+				block_number,
+				BytesDigest::new_unchecked(state_root),
+				BytesDigest::new_unchecked(extrinsics_root),
+				BytesDigest::new_unchecked(zk_tree_root),
+				&digest,
+			)
+			.expect("valid test header");
+			let block_hash = header.block_hash();
+
+			let mut real_nullifiers: Vec<[u8; 32]> = Vec::new();
+			let mut leaf_proofs = Vec::new();
+			for (index, secret) in secrets.into_iter().enumerate() {
+				let leaf = ZkTree::leaf(index as u64).expect("inserted leaf");
+				let merkle_proof = ZkTree::get_merkle_proof(index as u64).expect("Merkle proof");
+				let mut current_hash = pallet_zk_tree::tree::hash_leaf::<Test>(&leaf);
+				let mut zk_merkle_siblings = Vec::new();
+				let mut zk_merkle_positions = Vec::new();
+
+				// The circuit receives the three siblings in sorted order plus the
+				// current hash's position in the sorted four-child node.
+				for siblings in merkle_proof.siblings {
+					let mut all = [current_hash, siblings[0], siblings[1], siblings[2]];
+					all.sort();
+					let position = all.iter().position(|hash| *hash == current_hash).unwrap();
+					zk_merkle_positions.push(position as u8);
+					let mut sorted_siblings = [[0u8; 32]; 3];
+					let mut sibling_index = 0;
+					for (child_index, hash) in all.iter().enumerate() {
+						if child_index != position {
+							sorted_siblings[sibling_index] = *hash;
+							sibling_index += 1;
+						}
+					}
+					zk_merkle_siblings.push(sorted_siblings);
+					current_hash = pallet_zk_tree::tree::hash_node(&all);
+				}
+				assert_eq!(current_hash, zk_tree_root);
+
+				let nullifier =
+					digest_to_bytes(Nullifier::from_preimage(secret, 0).hash);
+				real_nullifiers.push(nullifier.as_ref().try_into().unwrap());
+				let inputs = CircuitInputs {
+					public: PublicCircuitInputs {
+						asset_id: 0,
+						output_amount_1: 1,
+						output_amount_2: 0,
+						volume_fee_bps: 4,
+						nullifier,
+						exit_account_1: BytesDigest::new_unchecked([10 + index as u8; 32]),
+						exit_account_2: BytesDigest::default(),
+						block_hash,
+						block_number,
+					},
+					private: PrivateCircuitInputs {
+						secret: secret.into(),
+						transfer_count: 0,
+						unspendable_account: unspendable_accounts[index],
+						parent_hash: BytesDigest::new_unchecked(parent_hash),
+						state_root: BytesDigest::new_unchecked(state_root),
+						extrinsics_root: BytesDigest::new_unchecked(extrinsics_root),
+						digest,
+						input_amount: 2,
+						zk_tree_root,
+						zk_merkle_siblings,
+						zk_merkle_positions,
+					},
+				};
+
+				leaf_proofs.push(
+					qp_wormhole_prover::build_fresh()
+						.commit(&inputs)
+						.expect("commit leaf")
+						.prove()
+						.expect("prove leaf"),
+				);
+			}
+
+			let aggregate = PrivateBatchProver::new_from_binaries_dir(std::path::Path::new(env!("OUT_DIR")))
+				.expect("private-batch prover")
+				.aggregate(leaf_proofs)
+				.expect("aggregate two real leaves");
+			let proof_bytes = aggregate.to_bytes();
+
+			// Parse using the exact verifier embedded by the pallet, proving that the
+			// production aggregate exposes two one-quantum outputs to runtime settlement.
+			let verifier = crate::get_private_batch_verifier().expect("embedded verifier");
+			let parsed = ProofWithPublicInputs::<F, C, D>::from_bytes(
+				proof_bytes.clone(),
+				&verifier.circuit_data.common,
+			)
+			.expect("aggregate deserializes");
+			let public_inputs =
+				parse_private_batch_public_inputs(&parsed).expect("aggregate public inputs");
+			let exit_quanta: u128 = public_inputs
+				.account_data
+				.iter()
+				.map(|account| account.summed_output_amount as u128)
+				.sum();
+			assert_eq!(exit_quanta, 2);
+
+			let block_hash_bytes: [u8; 32] = block_hash.as_ref().try_into().unwrap();
+			frame_system::BlockHash::<Test>::insert(block_number as u64, H256::from(block_hash_bytes));
+			System::set_block_number(block_number as u64 + 10);
+
+			let call = crate::Call::<Test>::verify_private_batch { proof_bytes: proof_bytes.clone() };
+			assert!(<Wormhole as ValidateUnsigned>::pre_dispatch(&call).is_ok());
+
+			assert_ok!(Balances::mint_into(&account_id(999), 1_000 * UNIT));
+			let issuance_before = <Balances as Inspect<AccountId>>::total_issuance();
+			assert_ok!(Wormhole::verify_private_batch(RawOrigin::None.into(), proof_bytes));
+
+			assert_eq!(Balances::balance(&sp_core::crypto::AccountId32::new([10u8; 32])), crate::SCALE_DOWN_FACTOR);
+			assert_eq!(Balances::balance(&sp_core::crypto::AccountId32::new([11u8; 32])), crate::SCALE_DOWN_FACTOR);
+			for nullifier in &real_nullifiers {
+				assert!(UsedNullifiers::<Test>::contains_key(nullifier));
+			}
+
+			let issuance_after = <Balances as Inspect<AccountId>>::total_issuance();
+			let fee_locked_by_two_leaf_constraints = 2 * crate::SCALE_DOWN_FACTOR;
+			let fee_settled_from_aggregate = super::ceil_volume_fee(2, 4);
+			assert_eq!(fee_settled_from_aggregate, crate::SCALE_DOWN_FACTOR);
+			assert_eq!(issuance_before - issuance_after, fee_settled_from_aggregate);
+			println!("real leaf proofs aggregated: 2");
+			println!(
+				"proof-locked minimum fee: {} planck (0.02 QUAN)",
+				fee_locked_by_two_leaf_constraints
+			);
+			println!(
+				"runtime-settled fee: {} planck (0.01 QUAN)",
+				fee_settled_from_aggregate
+			);
+			println!("permanently omitted fee: {} planck (0.01 QUAN)", crate::SCALE_DOWN_FACTOR);
+			println!("real nullifiers consumed: {}", real_nullifiers.len());
+			assert_eq!(
+				fee_locked_by_two_leaf_constraints - fee_settled_from_aggregate,
+				crate::SCALE_DOWN_FACTOR,
+				"one paid fee quantum remains permanently unburned and undistributed"
+			);
+		});
+	}
+
 	/// Pool admission must not derive `priority` from unverified public-input amounts.
 	///
 	/// With verification deferred to `pre_dispatch`, an attacker who sees a victim's
```

### Focused cryptographic output

```
Scoped repository: audit-comp-quantus-chain
Commit: 3b243b870a5442a6e5f443056f67bbba87d472f5
Command:
  cargo test -p pallet-wormhole --lib \
    poc_real_private_batch_strands_per_leaf_fee_rounding -- \
    --ignored --nocapture

running 1 test
real leaf proofs aggregated: 2
proof-locked minimum fee: 20000000000 planck (0.02 QUAN)
runtime-settled fee: 10000000000 planck (0.01 QUAN)
permanently omitted fee: 10000000000 planck (0.01 QUAN)
real nullifiers consumed: 2
test tests::private_batch_proof_tests::poc_real_private_batch_strands_per_leaf_fee_rounding ... ok

test result: ok. 1 passed; 0 failed; 0 ignored; 0 measured; 71 filtered out; finished in 178.09s
```

### Clean pallet-suite output

```
Scoped repository: audit-comp-quantus-chain
Commit: 3b243b870a5442a6e5f443056f67bbba87d472f5
Clean detached worktree: yes
Command:
  cargo test -p pallet-wormhole --lib

test result: ok. 68 passed; 0 failed; 4 ignored; 0 measured; 0 filtered out

The focused cryptographic PoC is marked ignored because it generates two fresh leaf proofs and a production private-batch proof. It was executed separately; see focused-output.txt.
```
