# 89219 bc low missing merkle leaf record lead to permanent freezing of the miner volume fee

**Submitted on Aug 17th 2026 at 21:45:29 UTC by @Destiny0629 for** [**Audit Comp | Quantus**](https://immunefi.com/audit-competition/audit-comp-quantus)

* **Report ID:** #89219
* **Report Type:** Blockchain/DLT
* **Report severity:** Low
* **Target:** <https://github.com/immunefi-team/audit-comp-quantus-chain>
* **Impacts:**
  * Contract fails to deliver promised returns, but doesn't lose value

## Description

## Brief/Intro

Every wormhole exit pays the block author's miner-fee share to a keyless wormhole address through event-free `increase_balance` without recording a zk-tree leaf, permanently freezing that fee.

## Vulnerability Details

**Affected component.** `pallet-wormhole` fee distribution inside the exit handler (`pallets/wormhole/src/lib.rs:954-985`)

**Root cause.** Funds at a keyless wormhole address are spendable only through a ZK membership proof which requires a recorded leaf. The user-exit path mints and records a leaf (`increase_balance` at `lib.rs:838` + `record_transfer` at `:864`), and mining rewards do too (`:496`), but the miner-fee path mints through `increase_balance` at `lib.rs:965` with no `record_transfer` — so no leaf exists and the keyless recipient has no private key, leaving the credit permanently unspendable.

**Trigger.** Ordinary block production: every exit that pays a nonzero volume fee in a block authored by a miner freezes that miner's fee share. This is distinct from the earlier Report-D F-03 issue (the address is now correctly derived, yet the credit is still frozen for lack of a leaf — an incomplete fix).

## Impact Details

Permanent, systemic non-delivery of the miner's volume-fee share — credited to a keyless address with no zk-leaf, so it can never be withdrawn and is frozen forever.

## References

* `pallets/wormhole/src/lib.rs:954-985` / `:965` — miner-fee `increase_balance` with no `record_transfer`
* `pallets/wormhole/src/lib.rs:838` / `:864` — user-exit path: `increase_balance` + `record_transfer` (the correct pattern)
* `pallets/wormhole/src/lib.rs:496` — mining-reward path: mint + `record_transfer`

## Recommended Mitigation Steps

After minting the miner fee at `lib.rs:965`, record a zk-tree leaf for the author's wormhole address (mirroring the user-exit `record_transfer` at `:864`) so the miner can prove membership and withdraw it.

## Proof of Concept

Add inside `mod private_batch_proof_tests` in `pallets/wormhole/src/tests.rs` (crate `pallet-wormhole`).

Then run:

```bash
cargo test -p pallet-wormhole exploit_miner_fee_frozen_no_leaf_recorded -- --nocapture
```

```rust
#[test]
fn exploit_miner_fee_frozen_no_leaf_recorded() {
    new_test_ext().execute_with(|| {
        let proof = deserialize_test_proof();
        let inputs = parse_private_batch_public_inputs(&proof).expect("parse");
        let real_exits =
            inputs.account_data.iter().filter(|a| a.summed_output_amount > 0).count();

        setup_valid_block_state_for_test_proof();
        set_miner_preimage_digest([0xD0u8; 32]); // a miner authors this block
        let leaves_before = pallet_zk_tree::LeafCount::<Test>::get();

        // The real permissionless unsigned extrinsic (ZK-verified in pre_dispatch).
        assert_ok!(Wormhole::verify_private_batch(RawOrigin::None.into(), get_test_proof_bytes()));

        let leaves_added = (pallet_zk_tree::LeafCount::<Test>::get() - leaves_before) as usize;
        let miner_fee_paid = System::events().iter().any(|r| matches!(
            r.event,
            RuntimeEvent::Wormhole(crate::Event::<Test>::MinerVolumeFeePaid { .. })
        ));
        println!(
            "[F1] miner_fee_paid={}; leaves_added={} (== {} user exit(s); the miner fee added no leaf)",
            miner_fee_paid, leaves_added, real_exits
        );
        assert!(miner_fee_paid, "the miner was paid a nonzero volume fee");
        // The miner fee added no leaf: leaves grew only by the user-exit count, so the
        // keyless miner address can never build a membership proof to spend the fee.
        assert_eq!(
            leaves_added, real_exits,
            "only user exits record leaves; the miner-fee credit is leaf-less => frozen"
        );
    });
}
```

```
running 1 test
[F1] miner_fee_paid=true; leaves_added=1 (== 1 user exit(s); the miner fee added no leaf)
test tests::private_batch_proof_tests::exploit_miner_fee_frozen_no_leaf_recorded ... ok
test result: ok. 1 passed; 0 failed; 0 ignored; 0 measured; 87 filtered out; finished in 0.01s
```
