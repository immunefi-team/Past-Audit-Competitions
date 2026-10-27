# 88912 bc high missing record transfer in wormhole miner fee settlement permanently freezes unclaimed yield

**Submitted on Aug 15th 2026 at 15:44:45 UTC by @IronsideSec for** [**Audit Comp | Quantus**](https://immunefi.com/audit-competition/audit-comp-quantus)

* **Report ID:** #88912
* **Report Type:** Blockchain/DLT
* **Report severity:** High
* **Target:** <https://github.com/immunefi-team/audit-comp-quantus-chain>
* **Impacts:**
  * Permanent freezing of unclaimed yield

## Description

## Brief/Intro

Every ordinary Wormhole exit batch can credit 50% of its volume fee to the block author's QPoW-derived Wormhole account, but `process_exit_bundle` uses event-free `Unbalanced::increase_balance` and never records the transfer in the ZK tree. The author's raw balance and `MinerVolumeFeePaid` event therefore increase while no leaf gives the keyless author ownership of the reward, permanently freezing 100% of that fee yield unless Root governance performs an exceptional recovery migration.

## Vulnerability Details

**Affected in-scope asset:** [The Substrate L1 node — runtime, FRAME pallets, QPoW consensus, quantus-node](https://github.com/immunefi-team/audit-comp-quantus-chain)

**Frozen commit:** `3b243b870a5442a6e5f443056f67bbba87d472f5`

### Root cause

The block author is intentionally a Wormhole account rather than an ordinary ML-DSA account. [`derive_wormhole_address`](https://github.com/immunefi-team/audit-comp-quantus-chain/blob/3b243b870a5442a6e5f443056f67bbba87d472f5/primitives/wormhole/src/lib.rs#L97-L110) derives the address as the second Poseidon hash of the 32-byte inner hash placed in the QPoW pre-runtime digest:

```rust
/// The inner_digest is the "first hash" from wormhole derivation: `hash(salt + secret)`.
/// The wormhole address is: `address = hash(hash(salt + secret))`.
pub fn derive_wormhole_address(inner_digest: [u8; 32]) -> Result<[u8; 32], &'static str> {
    rehash_to_bytes(&inner_digest)
}
```

[`extract_author_from_digest`](https://github.com/immunefi-team/audit-comp-quantus-chain/blob/3b243b870a5442a6e5f443056f67bbba87d472f5/primitives/wormhole/src/lib.rs#L113-L150) extracts that inner hash and returns the derived Wormhole account. The miner knows the Wormhole secret needed for a membership proof, but does not know an ML-DSA key whose public-key hash equals this independently derived account. [`Verify::verify`](https://github.com/immunefi-team/audit-comp-quantus-chain/blob/3b243b870a5442a6e5f443056f67bbba87d472f5/primitives/dilithium-crypto/src/traits.rs#L152-L199) accepts an ordinary signature only when the supplied ML-DSA public key hashes to the signed `AccountId`; finding such a key for the Wormhole-derived account would require an infeasible preimage search.

Consequently, a balance credit to this author is spendable only if the runtime also creates the ZK-tree leaf against which the miner can prove membership.

The same settlement function shows the correct invariant for user exits. After crediting each exit account with `increase_balance`, [`process_exit_bundle`](https://github.com/immunefi-team/audit-comp-quantus-chain/blob/3b243b870a5442a6e5f443056f67bbba87d472f5/pallets/wormhole/src/lib.rs#L816-L859) explicitly calls `record_transfer`:

```rust
match <T::Currency as Unbalanced<_>>::increase_balance(
    exit_account,
    *exit_balance,
    frame_support::traits::tokens::Precision::Exact,
) {
    Ok(_) => {},
    Err(e) => {
        // ... emit ExitMintFailed ...
        continue;
    }
}

let from_account: <T as Config>::WormholeAccountId = mint_account.clone().into();
let to_account: <T as Config>::WormholeAccountId = exit_account.clone().into();
Self::record_transfer(
    T::AssetId::default(),
    &from_account,
    &to_account,
    *exit_balance,
);
```

[`record_transfer`](https://github.com/immunefi-team/audit-comp-quantus-chain/blob/3b243b870a5442a6e5f443056f67bbba87d472f5/pallets/wormhole/src/lib.rs#L1273-L1319) assigns the recipient's next `TransferCount`, inserts the leaf into the ZK tree, and emits its leaf index. The transfer count, recipient, asset ID, and amount are part of the note that the proof consumes.

The miner-fee branch omits that second half. Both [`verify_private_batch` and `verify_public_batch`](https://github.com/immunefi-team/audit-comp-quantus-chain/blob/3b243b870a5442a6e5f443056f67bbba87d472f5/pallets/wormhole/src/lib.rs#L576-L645) reach `process_exit_bundle`. The runtime configures a four-basis-point volume fee and a 50/50 burn/miner split. After calculating the split, [`process_exit_bundle`](https://github.com/immunefi-team/audit-comp-quantus-chain/blob/3b243b870a5442a6e5f443056f67bbba87d472f5/pallets/wormhole/src/lib.rs#L937-L983) performs only the raw credit and success event:

```rust
if !miner_fee.is_zero() {
    let digest = frame_system::Pallet::<T>::digest();
    if let Some(author) = qp_wormhole::extract_author_from_digest::<
        <T as frame_system::Config>::AccountId,
        _,
    >(digest.logs.iter())
    {
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
            // ...
        }
    }
}
```

There is no `Self::record_transfer`, no `TransferProofRecorder::record_transfer_proof`, no `TransferCount` increment, and no leaf insertion. In this frozen fork, `Unbalanced::increase_balance` bypasses the ordinary balance mutation completion hook, so the runtime's event-scanning proof recorder also has no matched transfer event from which it could repair the omission.

Later base mining rewards do not absorb the missing fee. Mining Rewards records a separate leaf for the exact reward argument it pays; it does not commit the author's aggregate free balance. Each note has its own transfer count and nullifier, so a later note cannot retroactively authorize the earlier unrecorded credit.

### Why the finding remains independent of published issues

* **PR #619:** [PR #619](https://github.com/Quantus-Network/chain/pull/619) publicly observed `increase_balance` “without `record_transfer`,” but defined the problem as indexer drift and changed only event emission. This PoC successfully observes the added `MinerVolumeFeePaid` event and the raw balance, then demonstrates the distinct security failure left by that event-only fix: the keyless recipient still has no membership leaf and cannot claim the indexed balance. An indexer event is not a ZK ownership commitment.
* **Eiger F-03:** [Eiger `EQ-QNT-WORMHOLE-F-03`](https://github.com/immunefi-team/audit-comp-quantus-chain/blob/3b243b870a5442a6e5f443056f67bbba87d472f5/audits/EigerWormholeAudit.pdf#page=20) concerned incorrect author derivation. The PoC positively asserts that the correctly double-Poseidon-derived author receives the exact fee; the failure occurs afterward.
* **Eiger F-05:** [Eiger `EQ-QNT-WORMHOLE-F-05`](https://github.com/immunefi-team/audit-comp-quantus-chain/blob/3b243b870a5442a6e5f443056f67bbba87d472f5/audits/EigerWormholeAudit.pdf#page=29) concerned unsupported signed transfer variants. This is internal, event-free settlement outside those signed-call variants.
* **Published reward-quantization issue:** The real fixture happens to pay a half-quantum miner share, but the second PoC gives the miner one complete `0.01 QUAN` quantum and still creates no leaf. The missing-recorder defect therefore exists independently of amount quantization.

### Recommended remediation

Treat the miner fee as an atomic keyless-account credit-and-proof operation. Accumulate sub-quantum entitlements per miner, and whenever a whole quantum is available, credit exactly that amount and call `record_transfer` in the same transactional operation. If leaf recording fails, roll back the balance credit and do not emit `MinerVolumeFeePaid`. Historical missing-leaf balances require an explicit migration because later notes cannot absorb them.

## Impact Details

**Selected in-scope impact: High — Permanent freezing of unclaimed yield.**

For an exit expressed in the circuit's `0.01 QUAN` units, the frozen fee rule is:

```
fee_quanta = ceil(exit_quanta * 4 / 9,996)
miner_yield = 50% * fee_quanta * 0.01 QUAN
```

For large batches the miner's entitlement approaches `2.0008 bps` of exit volume. For each ordinary nonzero batch with a canonical QPoW author and a successful miner credit:

{% stepper %}
{% step %}

## The users' proof is verified and their input nullifiers are consumed.

{% endstep %}

{% step %}

## The author's correct Wormhole account receives the miner share in its raw free balance.

{% endstep %}

{% step %}

## `MinerVolumeFeePaid` reports that yield as successfully paid.

{% endstep %}

{% step %}

## No transfer count or Merkle leaf assigns the yield to the author.

{% endstep %}

{% step %}

## The Wormhole secret cannot authorize a normal ML-DSA transfer, and the miner cannot produce membership for a nonexistent leaf.

{% endstep %}

{% step %}

## Therefore 100% of that credited fee yield remains indefinitely frozen in the keyless account.

{% endstep %}
{% endstepper %}

The committed real-proof fixture credits `5,000,000,000` Planck (`0.005 QUAN`) while leaving the author's `TransferCount` at zero; the only new leaf belongs to the user exit. The aligned control credits exactly `10,000,000,000` Planck (`0.01 QUAN`) and still leaves the author without a leaf. This proves the loss is caused by missing ownership recording rather than the circuit's precision floor.

No malformed proof, race, compromised key, privileged role, or 51% attack is required. Any normal valid Wormhole exit can trigger the faulty path, and the loss accumulates across both private and public batches. There is no miner-controlled recovery path. Root governance could exceptionally force-transfer or migrate historical balances, but that does not make the yield claimable through the protocol and does not change the selected High impact from “Permanent freezing of unclaimed yield.”

## References

* [Quantus competition scope and impacts](https://immunefi.com/audit-competition/audit-comp-quantus/scope/)
* [Scoped chain asset](https://github.com/immunefi-team/audit-comp-quantus-chain)
* [QPoW author/Wormhole address derivation](https://github.com/immunefi-team/audit-comp-quantus-chain/blob/3b243b870a5442a6e5f443056f67bbba87d472f5/primitives/wormhole/src/lib.rs#L97-L150)
* [Private/public batch entry points](https://github.com/immunefi-team/audit-comp-quantus-chain/blob/3b243b870a5442a6e5f443056f67bbba87d472f5/pallets/wormhole/src/lib.rs#L576-L645)
* [Correct exit credit plus leaf recording](https://github.com/immunefi-team/audit-comp-quantus-chain/blob/3b243b870a5442a6e5f443056f67bbba87d472f5/pallets/wormhole/src/lib.rs#L816-L859)
* [Volume-fee calculation and miner split](https://github.com/immunefi-team/audit-comp-quantus-chain/blob/3b243b870a5442a6e5f443056f67bbba87d472f5/pallets/wormhole/src/lib.rs#L867-L897)
* [Vulnerable miner credit](https://github.com/immunefi-team/audit-comp-quantus-chain/blob/3b243b870a5442a6e5f443056f67bbba87d472f5/pallets/wormhole/src/lib.rs#L937-L983)
* [`record_transfer` leaf insertion](https://github.com/immunefi-team/audit-comp-quantus-chain/blob/3b243b870a5442a6e5f443056f67bbba87d472f5/pallets/wormhole/src/lib.rs#L1273-L1319)
* [Runtime volume-fee rates](https://github.com/immunefi-team/audit-comp-quantus-chain/blob/3b243b870a5442a6e5f443056f67bbba87d472f5/runtime/src/configs/mod.rs#L728-L752)
* [PR #619 indexer-event change](https://github.com/Quantus-Network/chain/pull/619)
* [Eiger Wormhole audit](https://github.com/immunefi-team/audit-comp-quantus-chain/blob/3b243b870a5442a6e5f443056f67bbba87d472f5/audits/EigerWormholeAudit.pdf)

## Proof of Concept

### Reproduction — one patch file against the scoped repository

This PoC runs entirely as local unit tests against the exact in-scope frozen commit. It does not contact a public network, require private keys, replace any production code, or require an attachment. The single patch file adds only two regression tests to the repository's existing `pallets/wormhole/src/tests.rs` test module:

* a real-proof test that executes the committed Plonky2 proof in `pallets/wormhole/test-data/private_batch.hex`; and
* a whole-quantum control that exercises the same shared `process_exit_bundle` settlement routine with an exactly representable `0.01 QUAN` miner fee.

{% stepper %}
{% step %}

## Clone the scoped asset and check out the frozen commit

```bash
git clone https://github.com/immunefi-team/audit-comp-quantus-chain.git
cd audit-comp-quantus-chain
git checkout 3b243b870a5442a6e5f443056f67bbba87d472f5
```

{% endstep %}

{% step %}

## Create the patch file

Create this one file in the repository root and paste the complete unified diff from the end of this section into it:

```
poc-miner-volume-fee-missing-leaf.patch
```

{% endstep %}

{% step %}

## Apply the patch and run the PoC tests

```bash
git apply --check poc-miner-volume-fee-missing-leaf.patch
git apply poc-miner-volume-fee-missing-leaf.patch
cargo test -p pallet-wormhole --lib poc_miner_volume_fee_ -- \
  --nocapture --test-threads=1
```

`--test-threads=1` only makes the printed evidence deterministic. It does not change runtime state or either assertion path. The repository's pinned Rust `1.93.0` toolchain is selected automatically.
{% endstep %}
{% endstepper %}

### Expected result

```
running 2 tests
test tests::exit_bundle_tests::poc_miner_volume_fee_aligned_share_still_has_no_leaf ... aligned_total_fee=20000000000
aligned_miner_fee=10000000000
aligned_miner_balance=10000000000
aligned_miner_transfer_count=0
aligned_leaf_count_before=0
aligned_leaf_count_after=1
aligned_miner_leaf_created=false
ok
test tests::private_batch_proof_tests::poc_miner_volume_fee_real_proof_has_no_withdrawable_leaf ... real_proof_exit_quanta=1998
real_proof_total_fee=10000000000
real_proof_miner_fee=5000000000
real_proof_miner_balance=5000000000
real_proof_miner_transfer_count=0
real_proof_leaf_count_before=0
real_proof_leaf_count_after=1
real_proof_miner_leaf_created=false
ok

test result: ok. 2 passed; 0 failed; 0 ignored; 0 measured; 71 filtered out; finished in 0.04s
```

The test output and assertions establish that:

* the repository's authentic private-batch proof verifies and settles normally;
* the correct QPoW-derived author receives the exact `5,000,000,000`-Planck raw balance and `MinerVolumeFeePaid` event;
* the real user exit inserts one leaf, proving that the ZK tree is active;
* the author's `TransferCount` remains zero and no newly inserted leaf belongs to the author;
* the whole-quantum control credits exactly `10,000,000,000` Planck (`0.01 QUAN`) and still creates no author leaf; and
* therefore the missing leaf is caused by the omitted recorder call, not by sub-quantum rounding.

The first test uses the real verifier and committed proof fixture. The second deliberately calls only the exact shared post-verification settlement function so that amount quantization is isolated as a control; it neither mocks nor replaces the vulnerable miner-credit code.

### Complete single-file PoC — `poc-miner-volume-fee-missing-leaf.patch`

Copy this file exactly as shown:

```diff
diff --git a/pallets/wormhole/src/tests.rs b/pallets/wormhole/src/tests.rs
index 4473b706..a918ad1f 100644
--- a/pallets/wormhole/src/tests.rs
+++ b/pallets/wormhole/src/tests.rs
@@ -1092,6 +1092,80 @@ mod private_batch_proof_tests {
 		});
 	}
 
+	/// PoC: a real verified exit credits the keyless QPoW author but creates no author leaf.
+	#[test]
+	fn poc_miner_volume_fee_real_proof_has_no_withdrawable_leaf() {
+		new_test_ext().execute_with(|| {
+			let proof = deserialize_test_proof();
+			let inputs = parse_private_batch_public_inputs(&proof).expect("Should parse");
+			let block_number = inputs.block_data.block_number as u64;
+			let block_hash_bytes: [u8; 32] =
+				inputs.block_data.block_hash.as_ref().try_into().unwrap();
+			frame_system::BlockHash::<Test>::insert(block_number, H256::from(block_hash_bytes));
+			System::set_block_number(block_number + 10);
+
+			let miner_preimage = [7u8; 32];
+			set_miner_preimage_digest(miner_preimage);
+			let author = sp_core::crypto::AccountId32::from(
+				qp_wormhole::derive_wormhole_address(miner_preimage).unwrap(),
+			);
+
+			let exit_quanta: u128 = inputs
+				.account_data
+				.iter()
+				.filter(|a| a.summed_output_amount > 0)
+				.map(|a| a.summed_output_amount as u128)
+				.sum();
+			let total_fee = super::ceil_volume_fee(exit_quanta, VolumeFeeRateBps::get() as u128);
+			let expected_miner_fee = total_fee - total_fee / 2;
+			assert_eq!(exit_quanta, 1_998);
+			assert_eq!(total_fee, 10_000_000_000);
+			assert_eq!(expected_miner_fee, 5_000_000_000);
+
+			assert_eq!(Balances::free_balance(&author), 0);
+			assert_eq!(crate::TransferCount::<Test>::get(&author), 0);
+			let leaf_count_before = ZkTree::leaf_count();
+
+			assert_ok!(Wormhole::verify_private_batch(
+				RawOrigin::None.into(),
+				get_test_proof_bytes()
+			));
+
+			System::assert_has_event(
+				crate::Event::<Test>::MinerVolumeFeePaid {
+					miner: author.clone(),
+					amount: expected_miner_fee,
+				}
+				.into(),
+			);
+			assert_eq!(Balances::free_balance(&author), expected_miner_fee);
+			assert_eq!(
+				crate::TransferCount::<Test>::get(&author),
+				0,
+				"fee balance exists, but no spendable ZK leaf was recorded"
+			);
+			let leaf_count_after = ZkTree::leaf_count();
+			assert_eq!(leaf_count_after, leaf_count_before + 1, "the user exit inserts one leaf");
+			let author_leaf_created = (leaf_count_before..leaf_count_after).any(|index| {
+				ZkTree::leaf(index).is_some_and(|leaf| leaf.to == author)
+			});
+
+			println!("real_proof_exit_quanta={exit_quanta}");
+			println!("real_proof_total_fee={total_fee}");
+			println!("real_proof_miner_fee={expected_miner_fee}");
+			println!("real_proof_miner_balance={}", Balances::free_balance(&author));
+			println!(
+				"real_proof_miner_transfer_count={}",
+				crate::TransferCount::<Test>::get(&author)
+			);
+			println!("real_proof_leaf_count_before={leaf_count_before}");
+			println!("real_proof_leaf_count_after={leaf_count_after}");
+			println!("real_proof_miner_leaf_created={author_leaf_created}");
+
+			assert!(!author_leaf_created, "no new ZK leaf belongs to the credited miner");
+		});
+	}
+
 	#[test]
 	fn test_verify_private_batch_cannot_replay() {
 		new_test_ext().execute_with(|| {
@@ -1820,6 +1894,63 @@ mod exit_bundle_tests {
 		});
 	}
 
+	/// Control PoC: the omission persists when the miner share is exactly one leaf quantum.
+	#[test]
+	fn poc_miner_volume_fee_aligned_share_still_has_no_leaf() {
+		new_test_ext().execute_with(|| {
+			System::set_block_number(1);
+			let miner_preimage = [7u8; 32];
+			set_miner_preimage_digest(miner_preimage);
+			let author = AccountId32::from(
+				qp_wormhole::derive_wormhole_address(miner_preimage).unwrap(),
+			);
+
+			// 2,500 output quanta require a two-quantum fee at 4 bps:
+			// ceil(2,500 * 4 / 9,996) = 2. The 50% miner share is one
+			// complete 0.01-QUAN leaf quantum, so quantization is not the cause.
+			let exit_quanta = 2_500u32;
+			let total_fee = super::ceil_volume_fee(
+				exit_quanta as u128,
+				VolumeFeeRateBps::get() as u128,
+			);
+			let expected_miner_fee = total_fee - total_fee / 2;
+			assert_eq!(total_fee, 2 * crate::SCALE_DOWN_FACTOR);
+			assert_eq!(expected_miner_fee, crate::SCALE_DOWN_FACTOR);
+
+			let leaf_count_before = ZkTree::leaf_count();
+			assert_ok!(Wormhole::process_exit_bundle(bundle(
+				vec![segment(&[1], &[(10, exit_quanta)])],
+				None,
+			)));
+
+			System::assert_has_event(
+				crate::Event::<Test>::MinerVolumeFeePaid {
+					miner: author.clone(),
+					amount: expected_miner_fee,
+				}
+				.into(),
+			);
+			let miner_balance = Balances::free_balance(&author);
+			let miner_transfer_count = crate::TransferCount::<Test>::get(&author);
+			let leaf_count_after = ZkTree::leaf_count();
+			assert_eq!(miner_balance, expected_miner_fee);
+			assert_eq!(miner_transfer_count, 0);
+			let author_leaf_created = (leaf_count_before..leaf_count_after).any(|index| {
+				ZkTree::leaf(index).is_some_and(|leaf| leaf.to == author)
+			});
+
+			println!("aligned_total_fee={total_fee}");
+			println!("aligned_miner_fee={expected_miner_fee}");
+			println!("aligned_miner_balance={miner_balance}");
+			println!("aligned_miner_transfer_count={miner_transfer_count}");
+			println!("aligned_leaf_count_before={leaf_count_before}");
+			println!("aligned_leaf_count_after={leaf_count_after}");
+			println!("aligned_miner_leaf_created={author_leaf_created}");
+
+			assert!(!author_leaf_created, "aligned fee share still has no author leaf");
+		});
+	}
+
 	#[test]
 	fn process_exit_bundle_burns_rebate_when_aggregator_mint_fails() {
 		new_test_ext().execute_with(|| {
```
