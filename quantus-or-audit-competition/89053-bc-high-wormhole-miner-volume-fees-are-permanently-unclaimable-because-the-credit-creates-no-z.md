# 89053 bc high wormhole miner volume fees are permanently unclaimable because the credit creates no zk tree leaf

**Submitted on Aug 16th 2026 at 19:37:20 UTC by @tinnohofficial for** [**Audit Comp | Quantus**](https://immunefi.com/audit-competition/audit-comp-quantus)

* **Report ID:** #89053
* **Report Type:** Blockchain/DLT
* **Report severity:** High
* **Target:** <https://github.com/immunefi-team/audit-comp-quantus-chain>
* **Impacts:**
  * Permanent freezing of unclaimed yield

## Description

## Brief/Intro

QPoW miners are paid to Poseidon-derived wormhole addresses. These addresses have no Dilithium signing key, so their funds can only be claimed through a ZK-tree leaf that commits the recipient and amount.

The wormhole pallet credits the miner's volume-fee share to the correct address, but does not create that leaf. The balance is visible in `System::Account`, yet the miner can neither sign from the address nor prove a wormhole exit for the fee. Every successfully credited miner volume fee is therefore permanently unclaimable without privileged recovery.

## Vulnerability Details

### Wormhole credits require a ZK-tree leaf

For a normal user exit, `process_exit_bundle` performs both required operations:

```rust
<T::Currency as Unbalanced<_>>::increase_balance(exit_account, *exit_balance, Precision::Exact)?;

Self::record_transfer(
    T::AssetId::default(),
    &from_account,
    &to_account,
    *exit_balance,
);
```

See `pallets/wormhole/src/lib.rs:822-859`.

`record_transfer` increments `TransferCount` and inserts a leaf containing the recipient, transfer count, asset ID, and exact amount:

```rust
let current_count = TransferCount::<T>::get(&leaf_to);
TransferCount::<T>::insert(&leaf_to, current_count.saturating_add(1));
T::ZkTree::record_transfer(leaf_to.into(), current_count.into(), asset_id, amount);
```

See `pallets/wormhole/src/lib.rs:1280-1310` and `pallets/zk-tree/src/lib.rs:114-147`.

Normal block rewards follow the same rule by calling `record_transfer_proof` immediately after minting: `pallets/mining-rewards/src/lib.rs:230-237`.

### The miner-fee branch omits the leaf

The miner-fee branch only changes the balance and emits an event:

```rust
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
    // ...
}
```

See `pallets/wormhole/src/lib.rs:942-983`. There is no `record_transfer` or `record_transfer_proof` call.

Nothing records the leaf indirectly:

* `increase_balance` is event-free, as documented at `pallets/wormhole/src/lib.rs:822-827` and tested by `exit_credits_emit_no_scannable_transfer_events`.
* `MinerVolumeFeePaid` is not one of the balance events scanned by `WormholeProofRecorderExtension`: `runtime/src/transaction_extensions.rs:329-395`.
* The settlement calls are bare unsigned extrinsics, so the extension's normal signed-transaction scan does not process them.

### The credited address cannot spend normally

`extract_author_from_digest` derives the author with `derive_wormhole_address(preimage)`: `primitives/wormhole/src/lib.rs:122-150`. `MINING.md:41-48` confirms that mining rewards use this wormhole address.

The miner therefore has only two possible spend paths, and both are unavailable for this fee:

1. A normal balance transfer is impossible because no Dilithium signing key exists for the Poseidon-derived address.
2. A wormhole exit is impossible because no leaf commits the miner-fee amount.

A later block reward does not recover the fee. Its leaf commits only the later reward amount, not the account's full free balance.

### Distinction from the public Eiger findings

This is not the previously reported author-derivation bug:

* `EQ-QNT-WORMHOLE-F-03` reported that the pallet decoded the raw preimage as the author. That was fixed by deriving the correct wormhole address. The current code now credits the correct address, but still omits the leaf needed to claim the credit.
* `EQ-QNT-WORMHOLE-F-05` concerned missing signed transfer call variants in the proof recorder. This fee is an event-free internal balance credit inside a bare unsigned settlement call, so that fix does not cover it.

The current defect has a different root cause and requires a different fix: record a leaf for the correctly derived author.

## Impact Details

The runtime assigns 50% of each computed wormhole volume fee to the block author: `runtime/src/configs/mod.rs:728-738`. Whenever that credit succeeds, the full miner share becomes permanently unclaimable.

The protocol emits `MinerVolumeFeePaid` and displays a real free balance, but the miner cannot spend or claim it. No later reward sweeps the missing amount. Recovery of already-stranded fees requires privileged governance or a storage migration.

This directly matches **High - Permanent freezing of unclaimed yield**: the frozen value is protocol-allocated miner-fee revenue, not user principal.

## Recommended Fix

Record a transfer immediately after each successful miner-fee credit:

```rust
Ok(_) => {
    let from = T::MintingAccount::get().into();
    let to = author.clone().into();
    Self::record_transfer(T::AssetId::default(), &from, &to, miner_fee);
    Self::deposit_event(Event::MinerVolumeFeePaid { miner: author, amount: miner_fee });
},
```

The settlement weight must also include this additional ZK-tree insertion. Previously stranded balances require a separate migration or privileged recovery.

## References

* `pallets/wormhole/src/lib.rs:822-859` - user exit credit followed by `record_transfer`.
* `pallets/wormhole/src/lib.rs:942-983` - miner-fee credit with no leaf.
* `pallets/wormhole/src/lib.rs:1280-1330` - leaf creation and transfer-count update.
* `pallets/zk-tree/src/lib.rs:114-147` - leaf fields, including the exact amount.
* `pallets/mining-rewards/src/lib.rs:230-237` - normal miner reward followed by proof recording.
* `primitives/wormhole/src/lib.rs:97-150` - wormhole author derivation.
* `runtime/src/configs/mod.rs:728-738` - volume-fee configuration.
* `MINING.md:41-48` - miner rewards use derived wormhole addresses.

## Proof of Concept

The PoC submits the pallet's production private-batch fixture through the real `verify_private_batch` call. In the same settlement:

* The user exit receives a balance and `TransferCount = 1`, proving its leaf was inserted.
* The block author receives the miner fee but remains at `TransferCount = 0`, proving no leaf was inserted.
* A second test shows that a later block-reward leaf commits only the reward and does not cover the stranded fee.

### Run

From `audit-comp-quantus-chain`:

{% stepper %}
{% step %}

### Copy the PoC

Copy the PoC to `pallets/wormhole/src/poc_miner_fee.rs`.
{% endstep %}

{% step %}

### Register the test module

Add the following above the existing `#[cfg(test)] mod tests;` in `pallets/wormhole/src/lib.rs`:

```rust
#[cfg(test)]
mod poc_miner_fee;
```

{% endstep %}

{% step %}

### Run the tests

```bash
cargo test -p pallet-wormhole --lib poc_miner_fee -- --nocapture
```

{% endstep %}
{% endstepper %}

### Observed Output

Tested against frozen chain commit `3b243b8`:

```
running 2 tests
author holds 57000000000000 planck but the only claimable leaf is worth 50000000000000 planck
test poc_miner_fee::poc::a_later_block_reward_leaf_does_not_sweep_the_stranded_fee ... ok
miner d2027e5ab7dda597109a170d67b9c932746ff615b99a22ef30c13f1f2c58651b credited 5000000000 planck with ZERO ZK leaves -> permanently unspendable
exit account 0404040404040404040404040404040404040404040404040404040404040404 credited and given TransferCount = 1
test poc_miner_fee::poc::miner_volume_fee_is_credited_without_a_zk_leaf ... ok

test result: ok. 2 passed; 0 failed
```

### PoC Code

```rust
//! PoC: the miner's share of the wormhole volume fee is credited to a keyless
//! wormhole address WITHOUT the compensating ZK-tree leaf, so it can never be spent.

#[cfg(test)]
mod poc {
	use crate::{mock::*, pallet::TransferCount};
	use frame_support::assert_ok;
	use frame_system::RawOrigin;
	use qp_wormhole_verifier::{parse_private_batch_public_inputs, ProofWithPublicInputs, C, F};
	use sp_core::{crypto::AccountId32, H256};

	const D: usize = 2;
	const PRIVATE_BATCH_PROOF_HEX: &str = include_str!("../test-data/private_batch.hex");

	fn proof_bytes() -> Vec<u8> {
		hex::decode(PRIVATE_BATCH_PROOF_HEX.trim()).expect("Invalid hex in test proof")
	}

	fn deserialize_proof() -> ProofWithPublicInputs<F, C, D> {
		let verifier = crate::get_private_batch_verifier().expect("verifier available");
		ProofWithPublicInputs::<F, C, D>::from_bytes(proof_bytes(), &verifier.circuit_data.common)
			.expect("proof deserializes")
	}

	#[test]
	fn miner_volume_fee_is_credited_without_a_zk_leaf() {
		new_test_ext().execute_with(|| {
			let proof = deserialize_proof();
			let inputs = parse_private_batch_public_inputs(&proof).expect("parse");

			let block_number = inputs.block_data.block_number as u64;
			let block_hash_bytes: [u8; 32] =
				inputs.block_data.block_hash.as_ref().try_into().unwrap();
			frame_system::BlockHash::<Test>::insert(block_number, H256::from(block_hash_bytes));
			System::set_block_number(block_number + 10);

			let miner_preimage = [7u8; 32];
			set_miner_preimage_digest(miner_preimage);
			let author = AccountId32::from(
				qp_wormhole::derive_wormhole_address(miner_preimage).expect("canonical limbs"),
			);

			let exit_account: AccountId32 = {
				let account = inputs
					.account_data
					.iter()
					.find(|account| account.summed_output_amount > 0)
					.expect("fixture has a real exit");
				let bytes: [u8; 32] = (*account.exit_account).as_ref().try_into().unwrap();
				AccountId32::from(bytes)
			};

			assert_eq!(TransferCount::<Test>::get(&author), 0);
			assert_eq!(TransferCount::<Test>::get(&exit_account), 0);
			let author_before = Balances::free_balance(author.clone());

			assert_ok!(Wormhole::verify_private_batch(RawOrigin::None.into(), proof_bytes()));

			let author_after = Balances::free_balance(author.clone());
			let miner_fee = author_after - author_before;

			assert!(miner_fee > 0, "miner received a volume-fee credit");
			assert_eq!(
				TransferCount::<Test>::get(&exit_account),
				1,
				"exit credit has a ZK leaf"
			);
			assert_eq!(
				TransferCount::<Test>::get(&author),
				0,
				"miner-fee credit has no ZK leaf"
			);

			let author_leaf_events = System::events()
				.into_iter()
				.filter(|event| {
					matches!(
						&event.event,
						RuntimeEvent::Wormhole(crate::Event::NativeTransferred { to, .. })
							if to == &author
					)
				})
				.count();
			assert_eq!(author_leaf_events, 0);

			println!(
				"miner {:?} credited {} planck with ZERO ZK leaves -> permanently unspendable",
				author, miner_fee
			);
			println!(
				"exit account {:?} credited and given TransferCount = {}",
				exit_account,
				TransferCount::<Test>::get(&exit_account)
			);
		});
	}

	#[test]
	fn a_later_block_reward_leaf_does_not_sweep_the_stranded_fee() {
		use qp_wormhole::TransferProofRecorder;

		new_test_ext().execute_with(|| {
			System::set_block_number(1);
			let miner_preimage = [7u8; 32];
			let author = AccountId32::from(
				qp_wormhole::derive_wormhole_address(miner_preimage).expect("canonical limbs"),
			);

			assert_ok!(
				<Balances as frame_support::traits::fungible::Unbalanced<AccountId>>::increase_balance(
					&author,
					7 * UNIT,
					frame_support::traits::tokens::Precision::Exact,
				)
			);

			let reward = 50 * UNIT;
			assert_ok!(<Balances as frame_support::traits::fungible::Mutate<AccountId>>::mint_into(
				&author, reward
			));
			Wormhole::record_transfer_proof(None, MINTING_ACCOUNT, author.clone(), reward);

			let claimable_amount = System::events()
				.into_iter()
				.find_map(|event| match event.event {
					RuntimeEvent::Wormhole(crate::Event::NativeTransferred {
						to, amount, ..
					}) if to == author => Some(amount),
					_ => None,
				})
				.expect("reward leaf recorded");

			assert_eq!(claimable_amount, reward);
			assert_eq!(Balances::free_balance(&author), 57 * UNIT);
			println!(
				"author holds {} planck but the only claimable leaf is worth {} planck",
				Balances::free_balance(&author),
				claimable_amount
			);
		});
	}
}
```
