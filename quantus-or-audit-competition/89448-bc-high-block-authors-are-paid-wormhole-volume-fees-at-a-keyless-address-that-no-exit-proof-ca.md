# 89448 bc high block authors are paid wormhole volume fees at a keyless address that no exit proof can unlock

**Submitted on Aug 19th 2026 at 12:02:53 UTC by @Pyro for** [**Audit Comp | Quantus**](https://immunefi.com/audit-competition/audit-comp-quantus)

* **Report ID:** #89448
* **Report Type:** Blockchain/DLT
* **Report severity:** High
* **Target:** <https://github.com/immunefi-team/audit-comp-quantus-chain>
* **Impacts:**
  * Permanent freezing of funds

## Description

A wormhole address holds no spendable balance. `process_exit_bundle` never debits the source address when it settles an exit. What makes value at such an address usable is a ZK-tree leaf, and `record_transfer` is the pallet's only leaf writer.

The exit-credit path pairs the credit with the recorder:

```rust
				match <T::Currency as Unbalanced<_>>::increase_balance(
					exit_account,
					*exit_balance,
					frame_support::traits::tokens::Precision::Exact,
				) {
```

```rust
				Self::record_transfer(
					T::AssetId::default(),
					&from_account,
					&to_account,
					*exit_balance,
				);
```

The miner volume fee uses the same event-free primitive and stops at the event:

```rust
					match <T::Currency as Unbalanced<_>>::increase_balance(
						&author,
						miner_fee,
						frame_support::traits::tokens::Precision::Exact,
					) {
						Ok(_) => {
							Self::deposit_event(Event::MinerVolumeFeePaid {
								miner: author,
								amount: miner_fee,
							});
						},
```

The author is `qp_wormhole::extract_author_from_digest`, which returns `derive_wormhole_address(preimage)`:

```rust
/// This hashes the inner_digest using Poseidon to get the wormhole account address.
/// The inner_digest is the "first hash" from wormhole derivation: `hash(salt + secret)`.
/// The wormhole address is: `address = hash(hash(salt + secret))`.
```

The payee is a Poseidon image, so no signing key exists for it. A ZK exit is the only way to move value sitting there, and an exit needs a leaf. The miner holds the secret and can satisfy the circuit, but this credit leaves it nothing to prove against.

Nothing repairs the omission later:

* `increase_balance` writes the balance without emitting a `Balances` event, so `WormholeProofRecorderExtension` cannot see it. The pallet's own test `exit_credits_emit_no_scannable_transfer_events` pins that this path emits nothing scannable.
* The settlement is an unsigned bare extrinsic, where that extension does not run at all.
* `on_initialize` records leaves only at block 1, so the genesis sweep is long past.
* The pallet exposes only `verify_private_batch` and `verify_public_batch`, so no admin path can insert the leaf.
* Root cannot insert one either. Root dispatched calls bypass the recorder extension by design (audit item EQ-QNT-WORMHOLE-F-05), so a rescue needs a separately funded signed transfer, per miner, per settlement.
* Leaves are per `(recipient, transfer_count)` and each commits one fixed amount, so the mining-rewards leaf does not cover the fee.

Both failure arms of the same branch add the unpaid fee back to `burn_amount`. The branch's contract is pay or burn. The success path does neither.

Example:

{% stepper %}
{% step %}

## A miner runs with a stable `rewards_inner_hash`

Its author account already exists from earlier block rewards.
{% endstep %}

{% step %}

## A user submits an exit

Any user submits a valid unsigned `verify_private_batch` settling a 1,000 QUAN exit.
{% endstep %}

{% step %}

## The volume fee is calculated

`volume_fee_for_exit` computes `ceil(100_000 * 4 / 9996) = 41` quanta, so the total fee is 410,000,000,000 planck.
{% endstep %}

{% step %}

## The miner fee is calculated

`VolumeFeesBurnRate` is 50 percent, so 205,000,000,000 planck burns and `miner_fee` is 205,000,000,000 planck.
{% endstep %}

{% step %}

## The author balance is credited

`increase_balance(&author, 205_000_000_000, Exact)` succeeds and `MinerVolumeFeePaid` is emitted.
{% endstep %}

{% step %}

## No leaf is recorded

`TransferCount` for the author is unchanged and no `NativeTransferred` event fires, so no leaf exists for the credit.
{% endstep %}

{% step %}

## The entitlement remains unchanged

The author's balance is 0.205 QUAN higher and its exitable entitlement is unchanged.
{% endstep %}

{% step %}

## Later exits repeat the issue

Every later exit repeats this, so all wormhole volume-fee revenue accrues to block authors and none is deliverable.
{% endstep %}
{% endstepper %}

## Recommendation

Record the leaf inside the same success arm, exactly as the sibling exit-credit branch does.

```diff
 					match <T::Currency as Unbalanced<_>>::increase_balance(
 						&author,
 						miner_fee,
 						frame_support::traits::tokens::Precision::Exact,
 					) {
 						Ok(_) => {
+							let from_account: <T as Config>::WormholeAccountId =
+								mint_account.clone().into();
+							let to_account: <T as Config>::WormholeAccountId =
+								author.clone().into();
+							Self::record_transfer(
+								T::AssetId::default(),
+								&from_account,
+								&to_account,
+								miner_fee,
+							);
 							Self::deposit_event(Event::MinerVolumeFeePaid {
 								miner: author,
 								amount: miner_fee,
 							});
 						},
```

Better, factor a helper that does `increase_balance` then `record_transfer`, and route every event-free credit in `process_exit_bundle` through it so no future branch can credit without recording.

## Proof of Concept

Verified against commit `3b243b8`. The test builds a cryptographically valid exit proof for the credited fee, submits it through the real unsigned extrinsic, and the chain reverts. The same construction for a recorded credit is accepted and pays out.

```bash
cargo test --release -p pallet-wormhole --lib -- poc_h01 --nocapture --ignored
```

### Setup

`pallets/wormhole` already carries `qp-wormhole-prover`, `qp-wormhole-circuit` and `qp-wormhole-aggregator` as dev-dependencies, so real proofs can be built in-tree. Two more are needed to drive the real emission path. In `pallets/wormhole/Cargo.toml`, `[dev-dependencies]`:

```toml
pallet-mining-rewards = { workspace = true, features = ["std"] }
pallet-treasury = { workspace = true, features = ["std"] }
```

In `pallets/wormhole/src/mock.rs`, add `MiningRewards: pallet_mining_rewards` to `construct_runtime!` and:

```rust
parameter_types! {
	pub const MaxSupply: Balance = 21_000_000 * UNIT;
	pub const EmissionDivisor: Balance = 15_163_560;
	pub const MiningUnit: Balance = UNIT;
}

/// Minimal treasury provider so the real mining-rewards emission path can run here.
pub struct MockTreasury;
impl pallet_treasury::TreasuryProvider for MockTreasury {
	type AccountId = AccountId;
	fn account_id() -> AccountId {
		AccountId::new([1u8; 32])
	}
	fn portion() -> Permill {
		Permill::from_percent(50)
	}
}

impl pallet_mining_rewards::Config for Test {
	type Currency = Balances;
	type AssetId = u32;
	// The real recorder: mining rewards insert real leaves into the real ZK tree.
	type ProofRecorder = Wormhole;
	type WeightInfo = ();
	type MaxSupply = MaxSupply;
	type EmissionDivisor = EmissionDivisor;
	type Treasury = MockTreasury;
	type MintingAccount = MintingAccount;
	type Unit = MiningUnit;
}
```

Shared prover plumbing, appended to `pallets/wormhole/src/tests.rs`:

```rust
/// Shared prover plumbing for the PoCs: builds real leaf proofs and real private
/// batches, so a PoC can submit a cryptographically valid exit to the chain.
#[cfg(test)]
pub(crate) mod poc_prover {
	use crate::mock::*;

	pub(crate) struct Batch {
		pub block_hash: [u8; 32],
		pub bytes: Vec<u8>,
	}

	/// The PoW preimage a miner publishes is the circuit's inner hash of its secret.
	pub(crate) fn miner_preimage(secret_bytes: [u8; 32]) -> [u8; 32] {
		use plonky2::{hash::poseidon2::Poseidon2Hash, plonk::config::Hasher};
		use qp_wormhole_circuit::unspendable_account::UNSPENDABLE_SALT;
		use qp_wormhole_inputs::BytesDigest;
		use qp_zk_circuits_common::utils::{bytes_to_digest, digest_to_bytes, string_to_felts};

		let secret_felts = bytes_to_digest(BytesDigest::new_unchecked(secret_bytes));
		let mut preimage = Vec::new();
		preimage.extend(string_to_felts(UNSPENDABLE_SALT).unwrap());
		preimage.extend(&secret_felts);
		*digest_to_bytes(Poseidon2Hash::hash_no_pad(&preimage).elements)
	}

	/// The pallet stores siblings only (its `hash_node` sorts internally). The circuit
	/// also wants a position hint, so derive both by walking the path.
	pub(crate) fn merkle_path(leaf_index: u64) -> (Vec<[[u8; 32]; 3]>, Vec<u8>) {
		let proof = ZkTree::get_merkle_proof(leaf_index).expect("leaf exists");
		let leaf = ZkTree::leaf(leaf_index).expect("leaf exists");
		let mut current = pallet_zk_tree::tree::hash_leaf::<Test>(&leaf);
		let (mut sibs, mut poss) = (Vec::new(), Vec::new());
		for level in &proof.siblings {
			let mut sorted = *level;
			sorted.sort();
			let mut all = [current, level[0], level[1], level[2]];
			all.sort();
			poss.push(all.iter().position(|h| *h == current).unwrap() as u8);
			sibs.push(sorted);
			current = pallet_zk_tree::tree::hash_node(&[current, level[0], level[1], level[2]]);
		}
		(sibs, poss)
	}

	/// Build a real leaf proof and aggregate it into a real private batch.
	///
	/// Returns the block hash the proof commits to, so the caller can decide whether
	/// the chain should be holding that header or a different one.
	#[allow(clippy::too_many_arguments)]
	pub(crate) fn build_batch(
		secret_bytes: [u8; 32],
		transfer_count: u64,
		zk_tree_root: [u8; 32],
		siblings: Vec<[[u8; 32]; 3]>,
		positions: Vec<u8>,
		input_quanta: u32,
		output_quanta: u32,
		exit_account: [u8; 32],
	) -> Result<Batch, String> {
		use qp_wormhole_aggregator::private_batch::prover::PrivateBatchProver;
		use qp_wormhole_circuit::{
			block_header::header::HeaderInputs,
			inputs::{CircuitInputs, PrivateCircuitInputs},
			nullifier::Nullifier,
			unspendable_account::UnspendableAccount,
		};
		use qp_wormhole_inputs::{BytesDigest, PublicCircuitInputs};
		use qp_zk_circuits_common::utils::digest_to_bytes;

		let secret = BytesDigest::new_unchecked(secret_bytes);
		let unspendable = *digest_to_bytes(UnspendableAccount::from_secret(secret).account_id);
		let nullifier = digest_to_bytes(Nullifier::from_preimage(secret, transfer_count).hash);
		let zero = [0u8; 32];
		let digest = [0u8; 110];

		let header = HeaderInputs::new(
			BytesDigest::new_unchecked(zero),
			1u32,
			BytesDigest::new_unchecked(zero),
			BytesDigest::new_unchecked(zero),
			BytesDigest::new_unchecked(zk_tree_root),
			&digest,
		)
		.map_err(|e| format!("header: {e:?}"))?;
		let block_hash = header.block_hash();

		let inputs = CircuitInputs {
			public: PublicCircuitInputs {
				asset_id: 0,
				output_amount_1: output_quanta,
				output_amount_2: 0,
				volume_fee_bps: 4,
				nullifier,
				exit_account_1: BytesDigest::new_unchecked(exit_account),
				exit_account_2: BytesDigest::default(),
				block_hash,
				block_number: 1,
			},
			private: PrivateCircuitInputs {
				secret: secret.into(),
				transfer_count,
				unspendable_account: BytesDigest::new_unchecked(unspendable),
				parent_hash: BytesDigest::new_unchecked(zero),
				state_root: BytesDigest::new_unchecked(zero),
				extrinsics_root: BytesDigest::new_unchecked(zero),
				digest,
				input_amount: input_quanta,
				zk_tree_root,
				zk_merkle_siblings: siblings,
				zk_merkle_positions: positions,
			},
		};

		let leaf_proof = qp_wormhole_prover::build_fresh()
			.commit(&inputs)
			.map_err(|e| format!("commit: {e:?}"))?
			.prove()
			.map_err(|e| format!("prove: {e:?}"))?;

		let bins = std::path::Path::new(env!("OUT_DIR"));
		let prover = PrivateBatchProver::new_from_binaries_dir(bins)
			.map_err(|e| format!("prover: {e:?}"))?;
		let aggregated =
			prover.aggregate(vec![leaf_proof]).map_err(|e| format!("aggregate: {e:?}"))?;

		Ok(Batch {
			block_hash: block_hash.as_ref().try_into().unwrap(),
			bytes: aggregated.to_bytes(),
		})
	}
}
```

The test:

```rust
#[cfg(test)]
mod poc_h01 {
	use super::poc_prover::{build_batch, merkle_path, miner_preimage};
	use crate::{
		mock::*,
		pallet::{Error, ExitBundle, ExitSegment},
	};
	use frame_support::{assert_ok, traits::fungible::Inspect};
	use frame_system::RawOrigin;
	use qp_wormhole_verifier::{BlockData, BytesDigest, PublicInputsByAccount};
	use sp_core::{crypto::AccountId32, H256};
	use sp_runtime::DispatchError;

	const EXIT: [u8; 32] = [8u8; 32];

	fn q(n: u32) -> u128 {
		n as u128 * crate::SCALE_DOWN_FACTOR
	}

	/// Submits a real, cryptographically valid exit proof for the miner volume fee and
	/// shows the chain rejects it, while the identical construction for a recorded
	/// credit is accepted and pays out.
	#[test]
	#[ignore = "generates real ZK proofs, run with --release"]
	fn poc_miner_fee_cannot_be_withdrawn() {
		let secret = [42u8; 32];
		let preimage = miner_preimage(secret);
		let author = AccountId32::from(qp_wormhole::derive_wormhole_address(preimage).unwrap());
		let exit_acct = AccountId32::new(EXIT);
		let mint = AccountId32::new([3u8; 32]);

		new_test_ext().execute_with(|| {
			System::set_block_number(5);
			set_miner_preimage_digest(preimage);

			// ---- 1. A 10,000 QUAN exit settles and pays the author its volume fee ----
			let bundle = ExitBundle {
				asset_id: 0,
				volume_fee_bps: VolumeFeeRateBps::get(),
				block_data: BlockData::default(),
				aggregator_address: None,
				segments: vec![ExitSegment {
					account_data: vec![PublicInputsByAccount {
						summed_output_amount: 1_000_000,
						exit_account: BytesDigest::new_unchecked([4u8; 32]),
					}],
					nullifiers: vec![BytesDigest::new_unchecked([9u8; 32])],
				}],
			};
			let before = Balances::balance(&author);
			Wormhole::process_exit_bundle(bundle).expect("settlement succeeds");
			let fee = Balances::balance(&author) - before;
			let fee_q = (fee / crate::SCALE_DOWN_FACTOR) as u32;
			println!("volume fee credited      : {} planck ({} quanta)", fee, fee_q);
			assert!(fee_q >= 2, "the fee clears the two-quantum circuit minimum");
			// `record_transfer` bumps this on every leaf written to the author, so 0 leaves.
			assert_eq!(Wormhole::transfer_count(&author), 0, "no leaf was recorded for it");

			// ---- 2. Give the author a recorded credit, to serve as the control ----
			Wormhole::record_transfer(0, &mint, &author, q(100));
			let control_idx = ZkTree::leaf_count() - 1;
			let live_root = ZkTree::root();
			let (sibs, poss) = merkle_path(control_idx);
			println!("live chain root          : {}", hex::encode(live_root));

			// ---- 3. Control: a real exit of the recorded credit, accepted on chain ----
			let control = build_batch(secret, 0, live_root, sibs.clone(), poss.clone(), 100, 99, EXIT)
				.expect("a recorded credit is provable");
			frame_system::BlockHash::<Test>::insert(1u64, H256::from(control.block_hash));
			let exit_before = Balances::balance(&exit_acct);
			assert_ok!(Wormhole::verify_private_batch(RawOrigin::None.into(), control.bytes));
			let paid = Balances::balance(&exit_acct) - exit_before;
			println!("control exit paid out    : {} planck", paid);
			assert_eq!(paid, q(99), "the withdrawal path is live and moves real funds");

			// ---- 4. Door 1: prove the fee against the chain's own root ----
			let door1 = build_batch(secret, 1, live_root, sibs, poss, fee_q, fee_q - 1, EXIT);
			println!("door 1 (live root)       : {:?}", door1.as_ref().err().map(|_| "UNPROVABLE"));
			assert!(door1.is_err(), "no leaf under the live root pays the fee");

			// ---- 5. Record the missing leaf, so the tree differs by exactly it ----
			// This is the single missing call from `process_exit_bundle`.
			let pre_fix_root = ZkTree::root();
			let pre_fix_leaves = ZkTree::leaf_count();
			Wormhole::record_transfer(0, &mint, &author, fee);
			let fixed_root = ZkTree::root();
			let fee_idx = ZkTree::leaf_count() - 1;
			let fee_leaf = ZkTree::leaf(fee_idx).unwrap();
			let (fee_sibs, fee_poss) = merkle_path(fee_idx);
			println!("root with the fee leaf   : {}", hex::encode(fixed_root));
			println!(
				"the tree differs by exactly one leaf: {} -> {}",
				pre_fix_leaves,
				ZkTree::leaf_count()
			);
			assert_ne!(fixed_root, pre_fix_root, "the live chain never held this root");
			assert_eq!(ZkTree::leaf_count(), pre_fix_leaves + 1, "exactly one leaf was added");
			assert_eq!(fee_leaf.to, author);
			assert_eq!(fee_leaf.amount, fee, "and it commits exactly the credited fee");

			// ---- 6. Door 2: the fee IS withdrawable under that root ----
			let door2 = build_batch(
				secret, 1, fixed_root, fee_sibs, fee_poss, fee_q, fee_q - 1, EXIT,
			)
			.expect("with the leaf present the exit is provable");

			// The chain's header still commits the live root, so it rejects the proof.
			let err = Wormhole::verify_private_batch(RawOrigin::None.into(), door2.bytes.clone())
				.expect_err("the chain must reject it");
			println!("door 2 against live chain: {:?}", err.error);
			assert_eq!(err.error, DispatchError::from(Error::<Test>::InvalidPublicInputs));

			// ---- 7. The only difference is the missing record_transfer ----
			frame_system::BlockHash::<Test>::insert(1u64, H256::from(door2.block_hash));
			let before2 = Balances::balance(&exit_acct);
			assert_ok!(Wormhole::verify_private_batch(RawOrigin::None.into(), door2.bytes));
			println!(
				"same proof, fee leaf present: paid {} planck",
				Balances::balance(&exit_acct) - before2
			);
		});
	}
}
```

Actual output:

```
volume fee credited      : 2005000000000 planck (200 quanta)
live chain root          : d71490900dbf68a9a8b284852ced45f482361fbf8bde07962a1b11ab4a194e1f
control exit paid out    : 990000000000 planck
door 1 (live root)       : Some("UNPROVABLE")
root with the fee leaf   : 9e1f1150893dc99df7e10bc51ebb03e28052f69e5194d9aaa632db3582b42382
the tree differs by exactly one leaf: 3 -> 4
door 2 against live chain: Module(ModuleError { index: 3, error: [0, 0, 0, 0], message: Some("InvalidPublicInputs") })
same proof, fee leaf present: paid 1990000000000 planck
```

Read in order:

{% stepper %}
{% step %}

## A volume fee is credited without a leaf

A 10,000 QUAN exit credits the author 200 quanta of volume fee and leaves its transfer count at zero, so no leaf commits it.
{% endstep %}

{% step %}

## The withdrawal path is live

A recorded credit at the same address is proven, submitted through `verify_private_batch`, accepted, and pays out 990,000,000,000 planck of real balance.
{% endstep %}

{% step %}

## Door 1

Proving the fee against the chain's own tree root is impossible. No leaf under that root pays it.
{% endstep %}

{% step %}

## A tree differing by exactly the missing leaf

Calling the single missing `record_transfer` adds one leaf, committing exactly the credited fee to exactly the author. The tree goes from 3 leaves to 4 and the root changes.
{% endstep %}

{% step %}

## Door 2

Under that root the exit is provable. Submitted against the live chain, whose header still commits the old root, it reverts with `InvalidPublicInputs`.
{% endstep %}

{% step %}

## The only difference is the missing call

Give the chain the header carrying the fee leaf, resubmit the identical bytes, and it is accepted and pays 1,990,000,000,000 planck.
{% endstep %}
{% endstepper %}

Step 4 is what makes the rejection specific. The proof is not rejected because it is malformed or aimed at the wrong block. It is rejected because the chain's tree is missing one leaf, and step 4 builds the tree that differs from the live one by exactly that leaf.
