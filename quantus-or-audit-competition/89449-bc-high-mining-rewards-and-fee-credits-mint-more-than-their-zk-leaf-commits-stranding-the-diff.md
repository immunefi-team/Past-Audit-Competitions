# 89449 bc high mining rewards and fee credits mint more than their zk leaf commits stranding the difference at a keyless miner address

**Submitted on Aug 19th 2026 at 12:04:39 UTC by @Pyro for** [**Audit Comp | Quantus**](https://immunefi.com/audit-competition/audit-comp-quantus)

* **Report ID:** #89449
* **Report Type:** Blockchain/DLT
* **Report severity:** High
* **Target:** <https://github.com/immunefi-team/audit-comp-quantus-chain>
* **Impacts:**
  * Permanent freezing of funds

## Description

`mint_reward` mints the raw reward and hands the same raw amount to the proof recorder:

```rust
					match T::Currency::mint_into(miner, reward) {
						Ok(_) => {
							T::ProofRecorder::record_transfer_proof(
								None, // Native token
								mint_account,
								miner.clone(),
								reward,
							);
```

`hash_leaf` commits only whole quanta:

```rust
	let amount_quantized =
		u32::try_from(amount_u128 / AMOUNT_SCALE_DOWN_FACTOR).unwrap_or(u32::MAX);
```

`AMOUNT_SCALE_DOWN_FACTOR` is 10,000,000,000 planck. The sub-quantum remainder never enters the Poseidon preimage, so no circuit can witness it, and `process_exit_bundle` reconstructs `summed_output_amount * SCALE_DOWN_FACTOR`. Entitlement stops at whole quanta while the balance is the raw amount.

`on_finalize` resolves the payee through `qp_wormhole::extract_author_from_digest`, which hashes the PoW preimage:

```rust
/// This hashes the inner_digest using Poseidon to get the wormhole account address.
/// The inner_digest is the "first hash" from wormhole derivation: `hash(salt + secret)`.
/// The wormhole address is: `address = hash(hash(salt + secret))`.
```

No signing key exists for a Poseidon image, so a ZK exit is the only way to spend a block reward, and an exit can claim only what the leaf commits.

There is no accumulator. `on_finalize` makes three independent `mint_reward` calls and each opens its own leaf at the next `transfer_count`, with its own floor. The nullifier binds to `(secret, transfer_count)`, so a leaf spends once and a later leaf cannot absorb an earlier remainder. Nothing repairs it afterwards either: `record_transfer` is the only leaf writer, `on_initialize` records only at block 1, and a leaf's committed amount is fixed at insert.

The fee leg fares worse than the reward leg. Transaction fees get their own `mint_reward` call, so they get their own leaf and their own floor. The quantum is 0.01 UNIT, and the runtime prices a typical 500-byte transfer at about 0.0005 UNIT of length fee. Any block whose fees total under one quantum commits a zero-value leaf, so the miner can withdraw none of them, while the runtime documents that "100% of transaction fees go to the block miner". The dead leaf still consumes a `transfer_count` index.

The runtime already names this hazard and enforces the invariant elsewhere. `VestingPayoutQuantum` is defined as `pallet_wormhole::SCALE_DOWN_FACTOR`, with the comment that "a sub-quantum transfer would be committed as a zero-value leaf, stranding funds paid to keyless beneficiaries". Pallet-vesting honours it:

```rust
		/// Round down to a multiple of the payout quantum.
		fn quantize_down(amount: BalanceOf<T>) -> BalanceOf<T> {
			let remainder = amount % T::PayoutQuantum::get();
			amount.checked_sub(&remainder).expect("remainder never exceeds the dividend")
		}
```

Vesting pays whole quanta and keeps the remainder in the pot for the next claim. Mining-rewards mints the remainder into an address that can never claim it.

The same chokepoint accepts unaligned credits from the event scanner, so a third-party transfer sized for a wormhole recipient loses its remainder the same way. `record_transfer_proof` rejects only a zero amount. There is no alignment check and no usable minimum.

The shortfall is permanent and recurs on every block. At the 12,000 ms target that is 7,200 block rewards a day, each stranding its own remainder at an address that can never claim it, while total issuance rises by the full raw amount and the unclaimable part consumes the 21,000,000 QUAN supply budget. Emission decays, so the shortfall grows as a share of the reward until the per-block reward itself falls under one quantum and strands in full.

### Example

{% stepper %}
{% step %}

## The chain starts from the Planck preset

Genesis issuance is 100,009,001,000,000,000 planck.
{% endstep %}

{% step %}

## The first block carries a PoW pre-runtime digest

It collects no transaction fees.
{% endstep %}

{% step %}

## Calculate the total reward

`on_finalize` computes:

```
total_reward = (2.1e19 - 100_009_001_000_000_000) / 15_163_560 = 1_378_303_709_617 planck
```

{% endstep %}

{% step %}

## Split the treasury portion

The 50 percent treasury portion floors to 689,151,854,808 planck, so `miner_reward` is 689,151,854,809 planck.
{% endstep %}

{% step %}

## Mint the full raw reward

`mint_into` credits the full 689,151,854,809 planck and total issuance rises by the same amount.
{% endstep %}

{% step %}

## Commit only whole quanta

`hash_leaf` commits:

```
689_151_854_809 / 10_000_000_000 = 68 quanta
```

This is worth 680,000,000,000 planck.
{% endstep %}

{% step %}

## Identify the stranded entitlement

9,151,854,809 planck of that block's reward is entitlement that was never created, 1.33 percent of the reward.
{% endstep %}

{% step %}

## Repeat the shortfall

The next block opens a new leaf at `transfer_count + 1` with no remainder carried in, and the shortfall repeats for every block author.
{% endstep %}
{% endstepper %}

## Recommendation

Apply the rule pallet-vesting already applies. Pay the miner one credit, quantized down to whole leaf quanta, and roll the remainder into the next block's bucket.

```diff
-		// Send fees to miner if any
-		Self::mint_reward(miner.as_ref(), tx_fees);
-
-		// Send block rewards to miner
-		Self::mint_reward(miner.as_ref(), miner_reward);
+		// One credit, one leaf. Pay whole leaf quanta only and carry the sub-quantum
+		// remainder into the next block, so nothing is minted that no leaf can commit.
+		let miner_total = tx_fees.saturating_add(miner_reward);
+		let quantum = T::LeafQuantum::get();
+		let remainder = miner_total % quantum;
+		Self::retain_unminted(remainder);
+		Self::mint_reward(miner.as_ref(), miner_total.saturating_sub(remainder));
```

`LeafQuantum` is a new pallet constant, wired in the runtime to `pallet_wormhole::SCALE_DOWN_FACTOR`, exactly as `VestingPayoutQuantum` already is.

Add the same guard at the shared chokepoint so no caller can bypass it. In `record_transfer_proof`, reject a native credit to a wormhole-form recipient that is not a multiple of `AMOUNT_SCALE_DOWN_FACTOR`, or that is below the two-quantum minimum the circuit fee relation needs for a nonzero output.

## Proof of Concept

Verified against commit `3b243b8`. The leaves come from the real emission path, the recipient is the address the PoW digest actually pays, and the failing withdrawal is a valid proof reverted on chain.

```bash
cargo test --release -p pallet-wormhole --lib -- poc_h02 --nocapture --ignored
```

### Setup

`pallets/wormhole` already carries `qp-wormhole-prover`, `qp-wormhole-circuit` and `qp-wormhole-aggregator` as dev-dependencies, so real proofs can be built in-tree. Two more are needed to drive the real emission path.

In `pallets/wormhole/Cargo.toml`, `[dev-dependencies]`:

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
mod poc_h02 {
	use super::poc_prover::{build_batch, merkle_path, miner_preimage};
	use crate::{mock::*, pallet::Error};
	use frame_support::{
		assert_ok,
		traits::{fungible::Inspect, Hooks},
	};
	use frame_system::RawOrigin;
	use sp_core::{crypto::AccountId32, H256};
	use sp_runtime::DispatchError;

	const EXIT: [u8; 32] = [8u8; 32];
	const QUANTUM: u128 = crate::SCALE_DOWN_FACTOR;

	/// Runs the real emission path, then submits real exit proofs showing the miner can
	/// withdraw only what its leaves commit, and that the accumulated remainder becomes
	/// withdrawable the moment the carry fix records it.
	#[test]
	#[ignore = "generates real ZK proofs, run with --release"]
	fn poc_reward_remainder_is_unexitable() {
		let secret = [42u8; 32];
		let preimage = miner_preimage(secret);
		let miner = AccountId32::from(qp_wormhole::derive_wormhole_address(preimage).unwrap());
		let exit_acct = AccountId32::new(EXIT);
		let mint = AccountId32::new([3u8; 32]);

		new_test_ext().execute_with(|| {
			System::set_block_number(5);
			set_miner_preimage_digest(preimage);
			// mining-rewards resolves its payee through this exact function.
			assert_eq!(
				qp_wormhole::extract_author_from_digest::<AccountId32, _>(
					System::digest().logs.iter()
				),
				Some(miner.clone()),
				"the emission path pays exactly this address"
			);

			// ---- 1. Twenty blocks of real emission ----
			for b in 1..=20u64 {
				MiningRewards::on_finalize(b);
			}
			let leaves: Vec<(u64, u128)> = (0..ZkTree::leaf_count())
				.filter_map(|i| {
					let l = ZkTree::leaf(i).unwrap();
					(l.to == miner).then_some((i, l.amount))
				})
				.collect();
			let stranded: u128 = leaves.iter().map(|(_, a)| a % QUANTUM).sum();
			println!("miner leaves over 20 blocks : {}", leaves.len());
			println!("stranded across them        : {} planck", stranded);
			assert_eq!(leaves.len(), 20, "one leaf per block, nothing merges them");
			assert!(stranded > QUANTUM, "the remainders exceed a full quantum");

			let (idx0, minted0) = leaves[0];
			let floored0 = (minted0 / QUANTUM) as u32;
			let live_root = ZkTree::root();
			let (sibs0, poss0) = merkle_path(idx0);
			println!(
				"leaf {}: minted {} planck, commits {} quanta, remainder {} planck",
				idx0,
				minted0,
				floored0,
				minted0 % QUANTUM
			);
			println!("live chain root             : {}", hex::encode(live_root));

			// ---- 2. Door 1: claim one quantum more against the chain's own root ----
			let door1 = build_batch(
				secret, 0, live_root, sibs0, poss0, floored0 + 1, floored0, EXIT,
			);
			println!("door 1 (live root)          : {:?}", door1.as_ref().err().map(|_| "UNPROVABLE"));
			assert!(door1.is_err(), "the leaf commits {floored0} quanta and no more");

			// ---- 3. Control: withdraw exactly what a real leaf commits ----
			let (idx1, minted1) = leaves[1];
			let floored1 = (minted1 / QUANTUM) as u32;
			let (sibs1, poss1) = merkle_path(idx1);
			let control =
				build_batch(secret, 1, live_root, sibs1, poss1, floored1, floored1 - 1, EXIT)
					.expect("the floored amount is provable");
			frame_system::BlockHash::<Test>::insert(1u64, H256::from(control.block_hash));
			let before = Balances::balance(&exit_acct);
			assert_ok!(Wormhole::verify_private_batch(RawOrigin::None.into(), control.bytes));
			let paid = Balances::balance(&exit_acct) - before;
			println!("control withdrew            : {} planck of {} minted", paid, minted1);
			assert_eq!(paid, (floored1 - 1) as u128 * QUANTUM);

			// ---- 4. Record the carry leaf, so the tree differs by exactly it ----
			// This is what the recommended remainder accumulator would eventually record.
			let pre_fix_root = ZkTree::root();
			let pre_fix_leaves = ZkTree::leaf_count();
			Wormhole::record_transfer(0, &mint, &miner, stranded);
			let fixed_root = ZkTree::root();
			let carry_idx = ZkTree::leaf_count() - 1;
			let carry_leaf = ZkTree::leaf(carry_idx).unwrap();
			let (carry_sibs, carry_poss) = merkle_path(carry_idx);
			let carry_q = (stranded / QUANTUM) as u32;
			println!("root with the carry leaf    : {}", hex::encode(fixed_root));
			println!(
				"the tree differs by exactly one leaf: {} -> {}",
				pre_fix_leaves,
				ZkTree::leaf_count()
			);
			assert_ne!(fixed_root, pre_fix_root, "the live chain never held this root");
			assert_eq!(ZkTree::leaf_count(), pre_fix_leaves + 1, "exactly one leaf was added");
			assert_eq!(carry_leaf.to, miner);
			assert_eq!(carry_leaf.amount, stranded, "and it commits the accumulated remainder");

			// ---- 5. Door 2: the remainder IS withdrawable under that root ----
			let door2 = build_batch(
				secret, 20, fixed_root, carry_sibs, carry_poss, carry_q, carry_q - 1, EXIT,
			)
			.expect("with the carry leaf present the exit is provable");

			let err = Wormhole::verify_private_batch(RawOrigin::None.into(), door2.bytes.clone())
				.expect_err("the chain must reject it");
			println!("door 2 against live chain   : {:?}", err.error);
			assert_eq!(err.error, DispatchError::from(Error::<Test>::InvalidPublicInputs));

			// ---- 6. The only difference is the missing carry ----
			frame_system::BlockHash::<Test>::insert(1u64, H256::from(door2.block_hash));
			let before2 = Balances::balance(&exit_acct);
			assert_ok!(Wormhole::verify_private_batch(RawOrigin::None.into(), door2.bytes));
			println!(
				"same proof, carry leaf present: paid {} planck",
				Balances::balance(&exit_acct) - before2
			);
		});
	}
}
```

### Actual output

```
miner leaves over 20 blocks : 20
stranded across them        : 48981930013 planck
leaf 0: minted 692449530321 planck, commits 69 quanta, remainder 2449530321 planck
live chain root             : 4fbfa7957af137d4d3646beebfb1ac64917abe1a87e111ca883e3d8b332d9dcd
door 1 (live root)          : Some("UNPROVABLE")
control withdrew            : 680000000000 planck of 692449484656 minted
root with the carry leaf    : 6838ff650df873981d7194c330999d553ba00036743c20a57c5c5c68fdf57391
the tree differs by exactly one leaf: 41 -> 42
door 2 against live chain   : Module(ModuleError { index: 3, error: [0, 0, 0, 0], message: Some("InvalidPublicInputs") })
same proof, carry leaf present: paid 30000000000 planck
```

Read in order:

{% stepper %}
{% step %}

## Confirm the payee

The payee is asserted against the same `extract_author_from_digest` the emission path calls, so it is the keyless miner address.
{% endstep %}

{% step %}

## Produce real emission leaves

Twenty blocks of real `on_finalize` produce twenty separate leaves. Nothing merges them, and 48,981,930,013 planck is stranded across them.
{% endstep %}

{% step %}

## Door 1

Claiming one quantum more against the chain's own root is unprovable. The leaf commits 69 quanta and no more.
{% endstep %}

{% step %}

## The ceiling

A legitimate exit withdraws 680,000,000,000 planck out of 692,449,484,656 planck credited.
{% endstep %}

{% step %}

## A tree differing by exactly the missing leaf

Recording the accumulated remainder, which is what the recommended carry does, adds one leaf committing exactly those 48,981,930,013 planck. The tree goes from 41 leaves to 42.
{% endstep %}

{% step %}

## Door 2

Under that root the remainder is withdrawable. Submitted against the live chain it reverts with `InvalidPublicInputs`.
{% endstep %}

{% step %}

## Withdraw with the carry leaf present

With the carry leaf present the same proof pays 30,000,000,000 planck.
{% endstep %}
{% endstepper %}

Step 5 is what makes the rejection specific. The remainders are not lost to an accounting quirk. They are real value that becomes withdrawable the moment a leaf commits them, and the fix that creates that leaf is the one in the recommendation above.
