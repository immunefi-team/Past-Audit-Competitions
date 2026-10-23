# 89134 bc high miner volume fee credits lack zk tree leaves permanently freezing block author fees

**Submitted on Aug 17th 2026 at 12:13:42 UTC by @Kangaroo59250 for** [**Audit Comp | Quantus**](https://immunefi.com/audit-competition/audit-comp-quantus)

* **Report ID:** #89134
* **Report Type:** Blockchain/DLT
* **Report severity:** High
* **Target:** <https://github.com/immunefi-team/audit-comp-quantus-chain>
* **Impacts:**
  * Permanent freezing of funds

## Description

## Summary

Every successful wormhole settlement credits the block author with a share of the settlement’s volume fee. The credit is applied to the author’s Poseidon-derived wormhole address, but no corresponding transfer proof—or ZK-tree leaf—is recorded.

Unlike a normal signature-controlled account, the derived block-author address is not backed by a Dilithium signing key. Value credited to it can only be recovered through the wormhole exit mechanism, which requires a leaf committing to `(to, transfer_count, asset_id, amount)` in a block header’s `zk_tree_root`.

Because the miner volume-fee credit has no leaf, the miner cannot construct a valid membership witness for it. Every successfully credited miner volume fee is therefore permanently unwithdrawable through the current protocol.

## Vulnerability Details

`process_exit_bundle` in `pallets/wormhole/src/lib.rs` handles three related value flows inconsistently.

{% stepper %}
{% step %}

## Exit payouts

At `pallets/wormhole/src/lib.rs:818-860`, each successful exit payout is credited with:

```rust
<T::Currency as Unbalanced<_>>::increase_balance(...)
```

The pallet then calls:

```rust
Self::record_transfer(...)
```

This creates a ZK-tree leaf for the credited amount, allowing the recipient to use it as an input to a later wormhole exit.
{% endstep %}

{% step %}

## Aggregator rebate

At `pallets/wormhole/src/lib.rs:919-933`, the optional aggregator rebate is credited with `increase_balance`, but no leaf is recorded.

An ordinary signature-controlled aggregator account can spend the rebate normally. An aggregator that deliberately selects a wormhole address would encounter the same locking behavior, but this is not the primary issue described in this report.
{% endstep %}

{% step %}

## Miner volume fee

At `pallets/wormhole/src/lib.rs:944-983`, the miner’s volume-fee share is credited to the block author with:

```rust
<T::Currency as Unbalanced<_>>::increase_balance(
    &author,
    miner_fee,
    frame_support::traits::tokens::Precision::Exact,
)
```

On success, the pallet emits:

```rust
Event::MinerVolumeFeePaid {
    miner: author,
    amount: miner_fee,
}
```

However, it does not call `Self::record_transfer` or `record_transfer_proof`. The miner’s balance increases, but the ZK tree contains no leaf representing that credit.

The author is obtained through `qp_wormhole::extract_author_from_digest` in `primitives/wormhole/src/lib.rs:109-151`. It takes the QPoW pre-runtime digest preimage and derives:

```
derive_wormhole_address(preimage) = rehash_to_bytes(preimage)
```

This produces the Poseidon-derived wormhole address to which mining compensation is paid. The miner can use the corresponding wormhole secret to spend valid leaves, but cannot sign an ordinary transaction from the derived address.

`Self::record_transfer`, at `pallets/wormhole/src/lib.rs:1280-1330`, is the production path that increments the recipient’s transfer count and inserts the transfer into `pallet-zk-tree`. Since the miner fee path never reaches it, no membership witness can be generated for the credited fee.

This is inconsistent with `pallets/mining-rewards/src/lib.rs:220-298`. Successful mining-reward mints are immediately followed by:

```rust
T::ProofRecorder::record_transfer_proof(
    None,
    mint_account,
    miner.clone(),
    reward,
);
```

The mining-rewards tests explicitly reference the earlier Eiger audit finding `EQ-QNT-WORMHOLE-F-03: Miner Rewards Silently Burned Due to Incorrect Preimage Handling`. The existing mining-reward implementation records leaves specifically so rewards sent to miner wormhole addresses remain withdrawable. The same protection is missing from the volume-fee distribution path.
{% endstep %}
{% endstepper %}

## Impact Details

The defect affects every miner volume-fee credit for which `increase_balance` succeeds.

The runtime volume-fee rate is four basis points:

```
fee_quanta = ceil(exit_quanta × 4 / 9996)
```

`VolumeFeesBurnRate` is 50%, so the block author receives:

```
miner_fee = fee − floor(fee × 50%)
```

For sufficiently large settlements, the frozen miner share approaches approximately `0.020008%` of the wormhole exit volume. Small settlements can have a higher effective rate because the fee is rounded up to a complete `0.01 QUAN` circuit quantum.

The smallest fee is one quantum, or `0.01 QUAN`, making the minimum miner share `0.005 QUAN`. The runtime existential deposit is only `0.001 QUAN`, so even a fresh block-author account can be created by the minimum credit. Consequently, the issue is not limited to miners whose addresses already hold mining rewards.

The affected balance is visible on-chain and announced as paid through `MinerVolumeFeePaid`, but the miner cannot withdraw it through either available authorization mechanism:

* The Poseidon-derived address has no corresponding Dilithium signing key.
* The missing ZK-tree leaf prevents construction of a valid wormhole exit proof.

The loss recurs on every settlement with a valid block author and accumulates over the chain’s lifetime.

Recovering historical credits requires a runtime upgrade. Historical successful credits can be identified from `MinerVolumeFeePaid` events where archival or indexed chain history is available, after which the missing leaves can be backfilled. Already-credited balances must not be minted a second time during remediation.

## Recommendation

Every representable credit to a miner wormhole address must be accompanied by a ZK-tree leaf, atomically with the balance credit.

A direct per-settlement `record_transfer` call is insufficient because the minimum miner share is `0.005 QUAN`, while ZK-tree leaves quantize amounts to `0.01 QUAN`. Recording a sub-quantum amount would produce a leaf whose encoded amount is zero.

Implement per-miner accumulation before crediting the keyless address:

{% stepper %}
{% step %}

## Add the fee to a pending amount

Add the new fee to a pending amount associated with the miner.
{% endstep %}

{% step %}

## Calculate the whole-quantum portion

```
recordable = pending − (pending mod SCALE_DOWN_FACTOR)
remainder  = pending mod SCALE_DOWN_FACTOR
```

{% endstep %}

{% step %}

## Credit and record the whole-quantum amount

If `recordable > 0`, credit that amount and record a transfer proof for exactly the same amount in one atomic operation.
{% endstep %}

{% step %}

## Retain the remainder

Retain the sub-quantum remainder without reporting it as paid.
{% endstep %}

{% step %}

## Define a terminal-dust policy

Define an explicit terminal-dust policy for miners that receive no future fees. Completely preserving arbitrary sub-quantum value would require increasing the circuit’s amount precision; otherwise the protocol must document an appropriate rounding, burning, or reassignment rule.
{% endstep %}
{% endstepper %}

Update the pallet’s weights and benchmarks to include the additional ZK-tree insertion.

For historical successful credits, use `MinerVolumeFeePaid` history to aggregate missing amounts per miner and insert leaves for the already-credited, whole-quantum portions. Do not mint those amounts again. Historical sub-quantum remainders should be handled under the chosen migration policy.

Add regression coverage asserting that:

* A whole-quantum miner-fee payment advances the author’s transfer count and inserts a matching leaf.
* Multiple sub-quantum fees are accumulated into a withdrawable whole-quantum leaf.
* No zero-value leaf is inserted.
* Failed credits do not create leaves or emit `MinerVolumeFeePaid`.

## References

* `pallets/wormhole/src/lib.rs:818-860` — exit payouts are credited and recorded
* `pallets/wormhole/src/lib.rs:919-933` — aggregator rebate credit
* `pallets/wormhole/src/lib.rs:944-983` — miner volume-fee credit without proof recording
* `pallets/wormhole/src/lib.rs:1082-1093` — quantized volume-fee calculation
* `pallets/wormhole/src/lib.rs:1280-1330` — transfer-count update and ZK-tree insertion
* `pallets/mining-rewards/src/lib.rs:220-298` — mining rewards record transfer proofs
* `primitives/wormhole/src/lib.rs:97-151` — miner wormhole-address derivation
* `pallets/zk-tree/src/tree.rs:135-149` — leaf amount quantization
* `runtime/src/lib.rs:97-102` — token unit and existential deposit
* `runtime/src/configs/mod.rs:730-736` — volume-fee parameters
* Repository commit: `3b243b870a5442a6e5f443056f67bbba87d472f5`

## Proof of Concept

Add the following test to `pallets/wormhole/src/tests.rs`, immediately before `test_verify_private_batch_cannot_replay`:

```rust
/// PoC: the miner volume-fee share is credited to the block author's
/// keyless wormhole address, but unlike exit payouts and mining rewards no
/// ZK-tree leaf (transfer proof) is recorded for it. Funds at a wormhole
/// address can only be recovered through a ZK exit proof, which requires a
/// covering leaf, so the credited fee is permanently unexitable.
#[test]
fn poc_miner_volume_fee_minted_without_zk_leaf() {
    new_test_ext().execute_with(|| {
        let proof = deserialize_test_proof();
        let inputs =
            parse_private_batch_public_inputs(&proof).expect("Should parse");

        // Set block-hash state to match the fixture proof.
        let block_number = inputs.block_data.block_number as u64;
        let block_hash_bytes: [u8; 32] =
            inputs.block_data.block_hash.as_ref().try_into().unwrap();

        frame_system::BlockHash::<Test>::insert(
            block_number,
            H256::from(block_hash_bytes),
        );
        System::set_block_number(block_number + 10);

        // Derive the block author's wormhole address through the same path
        // used by QPoW authorship.
        let miner_preimage = [7u8; 32];
        set_miner_preimage_digest(miner_preimage);

        let author = sp_core::crypto::AccountId32::from(
            qp_wormhole::derive_wormhole_address(miner_preimage)
                .expect("test preimage limbs are canonical"),
        );

        let exit_quanta: u128 = inputs
            .account_data
            .iter()
            .filter(|account| account.summed_output_amount > 0)
            .map(|account| account.summed_output_amount as u128)
            .sum();

        let total_fee = super::ceil_volume_fee(
            exit_quanta,
            VolumeFeeRateBps::get() as u128,
        );
        let expected_miner_fee = total_fee - total_fee / 2;

        assert!(
            expected_miner_fee > 0,
            "fixture must yield a non-zero miner fee"
        );

        let leaves_before = ZkTree::leaf_count();
        let author_balance_before = Balances::free_balance(author.clone());

        assert_ok!(Wormhole::verify_private_batch(
            RawOrigin::None.into(),
            get_test_proof_bytes(),
        ));

        // Control: the miner fee was successfully credited.
        assert_eq!(
            Balances::free_balance(author.clone()),
            author_balance_before + expected_miner_fee,
            "the miner fee must be credited to the block author"
        );

        // No transfer count was allocated for the credit.
        assert_eq!(
            crate::TransferCount::<Test>::get(author.clone()),
            0u64,
            "a recorded miner-fee credit would advance the transfer count"
        );

        // None of the leaves inserted while processing the settlement
        // represents the miner-fee credit.
        let fee_leaf_exists =
            (leaves_before..ZkTree::leaf_count()).any(|index| {
                let leaf = ZkTree::leaf(index)
                    .expect("leaf below leaf_count must exist");

                leaf.to == author && leaf.amount == expected_miner_fee
            });

        assert!(
            !fee_leaf_exists,
            "no ZK-tree leaf covers the miner volume-fee credit"
        );
    });
}
```

Run:

```
cargo test -p pallet-wormhole --lib poc_miner_volume_fee_minted_without_zk_leaf
```

On commit `3b243b870a5442a6e5f443056f67bbba87d472f5`, the test passes:

```
running 1 test
test tests::private_batch_proof_tests::poc_miner_volume_fee_minted_without_zk_leaf ... ok

test result: ok. 1 passed; 0 failed
```

The test demonstrates all three required conditions:

1. The miner fee is successfully credited to the author’s balance.
2. The author’s transfer count does not advance.
3. No newly inserted ZK-tree leaf covers the credited amount.
