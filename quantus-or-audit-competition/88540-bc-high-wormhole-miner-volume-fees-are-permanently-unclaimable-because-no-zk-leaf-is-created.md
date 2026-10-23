# 88540 bc high wormhole miner volume fees are permanently unclaimable because no zk leaf is created

**Submitted on Aug 13th 2026 at 17:50:51 UTC by @CarlosMB for** [**Audit Comp | Quantus**](https://immunefi.com/audit-competition/audit-comp-quantus)

* **Report ID:** #88540
* **Report Type:** Blockchain/DLT
* **Report severity:** High
* **Target:** <https://github.com/immunefi-team/audit-comp-quantus-chain>
* **Impacts:**
  * Permanent freezing of unclaimed yield

## Description

## Summary

QPoW pays the block author to a **Wormhole-derived account**. Such an account is spent through the Wormhole ZK system: the miner's secret derives the account, but a withdrawal also requires a Merkle leaf committing to `(recipient, transfer_count, asset_id, amount)`.

Normal Wormhole credits explicitly call `record_transfer()` after crediting balance. Mining rewards similarly call `record_transfer_proof()`. The miner's share of Wormhole **volume fees does neither**:

```rust
<T::Currency as Unbalanced<_>>::increase_balance(
    &author,
    miner_fee,
    Precision::Exact,
)?;

Self::deposit_event(Event::MinerVolumeFeePaid {
    miner: author,
    amount: miner_fee,
});
```

Therefore a successful exit increases the miner's real balance without increasing the amount represented by Wormhole leaves. The reward is not claimable by the miner through any normal protocol path.

## Why it cannot be recovered by the miner

* **Normal signed transaction:** signed accounts are derived from an ML-DSA public key, whereas the miner reward account is independently derived from the QPoW Wormhole preimage. Knowing the Wormhole secret does not provide a signing key for that `AccountId`.
* **Wormhole withdrawal:** the circuit requires an actual Merkle leaf for the miner and binds the spend to that leaf's exact quantized `input_amount`. No leaf is created for `miner_fee`.
* **Event scanner:** `Unbalanced::increase_balance()` is event-free; `MinerVolumeFeePaid` is not one of the `Balances::{Transfer, Minted, TransferOnHold, ReserveRepatriated}` events recorded by `WormholeProofRecorderExtension`.
* **Later mining rewards:** `MiningRewards::mint_reward()` creates a leaf for exactly the new `reward`; it never snapshots or merges the account's pre-existing balance. A later reward therefore does not absorb the missing volume fee.
* **Genesis recording:** balances are converted into genesis leaves only at block 1; later volume fees are never reconciled from balances.
* **Recovery/proxy/multisig paths:** they require prior signed account setup/control and do not create ownership of an already-derived Wormhole miner account.

Root or a runtime/state migration could repair the state, but that is privileged intervention, not a claim/recovery path available to the miner.

## Impact

Production charges a **4 bps** Wormhole volume fee and allocates **50%** to the block author. For every successful exit with a non-zero miner share:

```
miner balance      += miner_fee
Wormhole spendable += 0
```

The miner's protocol reward becomes permanently unclaimable unless privileged state intervention occurs. This maps to **BC-H07 — Permanent freezing of unclaimed yield**.

The bug is not limited to rounding dust: the recording call is missing for **all** miner volume fees, including amounts equal to or larger than the Wormhole `0.01 QUAN` quantum.

## Proof of Concept

This PoC uses a **real Plonky2 private-batch proof**, the real Wormhole settlement path, real `Balances`, and the real ZK tree. It deliberately generates a proof whose miner fee is **exactly 0.01 QUAN**, so the result cannot be dismissed as sub-quantum dust.

{% stepper %}
{% step %}

## Generate a valid proof with a full-quantum miner fee

In `pallets/wormhole/src/tests.rs`, inside `fixture_gen::build_test_private_batch_proof`, change only:

```rust
let input_amount = 5000u32;
let output_amount = 4998u32;
```

At 4 bps:

```
fee = ceil(4998 * 4 / 9996) = 2 quanta = 0.02 QUAN
miner share (50%)           = 1 quantum = 0.01 QUAN
```

Regenerate the repository's valid private-batch fixture:

```bash
cargo test -p pallet-wormhole --lib regenerate_test_fixture \
  --release -- --ignored --nocapture
```

{% endstep %}

{% step %}

## Add the regression test

Add this inside `private_batch_proof_tests`:

```rust
#[test]
fn e2e_miner_volume_fee_is_real_balance_but_never_becomes_zk_spendable() {
    use frame_support::traits::fungible::Mutate;
    use qp_wormhole::TransferProofRecorder;

    fn represented_native_balance(who: &AccountId) -> u128 {
        pallet_zk_tree::Leaves::<Test>::iter()
            .filter(|(_, leaf)| leaf.to == *who && leaf.asset_id == 0)
            // The circuit commits to the quantized amount, not arbitrary raw dust.
            .map(|(_, leaf)| (leaf.amount / crate::SCALE_DOWN_FACTOR) * crate::SCALE_DOWN_FACTOR)
            .sum()
    }

    new_test_ext().execute_with(|| {
        let proof = deserialize_test_proof();
        let inputs = parse_private_batch_public_inputs(&proof).unwrap();

        let block_number = inputs.block_data.block_number as u64;
        let block_hash = H256::from(
            <[u8; 32]>::try_from(inputs.block_data.block_hash.as_ref()).unwrap()
        );
        frame_system::BlockHash::<Test>::insert(block_number, block_hash);
        System::set_block_number(block_number + 10);

        // Same QPoW -> Wormhole author derivation used in production.
        let miner_preimage = [7u8; 32];
        set_miner_preimage_digest(miner_preimage);
        let miner = AccountId::from(
            qp_wormhole::derive_wormhole_address(miner_preimage).unwrap()
        );

        let balance_before = Balances::free_balance(&miner);
        let transfer_count_before = crate::TransferCount::<Test>::get(&miner);
        let represented_before = represented_native_balance(&miner);

        // Real valid Plonky2 proof -> real Wormhole settlement.
        assert_ok!(Wormhole::verify_private_batch(
            RawOrigin::None.into(),
            get_test_proof_bytes(),
        ));

        let miner_fee = System::events()
            .iter()
            .find_map(|r| match &r.event {
                RuntimeEvent::Wormhole(
                    crate::Event::<Test>::MinerVolumeFeePaid { miner: who, amount }
                ) if who == &miner => Some(*amount),
                _ => None,
            })
            .expect("valid exit must pay miner fee");

        // This proof was generated so the miner share is one full Wormhole quantum.
        assert_eq!(miner_fee, crate::SCALE_DOWN_FACTOR);

        // BUG: real balance increased...
        assert_eq!(
            Balances::free_balance(&miner),
            balance_before + miner_fee
        );

        // ...but no miner transfer/leaf was created.
        assert_eq!(
            crate::TransferCount::<Test>::get(&miner),
            transfer_count_before
        );
        assert_eq!(
            represented_native_balance(&miner),
            represented_before
        );

        // Now reproduce the successful MiningRewards::mint_reward path:
        // a later legitimate reward DOES create a leaf, but only for itself.
        let later_reward = 10 * crate::SCALE_DOWN_FACTOR;
        <Balances as Mutate<AccountId>>::mint_into(&miner, later_reward).unwrap();
        assert!(
            <Wormhole as TransferProofRecorder<AccountId, u32, u128>>::record_transfer_proof(
                None,
                MINTING_ACCOUNT,
                miner.clone(),
                later_reward,
            )
        );

        assert_eq!(
            crate::TransferCount::<Test>::get(&miner),
            transfer_count_before + 1
        );

        let real_delta = Balances::free_balance(&miner) - balance_before;
        let represented_delta = represented_native_balance(&miner) - represented_before;

        assert_eq!(real_delta, miner_fee + later_reward);
        assert_eq!(represented_delta, later_reward);

        // The unrecoverable gap survives a later correctly-recorded reward exactly.
        assert_eq!(real_delta - represented_delta, miner_fee);
    });
}
```

{% endstep %}

{% step %}

## Run the regression test

```bash
cargo test -p pallet-wormhole \
  e2e_miner_volume_fee_is_real_balance_but_never_becomes_zk_spendable \
  --release -- --nocapture
```

A passing test proves that settlement of a valid generated Wormhole proof credits a full-quantum miner fee to the miner's real balance without creating corresponding ZK spend capacity. It also proves that a later correctly-recorded reward does not absorb the missing balance.
{% endstep %}
{% endstepper %}

## Recommended fix

Do not credit miner volume fees directly to the Wormhole account without proof recording. Accumulate them and settle them through a path that creates Wormhole leaves in representable (`0.01 QUAN`) quanta, or merge them into the normal mining-reward settlement before recording the miner's leaf.
