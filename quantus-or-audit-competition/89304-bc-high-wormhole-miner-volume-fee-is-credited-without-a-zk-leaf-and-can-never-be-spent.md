# 89304 bc high wormhole miner volume fee is credited without a zk leaf and can never be spent

**Submitted on Aug 18th 2026 at 14:17:20 UTC by @Pyro for** [**Audit Comp | Quantus**](https://immunefi.com/audit-competition/audit-comp-quantus)

* **Report ID:** #89304
* **Report Type:** Blockchain/DLT
* **Report severity:** High
* **Target:** <https://github.com/immunefi-team/audit-comp-quantus-chain>
* **Impacts:**
  * Permanent freezing of unclaimed yield

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

The author is `qp_wormhole::extract_author_from_digest`, which returns `derive_wormhole_address(preimage)`. The miner holds the secret behind that address and can satisfy the exit circuit. It never receives a leaf, so there is nothing to prove against.

Nothing repairs the omission later:

* `increase_balance` writes the balance without emitting a `Balances` event, so `WormholeProofRecorderExtension` cannot see it.
* The settlement is an unsigned bare extrinsic, where that extension does not run at all.
* The pallet exposes only `verify_private_batch` and `verify_public_batch`, so no admin path can insert the leaf.
* Leaves are per `(recipient, transfer_count)` and each commits one fixed amount, so the mining-rewards leaf does not cover the fee.

Both failure arms of the same branch add the unpaid fee back to `burn_amount`. The branch's contract is pay or burn. The success path does neither.

{% stepper %}
{% step %}

## A miner runs with a stable `rewards_inner_hash`

Its author account already exists from earlier block rewards.
{% endstep %}

{% step %}

## A user settles an exit

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

## The balance is credited

`increase_balance(&author, 205_000_000_000, Exact)` succeeds and `MinerVolumeFeePaid` is emitted.
{% endstep %}

{% step %}

## No leaf is created

`TransferCount` for the author is unchanged and no `NativeTransferred` event fires, so no leaf exists for the credit.
{% endstep %}

{% step %}

## The entitlement remains unchanged

The author's balance is 0.205 QUAN higher and its exitable entitlement is unchanged.
{% endstep %}

{% step %}

## The issue repeats

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

Add to `mod private_batch_proof_tests` in `pallets/wormhole/src/tests.rs`. It reuses the committed proof fixture, so no node build is needed.

```bash
cargo test -p pallet-wormhole poc_miner_fee_credited_without_leaf -- --nocapture
```

```rust
#[test]
fn poc_miner_fee_credited_without_leaf() {
	new_test_ext().execute_with(|| {
		let inputs = parse_private_batch_public_inputs(&deserialize_test_proof()).unwrap();
		let block_number = inputs.block_data.block_number as u64;
		let block_hash: [u8; 32] = inputs.block_data.block_hash.as_ref().try_into().unwrap();
		frame_system::BlockHash::<Test>::insert(block_number, H256::from(block_hash));
		System::set_block_number(block_number + 10);

		// Block author, seeded exactly as QPoW does.
		let preimage = [7u8; 32];
		set_miner_preimage_digest(preimage);
		let author = sp_core::crypto::AccountId32::from(
			qp_wormhole::derive_wormhole_address(preimage).unwrap(),
		);

		let balance_before = Balances::free_balance(author.clone());
		let leaves_before = Wormhole::transfer_count(&author);

		assert_ok!(Wormhole::verify_private_batch(RawOrigin::None.into(), get_test_proof_bytes()));

		let credited = Balances::free_balance(author.clone()) - balance_before;
		let leaves_after = Wormhole::transfer_count(&author);

		// An exit account settled by the same call, for contrast.
		let exit = inputs.account_data.iter().find(|a| a.summed_output_amount > 0).unwrap();
		let exit_bytes: [u8; 32] = (*exit.exit_account).as_ref().try_into().unwrap();
		let exit_account = sp_core::crypto::AccountId32::from(exit_bytes);

		println!("author balance credited : {} planck", credited);
		println!("author leaves           : {} -> {}", leaves_before, leaves_after);
		println!("exit account leaves     : {}", Wormhole::transfer_count(&exit_account));

		assert!(credited > 0, "the miner fee was credited");
		assert_eq!(leaves_after, leaves_before, "but no leaf was recorded for it");
		assert!(Wormhole::transfer_count(&exit_account) > 0, "the exit account did get one");
	});
}
```

Output shape (the credited amount depends on the committed fixture's exit total):

```
author balance credited : <nonzero> planck
author leaves           : 0 -> 0
exit account leaves     : 1
```

Balance credited, leaf count unchanged. That pair is the whole finding.
