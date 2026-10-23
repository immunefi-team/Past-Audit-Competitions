# 89984 bc high missing zk tree registration for miner volume fees permanently freezes credited funds

**Submitted on Aug 22nd 2026 at 10:03:42 UTC by @Kalyantr for** [**Audit Comp | Quantus**](https://immunefi.com/audit-competition/audit-comp-quantus)

* **Report ID:** #89984
* **Report Type:** Blockchain/DLT
* **Report severity:** High
* **Target:** <https://github.com/immunefi-team/audit-comp-quantus-chain>
* **Impacts:**
  * Permanent freezing of funds

## Description

## Brief/Intro

`Wormhole::process_exit_bundle` credits a successful settlement’s miner volume fee to the current QPoW author’s Wormhole-derived address using `Unbalanced::increase_balance`. Unlike a successful user exit, however, this fee-credit path does not call `Wormhole::record_transfer`.

`record_transfer` is the path that increments the recipient’s `TransferCount` and inserts the corresponding leaf into the ZK tree. Consequently, the miner receives a balance credit but no ZK-tree commitment for that credit. The miner cannot construct the Merkle witness required for a later Wormhole withdrawal, permanently freezing the credited miner fee through all currently available permissionless paths.

## Vulnerability Details

A Wormhole address is controlled for withdrawal by knowledge of its Wormhole secret, not by a normal Dilithium account key. To withdraw a balance from such an address, the holder must produce a valid leaf proof proving that a committed ZK-tree leaf contains:

* The Wormhole recipient address
* Its per-recipient `TransferCount`
* The credited amount
* A valid Merkle path to a committed ZK-tree root

The normal exit-credit path correctly creates this required state:

```
// pallets/wormhole/src/lib.rs — process_exit_bundle

match <T::Currency as Unbalanced<_>>::increase_balance(
    exit_account,
    *exit_balance,
    Precision::Exact,
) {
    Ok(_) => {},
    Err(_) => { /* skip failed exit */ },
}

let from_account: T::WormholeAccountId = mint_account.clone().into();
let to_account: T::WormholeAccountId = exit_account.clone().into();

Self::record_transfer(
    T::AssetId::default(),
    &from_account,
    &to_account,
    *exit_balance,
);
```

`record_transfer` increments `TransferCount` and calls `T::ZkTree::record_transfer`, thereby committing the credit as a ZK-tree leaf. A future Wormhole withdrawal additionally requires knowledge of the Wormhole secret corresponding to the credited recipient.

The miner-fee path only credits the balance and emits an informational event:

```
// pallets/wormhole/src/lib.rs — process_exit_bundle

match <T::Currency as Unbalanced<_>>::increase_balance(
    &author,
    miner_fee,
    Precision::Exact,
) {
    Ok(_) => {
        Self::deposit_event(Event::MinerVolumeFeePaid {
            miner: author,
            amount: miner_fee,
        });
    },
    Err(e) => {
        // fee is redirected to burn amount
    },
}
```

There is no corresponding call to:

```
Self::record_transfer(...)
```

Therefore, a successful miner-fee payment has this state transition:

```
QPoW author inner hash
  → derived Wormhole author address
  → miner balance credited
  → MinerVolumeFeePaid emitted
  → no TransferCount increment
  → no ZK-tree leaf
  → no Merkle witness for a future Wormhole withdrawal
```

The current runtime exposes no permissionless public call that can register a ZK-tree leaf retroactively for an existing credited balance. Sending a later normal transfer to the same Wormhole address creates a leaf only for that later transfer; it cannot create a leaf for the already credited miner fee.

A governance-led runtime migration could potentially repair affected historical credits, but miners have no current permissionless recovery route.

## Impact Details

A valid Wormhole settlement with a nonzero fee permanently freezes the miner’s volume-fee credit.

Any holder of a valid Wormhole settlement proof can submit the unsigned settlement transaction with a nonzero fee. When it is included, the current QPoW block author receives a miner-fee credit that is not registered in the ZK tree. No privileged access, invalid proof, or malicious miner action is required.

The affected funds are miner-fee credits, not user deposit principal. They are correctly credited to the intended derived Wormhole address but cannot be withdrawn because the protocol does not commit the required leaf.

This matches the in-scope impact:

> Permanent freezing of funds.

## Distinction from PR #619

Commit `60ec991` added `MinerVolumeFeePaid` after the miner-fee balance credit to make the payment visible to indexers. It did not call `record_transfer` or otherwise register the credit in the ZK tree.

This report concerns the distinct security consequence: without the ZK-tree leaf and `TransferCount` update, the credited miner fee has no withdrawal witness and is permanently unexitable through permissionless protocol paths.

## Recommended Remediation

Record a Wormhole transfer immediately after each successful fee credit. This must happen before moving `author` into `MinerVolumeFeePaid`.

```rust
match <T::Currency as Unbalanced<_>>::increase_balance(
    &author,
    miner_fee,
    frame_support::traits::tokens::Precision::Exact,
) {
    Ok(_) => {
        let from: <T as Config>::WormholeAccountId = mint_account.clone().into();
        let to: <T as Config>::WormholeAccountId = author.clone().into();

        Self::record_transfer(
            T::AssetId::default(),
            &from,
            &to,
            miner_fee,
        );

        Self::deposit_event(Event::MinerVolumeFeePaid {
            miner: author,
            amount: miner_fee,
        });
    },
    Err(e) => {
        log::warn!(
            "Miner fee of {:?} could not be minted ({:?}); burning it instead",
            miner_fee,
            e
        );
        burn_amount_u128 = burn_amount_u128.saturating_add(miner_fee_u128);
    },
}
```

Apply the same pattern to the aggregator-rebate branch, but only after its `increase_balance(&aggregator_account, aggregator_fee, ...)` succeeds.

Add regression tests asserting that:

* A successful miner-fee credit increases `ZkTree::leaf_count()` by one additional leaf.
* `Wormhole::transfer_count(&miner)` increases by one.
* The newly recorded leaf belongs to the miner and has `amount == miner_fee`.
* The equivalent assertions hold for a successful aggregator rebate to a Wormhole address.

Finally, rebenchmark/update the affected settlement weights, because private settlement can now insert one additional leaf for the miner fee and public settlement can insert up to two additional leaves (miner fee plus aggregator rebate).

This forward fix does not recover already unrecorded historical credits. Those require a separate governance-led migration or compensation procedure based on reconstructable historical fee payments.

## Proof of Concept

Add this test inside the existing `private_batch_proof_tests` module in `pallets/wormhole/src/tests.rs`.

Run this command:

```bash
cargo test -p pallet-wormhole tests::private_batch_proof_tests::poc_miner_volume_fee_credit_has_no_zk_leaf -- --exact --nocapture
```

```rust
#[test]
	fn poc_miner_volume_fee_credit_has_no_zk_leaf() {
		use sp_runtime::traits::ValidateUnsigned;

		new_test_ext().execute_with(|| {
			let proof = deserialize_test_proof();
			let inputs = parse_private_batch_public_inputs(&proof).expect("fixture must parse");

			let block_number = inputs.block_data.block_number as u64;
			let block_hash_bytes: [u8; 32] =
				inputs.block_data.block_hash.as_ref().try_into().expect("32-byte hash");
			frame_system::BlockHash::<Test>::insert(block_number, H256::from(block_hash_bytes));
			System::set_block_number(block_number + 10);

			// This is the same pre-runtime-digest route QPoW uses. The runtime derives
			// the credited AccountId from this preimage, rather than receiving a
			// normal transaction signer.
			let miner_preimage = [7u8; 32];
			set_miner_preimage_digest(miner_preimage);
			let miner = sp_core::crypto::AccountId32::from(
				qp_wormhole::derive_wormhole_address(miner_preimage)
					.expect("test preimage limbs are canonical"),
			);

			let exit_leaf_count = inputs
				.account_data
				.iter()
				.filter(|account| account.summed_output_amount > 0)
				.count() as u64;
			let exit_quanta: u128 = inputs
				.account_data
				.iter()
				.filter(|account| account.summed_output_amount > 0)
				.map(|account| account.summed_output_amount as u128)
				.sum();
			let total_fee = super::ceil_volume_fee(exit_quanta, VolumeFeeRateBps::get() as u128);
			let miner_fee = total_fee - total_fee / 2;
			assert!(miner_fee > 0, "fixture must produce a miner fee");

			let balance_before = Balances::free_balance(miner.clone());
			let leaves_before = ZkTree::leaf_count();
			let transfer_count_before = Wormhole::transfer_count(&miner);
			let proof_bytes = get_test_proof_bytes();
			let call = crate::Call::<Test>::verify_private_batch {
				proof_bytes: proof_bytes.clone(),
			};

			// In a real unsigned extrinsic this runs before dispatch. It performs the
			// full ZK verification; the dispatch body below deliberately performs only
			// cheap parsing before it settles the already-verified bundle.
			assert!(
				<Wormhole as ValidateUnsigned>::pre_dispatch(&call).is_ok(),
				"the real proof fixture must pass full ZK verification before settlement"
			);

			assert_ok!(Wormhole::verify_private_batch(
				RawOrigin::None.into(),
				proof_bytes,
			));

			// The fee is real: a successful balance credit and its cosmetic event exist.
			assert_eq!(Balances::free_balance(miner.clone()), balance_before + miner_fee);
			System::assert_has_event(
				crate::Event::<Test>::MinerVolumeFeePaid { miner: miner.clone(), amount: miner_fee }
					.into(),
			);

			// Only the users' exit credits produced leaves. The miner credit produced
			// neither the leaf nor the count that a leaf proof requires.
			assert_eq!(ZkTree::leaf_count(), leaves_before + exit_leaf_count);
			assert_eq!(Wormhole::transfer_count(&miner), transfer_count_before);
			assert!(
				(0..ZkTree::leaf_count()).all(|index| {
					let leaf = ZkTree::leaf(index).expect("all indices below leaf_count exist");
					leaf.to != miner
				}),
				"the credited miner must have no ZK-tree leaf"
			);
			println!(
				"PoC reproduced: miner fee = {miner_fee}; miner balance = {}; \
				 ZK leaves = {leaves_before} -> {}; miner TransferCount = {transfer_count_before} -> {}",
				Balances::free_balance(miner.clone()),
				ZkTree::leaf_count(),
				Wormhole::transfer_count(&miner),
			);
		});
	}
```

The test passes with the current code and produces:

```
PoC reproduced:
miner fee = 5000000000
miner balance = 5000000000
ZK leaves = 0 -> 1
miner TransferCount = 0 -> 0
```
