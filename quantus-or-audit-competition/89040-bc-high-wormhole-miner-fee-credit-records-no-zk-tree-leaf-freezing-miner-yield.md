# 89040 bc high wormhole miner fee credit records no zk tree leaf freezing miner yield

**Submitted on Aug 16th 2026 at 17:37:40 UTC by @AlexSR for** [**Audit Comp | Quantus**](https://immunefi.com/audit-competition/audit-comp-quantus)

* **Report ID:** #89040
* **Report Type:** Blockchain/DLT
* **Report severity:** High
* **Target:** <https://github.com/immunefi-team/audit-comp-quantus-chain>
* **Impacts:**
  * Permanent freezing of funds

## Description

## Summary

On this chain a native balance at a Poseidon-derived wormhole address is spendable only through a zk-tree leaf. `process_exit_bundle` follows that rule for two credits and breaks it for a third. The exit-principal credit records a leaf (`pallets/wormhole/src/lib.rs:854`) and the block-reward credit records a leaf (`pallets/mining-rewards/src/lib.rs:232`), but the volume-fee credit to the block author increases the author's balance and stops there (`pallets/wormhole/src/lib.rs:955-964`). The credited fee gets no leaf, so no withdrawal proof can be built for it, and the address is keyless, so no ordinary signed transfer can move it either. The miner cannot recover the fee by any permissionless action.

This happens on ordinary operation with no attacker. Every fee-bearing wormhole exit produces a nonzero miner fee under the shipped config (`VolumeFeeRateBps = 4`, `VolumeFeesBurnRate = 50%`, `runtime/src/configs/mod.rs:734-736`), and each such exit strands the miner's share once the fee is credited to the block author (the code otherwise burns it, `lib.rs:966-982`). The miner's share is quantized to whole `0.01 DEV` units by the leaf format (`DEV` is the token symbol, `node/src/chain_spec.rs:15`; source comments call it `QUAN`), so the stranded, recoverable-if-fixed value is `floor(miner_fee / 1e10)` units. That is zero only for the smallest fee tier and nonzero for every exit of `25 DEV` or more, which is where this report is scoped.

Verified on the frozen in-scope tree (commit `3b243b87`, `pallets/wormhole/src/lib.rs` is fork-authored and compiles into the runtime Wasm unconditionally). A drop-in unit test drives the pallet's real `process_exit_bundle` and asserts the miner is credited while the zk-tree leaf count does not increase for the fee. Root cause, reachability, the duplicate check, the fix and references are in the Validation Steps field.

## Root cause

The volume-fee success arm credits the block author but omits the `record_transfer` call that the two other keyless-address credits (the exit principal and the block reward) both make.

The exit path sets the invariant in the same function: each exit account is credited event-free with `increase_balance` and then gets a leaf via `record_transfer` (`lib.rs:854`). The `NOTE` above (`lib.rs:822-827`) says the event-free credit is deliberate precisely because the path records its proof "via `record_transfer` below."

`pallet-mining-rewards` credits the identical wormhole address (same `extract_author_from_digest`, `mining-rewards/src/lib.rs:207`) and records a leaf via `record_transfer_proof` after minting (`:230-232`).

The volume-fee arm does neither follow-up. It credits the author with `increase_balance` (`lib.rs:955-959`), emits `MinerVolumeFeePaid` (`:961-964`), and returns. No `record_transfer`, no `record_transfer_proof`. The credit lands as a bare balance with no note behind it.

This is the residue of a prior fix. The March-2026 audit finding EQ-QNT-WORMHOLE-F-03 corrected how the author is derived at this site (now `extract_author_from_digest` at `:946`); it never added a leaf, and the missing-leaf defect it left behind is what this report describes. See the "Not a duplicate" section in Validation Steps.

## Why a missing leaf is a permanent freeze

The credit lands on a keyless address whose only spend instrument is a zk-tree leaf, and no permissionless path can create the missing leaf.

The recipient is keyless. A signable account id on this chain is `poseidon(dilithium_public_key)` (`primitives/dilithium-crypto/src/scheme_macro.rs:244`). The block-author address is `poseidon(poseidon(preimage))`, derived from the PoW pre-runtime digest (`primitives/wormhole/src/lib.rs:101,109,146-148`). No Dilithium keypair can be found for that address (Poseidon preimage resistance), so no ordinary signed `Balances` transfer can move the balance. The runtime states the same invariant: value credited to a keyless beneficiary with no matching leaf is "unexitable" (`runtime/src/configs/mod.rs:583-586`).

Nothing can supply the leaf after the fact. `pallet-zk-tree` exposes no extrinsic (no `#[pallet::call]` in `pallets/zk-tree/src/lib.rs`); `insert_leaf` is an internal function reachable only through `record_transfer`/`record_transfer_proof`. The runtime's event scanner records a leaf only for four Balances events, `Transfer`, `Minted`, `TransferOnHold` and `ReserveRepatriated` (`runtime/src/transaction_extensions.rs:339-377`), and `increase_balance` emits none of them (the reason the exit path uses it, `lib.rs:822-827`, pinned by the test `exit_credits_emit_no_scannable_transfer_events`). Nothing consumes the `MinerVolumeFeePaid` event. The miner has no path to recover it; only privileged Root governance can, by moving the balance with a scheduler-enacted `force_transfer` (Root-only, `runtime/src/transaction_extensions.rs:291-292`) or a runtime migration that adds the leaf.

## Trigger and preconditions

No attacker and no privilege. The victim is the block author. The trigger is any honest, permissionless wormhole exit whose minted amount is at least `25 DEV`, submitted through the `ensure_none` extrinsics `verify_private_batch`/`verify_public_batch` (`lib.rs:589,626`). Under the shipped config every fee-bearing exit produces `miner_fee = k * 5e9 > 0`; the minimum `5e9` planck exceeds the existential deposit, so the credit lands rather than routing to the burn fallback; and `k >= 2`, a nonzero recoverable amount, holds for any exit of `25 DEV` or more. The precondition derivation is in the Validation Steps field.

Sequence:

{% stepper %}
{% step %}

## Submit a wormhole exit

A user submits a wormhole exit of `25 DEV` or more. `process_exit_bundle` mints the exit principal and records its leaf (`lib.rs:854`).
{% endstep %}

{% step %}

## Credit the miner fee

The same call computes `miner_fee = k * 5e9` planck, finds the block author from the digest, and credits it with `increase_balance` (`lib.rs:955-964`). No leaf is recorded.
{% endstep %}

{% step %}

## Freeze the credited balance

The miner now holds `k * 0.005 DEV` of balance at a keyless address with no note. `floor(k/2)` quanta of it were recoverable had the leaf been written; they are now frozen with no path to a withdrawal proof.
{% endstep %}
{% endstepper %}

## Impact

Permanent freezing of the miner's volume-fee yield. Per qualifying exit the frozen, recoverable-if-fixed amount is `floor(k/2)` quanta (`>= 0.01 DEV` for any exit of `25 DEV` or more): the miner's half of the volume fee, rounded down to whole quanta. The loss accrues on every fee-bearing exit above the threshold for as long as the code is unpatched, so the aggregate grows without bound with exit volume even though each instance is small.

The direction is a freeze, not a transfer. The fee is credited to the miner's own address and is not burned and not redirected; it cannot be spent. There is no state corruption, no user-principal loss (the exit principal is delivered correctly with its leaf), and no inflation: the miner credit uses `increase_balance`, which raises the balance without raising total issuance (`lib.rs:884-885`).

## Severity

> Permanent freezing of unclaimed yield

This report files at **High** on that rung: the chain credits and announces miner yield but omits the only bearer leaf with which the miner can claim it. That share is yield earned for producing the block, and it is credited but never spendable, so it is permanently frozen unclaimed yield.

## Proof of Concept

`miner_fee_missing_leaf_poc.rs` (attached) holds two drop-in unit tests. Append them to `pallets/wormhole/src/tests.rs`; they reuse the mock runtime, the direct-`process_exit_bundle` pattern of the existing `exit_bundle_tests` module, and the `set_miner_preimage_digest` helper.

* `poc_miner_fee_credited_without_leaf_is_frozen` builds a `k >= 2` exit (minted 25 DEV), seeds a block-author digest, calls `process_exit_bundle`, then asserts the miner is credited (`MinerVolumeFeePaid`, balance up by `miner_fee`) while the zk-tree `LeafCount` rises by exactly one, the exit principal's leaf. A second leaf would exist if the fee had recorded one; it does not.
* `poc_k1_tier_has_zero_recoverable_value` shows the sub-quantum tier this report does not file: a 20 DEV exit yields `miner_fee = 5e9` planck, and `miner_fee / AMOUNT_SCALE_DOWN_FACTOR = 0`, so even the fixed path records a zero-value leaf.

```bash
git clone https://github.com/immunefi-team/audit-comp-quantus-chain && cd audit-comp-quantus-chain
git checkout 3b243b870a5442a6e5f443056f67bbba87d472f5   # frozen audited commit
cat /path/to/miner_fee_missing_leaf_poc.rs >> pallets/wormhole/src/tests.rs
cargo test -p pallet-wormhole --lib poc_miner_fee_missing_leaf -- --nocapture
```

Commands, expected results, and the build environment are in `RUN.md`; the verbatim run output is in `POC_EXECUTED_GREEN.md`. These are single in-process unit tests; no node or network is involved.

### What the PoC proves vs. stubs

| Claim                                                                                  | Status                                                                                                                                                   |
| -------------------------------------------------------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `process_exit_bundle` credits the miner (`MinerVolumeFeePaid`, balance up)             | Executed green (test 1)                                                                                                                                  |
| The miner-fee credit records no leaf (`LeafCount` delta == 1, the exit principal only) | Executed green (test 1)                                                                                                                                  |
| `k >= 2` recoverable-if-fixed value is nonzero; `k = 1` is zero                        | Executed green (test 2); Argued from source (`tree.rs:146-148`)                                                                                          |
| Recipient is keyless: `poseidon(poseidon(preimage))` vs signable `poseidon(pubkey)`    | Argued from source (`scheme_macro.rs:244`; `primitives/wormhole/src/lib.rs:101,109`)                                                                     |
| No path can supply the missing leaf                                                    | Argued from source (no `#[pallet::call]` in `pallets/zk-tree/src/lib.rs`; scanner `transaction_extensions.rs:339-377`; no `MinerVolumeFeePaid` consumer) |
| Shipped config makes `miner_fee > 0` and above the ED on every fee-bearing exit        | Argued from source (`configs/mod.rs:734-736`; `runtime/src/lib.rs:98,102`)                                                                               |
| The fix compiles and the patch applies                                                 | Authored; not part of the executed run (`add_miner_fee_leaf.patch`)                                                                                      |

```rust
// Proof of concept: the wormhole volume-fee credit to the block author records
// no zk-tree leaf, so the credited fee is unspendable (permanently frozen at a
// keyless address).
//
// This is a drop-in unit test for the wormhole pallet. Append it to
// `pallets/wormhole/src/tests.rs`. It reuses that suite's mock runtime, its
// direct-`process_exit_bundle` pattern (see the `exit_bundle_tests` module) and
// its `set_miner_preimage_digest` helper (see the existing miner-fee test).
//
// It drives the real `process_exit_bundle` with a hand-built bundle and a
// seeded block-author digest, then asserts:
//   1. the miner is credited the volume fee (MinerVolumeFeePaid, balance up), and
//   2. the zk-tree leaf count increases by exactly one (the exit principal's
//      leaf), so the miner fee added no leaf and has no withdrawal note.
//
// No node, network, or live state is involved; it is a single in-process test.

#[cfg(test)]
mod poc_miner_fee_missing_leaf {
	use crate::mock::*;
	use crate::pallet::{ExitBundle, ExitSegment};
	use frame_support::assert_ok;
	use frame_support::traits::fungible::Inspect;
	use qp_wormhole_verifier::{BlockData, BytesDigest, PublicInputsByAccount};
	use sp_core::crypto::AccountId32;

	fn digest(byte: u8) -> BytesDigest {
		BytesDigest::new_unchecked([byte; 32])
	}

	// One segment: a single real exit account plus two nullifiers.
	fn bundle_with(exit_byte: u8, quantized_amount: u32) -> ExitBundle {
		ExitBundle {
			asset_id: 0,
			volume_fee_bps: VolumeFeeRateBps::get(),
			block_data: BlockData::default(),
			aggregator_address: None,
			segments: vec![ExitSegment {
				account_data: vec![PublicInputsByAccount {
					summed_output_amount: quantized_amount,
					exit_account: digest(exit_byte),
				}],
				nullifiers: vec![digest(1), digest(2)],
			}],
		}
	}

	/// k >= 2 tier (the tier this report files). Minted exit = 25 DEV
	/// (quantized 2500). fee = 4 bps, burn = 50%, so miner_fee = 1e10 planck =
	/// one quantum. The miner is credited, but no zk-tree leaf is recorded for
	/// the fee, so the balance is frozen at the keyless block-author address.
	#[test]
	fn poc_miner_fee_credited_without_leaf_is_frozen() {
		new_test_ext().execute_with(|| {
			System::set_block_number(1);

			// Seed the block author via the PoW pre-runtime digest, exactly as
			// QPoW does. The derived address is poseidon(poseidon(preimage)) -
			// a keyless wormhole address, not poseidon(dilithium_pubkey).
			let miner_preimage = [7u8; 32];
			set_miner_preimage_digest(miner_preimage);
			let author = AccountId32::from(
				qp_wormhole::derive_wormhole_address(miner_preimage)
					.expect("test preimage limbs are canonical"),
			);

			// Expected miner fee under the shipped config.
			let fee_bps = VolumeFeeRateBps::get() as u128;
			let total_fee = super::ceil_volume_fee(2500u128, fee_bps);
			let miner_fee = total_fee - total_fee / 2;
			assert!(miner_fee > 0, "config must produce a nonzero miner fee");

			// k >= 2: the recoverable-if-fixed leaf value is nonzero. The leaf
			// format floors amount / AMOUNT_SCALE_DOWN_FACTOR to a u32.
			let recoverable_quanta =
				miner_fee / pallet_zk_tree::tree::AMOUNT_SCALE_DOWN_FACTOR;
			assert!(
				recoverable_quanta >= 1,
				"this report is scoped to the k>=2 tier (nonzero recoverable value)"
			);

			let leaves_before = pallet_zk_tree::LeafCount::<Test>::get();
			let author_balance_before = Balances::balance(&author);

			assert_ok!(Wormhole::process_exit_bundle(bundle_with(10, 2500)));

			// (1) The miner was credited the fee as a bare balance.
			System::assert_has_event(
				crate::Event::<Test>::MinerVolumeFeePaid {
					miner: author.clone(),
					amount: miner_fee,
				}
				.into(),
			);
			assert_eq!(
				Balances::balance(&author),
				author_balance_before + miner_fee,
				"miner fee must be credited",
			);

			// (2) Exactly one leaf was created: the exit principal's. If the
			// miner-fee credit had recorded a leaf, the delta would be two.
			let leaves_after = pallet_zk_tree::LeafCount::<Test>::get();
			assert_eq!(
				leaves_after - leaves_before,
				1,
				"one exit account => one leaf; the miner fee recorded no leaf",
			);
		});
	}

	/// k = 1 tier (the sub-quantum tier this report does NOT file). Minted exit
	/// = 20 DEV (quantized 2000) => miner_fee = 5e9 planck, below one quantum,
	/// so even the fixed path would record a zero-value leaf and nothing is
	/// recoverable. This case is conceded to the programme's known issue on
	/// rewards below the quantized minimum, and is separated out here.
	#[test]
	fn poc_k1_tier_has_zero_recoverable_value() {
		new_test_ext().execute_with(|| {
			let fee_bps = VolumeFeeRateBps::get() as u128;
			let total_fee = super::ceil_volume_fee(2000u128, fee_bps);
			let miner_fee = total_fee - total_fee / 2;
			let recoverable_quanta =
				miner_fee / pallet_zk_tree::tree::AMOUNT_SCALE_DOWN_FACTOR;
			assert_eq!(
				recoverable_quanta, 0,
				"k=1 fee is sub-quantum; recoverable value is zero even with the fix",
			);
		});
	}
}
```
