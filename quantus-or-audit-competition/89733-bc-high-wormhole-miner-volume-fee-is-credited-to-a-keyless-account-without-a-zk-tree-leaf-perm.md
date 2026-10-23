# 89733 bc high wormhole miner volume fee is credited to a keyless account without a zk tree leaf permanently freezing it

**Submitted on Aug 20th 2026 at 20:31:52 UTC by @santin0xyz for** [**Audit Comp | Quantus**](https://immunefi.com/audit-competition/audit-comp-quantus)

* **Report ID:** #89733
* **Report Type:** Blockchain/DLT
* **Report severity:** High
* **Target:** <https://github.com/immunefi-team/audit-comp-quantus-chain>
* **Impacts:**
  * Permanent freezing of unclaimed yield

## Description

### Summary

Every wormhole exit settles a volume fee. Half is burned; the other half is credited to the QPoW block author (`MinerVolumeFeePaid`). The author's account is, by construction, a keyless Poseidon output, so its only spending path is a wormhole ZK exit — which spends a **specific ZK-tree leaf**. `process_exit_bundle` credits the fee with `Unbalanced::increase_balance` and never records that leaf, so the value lands in a balance that can never be moved again through any non-privileged path.

`pallet-mining-rewards` pays the *same account* in the *same block* and does record its leaf. The two pallets disagree on whether a credit to a keyless account needs one.

### Why the author account cannot be signable

`qp_wormhole::extract_author_from_digest` (`primitives/wormhole/src/lib.rs:122`) takes the miner's 32-byte pre-runtime preimage and always returns `derive_wormhole_address(preimage)` = `rehash_to_bytes(...)`, i.e. `Poseidon` over 4 felts. There is no code path that pays a miner-chosen raw address.

A signable account on this chain is `hash_bytes(dilithium_pubkey)` — Poseidon over the felt encoding of a 1952/2592-byte ML-DSA key (`primitives/dilithium-crypto/src/scheme_macro.rs:239-244`). Landing a Poseidon-of-4-felts output on a signable account would require a Poseidon preimage attack, so the payout address is keyless by construction.

For such an account the only spend is the ZK exit, and the circuit spends one concrete leaf `(to, transfer_count, asset_id, amount)` under `(out_1 + out_2) · 10000 ≤ input_amount · (10000 − fee_bps)`, where `input_amount` is bound into the leaf hash (`qp-wormhole-circuit`, `ZkLeafTargets::collect_for_hash`) and the leaf must reproduce `header.zk_tree_root`. The account's free balance never enters the circuit. **A credit with no leaf is not ZK-spendable** — the runtime states this itself when documenting the known Root `force_transfer` gap in `runtime/src/transaction_extensions.rs`.

### The asymmetry

`pallets/mining-rewards/src/lib.rs:229` — correct:

```rust
match T::Currency::mint_into(miner, reward) {
    Ok(_) => {
        T::ProofRecorder::record_transfer_proof(None, mint_account, miner.clone(), reward);
        Self::deposit_event(Event::MinerRewarded { miner: miner.clone(), reward });
```

`pallets/wormhole/src/lib.rs:955` — the bug:

```rust
match <T::Currency as Unbalanced<_>>::increase_balance(&author, miner_fee, Precision::Exact) {
    Ok(_) => {
        Self::deposit_event(Event::MinerVolumeFeePaid { miner: author, amount: miner_fee });
    },
```

The comment at `pallets/wormhole/src/lib.rs:822` justifies the event-free primitive for the **user exit**, which records its own leaf 26 lines later (`:854`). The same primitive was carried to the miner-fee credit, which has no leaf of its own.

No fallback covers it: this arm never calls `record_transfer`, and `increase_balance` emits no `Minted`/`Transfer` event, so even if a submission went through `WormholeProofRecorderExtension` the scanner would not pick the credit up. The intended admission path is a bare unsigned extrinsic (`ValidateUnsigned`), for which that extension's event scan does not run.

## Proof of Concept

Three tests in a single self-contained file, `poc.rs` (attached). It borrows nothing from the existing test module, so installing it takes two steps and no patch:

{% stepper %}
{% step %}

## Save the attached file

Save the attached file as `pallets/wormhole/src/poc.rs`.
{% endstep %}

{% step %}

## Add the test module

Add two lines to `pallets/wormhole/src/lib.rs`, next to the existing `mod tests;` (line 164):

```rust
#[cfg(test)]
mod poc;
```

`#[cfg(test)]` must sit on `mod poc` itself — it does not inherit from `mod tests`. With that attribute the module compiles out of every non-test build, so the behaviour under test is the unmodified frozen commit. `git diff` after installing shows exactly `1 file changed, 2 insertions(+)`.
{% endstep %}
{% endstepper %}

```bash
cargo test -p pallet-wormhole --lib poc -- --nocapture --test-threads=1
```

```
PoC 1: exit 2500 quanta, fee 2 quanta, author credited 10000000000 planck (one whole quantum) with 0 leaves
PoC 2: author balance 3005000000000 planck, leaf claim 3000000000000 planck, unprovable surplus 5000000000 planck
PoC 3: a 1-quantum fee splits into two half-quanta; hash_leaf commits 0 each

test result: ok. 3 passed; 0 failed; 0 ignored
```

Full pallet suite with the module installed: `71 passed; 0 failed; 3 ignored`.

The evidence in all three is `TransferCount`: `record_transfer` bumps it on every leaf insert and is its only writer, so a count that does not move proves no leaf was created for the credit.

**What each test proves.** `poc1_whole_quantum_fee_is_credited_without_a_leaf` is the one that isolates the cause. `hash_leaf` commits `amount / AMOUNT_SCALE_DOWN_FACTOR`, so a sub-quantum credit would be unrepresentable in a leaf even if one were recorded; this test picks an exit volume whose author share is exactly one whole quantum, which rules quantization out. The credit is representable, a leaf for it would commit amount 1, and it is still frozen.

`poc2_real_proof_fee_survives_an_existing_reward_leaf` runs the same on the checked-in proof, starting from the mainnet precondition that the author already holds a block-reward leaf, and shows that leaf does not cover the fee; it also carries the control that every exit the bundle minted did receive its own leaf.

`poc3_odd_fee_splits_below_leaf_resolution` is why fix step 1 exists.

Neither of the first two attempts an exit and observes it fail, because it is not needed: there is no circuit path of the form "I know the secret, therefore I may take this account's free balance". The absence of a leaf covering the credit is the whole proof.

## Impact

Every wormhole exit permanently strands the block author's entire share of the volume fee — with the mainnet configuration (`runtime/src/configs/mod.rs:734`: `VolumeFeeRateBps = 4`, `VolumeFeesBurnRate = 50%`), **50% of every fee collected, on every settlement, forever**. The settled fee is `ceil(exit_quanta · bps / (10000 − bps))` quanta, so the effective rate is 10 000 bps on a 1-quantum exit, 400 bps at 25 quanta, and only approaches the nominal 4 bps at volume (4.01 bps at 10^6 quanta). Half of whatever is collected is stranded.

This is not depositor principal — the user already received `output` and the fee was deducted in-circuit. It is the author's compensation, i.e. **unclaimed yield**: credited, announced by an event, inert forever.

Second-order, not the primary claim: settlement extrinsics are `Pays::No` and impose a full plonky2 verification on the author in `ValidateUnsigned::pre_dispatch`. The volume fee is the only compensation, so making it unspendable removes the reason to include wormhole settlements at all. That does not stop the network; it can stop the wormhole.

## Severity rationale

High — *Permanent freezing of unclaimed yield*. It is protocol yield, not user principal, and `Balances::force_transfer` is Root, so governance could recover stranded fees account-by-account: impracticable at scale, but it exists, so the Critical qualifier *"fix requires hardfork"* does not apply.

## Fix

Two changes; the first alone is not sufficient.

{% stepper %}
{% step %}

## Keep the author's share on a quantum boundary

At `:890`, floor the share and hold the remainder aside:

```rust
// pallets/wormhole/src/lib.rs:890 — replaces the existing `let miner_fee_u128 = ...`
let mut miner_fee_u128 = total_fee_u128.saturating_sub(burn_amount_u128);
// Keep the author's share representable in a ZK leaf (`hash_leaf` commits
// `amount / AMOUNT_SCALE_DOWN_FACTOR`).
let remainder = miner_fee_u128 % crate::SCALE_DOWN_FACTOR;
miner_fee_u128 -= remainder;
```

At `:937`, *after* the aggregator-rebate block at `:894` has taken its share, fold the remainder into the burn:

```rust
// pallets/wormhole/src/lib.rs:937 — immediately BEFORE `let miner_fee: BalanceOf<T> = ...`
burn_amount_u128 = burn_amount_u128.saturating_add(remainder);
```

Do not merge the blocks, and do not recompute `burn_amount_u128` as `total_fee_u128 - miner_fee_u128`. The rebate is `VolumeFeesAggregatorRate × burn_amount_u128` and subtracts itself from that bucket at `:919`, so folding the remainder in beforehand hands the aggregator a share of it (measured: **5e9 instead of 2.5e9 planck**, breaking `process_exit_bundle_fee_and_rebate_exclude_denied_segment` and `test_verify_public_batch_succeeds_and_pays_aggregator`), and recomputing afterwards burns the rebate twice. The existing `if !miner_fee.is_zero()` guard covers the floor case, where the share rounds to zero and the whole fee is burned.

Small exits then burn the whole fee instead of paying something unspendable — better than the status quo, but a policy change. To keep the revenue, accrue the share in storage and settle it once it reaches a quantum.
{% endstep %}

{% step %}

## Record the leaf

Keep `increase_balance`: the backing value already exists in the locked deposit, so `mint_into` would inflate `TotalIssuance` on top of value already counted.

```rust
Ok(_) => {
    // The block author is always `Poseidon(pre-runtime preimage)` — a keyless account
    // whose only spending path is a wormhole exit, which needs a leaf.
    let from_account: <T as Config>::WormholeAccountId = mint_account.clone().into();
    let to_account: <T as Config>::WormholeAccountId = author.clone().into();
    Self::record_transfer(T::AssetId::default(), &from_account, &to_account, miner_fee);

    Self::deposit_event(Event::MinerVolumeFeePaid { miner: author, amount: miner_fee });
},
```

`mint_account` is already in scope, and this arm sits inside `if !miner_fee.is_zero()`, so change 1's zero case never reaches it. Follow-ups: re-benchmark both settlement calls for the extra `record_transfer` (one `TransferCount` read/write plus a flat-priced leaf insert per `pallet_zk_tree::INSERT_LEAF_DB_OPS`), and pin the leaf in `test_verify_private_batch_emits_miner_volume_fee_paid`, which today asserts only the balance delta — precisely why this survived review.
{% endstep %}
{% endstepper %}

## Proof of Concept

````rust
//! PoC: the wormhole volume fee paid to the block author is credited without a ZK-tree leaf,
//! which makes it permanently unspendable.
//!
//! Install:
//! 1. Save this file as `pallets/wormhole/src/poc.rs`.
//! 2. Add these two lines in `pallets/wormhole/src/lib.rs`, next to the existing `mod tests;`.
//!    `#[cfg(test)]` must sit on `mod poc` itself — it does not inherit from `mod tests`:
//!
//!    ```ignore
//!    #[cfg(test)]
//!    mod poc;
//!    ```
//!
//!    The module then compiles out of every non-test build. Run:
//!    `cargo test -p pallet-wormhole --lib poc -- --nocapture --test-threads=1`

#![allow(clippy::unwrap_used)]

use crate::{
	mock::*,
	pallet::{ExitBundle, ExitSegment},
};
use frame_support::{
	assert_ok,
	traits::fungible::{Inspect, Mutate},
};
use frame_system::RawOrigin;
use qp_wormhole_verifier::{
	parse_private_batch_public_inputs, BlockData, BytesDigest, ProofWithPublicInputs,
	PublicInputsByAccount, C, F,
};
use sp_core::{crypto::AccountId32, H256};

const D: usize = 2;
const PROOF_HEX: &str = include_str!("../test-data/private_batch.hex");

fn proof_bytes() -> Vec<u8> {
	hex::decode(PROOF_HEX.trim()).unwrap()
}

fn proof() -> ProofWithPublicInputs<F, C, D> {
	let verifier = crate::get_private_batch_verifier().unwrap();
	ProofWithPublicInputs::<F, C, D>::from_bytes(proof_bytes(), &verifier.circuit_data.common)
		.unwrap()
}

/// Re-derives the circuit's quantized-ceiling fee rule independently of the code under test.
fn ceil_fee(exit_quanta: u128, bps: u128) -> u128 {
	exit_quanta.saturating_mul(bps).div_ceil(10_000 - bps) * crate::SCALE_DOWN_FACTOR
}

fn author_for(preimage: [u8; 32]) -> AccountId {
	AccountId32::from(qp_wormhole::derive_wormhole_address(preimage).unwrap())
}

fn scaled(quanta: u32) -> u128 {
	(quanta as u128) * crate::SCALE_DOWN_FACTOR
}

fn leaves_of(who: &AccountId) -> u64 {
	crate::TransferCount::<Test>::get(who)
}

fn one_segment_bundle(nullifier: u8, exit_account: u8, quanta: u32) -> ExitBundle {
	ExitBundle {
		asset_id: 0,
		volume_fee_bps: VolumeFeeRateBps::get(),
		block_data: BlockData::default(),
		aggregator_address: None,
		segments: vec![ExitSegment {
			account_data: vec![PublicInputsByAccount {
				summed_output_amount: quanta,
				exit_account: BytesDigest::new_unchecked([exit_account; 32]),
			}],
			nullifiers: vec![BytesDigest::new_unchecked([nullifier; 32])],
		}],
	}
}

/// Makes the fixture's block hash resolvable on chain and moves past it.
fn seed_fixture_block(number: u32, hash: BytesDigest) {
	let bytes: [u8; 32] = hash.as_ref().try_into().unwrap();
	frame_system::BlockHash::<Test>::insert(number as u64, H256::from(bytes));
	System::set_block_number(number as u64 + 10);
}

/// Primary PoC. `hash_leaf` commits `amount / AMOUNT_SCALE_DOWN_FACTOR`, so a sub-quantum credit
/// would be unrepresentable in a leaf even if one were recorded. This picks an exit volume whose
/// author share is exactly one whole quantum, which rules quantization out as an explanation:
/// the credit is representable, a leaf for it would commit amount 1, and it is still frozen.
#[test]
fn poc1_whole_quantum_fee_is_credited_without_a_leaf() {
	new_test_ext().execute_with(|| {
		System::set_block_number(1);

		// ceil(2500 * 4 / 9996) = 2 quanta of fee, split into 1 burned and 1 to the author.
		const EXIT_QUANTA: u32 = 2500;

		let preimage = [9u8; 32];
		set_miner_preimage_digest(preimage);
		let author = author_for(preimage);

		let total_fee = Wormhole::volume_fee_for_exit(scaled(EXIT_QUANTA), VolumeFeeRateBps::get());
		let burn = VolumeFeesBurnRate::get() * total_fee;
		let fee = total_fee - burn;
		assert_eq!(total_fee, 2 * crate::SCALE_DOWN_FACTOR);
		assert_eq!(fee, crate::SCALE_DOWN_FACTOR, "author share is one whole quantum");
		assert_eq!(fee / crate::SCALE_DOWN_FACTOR, 1, "a leaf here would commit amount 1, not 0");

		// Endowed so the burn assertion is a real subtraction rather than a saturation at zero.
		assert_ok!(<Balances as Mutate<AccountId>>::mint_into(
			&AccountId32::new([99u8; 32]),
			1_000 * UNIT
		));
		let issuance_before = <Balances as Inspect<AccountId>>::total_issuance();
		let tree_before = pallet_zk_tree::LeafCount::<Test>::get();

		assert_ok!(Wormhole::process_exit_bundle(one_segment_bundle(1, 10, EXIT_QUANTA)));

		assert_eq!(Balances::free_balance(author.clone()), fee, "the fee was credited");
		assert_eq!(leaves_of(&author), 0, "BUG: whole-quantum credit, keyless account, no leaf");

		// The exit account of the same bundle did get its leaf, and the tree grew by exactly
		// that one, so the omission is specific to the author-fee path.
		assert_eq!(leaves_of(&AccountId32::new([10u8; 32])), 1);
		assert_eq!(pallet_zk_tree::LeafCount::<Test>::get() - tree_before, 1);

		// The burn half really was destroyed, so the author's half is not uncollected revenue:
		// the user paid it and nobody can spend it.
		assert_eq!(<Balances as Inspect<AccountId>>::total_issuance(), issuance_before - burn);

		println!(
			"PoC 1: exit {EXIT_QUANTA} quanta, fee {} quanta, author credited {fee} planck \
			 (one whole quantum) with 0 leaves",
			total_fee / crate::SCALE_DOWN_FACTOR
		);
	});
}

/// The same on the repository's real proof, starting from the mainnet precondition: the author
/// already holds a block-reward leaf, since `mint_reward` inserts one every block for that same
/// account. It does not help, because each leaf commits its own `amount` and the circuit binds
/// `input_amount` to the leaf hash, so a reward leaf authorises spending the reward and no more.
#[test]
fn poc2_real_proof_fee_survives_an_existing_reward_leaf() {
	new_test_ext().execute_with(|| {
		let inputs = parse_private_batch_public_inputs(&proof()).unwrap();
		seed_fixture_block(inputs.block_data.block_number, inputs.block_data.block_hash);

		let preimage = [7u8; 32];
		set_miner_preimage_digest(preimage);
		let author = author_for(preimage);

		// Exactly what `pallet_mining_rewards::mint_reward` does every block.
		let reward: Balance = 3 * UNIT;
		assert_ok!(<Balances as Mutate<AccountId>>::mint_into(&author, reward));
		Wormhole::record_transfer(0u32, &MintingAccount::get(), &author, reward);

		let reward_leaf_idx = pallet_zk_tree::LeafCount::<Test>::get() - 1;
		assert_eq!(pallet_zk_tree::Leaves::<Test>::get(reward_leaf_idx).unwrap().to, author);
		assert_eq!(leaves_of(&author), 1);
		let tree_before = pallet_zk_tree::LeafCount::<Test>::get();

		let exit_quanta: u128 = inputs
			.account_data
			.iter()
			.filter(|a| a.summed_output_amount > 0)
			.map(|a| a.summed_output_amount as u128)
			.sum();
		let total_fee = ceil_fee(exit_quanta, VolumeFeeRateBps::get() as u128);
		let fee = total_fee - total_fee / 2;
		assert!(fee > 0);

		assert_ok!(Wormhole::verify_private_batch(RawOrigin::None.into(), proof_bytes()));

		assert_eq!(Balances::free_balance(author.clone()), reward + fee, "reward plus fee");
		assert_eq!(leaves_of(&author), 1, "BUG: credited, but no leaf was added");

		// The one leaf still commits only the reward, so the most this account can ever prove
		// is `reward`; the surplus contributed by the fee is out of reach.
		let leaf = pallet_zk_tree::Leaves::<Test>::get(reward_leaf_idx).unwrap();
		assert_eq!(leaf.amount, reward, "the only leaf commits the reward, not reward + fee");
		assert!(Balances::free_balance(author.clone()) > leaf.amount);
		assert_eq!(
			pallet_zk_tree::Leaves::<Test>::iter().filter(|(_, l)| l.to == author).count(),
			1,
			"no other leaf anywhere names the author"
		);

		// Control: every exit the bundle minted did receive a leaf, and the tree grew by that
		// count alone.
		let mut minted = 0u64;
		for slot in inputs.account_data.iter() {
			let bytes: [u8; 32] = (*slot.exit_account).as_ref().try_into().unwrap();
			if bytes == [0u8; 32] || slot.summed_output_amount == 0 {
				continue;
			}
			minted += 1;
			assert!(leaves_of(&AccountId32::from(bytes)) > 0, "every minted exit got a leaf");
		}
		assert!(minted > 0);
		assert_eq!(pallet_zk_tree::LeafCount::<Test>::get() - tree_before, minted);

		println!(
			"PoC 2: author balance {} planck, leaf claim {} planck, unprovable surplus {} planck",
			Balances::free_balance(author.clone()),
			leaf.amount,
			Balances::free_balance(author) - leaf.amount
		);
	});
}

/// Companion defect, and the reason recording a leaf is necessary but not sufficient.
/// `volume_fee_for_exit` returns whole quanta, but `VolumeFeesBurnRate` then halves it. On an
/// odd number of quanta the author's share is `x.5` and `hash_leaf`'s floor commits `x`; on the
/// one-quantum fee that every small exit hits, it commits zero.
#[test]
fn poc3_odd_fee_splits_below_leaf_resolution() {
	new_test_ext().execute_with(|| {
		for exit_quanta in [1u32, 25, 2499] {
			let total_fee =
				Wormhole::volume_fee_for_exit(scaled(exit_quanta), VolumeFeeRateBps::get());
			let fee = total_fee - VolumeFeesBurnRate::get() * total_fee;

			assert_eq!(total_fee, crate::SCALE_DOWN_FACTOR, "one whole quantum of fee");
			assert_eq!(fee, crate::SCALE_DOWN_FACTOR / 2, "half a quantum to the author");
			assert_eq!(fee / crate::SCALE_DOWN_FACTOR, 0, "a leaf here would commit amount 0");
		}
		println!("PoC 3: a 1-quantum fee splits into two half-quanta; hash_leaf commits 0 each");
	});
}
````
