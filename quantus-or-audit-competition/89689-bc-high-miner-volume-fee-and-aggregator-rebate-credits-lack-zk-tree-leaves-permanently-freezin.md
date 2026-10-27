# 89689 bc high miner volume fee and aggregator rebate credits lack zk tree leaves permanently freezing funds

**Submitted on Aug 20th 2026 at 15:40:27 UTC by @abarbatei for** [**Audit Comp | Quantus**](https://immunefi.com/audit-competition/audit-comp-quantus)

* **Report ID:** #89689
* **Report Type:** Blockchain/DLT
* **Report severity:** High
* **Target:** <https://github.com/immunefi-team/audit-comp-quantus-chain>
* **Impacts:**
  * Permanent freezing of funds

## Description

## Brief/Intro

When a wormhole exit bundle is settled on-chain, the volume fee owed to the block-producing miner and the rebate owed to the aggregator are credited via `Unbalanced::increase_balance`, but no ZK-tree leaf is created for either credit. Because miner addresses are always wormhole-derived (keyless Poseidon2 hashes with no corresponding private key), and aggregator addresses may also be wormhole-derived, the credited funds can never be spent: the ZK exit circuit requires a Merkle proof rooted in a leaf that does not exist, and no private key exists to sign a conventional extrinsic. This results in permanent freezing of protocol fees that accumulates with every wormhole exit settlement.

## Vulnerability Details

The wormhole pallet's `Pallet::process_exit_bundle` function handles three categories of on-chain credit during exit settlement:

1. **Exit credits to users** (correct pattern)
2. **Miner volume-fee share** (missing leaf)
3. **Aggregator rebate** (missing leaf)

### Correct pattern: exit credits

User exit credits correctly follow a two-step process: `increase_balance` to credit the account, then `Pallet::record_transfer` to insert a leaf into the ZK Merkle tree.

[`lib.rs::process_exit_bundle` lines 828-859](https://github.com/immunefi-team/audit-comp-quantus-chain/blob/3b243b870a5442a6e5f443056f67bbba87d472f5/pallets/wormhole/src/lib.rs#L828-L859)

```rust
match <T::Currency as Unbalanced<_>>::increase_balance(
    exit_account,
    *exit_balance,
    frame_support::traits::tokens::Precision::Exact,
) {
    Ok(_) => {},
    Err(e) => {
        // ... error handling, skip this exit ...
        continue;
    },
}

minted_exit_amount = minted_exit_amount.saturating_add(*exit_balance);

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

### Bug: miner volume-fee credit omits record\_transfer

The miner's share of the volume fee is credited via `increase_balance` but no `Pallet::record_transfer` call follows. Only a `MinerVolumeFeePaid` event is emitted.

[`lib.rs::process_exit_bundle` lines 955-973](https://github.com/immunefi-team/audit-comp-quantus-chain/blob/3b243b870a5442a6e5f443056f67bbba87d472f5/pallets/wormhole/src/lib.rs#L955-L973)

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

### Bug: aggregator rebate credit omits record\_transfer

The aggregator rebate suffers the same omission.

[`lib.rs::process_exit_bundle` lines 919-933](https://github.com/immunefi-team/audit-comp-quantus-chain/blob/3b243b870a5442a6e5f443056f67bbba87d472f5/pallets/wormhole/src/lib.rs#L919-L933)

```rust
match <T::Currency as Unbalanced<_>>::increase_balance(
    &aggregator_account,
    aggregator_fee,
    frame_support::traits::tokens::Precision::Exact,
) {
    Ok(_) => {},
    Err(e) => {
        log::warn!(
            "Aggregator rebate of {:?} could not be minted ({:?}); burning it instead",
            aggregator_fee,
            e
        );
        burn_amount_u128 = burn_amount_u128.saturating_add(aggregator_fee_u128);
    },
}
```

### Why there is no alternative spend path for miner addresses

The miner (block author) address is always derived via `qp_wormhole::extract_author_from_digest`, which calls `derive_wormhole_address(preimage)`.

[`lib.rs::extract_author_from_digest` lines 122-151](https://github.com/immunefi-team/audit-comp-quantus-chain/blob/3b243b870a5442a6e5f443056f67bbba87d472f5/primitives/wormhole/src/lib.rs#L122-L151)

```rust
pub fn extract_author_from_digest<'a, AccountId, I>(digest: I) -> Option<AccountId>
where
    AccountId: Decode,
    I: IntoIterator<Item = &'a DigestItem>,
{
    // ...
    if data.len() != 32 {
        return None;
    }
    let preimage: [u8; 32] = data.as_slice().try_into().ok()?;
    let address_bytes = derive_wormhole_address(preimage).ok()?;
    return AccountId::decode(&mut &address_bytes[..]).ok();
}
```

[`lib.rs::derive_wormhole_address` lines 109-111](https://github.com/immunefi-team/audit-comp-quantus-chain/blob/3b243b870a5442a6e5f443056f67bbba87d472f5/primitives/wormhole/src/lib.rs#L109-L111)

```rust
pub fn derive_wormhole_address(inner_digest: [u8; 32]) -> Result<[u8; 32], &'static str> {
    rehash_to_bytes(&inner_digest)
}
```

The address is a Poseidon2 hash output. No private key corresponds to this address. This means:

* **Signed extrinsics:** Impossible. No private key exists to produce a valid signature.
* **ZK exit proof:** Impossible. The ZK circuit requires a Merkle proof against a leaf in the ZK tree, but no leaf was ever created for these credits.
* **Proxy or recovery pallets:** Require initial setup from the target account, which itself requires a signature.

### Contrast with mining block rewards (correct pattern)

The `pallet-mining-rewards` correctly records a transfer proof after minting block rewards to the same wormhole-derived miner address:

[`lib.rs::mint_reward` lines 230-237](https://github.com/immunefi-team/audit-comp-quantus-chain/blob/3b243b870a5442a6e5f443056f67bbba87d472f5/pallets/mining-rewards/src/lib.rs#L230-L237)

```rust
match T::Currency::mint_into(miner, reward) {
    Ok(_) => {
        T::ProofRecorder::record_transfer_proof(
            None, // Native token
            mint_account,
            miner.clone(),
            reward,
        );
        // ...
    },
```

For reference, `Pallet::record_transfer` creates the ZK-tree leaf that enables future withdrawal via ZK exit proofs.

### Fee parameters

The volume fee parameters are configured in the runtime:

[`mod.rs` lines 728-738](https://github.com/immunefi-team/audit-comp-quantus-chain/blob/3b243b870a5442a6e5f443056f67bbba87d472f5/runtime/src/configs/mod.rs#L728-L738)

```rust
pub const VolumeFeeRateBps: u32 = 4;
pub const VolumeFeesBurnRate: Permill = Permill::from_percent(50);
pub const VolumeFeesAggregatorRate: Permill = Permill::from_percent(50);
```

## Impact Details

**Permanent freezing of miner volume-fee revenue.** Every wormhole exit settlement permanently freezes the miner's share of the volume fee. The frozen amount per settlement equals 0.02% of exit volume (4 bps fee rate, 50% burn rate, remaining 50% to miner).

This is not a one-time event. It occurs on every single wormhole exit bundle settlement during normal protocol operation, with no attacker action required. The frozen amount accumulates indefinitely over the lifetime of the chain.

For public-batch exits, the aggregator rebate (50% of the burn portion, i.e., 0.01% of exit volume) is also frozen if the aggregator uses a wormhole-derived address.

The impact is classified as permanent freezing of funds:

* Funds are credited to the miner's on-chain balance (visible via state queries).
* No mechanism exists to move the funds: no private key (Poseidon2-derived address), no ZK leaf (omitted `record_transfer`), no governance override in scope.
* The loss compounds with every exit settlement, growing without bound.
* While individual amounts are small (protocol fee fractions), the cumulative effect degrades miner incentive alignment over the chain's lifetime.

## Mitigation

Add `Pallet::record_transfer` calls after each `increase_balance` for the miner volume fee and the aggregator rebate, matching the pattern used for exit credits.

For the miner volume-fee credit (after line 964):

```rust
Ok(_) => {
    // Record transfer proof so the miner can exit via ZK proof
    let from_account: <T as Config>::WormholeAccountId = mint_account.clone().into();
    let to_account: <T as Config>::WormholeAccountId = author.clone().into();
    Self::record_transfer(
        T::AssetId::default(),
        &from_account,
        &to_account,
        miner_fee,
    );
    Self::deposit_event(Event::MinerVolumeFeePaid {
        miner: author,
        amount: miner_fee,
    });
},
```

For the aggregator rebate credit (after line 923):

```rust
Ok(_) => {
    // Record transfer proof so the aggregator can exit via ZK proof
    let from_account: <T as Config>::WormholeAccountId = mint_account.clone().into();
    let to_account: <T as Config>::WormholeAccountId = aggregator_account.clone().into();
    Self::record_transfer(
        T::AssetId::default(),
        &from_account,
        &to_account,
        aggregator_fee,
    );
},
```

> The `mint_account` variable is already available in scope at both locations, having been initialized at line 742.

## References

* [`Pallet::process_exit_bundle`](https://github.com/immunefi-team/audit-comp-quantus-chain/blob/3b243b870a5442a6e5f443056f67bbba87d472f5/pallets/wormhole/src/lib.rs#L737-L983): full settlement logic
* [`Pallet::record_transfer`](https://github.com/immunefi-team/audit-comp-quantus-chain/blob/3b243b870a5442a6e5f443056f67bbba87d472f5/pallets/wormhole/src/lib.rs#L1280-L1310): ZK-tree leaf insertion
* [`qp_wormhole::extract_author_from_digest`](https://github.com/immunefi-team/audit-comp-quantus-chain/blob/3b243b870a5442a6e5f443056f67bbba87d472f5/primitives/wormhole/src/lib.rs#L122-L151): miner address derivation
* [`qp_wormhole::derive_wormhole_address`](https://github.com/immunefi-team/audit-comp-quantus-chain/blob/3b243b870a5442a6e5f443056f67bbba87d472f5/primitives/wormhole/src/lib.rs#L109-L111): Poseidon2 hash address derivation
* [`MiningRewards::mint_reward`](https://github.com/immunefi-team/audit-comp-quantus-chain/blob/3b243b870a5442a6e5f443056f67bbba87d472f5/pallets/mining-rewards/src/lib.rs#L220-L237): correct pattern with `record_transfer_proof`
* [Runtime fee parameters](https://github.com/immunefi-team/audit-comp-quantus-chain/blob/3b243b870a5442a6e5f443056f67bbba87d472f5/runtime/src/configs/mod.rs#L728-L738): `VolumeFeeRateBps`, `VolumeFeesBurnRate`, `VolumeFeesAggregatorRate`
* Pinned commit: [3b243b8](https://github.com/immunefi-team/audit-comp-quantus-chain/commit/3b243b870a5442a6e5f443056f67bbba87d472f5)

## Proof of Concept

### Overview

This is an integration test that exercises the full `verify_private_batch` extrinsic dispatch with a real pre-computed ZK proof fixture (`test-data/private_batch.hex`). The proof undergoes real plonky2 deserialization, block-state validation, nullifier checks, exit minting, and fee distribution — the complete on-chain settlement path.

The test demonstrates both the defect and the impact:

* **Part A (defect):** After full dispatch, the miner is credited via `increase_balance` but no `record_transfer` is called — no ZK-tree leaf, no `NativeTransferred` event, no `TransferCount` increment.
* **Part B (impact):** The ZK tree is walked to confirm no leaf exists for the miner's volume-fee credit. The miner address is a Poseidon2 hash with no private key, so without a ZK leaf there is no spend path — funds are permanently frozen.

{% stepper %}
{% step %}

### Setup

Save the test file as `pallets/wormhole/src/proof_vol_fee_no_leaf.rs` in the `audit-comp-quantus-chain` repository at commit `3b243b8`.

Then temporarily add the module declaration to `pallets/wormhole/src/lib.rs`:

```rust
#[cfg(test)]
mod proof_vol_fee_no_leaf;
```

{% endstep %}

{% step %}

### Execution command

```bash
cargo test -p pallet-wormhole proof_vol_fee_no_leaf --release -- --nocapture
```

{% endstep %}
{% endstepper %}

### Proof source

```rust
//! Proof: miner volume-fee credit omits record_transfer, permanently freezing funds.
//!
//! Integration test exercising the full verify_private_batch dispatch with a real
//! ZK proof fixture. Demonstrates both the defect (missing record_transfer for the
//! miner's volume-fee share) and the impact (no ZK-tree leaf exists for the credit,
//! and the miner address is keyless, so the funds are permanently inaccessible).

use crate::mock::*;
use crate::pallet::TransferCount;
use frame_support::{assert_ok, traits::fungible::Inspect};
use frame_system::RawOrigin;
use qp_wormhole_verifier::parse_private_batch_public_inputs;
use sp_core::{crypto::AccountId32, H256};

const PRIVATE_BATCH_PROOF_HEX: &str = include_str!("../test-data/private_batch.hex");

fn get_test_proof_bytes() -> Vec<u8> {
    hex::decode(PRIVATE_BATCH_PROOF_HEX.trim()).expect("Invalid hex in test proof")
}

fn ceil_volume_fee(exit_quanta: u128, fee_bps: u128) -> u128 {
    exit_quanta
        .saturating_mul(fee_bps)
        .div_ceil(10_000 - fee_bps)
        * crate::SCALE_DOWN_FACTOR
}

fn canonicalize(account: &AccountId) -> AccountId {
    let bytes: [u8; 32] = *account.as_ref();
    AccountId32::from(pallet_zk_tree::tree::canonicalize_account_bytes(bytes))
}

#[test]
fn miner_volume_fee_has_no_zk_leaf_full_dispatch() {
    new_test_ext().execute_with(|| {
        // ── 1. Parse the real ZK proof fixture to extract expected public inputs ──
        let proof_bytes = get_test_proof_bytes();
        let verifier =
            crate::get_private_batch_verifier().expect("Verifier should be available");
        let proof = qp_wormhole_verifier::ProofWithPublicInputs::<
            qp_wormhole_verifier::F,
            qp_wormhole_verifier::C,
            2,
        >::from_bytes(proof_bytes.clone(), &verifier.circuit_data.common)
        .expect("Proof should deserialize");
        let inputs = parse_private_batch_public_inputs(&proof).expect("Should parse");

        // ── 2. Set up on-chain block state matching the proof ──
        let block_number = inputs.block_data.block_number as u64;
        let block_hash_bytes: [u8; 32] =
            inputs.block_data.block_hash.as_ref().try_into().unwrap();
        frame_system::BlockHash::<Test>::insert(
            block_number,
            H256::from(block_hash_bytes),
        );
        System::set_block_number(block_number + 10);

        // ── 3. Seed a block author via the pre-runtime digest (QPoW path) ──
        let miner_preimage = [7u8; 32];
        set_miner_preimage_digest(miner_preimage);
        let miner = AccountId32::from(
            qp_wormhole::derive_wormhole_address(miner_preimage)
                .expect("test preimage limbs are canonical"),
        );

        assert_eq!(
            qp_wormhole::extract_author_from_digest::<AccountId, _>(
                frame_system::Pallet::<Test>::digest().logs.iter()
            ),
            Some(miner.clone()),
            "miner must be wormhole-derived"
        );

        // ── 4. Snapshot pre-dispatch state ──
        let miner_balance_before = Balances::balance(&miner);
        let leaf_count_before = ZkTree::leaf_count();
        let canonical_miner = canonicalize(&miner);
        let miner_tc_before = TransferCount::<Test>::get(&canonical_miner);

        let exit_quanta: u128 = inputs
            .account_data
            .iter()
            .filter(|a| a.summed_output_amount > 0)
            .map(|a| a.summed_output_amount as u128)
            .sum();
        let fee_bps = VolumeFeeRateBps::get() as u128;
        let total_fee = ceil_volume_fee(exit_quanta, fee_bps);
        let burn_bucket = sp_runtime::Permill::from_percent(50) * total_fee;
        let expected_miner_fee = total_fee - burn_bucket;
        assert!(
            expected_miner_fee > 0,
            "fixture must produce a nonzero miner fee"
        );

        System::reset_events();

        // ══════════════════════════════════════════════════════════════════════
        // DISPATCH: Full verify_private_batch with real ZK proof verification
        // ══════════════════════════════════════════════════════════════════════
        assert_ok!(Wormhole::verify_private_batch(
            RawOrigin::None.into(),
            proof_bytes,
        ));

        // ══════════════════════════════════════════════════════════════════════
        // PART A — THE DEFECT: miner credited without a ZK-tree leaf
        // ══════════════════════════════════════════════════════════════════════

        // A.1 Miner balance DID increase (funds were credited)
        assert_eq!(
            Balances::balance(&miner),
            miner_balance_before + expected_miner_fee,
            "miner balance must increase by the volume-fee share"
        );

        // A.2 MinerVolumeFeePaid event WAS emitted
        System::assert_has_event(
            crate::Event::<Test>::MinerVolumeFeePaid {
                miner: miner.clone(),
                amount: expected_miner_fee,
            }
            .into(),
        );

        // A.3 No NativeTransferred event for the miner fee — proves record_transfer
        //     was never called for the miner's volume-fee credit
        let miner_native_transferred = frame_system::Pallet::<Test>::events()
            .iter()
            .any(|record| {
                matches!(
                    &record.event,
                    RuntimeEvent::Wormhole(
                        crate::Event::<Test>::NativeTransferred { to, amount, .. }
                    ) if *to == miner.clone().into() && *amount == expected_miner_fee
                )
            });
        assert!(
            !miner_native_transferred,
            "BUG: no NativeTransferred event for miner fee — \
             record_transfer was never called"
        );

        // A.4 TransferCount for the miner address is unchanged
        assert_eq!(
            TransferCount::<Test>::get(&canonical_miner),
            miner_tc_before,
            "BUG: miner TransferCount unchanged — no ZK leaf was inserted"
        );

        // ══════════════════════════════════════════════════════════════════════
        // PART B — IMPACT: miner funds are permanently frozen
        // ══════════════════════════════════════════════════════════════════════

        // B.1 No ZK-tree leaf exists for the miner's volume-fee credit.
        //     Walk every leaf inserted during this dispatch and confirm none
        //     belongs to the miner for the fee amount.
        let leaf_count_after = ZkTree::leaf_count();
        let miner_canonical_bytes: [u8; 32] = *canonical_miner.as_ref();
        for idx in leaf_count_before..leaf_count_after {
            let leaf = ZkTree::leaf(idx).expect("leaf must exist at this index");
            let leaf_to_bytes: [u8; 32] = *leaf.to.as_ref();
            assert!(
                leaf_to_bytes != miner_canonical_bytes
                    || leaf.amount != expected_miner_fee,
                "no ZK leaf should exist for the miner's volume-fee credit, \
                 but found one at index {}",
                idx
            );
        }

        // B.2 The miner address is a Poseidon2 hash output with no private key.
        //     Combined with the absence of a ZK leaf, there is no spend path:
        //       - Signed extrinsics: impossible (no private key for Poseidon2 hash)
        //       - ZK exit proof: impossible (no leaf → no Merkle inclusion proof)
        //       - Proxy/recovery pallets: require initial setup from the account
        //     Therefore the volume-fee credit is permanently frozen.
        //
        //     This is not a one-time event: it triggers on every wormhole exit
        //     settlement, accumulating frozen funds over the chain's lifetime.
    });
}
```

### Impact assertions

The test proves permanent freezing of funds in two parts:

* **Part A — The defect (miner credited without leaf):**
  1. Miner balance increases by the volume-fee share, confirming `increase_balance` executed.
  2. `MinerVolumeFeePaid` event emitted, confirming the fee credit path ran.
  3. No `NativeTransferred` event for the miner fee — proves `record_transfer` was never called.
  4. `TransferCount` for the miner address unchanged — independently confirms no ZK-tree leaf insertion.
* **Part B — Impact (funds permanently frozen):**
  1. No ZK-tree leaf matches the miner's canonical address for the fee amount — confirmed by walking every leaf inserted during dispatch.
  2. The miner address is a Poseidon2 hash with no private key. Without a ZK leaf, there is no spend path: no signed extrinsic (no key), no ZK exit proof (no leaf), no proxy/recovery (requires initial setup from the account).

### Reproducibility

* Pinned commit: `3b243b870a5442a6e5f443056f67bbba87d472f5`
* Uses the pallet's own pre-computed ZK proof fixture (`test-data/private_batch.hex`)
* Exercises the full `verify_private_batch` extrinsic dispatch path
* No target code modifications needed
