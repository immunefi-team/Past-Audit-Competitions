# 88881 bc high wormhole miner volume fee is minted without its zk tree leaf permanently freezing every exit fee&#x20;

**Submitted on Aug 15th 2026 at 11:51:08 UTC by @jerusha for** [**Audit Comp | Quantus**](https://immunefi.com/audit-competition/audit-comp-quantus)

* **Report ID:** #88881
* **Report Type:** Blockchain/DLT
* **Report severity:** High
* **Target:** <https://github.com/immunefi-team/audit-comp-quantus-chain>
* **Impacts:**
  * Permanent freezing of funds (fix requires hardfork)

## Description

## Brief/Intro

Every wormhole exit charges users a volume fee. Half is burned and the other half is credited to the block author as payment. The author account is always a wormhole address, derived as `Poseidon(Poseidon(salt+secret))`, which by construction has no private key. Value held at such an address can only be moved by a ZK exit proof, and every exit proof requires a ZK-tree leaf committing to that value. The exit path records such a leaf, and the mining-rewards pallet records one for the very same account, but the volume-fee credit records none. The fee is therefore taken from users, credited as a number that nothing in the runtime can ever spend, and lost forever. This is not a transfer that fails loudly; it succeeds, emits a success event, and leaves the value unrecoverable.

## Vulnerability Details

### Why value at the author's address needs a leaf

The block author is derived from the pre-runtime digest and is unconditionally passed through `derive_wormhole_address`, at `chain/primitives/wormhole/src/lib.rs:143-149`:

```rust
let preimage: [u8; 32] = data.as_slice().try_into().ok()?;
let address_bytes = derive_wormhole_address(preimage).ok()?;
return AccountId::decode(&mut &address_bytes[..]).ok();
```

`derive_wormhole_address` is `rehash_to_bytes(&inner_digest)` (`primitives/wormhole/src/lib.rs:109-111`), so the on-chain account is `Poseidon(Poseidon(salt+secret))`. There is no miner input that makes this a signature-controlled account: a keyed account is `Poseidon(public_key_bytes)` over a \~2 KB ML-DSA key (`chain/primitives/dilithium-crypto/src/scheme_macro.rs:244`), so matching one would require a Poseidon preimage collision.

The pallet's own comment states the resulting model, at `chain/pallets/wormhole/src/lib.rs:876-886`:

```rust
// Original deposit locked `input_amount` in an unspendable account (tokens still
// exist). On exit we mint `output_amount` to user, where: input >= output + fee
//   - Minting exit amounts: increases balances but NOT issuance
//   - Minting miner fee: increases balance but NOT issuance (increase_balance)
```

Exits mint to the destination and never debit the wormhole address, so raw balance at such an address is inert. The ZK-tree leaf is the bearer instrument. A withdrawal proves membership of a leaf and draws `input_amount` from that leaf, so a credit with no leaf can never be withdrawn by anyone.

### The defect

Three credits go to that same keyless author account. Two record a leaf. One does not.

| Credit       | Location                                | Records leaf                                        |
| ------------ | --------------------------------------- | --------------------------------------------------- |
| Exit mint    | `pallets/wormhole/src/lib.rs:852`       | yes, `Self::record_transfer(...)`                   |
| Block reward | `pallets/mining-rewards/src/lib.rs:230` | yes, `T::ProofRecorder::record_transfer_proof(...)` |
| Volume fee   | `pallets/wormhole/src/lib.rs:955`       | **no**                                              |

The exit path, `chain/pallets/wormhole/src/lib.rs:849-859`:

```rust
minted_exit_amount = minted_exit_amount.saturating_add(*exit_balance);
// Record transfer proof for the minted tokens
let from_account: <T as Config>::WormholeAccountId = mint_account.clone().into();
let to_account: <T as Config>::WormholeAccountId = exit_account.clone().into();
Self::record_transfer(T::AssetId::default(), &from_account, &to_account, *exit_balance);
```

The miner-fee path, `chain/pallets/wormhole/src/lib.rs:951-965`:

```rust
match <T::Currency as Unbalanced<_>>::increase_balance(
    &author, miner_fee, frame_support::traits::tokens::Precision::Exact,
) {
    Ok(_) => {
        Self::deposit_event(Event::MinerVolumeFeePaid { miner: author, amount: miner_fee });
    },
    ...
}
```

There is no `record_transfer` anywhere in that branch. `grep -n "record_transfer(" pallets/wormhole/src/lib.rs` returns only line 486 (genesis), line 854 (exits), and the recorder trait implementations.

Both pallets resolve the account through the same helper, so this is provably the same account: `pallets/mining-rewards/src/lib.rs:207` calls `qp_wormhole::extract_author_from_digest`, and `pallets/wormhole/src/lib.rs:946` calls it too. The author's balance therefore grows by fees that have no backing leaf, while its leaf set only ever reflects block rewards and exits.

### No other mechanism recovers it

The runtime has a second, event-driven recorder, `WormholeProofRecorderExtension`, which scans a transaction's events and records a leaf for transfers into wormhole accounts. It cannot see this credit, for two independent reasons.

First, `Unbalanced::increase_balance` emits no event at all. It is the default trait method at `chain/frame/support/src/traits/tokens/fungible/regular.rs:205-227`, which calls `write_balance` and never `done_mint_into`. Only `done_mint_into` deposits `Balances::Minted` (`chain/pallets/balances/src/impl_fungible.rs:192-193`, a file I verified is byte-identical to upstream `pallet-balances-46.0.0`). The pallet relies on this deliberately, and says so at `pallets/wormhole/src/lib.rs:820-827`, because exit credits already record their own proof and a `Minted` event would double-record them. The same event-free property that protects exits is what hides the miner fee.

Second, `verify_private_batch` and `verify_public_batch` are bare unsigned extrinsics (`ensure_none`, `pallets/wormhole/src/lib.rs:589` and `:626`), so the extension's `post_dispatch` never runs for them.

With no event and no explicit call, no leaf is created on any path.

### Evidence this is an oversight rather than a decision

Commit `60ec991` ("Emit MinerVolumeFeePaid event for wormhole exit volume fees", #619) states the problem and then fixes only its visibility: "The miner's share of the wormhole exit volume fee is minted via increase\_balance **without record\_transfer**, so no wormhole-pallet event carries it." The author noticed the missing `record_transfer` and added an event instead of a leaf.

The existing test covers the same blind spot. `pallets/wormhole/src/tests.rs:1035-1093` asserts the event fires and that `Balances::free_balance(expected_author)` increased, and never asserts that a leaf or `TransferCount` increased, which is exactly the assertion that would have caught this.

Nothing in the repository documents the fee as intentionally unspendable. `rg -i 'volume fee|miner fee|MinerVolumeFee' docs/ README.md` returns no hits.

### Not a duplicate, and not a known issue

The competition's Known Issues are referendum member votes remaining after removal, plaintext miner-to-node connections, and future zk-tree depth plus mining-reward quantization limits. This is none of them.

The closest prior finding is EQ-QNT-WORMHOLE-F-03, "Miner Rewards Silently Burned Due to Incorrect Preimage Handling", in `chain/audits/EigerWormholeAudit.pdf` (Equilibrium, 2026-03-20). That finding is a different bug with a different root cause, and it has been fixed. It reported that the pallet decoded the digest payload directly as an `AccountId` instead of hashing it, so the author lookup failed and the fee was burned. Its three recommendations, quoted from the report, are: fix the extraction logic to hash the preimage, emit an error or event when extraction fails, and add tests. It says nothing about recording a ZK leaf.

The project implemented exactly those three. Extraction now hashes correctly (`primitives/wormhole/src/lib.rs:148`), a `MinerVolumeFeePaid` event now exists, and a test now exists. The result is that the fee now reaches the correct address and is still unusable, for a reason the prior audit never examined. My finding is the residual defect that the fix left behind: a missing `record_transfer`, not a wrong address.

I searched all four audit reports for any treatment of this. Extracting each with `pdftotext -layout` and grepping, no report links `record_transfer` to the miner or author fee, and none contains "no leaf", "without a leaf" or "missing leaf" in that context. EQ-QNT-WORMHOLE-F-05 does concern the proof recorder, but it is about the transaction extension pattern-matching on specific call variants, not about this credit.

## Impact Details

What is lost: the miner's share of the volume fee on every wormhole exit. With `VolumeFeeRateBps = 4` and `VolumeFeesBurnRate = 50%` (`chain/runtime/src/configs/mod.rs:734-736`), that is 0.020 percent of all wormhole exit volume, or 200 QUAN per 1,000,000 QUAN exited. It accrues without bound for as long as the wormhole is used.

Who is harmed: both sides of the fee. Users pay it, because the circuit enforces `input >= output + fee` so their exit is reduced by the full fee. Miners never receive their half, because the credit has no leaf. The value is taken from users and delivered to nobody.

Why the funds are frozen rather than merely misrouted: the tokens exist in the author's `free_balance`. No signature can move them, since the account is a Poseidon-derived wormhole address with no private key. No ZK exit can move them, since exits draw on leaves and no leaf commits to this value. There is no extrinsic in any pallet that can mint a leaf for an arbitrary past credit.

Realistic attacker: none required. This fires on every honest exit, with no attacker and no unusual conditions. An attacker can amplify it deliberately, since wormhole exits are `Pays::No`: round-tripping value through the wormhole destroys 0.02 percent of the round-trip volume into dead balance at zero transaction-fee cost, shrinking the usable float.

Secondary accounting effect: `increase_balance` does not update `TotalIssuance` (see the pallet's own supply-accounting comment at `lib.rs:876-886`), so these frozen tokens are balance with no issuance backing.

Honest scoping of the workaround. Future fees are fixed by a one-line change plus a runtime upgrade, which on Substrate is forkless. Already-frozen fees are not recoverable by any normal path; restoring them would require governance to write ZK-tree storage directly via `set_storage`, which is a privileged out-of-band intervention rather than a protocol operation. I am claiming Critical because the impact type is permanent freezing of funds, and because the frozen amount is unrecoverable through any dispatchable, while noting plainly that the amount per exit is small and that the forward fix does not itself need a hard fork.

## References

* `chain/pallets/wormhole/src/lib.rs:951-965` (the miner-fee credit with no `record_transfer`)
* `chain/pallets/wormhole/src/lib.rs:849-859` (the exit path that does record a leaf)
* `chain/pallets/mining-rewards/src/lib.rs:230` (same account, records a leaf)
* `chain/primitives/wormhole/src/lib.rs:109-111, 143-149` (author is always a wormhole address)
* `chain/pallets/wormhole/src/lib.rs:876-886` (supply-accounting comment: deposits sit in an unspendable account)
* `chain/frame/support/src/traits/tokens/fungible/regular.rs:205-227` (`increase_balance` emits no event)
* `chain/pallets/balances/src/impl_fungible.rs:192-193` (`Minted` only from `done_mint_into`)
* `chain/pallets/wormhole/src/lib.rs:589, 626` (bare unsigned extrinsics, so no `post_dispatch` scan)
* `chain/runtime/src/configs/mod.rs:734-736` (`VolumeFeeRateBps = 4`, `VolumeFeesBurnRate = 50%`)
* `chain/pallets/wormhole/src/tests.rs:1035-1093` (existing test that checks balance but not the leaf)
* Dedup baseline: `chain/audits/EigerWormholeAudit.pdf`, EQ-QNT-WORMHOLE-F-03 and its recommendations
* Repository at the audited commit: <https://github.com/immunefi-team/audit-comp-quantus-chain/tree/audit-comp-ready>

## Proof of Concept

A unit test against the real pallet, using the repository's own private-batch proof fixture. It exercises the production `verify_private_batch` path, so the credit and the missing leaf are the real ones, not mocked. No network is touched, in line with the program's testing rules.

### Where it goes

Append the test below to the `private_batch_proof_tests` module in `chain/pallets/wormhole/src/tests.rs`, at commit `3b243b870a5442a6e5f443056f67bbba87d472f5`. It sits naturally after `test_verify_private_batch_emits_miner_volume_fee_paid`, whose setup it mirrors.

### Run it

```bash
export LIBCLANG_PATH=/Library/Developer/CommandLineTools/usr/lib   # macOS only; Linux: apt install libclang-dev
cargo test -p pallet-wormhole poc_miner_volume_fee -- --nocapture
```

### The test

```rust
/// PoC: the miner's volume-fee credit is permanently frozen.
///
/// The block author is a wormhole address (hash(hash(salt+secret))), which has no
/// private key by construction. Value at such an address is movable ONLY by a ZK exit,
/// which requires a ZK-tree leaf. Exits record a leaf (lib.rs:852) and mining-rewards
/// records one for this SAME account (mining-rewards/lib.rs:230) -- but the wormhole
/// miner volume fee is credited with a bare `increase_balance` and records none.
///
/// This test asserts the balance rises while NO leaf is created for the author,
/// i.e. the credited tokens can never be moved by anyone, ever.
#[test]
fn poc_miner_volume_fee_is_credited_without_a_zk_leaf_and_is_unspendable() {
    new_test_ext().execute_with(|| {
        let proof = deserialize_test_proof();
        let inputs = parse_private_batch_public_inputs(&proof).expect("Should parse");

        let block_number = inputs.block_data.block_number as u64;
        let block_hash_bytes: [u8; 32] =
            inputs.block_data.block_hash.as_ref().try_into().unwrap();
        frame_system::BlockHash::<Test>::insert(block_number, H256::from(block_hash_bytes));
        System::set_block_number(block_number + 10);

        let miner_preimage = [7u8; 32];
        set_miner_preimage_digest(miner_preimage);
        let expected_author = sp_core::crypto::AccountId32::from(
            qp_wormhole::derive_wormhole_address(miner_preimage)
                .expect("test preimage limbs are canonical"),
        );

        let exit_quanta: u128 = inputs
            .account_data
            .iter()
            .filter(|a| a.summed_output_amount > 0)
            .map(|a| a.summed_output_amount as u128)
            .sum();
        let num_exits = inputs
            .account_data
            .iter()
            .filter(|a| a.summed_output_amount > 0)
            .count() as u64;
        let total_fee = super::ceil_volume_fee(exit_quanta, VolumeFeeRateBps::get() as u128);
        let expected_miner_fee = total_fee - total_fee / 2;
        assert!(expected_miner_fee > 0, "fixture must produce a non-zero miner fee");

        let author_balance_before = Balances::free_balance(expected_author.clone());
        let author_leaves_before = crate::TransferCount::<Test>::get(&expected_author);
        let tree_leaves_before = pallet_zk_tree::LeafCount::<Test>::get();

        assert_ok!(Wormhole::verify_private_batch(
            RawOrigin::None.into(),
            get_test_proof_bytes()
        ));

        let author_leaves_after = crate::TransferCount::<Test>::get(&expected_author);
        let tree_leaves_after = pallet_zk_tree::LeafCount::<Test>::get();

        // (1) The miner IS credited real spendable-looking balance.
        assert_eq!(
            Balances::free_balance(expected_author.clone()),
            author_balance_before + expected_miner_fee,
            "miner fee should be credited to the author's balance"
        );

        // (2) But NO ZK-tree leaf is created for the author -- so nothing can move it.
        assert_eq!(
            author_leaves_after, author_leaves_before,
            "VULNERABILITY: author received balance but transfer_count did not increase, \
             so no ZK leaf backs the miner fee and it can never be exited"
        );

        // (3) The tree grew by exactly the exit count -- the miner fee added no leaf.
        assert_eq!(
            tree_leaves_after - tree_leaves_before,
            num_exits,
            "tree grew only by the number of exits; the miner fee contributed no leaf"
        );

        // (4) The account is keyless: it is a Poseidon-derived wormhole address, so no
        //     signature can ever authorize a transfer out of it either.
        assert_eq!(
            <[u8; 32]>::from(expected_author.clone()),
            qp_wormhole::derive_wormhole_address(miner_preimage).unwrap(),
            "author is a wormhole (keyless) address"
        );
    });
}
```

### Expected output

```
running 1 test
test tests::private_batch_proof_tests::poc_miner_volume_fee_is_credited_without_a_zk_leaf_and_is_unspendable ... ok

test result: ok. 1 passed; 0 failed; 0 ignored; 0 measured; 71 filtered out; finished in 0.06s
```

The test passing is the finding. Each assertion pins one link of the chain:

1. The author's free balance increases by exactly the expected miner fee, so the credit really happens and really is the fee users paid.
2. `TransferCount` for the author is unchanged, so `record_transfer` was never called for it and no leaf commits to that value.
3. The ZK tree grew by exactly the number of exits in the bundle, confirming the only leaves created were the exits' own. The fee contributed nothing.
4. The author account equals `derive_wormhole_address(preimage)`, so it is a Poseidon-derived wormhole address with no private key.

Together: value was credited to an account that cannot sign, and no leaf was created that would let it be exited. Both spending routes are closed, so the credited amount is permanently frozen.

To see the contrast, change assertion (2) to `assert_ne!` and the test fails, showing there genuinely is no leaf. Running the neighbouring `test_verify_private_batch_emits_miner_volume_fee_paid` shows the project's existing coverage stops at the balance check.
