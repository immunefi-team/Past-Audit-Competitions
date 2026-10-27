# 88566 bc high pr 656 incomplete remediation bundle level rounding under settles per leaf wormhole fees

**Submitted on Aug 13th 2026 at 20:49:10 UTC by @CarlosMB for** [**Audit Comp | Quantus**](https://immunefi.com/audit-competition/audit-comp-quantus)

* **Report ID:** #88566
* **Report Type:** Blockchain/DLT
* **Report severity:** High
* **Target:** <https://github.com/immunefi-team/audit-comp-quantus-chain>
* **Impacts:**
  * Permanent freezing of unclaimed yield

## Description

## Summary

PR #656 changed Wormhole settlement to use quantized-ceiling semantics:

```
fee_quanta = ceil(exit_quanta * bps / (10000 - bps))
```

The remediation computes that ceiling once over the aggregate output of the exit bundle and states that this reproduces the minimum fee locked by the proofs.

That assumption is incorrect.

The Wormhole leaf circuit enforces the fee independently for every leaf:

```
(output_1 + output_2) * 10000
    <= input_amount * (10000 - volume_fee_bps)
```

At the production `4 bps` rate, a leaf with `2q` input can output at most `1q`, where `1q = 0.01 QUAN`:

```
1 * 10000 <= 2 * 9996    // valid
2 * 10000 <= 2 * 9996    // invalid
```

Therefore every such leaf is forced by the circuit to leave a full `1q` gap.

However, `process_exit_bundle()` first aggregates the successfully minted outputs and then calls `volume_fee_for_exit()` only once:

```rust
let total_exit_u128: u128 = minted_exit_amount.try_into()?;

let total_fee_u128 =
    Self::volume_fee_for_exit(
        total_exit_u128,
        T::VolumeFeeRateBps::get(),
    );
```

The proofs therefore enforce the equivalent of:

```
Σ ceil(per_leaf_fee)
```

while the runtime settles:

```
ceil(Σ outputs * fee_rate)
```

These are not equivalent because:

```
ceil(Σx) < Σceil(x)
```

can hold strictly.

As a result, valid proofs can permanently consume value as a protocol-enforced fee gap that is never included in burn, miner, or aggregator settlement.

This is an **incomplete remediation of PR #656**, not a claim that bundle-level rounding itself is undocumented.

## Root cause

The leaf circuit applies its balance/fee inequality separately to each consumed input.

The runtime instead derives one fee from the total successfully minted output:

```rust
let total_fee_u128 =
    Self::volume_fee_for_exit(total_exit_u128, T::VolumeFeeRateBps::get());
```

`volume_fee_for_exit()` performs only one ceiling:

```rust
(minted_exit / SCALE_DOWN_FACTOR)
    .saturating_mul(fee_bps)
    .div_ceil(10_000 - fee_bps)
    .saturating_mul(SCALE_DOWN_FACTOR)
```

PR #656 introduced this calculation to fix a previous bug where proof-locked fee value silently disappeared from burn/miner/aggregator allocation.

The remediation relies on:

```
ceil(Σx) <= Σceil(x)
```

but that only proves that bundle settlement does not exceed the amount locked across individual proofs. It does **not** prove that the complete proof-side minimum is settled.

The E2E below demonstrates a strict inequality.

## Impact

Consider seven valid Wormhole leaves:

```
input_amount     = 2q
output_amount_1  = 1q
output_amount_2  = 0
volume_fee_bps   = 4
```

For every leaf, the ZK circuit prevents the user from outputting the complete `2q`; `1q` is the maximum valid output.

For one 7-leaf private batch:

```
Total consumed input       = 14q
Total output               =  7q
Circuit-forced gap         =  7q

Runtime settlement:
ceil(7 * 4 / 9996)         =  1q

Unsettled forced gap       =  6q
                            = 0.06 QUAN
```

This is not voluntary user underpayment or arbitrary change left behind: the circuit itself makes a `2q -> 2q` exit invalid.

After successful settlement, all seven nullifiers are consumed. The `6q` difference is:

* not returned through an exit output;
* not included in the runtime fee;
* not burned;
* not allocated to the miner;
* not allocated to the public-batch aggregator; and
* not recoverable by spending the original leaves again.

The audited production build uses:

```
NUM_LEAF_PROOFS = 7
NUM_PRIVATE_BATCH_PROOFS = 53
```

and the audited runtime exposes `verify_public_batch()`, which ultimately uses the same `process_exit_bundle()` settlement path.

A full public batch can therefore contain `53 * 7 = 371` underlying leaves. For the same valid minimum-size pattern:

```
Circuit-forced gap = 371q = 3.71 QUAN

Runtime settlement:
ceil(371 * 4 / 9996)
= 1q
= 0.01 QUAN

Unsettled difference:
370q
= 3.70 QUAN
```

The public-batch figure is the scaled consequence of the same root cause demonstrated directly by the private-batch E2E below.

## Why this is an incomplete remediation of PR #656

The existing bundle-level ceiling is publicly documented; this report does not rely on it being unexpected.

The issue is that PR #656 states that the new calculation reproduces the proof-side minimum while using the relation:

```
ceil(Σx) <= Σceil(x)
```

The E2E proves that the inequality can be strict:

```
Σ per-leaf minimum = 7q
bundle-level fee   = 1q
```

Therefore the previous shortfall class remains reachable for multiple small leaves: value that the ZK constraints force to remain outside the outputs is still omitted from fee settlement.

This also conflicts with the documented 4-bps fee policy for small exits, where the circuit floor is `0.01 QUAN` per nonzero small exit / pre-split chunk. Aggregating several such leaves should not make the individually enforced fee gaps disappear from settlement.

## Proof of Concept

## Runnable E2E PoC

The following test creates **7 real Wormhole leaves**, generates **7 real Plonky2 leaf proofs**, aggregates them into a private batch, passes both `validate_unsigned()` and `pre_dispatch()`, settles the batch through `verify_private_batch()`, and confirms that:

```
proof-enforced gap = 7q
runtime settlement = 1q
stranded residual  = 6q = 0.06 QUAN
7/7 nullifiers     = consumed
```

Add the test to:

```
pallets/wormhole/src/tests.rs
```

```rust
#[test]
fn e2e_multi_leaf_rounding_under_settles_six_quanta() {
    use crate::{mock::*, pallet::UsedNullifiers};
    use frame_support::{
        assert_ok,
        traits::fungible::{Inspect, Mutate},
    };
    use frame_system::RawOrigin;
    use qp_wormhole_aggregator::private_batch::prover::PrivateBatchProver;
    use qp_wormhole_circuit::{
        block_header::header::HeaderInputs,
        inputs::{CircuitInputs, PrivateCircuitInputs},
        nullifier::Nullifier,
        unspendable_account::UnspendableAccount,
        zk_merkle_proof::{ZkLeafData, ZkMerkleProofData},
    };
    use qp_wormhole_inputs::{BytesDigest, PublicCircuitInputs};
    use qp_wormhole_verifier::{
        parse_private_batch_public_inputs, ProofWithPublicInputs, C, F,
    };
    use qp_zk_circuits_common::{
        serialization::digest_to_bytes as felts_to_bytes,
        utils::digest_to_bytes,
    };
    use sp_core::{crypto::AccountId32, H256};
    use sp_runtime::{
        traits::ValidateUnsigned,
        transaction_validity::TransactionSource,
    };

    const D: usize = 2;
    const N: usize = 7;
    const INPUT_Q: u32 = 2;
    const OUTPUT_Q: u32 = 1;
    const FEE_BPS: u32 = 4;

    assert_eq!(crate::circuit_config::NUM_LEAF_PROOFS, N);
    assert_eq!(VolumeFeeRateBps::get(), FEE_BPS);

    let bins_dir = std::env::temp_dir().join("quantus-rounding-e2e-bins");
    let _ = std::fs::remove_dir_all(&bins_dir);
    std::fs::create_dir_all(&bins_dir).unwrap();

    qp_wormhole_circuit_builder::generate_all_circuit_binaries(
        &bins_dir, true, N, None,
    )
    .unwrap();

    new_test_ext().execute_with(|| {
        System::set_block_number(1);

        let quantum = crate::SCALE_DOWN_FACTOR;
        let input_base = INPUT_Q as u128 * quantum;
        let output_base = OUTPUT_Q as u128 * quantum;

        let block_number = 1u32;
        let parent_hash = [0u8; 32];
        let state_root: [u8; 32] = [
            0x7d,0x5f,0x04,0x3e,0x06,0x8b,0xe9,0x69,
            0x1e,0xfb,0xc3,0xc1,0xd4,0x98,0x78,0x8b,
            0x5d,0xc5,0xc7,0xd6,0x5f,0x41,0xc0,0xe2,
            0x4e,0x22,0x11,0xc3,0x99,0x7c,0x08,0x11,
        ];
        let extrinsics_root = [0u8; 32];
        let digest: [u8; 110] = [
            8,6,112,111,119,95,128,233,182,183,107,158,1,115,
            19,219,126,253,86,30,208,176,70,21,45,180,229,9,
            62,91,4,6,53,245,52,48,38,123,225,5,112,111,119,
            95,1,1,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,
            0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,
            0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,
            0,0,0,0,0,0,18,79,226,
        ];

        let funding_account = account_id(999);
        let mut secrets = Vec::new();
        let mut unspendable_accounts = Vec::new();

        // Create seven real 2q Wormhole leaves.
        for i in 0..N {
            let secret = BytesDigest::new_unchecked([42 + i as u8; 32]);
            let ua_digest = UnspendableAccount::from_secret(secret).account_id;
            let ua_bytes = digest_to_bytes(ua_digest);
            let ua = AccountId32::new(*ua_bytes);

            assert_ok!(Balances::mint_into(&ua, input_base));
            Wormhole::record_transfer(0, &funding_account, &ua, input_base);

            secrets.push(secret);
            unspendable_accounts.push(ua);
        }

        let zk_tree_root = ZkTree::root();

        let header_inputs = HeaderInputs::new(
            BytesDigest::new_unchecked(parent_hash),
            block_number,
            BytesDigest::new_unchecked(state_root),
            BytesDigest::new_unchecked(extrinsics_root),
            BytesDigest::new_unchecked(zk_tree_root),
            &digest,
        )
        .unwrap();

        let block_hash = header_inputs.block_hash();

        let mut leaf_proofs = Vec::with_capacity(N);
        let mut exit_accounts = Vec::with_capacity(N);

        for i in 0..N {
            let leaf = ZkTree::leaf(i as u64).unwrap();
            let ua_bytes: [u8; 32] = *unspendable_accounts[i].as_ref();

            let merkle = ZkTree::get_merkle_proof(i as u64).unwrap();
            let leaf_hash = pallet_zk_tree::tree::hash_leaf::<Test>(&leaf);

            let path = ZkMerkleProofData::from_unsorted(
                zk_tree_root,
                merkle.siblings.clone(),
                leaf_hash,
                ZkLeafData::new(
                    ua_bytes, 0, 0, INPUT_Q, OUTPUT_Q, 0, FEE_BPS,
                ),
                true,
            )
            .unwrap();

            let sorted_siblings = path
                .siblings
                .iter()
                .map(|level| [
                    felts_to_bytes(&level[0]),
                    felts_to_bytes(&level[1]),
                    felts_to_bytes(&level[2]),
                ])
                .collect();

            let nullifier =
                digest_to_bytes(Nullifier::from_preimage(secrets[i], 0).hash);

            let exit = BytesDigest::new_unchecked([0x80 + i as u8; 32]);
            let exit_bytes: [u8; 32] = *exit;
            exit_accounts.push(AccountId32::new(exit_bytes));

            let inputs = CircuitInputs {
                public: PublicCircuitInputs {
                    asset_id: 0,
                    output_amount_1: OUTPUT_Q,
                    output_amount_2: 0,
                    volume_fee_bps: FEE_BPS,
                    nullifier,
                    exit_account_1: exit,
                    exit_account_2: BytesDigest::default(),
                    block_hash,
                    block_number,
                },
                private: PrivateCircuitInputs {
                    secret: secrets[i].into(),
                    transfer_count: 0,
                    unspendable_account: BytesDigest::new_unchecked(ua_bytes),
                    parent_hash: BytesDigest::new_unchecked(parent_hash),
                    state_root: BytesDigest::new_unchecked(state_root),
                    extrinsics_root: BytesDigest::new_unchecked(extrinsics_root),
                    digest,
                    input_amount: INPUT_Q,
                    zk_tree_root,
                    zk_merkle_siblings: sorted_siblings,
                    zk_merkle_positions: path.positions,
                },
            };

            leaf_proofs.push(
                qp_wormhole_prover::build_fresh()
                    .commit(&inputs).unwrap()
                    .prove().unwrap()
            );
        }

        let batch_prover =
            PrivateBatchProver::new_from_binaries_dir(&bins_dir).unwrap();

        let batch = batch_prover.aggregate(leaf_proofs).unwrap();
        let proof_bytes = batch.to_bytes();

        let verifier = crate::get_private_batch_verifier().unwrap();
        let parsed = ProofWithPublicInputs::<F, C, D>::from_bytes(
            proof_bytes.clone(),
            &verifier.circuit_data.common,
        )
        .unwrap();

        let public = parse_private_batch_public_inputs(&parsed).unwrap();

        assert_eq!(public.nullifiers.len(), N);
        assert!(public.nullifiers.iter().all(|n| {
            let bytes: &[u8; 32] = n;
            bytes.iter().any(|b| *b != 0)
        }));

        let exit_quanta: u128 = public
            .account_data
            .iter()
            .map(|a| a.summed_output_amount as u128)
            .sum();

        assert_eq!(exit_quanta, 7);

        let block_hash_bytes: [u8; 32] =
            block_hash.as_ref().try_into().unwrap();

        frame_system::BlockHash::<Test>::insert(
            block_number as u64,
            H256::from(block_hash_bytes),
        );
        System::set_block_number(block_number as u64 + 10);

        let call = crate::Call::<Test>::verify_private_batch {
            proof_bytes: proof_bytes.clone(),
        };

        // Real pool and cryptographic inclusion paths.
        assert_ok!(<Wormhole as ValidateUnsigned>::validate_unsigned(
            TransactionSource::External,
            &call,
        ));
        assert_ok!(<Wormhole as ValidateUnsigned>::pre_dispatch(&call));

        let proof_forced_fee =
            N as u128 * (INPUT_Q - OUTPUT_Q) as u128 * quantum;

        let runtime_fee =
            Wormhole::volume_fee_for_exit(exit_quanta * quantum, FEE_BPS);

        assert_eq!(proof_forced_fee, 7 * quantum);
        assert_eq!(runtime_fee, quantum);
        assert_eq!(proof_forced_fee - runtime_fee, 6 * quantum);

        let issuance_before = Balances::total_issuance();

        // No author: all runtime-recognized fee is burned.
        assert_ok!(Wormhole::verify_private_batch(
            RawOrigin::None.into(),
            proof_bytes,
        ));

        let issuance_after = Balances::total_issuance();

        let total_exit_balance: u128 = exit_accounts
            .iter()
            .map(|a| Balances::free_balance(a.clone()))
            .sum();

        assert_eq!(total_exit_balance, 7 * output_base);

        // Only 1q was actually settled.
        assert_eq!(issuance_before - issuance_after, quantum);

        // Nevertheless all 7 inputs are consumed.
        for nullifier in &public.nullifiers {
            let bytes: [u8; 32] = nullifier.as_ref().try_into().unwrap();
            assert!(UsedNullifiers::<Test>::contains_key(bytes));
        }

        // The original leaves cannot be reclaimed.
        let replay = crate::Call::<Test>::verify_private_batch {
            proof_bytes: batch.to_bytes(),
        };
        assert!(<Wormhole as ValidateUnsigned>::pre_dispatch(&replay).is_err());

        assert_eq!(
            proof_forced_fee - (issuance_before - issuance_after),
            6 * quantum
        );
    });

    let _ = std::fs::remove_dir_all(&bins_dir);
}
```

Run:

```bash
SKIP_WASM_BUILD=1 cargo test -p pallet-wormhole \
  e2e_multi_leaf_rounding_under_settles_six_quanta \
  --release -- --nocapture
```

Observed:

```
test ...e2e_multi_leaf_rounding_under_settles_six_quanta ... ok
1 passed; 0 failed; 71 filtered out
```

This E2E proves the root cause on a real private batch. The public-batch impact follows from the same settlement function and the production `53 × 7` aggregation configuration:

```
371 leaves
× 1q proof-enforced fee each
= 371q = 3.71 QUAN

runtime settlement:
ceil(371 * 4 / 9996)
= 1q = 0.01 QUAN

stranded:
370q = 3.70 QUAN
```

## Recommended fix

The runtime and circuit must use the same rounding boundary.

One option is to authenticate and aggregate the fee contribution enforced for each real leaf and expose that total to settlement, so the runtime settles:

```
total_fee =
    Σ ceil(output_i * fee_bps / (10000 - fee_bps))
```

instead of:

```
total_fee =
    ceil(Σ output_i * fee_bps / (10000 - fee_bps))
```

Alternatively, change the ZK construction so that the fee constraint itself is enforced once over exactly the same aggregate value used by the runtime.

The critical requirement is that proof enforcement and runtime settlement use identical aggregation/rounding semantics.
