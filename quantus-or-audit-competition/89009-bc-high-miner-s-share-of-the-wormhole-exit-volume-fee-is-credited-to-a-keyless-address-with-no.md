# 89009 bc high miner s share of the wormhole exit volume fee is credited to a keyless address with no zk leaf permanently freezing it

**Submitted on Aug 16th 2026 at 13:50:16 UTC by @Naveen0x for** [**Audit Comp | Quantus**](https://immunefi.com/audit-competition/audit-comp-quantus)

* **Report ID:** #89009
* **Report Type:** Blockchain/DLT
* **Report severity:** High
* **Target:** <https://github.com/immunefi-team/audit-comp-quantus-chain>
* **Impacts:**
  * Permanent freezing of funds (fix requires hardfork)

## Description

## Summary

Every wormhole exit pays a volume fee. Half of it is burned and the other half is credited to the block author as mining revenue. That second half can never be moved by anyone.

The recipient address is derived by hashing the miner's PoW digest preimage, which produces a wormhole-form address — a Poseidon image, not a public key. No private key corresponds to it, so nothing can sign for it. A wormhole-form address has exactly one spending path: proving a leaf in the ZK tree. But the credit is performed with an event-free balance mutation and is not paired with a `record_transfer` call, so no leaf is ever created.

The credited value is therefore both unspendable (no key) and unexitable (no leaf). It accrues on every exit for the life of the chain, and neither the miner nor the project can recover it.

Every other credit path in the codebase handles this correctly. `pallet_mining_rewards` pays the *same* derived author address and pairs its `mint_into` with `record_transfer_proof`; `pallet_vesting` and `pallet_reversible_transfers` do the same for their hook-context credits; and the wormhole pallet's own exit credit is paired with `record_transfer` twenty-six lines earlier in the same function. Only the fee credits in the settlement block omit it.

**Target:** `pallets/wormhole` — exit fee settlement\
**Commit:** `3b243b870a5442a6e5f443056f67bbba87d472f5`

{% hint style="info" %}
This report is self-contained. The complete proof of concept is included below; there are no separate attachments to open, and no patches or modifications to the pallet or its mock are required to run it.
{% endhint %}

## Vulnerability details

`process_exit_bundle` in `pallets/wormhole/src/lib.rs` settles fees in the block spanning roughly lines 890–996. Two balance increases happen there:

```rust
// ~:919 — aggregator rebate
increase_balance(&aggregator_account, aggregator_fee, Exact)
// ~:955 — the block author's share of the volume fee
increase_balance(&author, miner_fee, Exact)
```

Neither is followed by a leaf-creating call. Enumerating every `record_transfer` call site in the pallet:

| line             | context                                                                 |
| ---------------- | ----------------------------------------------------------------------- |
| `:486`           | genesis sweep, block 1 only                                             |
| `:854`           | the exit credit, correctly paired with the `increase_balance` at `:828` |
| `:1367`, `:1386` | inside `record_transfer_proof`, the entry point other pallets use       |

There is no call anywhere between `:890` and `:996`. These two fee credits are the only balance-increasing operations in the pallet that produce no leaf.

**Why the recipient cannot sign.** At `:946` the author is resolved through `qp_wormhole::extract_author_from_digest`, which takes the first `PreRuntime` digest item for the PoW engine and passes it to `derive_wormhole_address` (`primitives/wormhole/src/lib.rs:109`), i.e. `rehash_to_bytes(preimage)`. The result is a Poseidon image. The miner supplies the preimage on the command line and the consensus client stamps it into the digest; the miner never controls the resulting address and no keypair maps to it.

**Why nothing else rescues the credit.** The runtime's proof-recorder extension builds leaves by scanning `Balances::{Transfer, Minted, TransferOnHold, ReserveRepatriated}` events. `Unbalanced::increase_balance` emits none of them — deliberately, per the in-code note at `:822-827` explaining that switching to `mint_into` would double-record the exit credit. There are two independent reasons the scanner cannot help: no scannable event is emitted at all, and the settlement runs inside a bare unsigned extrinsic, while transaction extensions only execute for transactions. The only signal emitted is the pallet's own `MinerVolumeFeePaid`, which the recorder does not match.

So the pallet already knows that a credit to a wormhole-form address needs an accompanying leaf, and does it correctly a few lines earlier in the same function — it just doesn't do it for the miner fee.

## Impact

* **Permanent, unrecoverable loss of miner revenue on every exit.** The chain reports the balance as the miner's, and the miner can never spend or exit it.
* **The loss is unbounded in aggregate.** It is a fixed fraction of exit volume with no cap, and it accumulates for the life of the chain.
* **It is silent.** The pallet emits its own fee-paid event, so the payment looks successful. Balance queries, block explorers and indexers all show the fee as paid. The loss is only visible if you separately ask whether a corresponding leaf exists.
* **It removes the miner's only compensation for including an exit.** Exit extrinsics are unsigned (`ensure_none`, `:589`/`:626`) and explicitly fee-free — the dispatch returns `Pays::No` on both the success and error paths (`:603`, `:638`, `:998`). A miner earns no transaction fee for including an exit, so the volume-fee share is the entire reward for that work, and it is exactly the part that cannot be collected.

Magnitude, at the configured constants (`VolumeFeeRateBps = 4`, `VolumeFeesBurnRate = 50%`):

| exit volume (QUAN) | frozen (bps of volume) |
| ------------------ | ---------------------- |
| 100                | 2.50                   |
| 10,000             | 2.005                  |
| 1,000,000          | 2.0008                 |

The rate converges on roughly **2 bps of all exit volume** — a 4 bps fee of which half is burned and half is frozen. Small exits round worse because of the fee's ceiling rounding and one-quantum minimum; even a single-quantum exit freezes half a quantum (5,000,000,000 planck), which sits above the existential deposit, so the below-ED burn fallback at `:966-974` never masks the credit.

One secondary effect the team may want to confirm: `increase_balance` raises total issuance, and `pallet_mining_rewards` budgets each block reward from `MaxSupply - total_issuance()`. If that holds, frozen fee credits also permanently consume emission headroom that would otherwise be paid out as block rewards. I have not measured this and it is not part of the claim.

## Proof of Concept

The test below drives the real `verify_private_batch` unsigned extrinsic using the repository's own proof fixture (`test-data/private_batch.hex`), seeding the block author through the PoW pre-runtime digest exactly as production does. It deliberately mirrors the project's existing `test_verify_private_batch_emits_miner_volume_fee_paid`, and then asserts the one thing that test never checks: whether the miner's credit produced a ZK leaf.

The control is inside the same call. The exit recipients *do* receive leaves during this very extrinsic, so leaf recording demonstrably works on this code path, in this call, in this block — the miner's zero is a real absence, not a broken probe.

{% stepper %}
{% step %}

## Append the test module

Append the module below to `pallets/wormhole/src/tests.rs`.
{% endstep %}

{% step %}

## Run the test

```bash
cargo test -p pallet-wormhole --lib miner_fee_records_no_leaf -- --nocapture
```

Nothing else is required — no patch, no new dependency, no change to the pallet or its mock.
{% endstep %}
{% endstepper %}

### The test

```rust
#[cfg(test)]
mod wormhole_miner_fee_leaf {
	use crate::mock::*;
	use frame_support::assert_ok;
	use frame_system::RawOrigin;
	use qp_wormhole_verifier::{parse_private_batch_public_inputs, ProofWithPublicInputs, C, D, F};
	use sp_core::H256;

	const PRIVATE_BATCH_PROOF_HEX: &str = include_str!("../test-data/private_batch.hex");

	/// Leaves in the ZK tree whose recipient is `who`, as `(transfer_count, amount)`.
	/// Leaves are keyed on the canonical (Goldilocks-reduced) recipient, so the lookup
	/// canonicalizes too. This is the only representation the withdrawal circuit can
	/// spend: an exit mints fresh value rather than debiting a balance, so a leaf is
	/// the necessary and sufficient condition for a wormhole address's value to be
	/// reachable at all.
	fn leaves_for(who: &sp_core::crypto::AccountId32) -> Vec<(u64, u128)> {
		let key: [u8; 32] = pallet_zk_tree::tree::canonicalize_account_bytes(
			<[u8; 32]>::try_from(who.as_ref()).unwrap(),
		);
		pallet_zk_tree::Leaves::<Test>::iter()
			.filter(|(_, leaf)| <[u8; 32]>::try_from(leaf.to.as_ref()).unwrap() == key)
			.map(|(_, leaf)| (leaf.transfer_count, leaf.amount))
			.collect()
	}

	/// Drives the real `verify_private_batch` unsigned extrinsic with the repository's
	/// own proof fixture, seeding the block author through the PoW pre-runtime digest
	/// exactly as production does. Mirrors the project's existing
	/// `test_verify_private_batch_emits_miner_volume_fee_paid`, and additionally asserts
	/// the one thing that test never checks: whether the miner's credit produced a leaf.
	///
	/// The control is inside the same call — the exit credits DO receive leaves during
	/// this very extrinsic, so leaf recording demonstrably works on this code path and
	/// the miner's zero is a real absence rather than a broken probe.
	#[test]
	fn miner_fee_records_no_leaf() {
		new_test_ext().execute_with(|| {
			let proof_bytes = hex::decode(PRIVATE_BATCH_PROOF_HEX.trim()).expect("fixture hex");
			let verifier = crate::get_private_batch_verifier().expect("verifier");
			let proof = ProofWithPublicInputs::<F, C, D>::from_bytes(
				proof_bytes.clone(),
				&verifier.circuit_data.common,
			)
			.expect("fixture deserializes");
			let inputs = parse_private_batch_public_inputs(&proof).expect("parse");

			// Anchor the proof's referenced block, as the project's own test does.
			let block_number = inputs.block_data.block_number as u64;
			let block_hash_bytes: [u8; 32] =
				inputs.block_data.block_hash.as_ref().try_into().unwrap();
			frame_system::BlockHash::<Test>::insert(block_number, H256::from(block_hash_bytes));
			System::set_block_number(block_number + 10);

			// Seed the block author through the PoW pre-runtime digest — the production
			// path. The resulting address is `rehash_to_bytes(preimage)`, a Poseidon
			// image, so no private key corresponds to it and nothing can sign for it.
			let miner_preimage = [7u8; 32];
			set_miner_preimage_digest(miner_preimage);
			let author = sp_core::crypto::AccountId32::from(
				qp_wormhole::derive_wormhole_address(miner_preimage).expect("canonical limbs"),
			);

			let author_before = Balances::free_balance(author.clone());
			let leaves_before = pallet_zk_tree::Leaves::<Test>::iter().count();

			// The real entry point: the unsigned extrinsic, with a real proof.
			assert_ok!(Wormhole::verify_private_batch(RawOrigin::None.into(), proof_bytes));

			let author_after = Balances::free_balance(author.clone());
			let leaves_after = pallet_zk_tree::Leaves::<Test>::iter().count();
			let author_leaves = leaves_for(&author);

			println!("miner credited        : {}", author_after - author_before);
			println!("leaves created (call) : {}", leaves_after - leaves_before);
			println!("leaves owned by miner : {author_leaves:?}");

			// 1. The miner is paid.
			assert!(
				author_after > author_before,
				"the real extrinsic must credit the miner's volume-fee share"
			);

			// 2. Same-call control: leaf recording works on this path — the exit
			//    credits received leaves during this very extrinsic.
			assert!(
				leaves_after > leaves_before,
				"control: this call must record leaves for the exit credits"
			);

			// 3. The defect: the miner's credit produced none of them.
			assert!(
				author_leaves.is_empty(),
				"miner's volume-fee credit must have produced NO zk leaf"
			);
		});
	}
}
```

### Output

```
miner credited        : 5000000000
leaves created (call) : 1
leaves owned by miner : []
```

The extrinsic verified a real proof, settled it, wrote a leaf for one credit and omitted one for the other. Since the miner's address is a Poseidon image with no corresponding key, and a leaf is the only other way to move value out of such an address, that 5,000,000,000 planck is now unreachable.

### Scope of what this test executes

Calling the dispatchable directly exercises `ensure_none`, the real proof bytes, `pre_validate_private_batch_proof`, bundle construction and the full settlement path. It does not execute ZK verification, because that gate lives in `ValidateUnsigned::pre_dispatch` and is not invoked when the dispatchable is called directly — as the pallet's own comment at `:591-596` explains. That does not weaken the finding: the defect is in fee settlement, which runs *after* verification, so a genuine proof reaches exactly the same settlement code. The project's own test uses this identical entry point to assert the fee is paid.

### Optional additional check

If you wire `pallet_mining_rewards` into the wormhole mock, you can confirm the cause from the other direction: calling `MiningRewards::on_finalize` credits a block reward to the *same* keyless address and does produce a leaf, because `mint_reward` pairs `mint_into` with `record_transfer_proof`. The address type is not what blocks leaf creation — the missing `record_transfer` call is. This is not required to reproduce the finding; the same-call control above already establishes it.

## Recommended fix

Pair the credit with a leaf, as every other credit path in the codebase does:

```rust
// after the increase_balance at ~:955
Self::record_transfer(asset_id, &minting_account, &author, miner_fee);
```

This mirrors the exit credit at `:828`/`:854` and `mint_reward` in `pallet_mining_rewards`. The same change applies to the aggregator rebate at `:919` whenever the chosen address is wormhole-form.

Two supporting measures worth considering:

1. **Make the invariant enforceable instead of conventional.** Route every credit to a wormhole-form address through a single helper that performs the balance mutation and the leaf insertion together, so a future edit cannot separate them.
2. **Recover value already frozen.** Credits made before the fix have no leaf and remain unreachable. A runtime upgrade can enumerate the affected credits and either record the missing leaves or reassign the balances, and should do so as part of the fix.

## A note on the aggregator rebate

The rebate at `:919` is leaf-free in the same way, but its recipient comes from a proof public input that the aggregator chooses. An aggregator naming an ordinary account keeps spendable funds; one naming a wormhole-form address freezes them. Because the aggregator can avoid the loss and the miner cannot, this report claims only the miner-fee path. The rebate is included as a related observation, not a separate claim.

## Severity rationale

The frozen value is miner revenue, which maps most precisely to *permanent freezing of unclaimed yield* (High), and that is the row claimed.

I am deliberately not claiming a Critical row. The *permanent freezing of funds* rows do not fit: the classification separates "funds" from "unclaimed yield" explicitly, and the value here is miner revenue that was never claimable in the first place. The row requiring a hardfork is also wrong on its face, since a runtime upgrade both stops the ongoing loss and can recover the already-frozen balances.

Per-exit amounts are small — the measured example is 0.005 QUAN on a single exit. The claim is not that any one exit matters; it is that the loss is a fixed fraction of all exit volume, applies to every exit, is irreversible, and compounds without bound while remaining invisible in balances and events.

## Relationship to a previously disclosed issue

A prior audit reported that miner rewards were misdirected because the author was derived by decoding the digest preimage directly instead of hashing it, and without filtering on the PoW engine id, so the fee reached an account the miner did not control. That issue is **fixed at this commit**: the author is now hashed and the engine id filtered, and extraction succeeds.

This report concerns the post-fix code and is the opposite situation. Extraction now works, so the fee reaches the *correct* miner address — the very same address `pallet_mining_rewards` pays. The defect is that this one credit is not accompanied by a `record_transfer`, so no leaf is created for it. The distinction holds on every axis:

* **Root cause:** a missing `record_transfer` at `:955`, not a wrong derivation.
* **Recipient:** the correct, intended miner address, not a raw-preimage address.
* **Symptom:** value credited to the right place but unspendable and unexitable, not value sent to the wrong place.
* **Fix:** pair the credit with a leaf, not change the derivation.

The asymmetry makes this concrete: mining-rewards credits the same address and does pair its mint with a leaf, so the miner's block rewards at that address are exitable. Only the wormhole volume-fee share is not. The miner realizes their block reward and loses their fee share, from one and the same address.

## Limitations

The demonstration runs against the pallet's own mock runtime, which wires the real balances, zk-tree and wormhole pallets. As stated above, ZK verification itself is not executed because it lives in `pre_dispatch`; everything downstream of it, including all of fee settlement, is.

The magnitude figures follow from the fee constants at this commit and will change if those constants change. The mock's `VolumeFeesBurnRate` is 50%, matching the runtime. No attempt has been made to quantify historical loss on any live network.
