# 89197 bc high missing zk leaf recording permanently freezes miner fee balances credited to keyless wormhole accounts

**Submitted on Aug 17th 2026 at 19:04:32 UTC by @Pig46940 for** [**Audit Comp | Quantus**](https://immunefi.com/audit-competition/audit-comp-quantus)

* **Report ID:** #89197
* **Report Type:** Blockchain/DLT
* **Report severity:** High
* **Target:** <https://github.com/immunefi-team/audit-comp-quantus-chain>
* **Impacts:**
  * Permanent freezing of funds

## Description

### Brief/Intro

When Quantus settles a Wormhole private- or public-batch proof, it credits the block author's share of the volume fee directly to the author's keyless Wormhole account. The credit increases the miner's on-chain free balance and Quantus emits `MinerVolumeFeePaid`, but the settlement path does not insert the corresponding transfer leaf into the ZK tree. Because a Wormhole account has no ordinary signing key and can spend only by proving a recorded leaf, the credited miner-fee balance cannot be transferred or withdrawn. This happens during normal settlement and requires no malicious or privileged actor.

### Vulnerability Details

Every balance credit to a keyless Wormhole account must have a corresponding ZK-tree leaf committing the recipient, transfer count, asset ID, and amount. Without that leaf, the account may have a positive on-chain balance but has no authorization path for spending it.

#### Quantus explicitly documents the required credit-and-record sequence

The normal user-exit path credits an exit account with the event-free `Unbalanced::increase_balance` primitive and then explicitly calls `record_transfer`. The developers' comment explains why both operations are required:

* `pallets/wormhole/src/lib.rs`

```rust
				// NOTE: this must stay `Unbalanced::increase_balance` (event-free). The runtime's
				// `WormholeProofRecorderExtension` records a transfer proof for every
				// `Balances::Minted` event it scans, and this exit already records its own proof
				// via `record_transfer` below — switching to `mint_into` (which emits `Minted`)
				// could double-record the credit. Pinned by the test
				// `exit_credits_emit_no_scannable_transfer_events`.
				match <T::Currency as Unbalanced<_>>::increase_balance(
					exit_account,
					*exit_balance,
					frame_support::traits::tokens::Precision::Exact,
				) {
					Ok(_) => {},
```

* `pallets/wormhole/src/lib.rs`

```rust
				// Record transfer proof for the minted tokens
				let from_account: <T as Config>::WormholeAccountId = mint_account.clone().into();
				let to_account: <T as Config>::WormholeAccountId = exit_account.clone().into();
				Self::record_transfer(
					T::AssetId::default(),
					&from_account,
					&to_account,
					*exit_balance,
				);
```

This is not an inferred requirement. Quantus deliberately uses an event-free credit and pairs it with an explicit recorder call. Using `mint_into` would cause the runtime extension to record the leaf from `Balances::Minted`; using `increase_balance` requires the pallet to record the leaf itself.

#### The miner-fee path performs only half of the required sequence

The miner-fee path uses the same event-free balance operation, but its success branch emits only `MinerVolumeFeePaid`:

* `pallets/wormhole/src/lib.rs`

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

There is no `record_transfer` or `record_transfer_proof` call after the successful credit. Consequently:

1. The account's free balance increases.
2. Its `TransferCount` entry does not increase.
3. No leaf representing the miner fee is inserted into `pallet-zk-tree`.
4. Quantus nevertheless emits an event stating that the miner fee was paid.

#### The transaction extension cannot recover the omitted leaf

`WormholeProofRecorderExtension` scans events produced by an extrinsic and records only four balance-event variants:

* `runtime/src/transaction_extensions.rs`

```rust
match event_record.event {
    RuntimeEvent::Balances(pallet_balances::Event::Transfer { .. }) => { /* ... */ },
    RuntimeEvent::Balances(pallet_balances::Event::Minted { .. }) => { /* ... */ },
    RuntimeEvent::Balances(pallet_balances::Event::TransferOnHold { .. }) => { /* ... */ },
    RuntimeEvent::Balances(pallet_balances::Event::ReserveRepatriated { .. }) => { /* ... */ },
    _ => None, // Ignore all other events
}
```

The miner fee cannot match any of these branches:

* `increase_balance` is intentionally event-free, so it produces no `Balances::Minted` event.
* The only success event is `pallet_wormhole::Event::MinerVolumeFeePaid`.
* `MinerVolumeFeePaid` is not matched by the extension and therefore reaches `_ => None`.
* The miner-fee code contains no explicit fallback recorder call.

#### The credited destination is necessarily a keyless Wormhole account

`process_exit_bundle` obtains the author through `qp_wormhole::extract_author_from_digest`, which derives the payout address by re-hashing the miner-supplied digest, the same keyless construction used throughout Wormhole. No signing key corresponds to it; the only spending mechanism is a Wormhole proof built from the secret and a recorded ZK-tree leaf.

A later, unrelated deposit cannot repair this credit: each leaf commits its own amount and transfer count, so a proof can only withdraw the value a leaf actually represents. The positive balance alone is not spendable.

#### Fee quantization must also be considered by the fix

The production configuration charges a four-basis-point volume fee and burns 50%, leaving 50% for the miner. ZK leaves encode amounts in quanta of `10_000_000_000` planck. Although the total fee is rounded to a complete quantum, splitting it can create a sub-quantum miner payment:

```
total fee       = 10,000,000,000 planck
50% miner share =  5,000,000,000 planck
leaf quantum    = 10,000,000,000 planck
```

Therefore, merely adding `record_transfer(miner_fee)` is insufficient for every payment: a 5,000,000,000-planck leaf encodes an amount of zero. The remediation must both record miner-fee credits and preserve sub-quantum remainders, for example by accumulating them until a full leaf quantum is payable or by rounding before crediting while retaining the remainder for a later payment.

## Impact Details

Selected impact: **Critical — Permanent freezing of funds**.

The affected value is not merely a prospective reward or an entry in an unclaimed-yield accumulator. Quantus has already:

* incorporated the fee into the miner's ordinary `free_balance` through `increase_balance`;
* emitted `MinerVolumeFeePaid`, representing the payment as completed; and
* removed that amount from the fee settlement flow rather than retaining it for a future claim.

Nevertheless, the miner cannot use either available spending mechanism:

* An ordinary signed transfer is impossible because the destination is a keyless Wormhole address, not the miner's signing account.
* A Wormhole withdrawal is impossible because no leaf exists for the credited fee.

Thus, Quantus records the amount as an existing account balance while omitting the only authorization object capable of spending it. The entire successfully credited miner fee is frozen, not only the sub-quantum remainder. The loss accumulates whenever private- or public-batch settlement pays a nonzero fee to a block author.

There is no recovery action available to the affected miner. Restoring existing balances would require protocol-level intervention, such as a runtime migration that reconstructs valid leaves or moves the affected balances. The report does not claim that a hard fork is necessarily required; the permanent-freeze characterization refers to the absence of any holder-accessible recovery or spending path.

The finding does not depend on unmodified Substrate behavior, leaked credentials, governance participation, a malicious genesis block, or access to the trusted miner-to-node connection.

## References

* Developer comment documenting the event-free credit and mandatory explicit recorder pairing: <https://github.com/immunefi-team/audit-comp-quantus-chain/blob/3b243b870a5442a6e5f443056f67bbba87d472f5/pallets/wormhole/src/lib.rs#L816-L859>
* Miner-fee credit without a recorder call: <https://github.com/immunefi-team/audit-comp-quantus-chain/blob/3b243b870a5442a6e5f443056f67bbba87d472f5/pallets/wormhole/src/lib.rs#L937-L974>
* Exhaustive event match in `WormholeProofRecorderExtension`: <https://github.com/immunefi-team/audit-comp-quantus-chain/blob/3b243b870a5442a6e5f443056f67bbba87d472f5/runtime/src/transaction\\_extensions.rs#L331-L395>
* `record_transfer` increments `TransferCount` and inserts the ZK-tree leaf: <https://github.com/immunefi-team/audit-comp-quantus-chain/blob/3b243b870a5442a6e5f443056f67bbba87d472f5/pallets/wormhole/src/lib.rs#L1257-L1329>
* Wormhole author derivation from the QPoW digest: <https://github.com/immunefi-team/audit-comp-quantus-chain/blob/3b243b870a5442a6e5f443056f67bbba87d472f5/primitives/wormhole/src/lib.rs#L97-L145>
* Production fee configuration: <https://github.com/immunefi-team/audit-comp-quantus-chain/blob/3b243b870a5442a6e5f443056f67bbba87d472f5/runtime/src/configs/mod.rs#L728-L753>
* ZK-leaf amount quantization: <https://github.com/immunefi-team/audit-comp-quantus-chain/blob/3b243b870a5442a6e5f443056f67bbba87d472f5/pallets/zk-tree/src/tree.rs#L99-L151>

## Proof of Concept

{% stepper %}
{% step %}

### Insert the test

In `pallets/wormhole/src/tests.rs`, locate the end of `test_verify_private_batch_cannot_replay`. Insert the following function **after that test's closing brace and immediately before** the comment `/// Regenerate the test fixture when circuit parameters change`. This position is inside `private_batch_proof_tests` but outside every existing function.

```rust
#[test]
fn poc_miner_fee_balance_increases_without_zk_leaf() {
    new_test_ext().execute_with(|| {
        // Parse the repository's valid private-batch fixture and install the
        // historical block state required by the verifier.
        let proof = deserialize_test_proof();
        let inputs = parse_private_batch_public_inputs(&proof).expect("Should parse");

        let block_number = inputs.block_data.block_number as u64;
        let block_hash_bytes: [u8; 32] =
            inputs.block_data.block_hash.as_ref().try_into().unwrap();
        frame_system::BlockHash::<Test>::insert(
            block_number,
            H256::from(block_hash_bytes),
        );
        System::set_block_number(block_number + 10);

        // Simulate the QPoW pre-runtime digest. Quantus derives the block-author
        // destination as a keyless Wormhole account from this inner digest.
        let miner_preimage = [7u8; 32];
        set_miner_preimage_digest(miner_preimage);
        let miner = sp_core::crypto::AccountId32::from(
            qp_wormhole::derive_wormhole_address(miner_preimage)
                .expect("test preimage limbs are canonical"),
        );
        let miner_bytes: &[u8] =
            <sp_core::crypto::AccountId32 as AsRef<[u8]>>::as_ref(&miner);

        // Ensure no normal user exit can accidentally account for a leaf or
        // balance change belonging to the miner.
        assert!(
            inputs
                .account_data
                .iter()
                .filter(|entry| entry.summed_output_amount > 0)
                .all(|entry| entry.exit_account.as_ref() != miner_bytes),
            "fixture exit recipient must not equal the miner"
        );

        let expected_user_exit_leaves = inputs
            .account_data
            .iter()
            .filter(|entry| {
                entry.summed_output_amount > 0 &&
                    entry.exit_account.as_ref() != &[0u8; 32]
            })
            .count() as u64;
        assert!(expected_user_exit_leaves > 0, "fixture must contain a real exit");

        // Independently reproduce the mock runtime's configured 4-bps fee and
        // 50/50 burn/miner split.
        let exit_quanta: u128 = inputs
            .account_data
            .iter()
            .filter(|entry| entry.summed_output_amount > 0)
            .map(|entry| entry.summed_output_amount as u128)
            .sum();
        let total_fee =
            super::ceil_volume_fee(exit_quanta, VolumeFeeRateBps::get() as u128);
        let expected_miner_fee = total_fee - total_fee / 2;
        assert!(expected_miner_fee > 0, "fixture must produce a miner fee");

        let balance_before = Balances::free_balance(&miner);
        let transfer_count_before =
            crate::pallet::TransferCount::<Test>::get(&miner);
        let leaf_count_before = ZkTree::leaf_count();

        System::reset_events();
        assert_ok!(Wormhole::verify_private_batch(
            RawOrigin::None.into(),
            get_test_proof_bytes()
        ));

        // The protocol considers the payment completed: the free balance grows
        // and MinerVolumeFeePaid is emitted.
        assert_eq!(
            Balances::free_balance(&miner),
            balance_before + expected_miner_fee,
            "the miner fee was credited to free balance"
        );
        System::assert_has_event(
            crate::Event::<Test>::MinerVolumeFeePaid {
                miner: miner.clone(),
                amount: expected_miner_fee,
            }
            .into(),
        );

        // Broken invariant: the same credit creates neither the miner's transfer
        // record nor an additional ZK-tree leaf.
        assert_eq!(
            crate::pallet::TransferCount::<Test>::get(&miner),
            transfer_count_before,
            "the credited miner received no transfer record"
        );
        assert_eq!(
            ZkTree::leaf_count(),
            leaf_count_before + expected_user_exit_leaves,
            "the tree contains only user-exit leaves and no miner-fee leaf"
        );

        // increase_balance is event-free. The settlement emitted no Balances
        // event from which WormholeProofRecorderExtension could recover the leaf.
        assert!(
            System::events().iter().all(|record| !matches!(
                record.event,
                RuntimeEvent::Balances(
                    pallet_balances::Event::<Test>::Transfer { .. } |
                        pallet_balances::Event::<Test>::Minted { .. }
                )
            )),
            "settlement emitted no balance event the recorder extension could use"
        );
    });
}
```

{% endstep %}

{% step %}

### Run the PoC

Run only this PoC from the repository root:

```bash
cargo test -p pallet-wormhole \
  poc_miner_fee_balance_increases_without_zk_leaf \
  -- --nocapture
```

{% endstep %}

{% step %}

### Observe the results

The independent test passes and establishes all parts of the broken transition:

* `Balances::free_balance(miner)` increases by `expected_miner_fee`.
* Quantus emits `MinerVolumeFeePaid` for the same account and amount.
* `TransferCount[miner]` remains unchanged.
* The ZK-tree leaf count increases only for ordinary user exits.
* No additional leaf represents the miner-fee credit.
* No `Balances::Transfer` or `Balances::Minted` event exists for the extension to scan.

This reproduces the complete broken state transition in the production settlement path: Quantus finalizes payment into a keyless Wormhole account while omitting the only proof material that can authorize spending that balance.
{% endstep %}
{% endstepper %}
