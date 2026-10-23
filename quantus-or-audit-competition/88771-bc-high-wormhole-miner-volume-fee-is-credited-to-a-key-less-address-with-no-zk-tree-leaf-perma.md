# 88771 bc high wormhole miner volume fee is credited to a key less address with no zk tree leaf permanently freezing it

**Submitted on Aug 15th 2026 at 00:36:25 UTC by @Naresh\_Kandula for** [**Audit Comp | Quantus**](https://immunefi.com/audit-competition/audit-comp-quantus)

* **Report ID:** #88771
* **Report Type:** Blockchain/DLT
* **Report severity:** High
* **Target:** <https://github.com/immunefi-team/audit-comp-quantus-chain>
* **Impacts:**
  * Permanent freezing of funds

## Description

### Summary

Every wormhole exit charges a volume fee. That fee is split three ways — burn, aggregator rebate, miner share — and the miner share is credited to the block author with `Unbalanced::increase_balance`.

The block author on this chain is, by construction, a **wormhole address**: `hash(hash(salt + secret))`. It has no private key and can never be signed for. In this design a balance sitting at such an address is not spendable on its own — the only way value there becomes spendable is by proving a ZK-tree leaf, which mints fresh funds elsewhere. Every other credit to this same class of address in the codebase pairs the credit with a leaf, for exactly this reason.

The miner-fee credit creates **no leaf**. The tokens are therefore permanently unspendable by anyone, by any path, forever — while `TotalIssuance` grows by that amount and the runtime emits `MinerVolumeFeePaid`, reporting the miner as paid.

At the shipped configuration this destroys **0.02% of all wormhole exit volume**, continuously and without bound, and it does so silently.

### Root cause

**1 — the credit, with no `record_transfer`** (`chain/pallets/wormhole/src/lib.rs:955-965`):

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

**2 — the author is a key-less wormhole address.** `chain/pallets/wormhole/src/lib.rs:946` calls `qp_wormhole::extract_author_from_digest`, which reads the 32-byte QPoW pre-runtime preimage and returns `derive_wormhole_address(preimage)` (`chain/primitives/wormhole/src/lib.rs:109-151`). That file documents the construction itself, at `:100-101`:

```rust
/// The inner_digest is the "first hash" from wormhole derivation: `hash(salt + secret)`.
/// The wormhole address is: `address = hash(hash(salt + secret))`.
```

This is precisely the circuit's unspendable-account construction. It cannot coincide with a Dilithium account — whose address is a hash of a public key — without a Poseidon2 preimage break, so no signature can ever move funds from it.

**3 — the same function, twelve lines earlier, does it correctly** (`chain/pallets/wormhole/src/lib.rs:850-858`):

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

**4 — and the sibling pallet crediting the identical author address does it correctly too** (`chain/pallets/mining-rewards/src/lib.rs:220-241`):

```rust
match T::Currency::mint_into(miner, reward) {
    Ok(_) => {
        T::ProofRecorder::record_transfer_proof(
            None, // Native token
            mint_account,
            miner.clone(),
            reward,
        );
```

Both of those credit the *same class of key-less address* from the *same minting sentinel*, and both pair the credit with a leaf. Only the miner volume-fee branch omits it.

**5 — no other mechanism compensates.** The runtime's `WormholeProofRecorderExtension` cannot cover it:

* It records proofs only from `Balances::Transfer`, `Balances::Minted`, `Balances::TransferOnHold` and `Balances::ReserveRepatriated` events (`chain/runtime/src/transaction_extensions.rs:339-372`).
* `Unbalanced::increase_balance` resolves to `write_balance`, which emits **no** event. `Event::Minted` is emitted only from `done_mint_into` — the `Mutate::mint_into` path — at `chain/pallets/balances/src/impl_fungible.rs:192-193`. `increase_balance` does not go through it. This is exactly why `mining-rewards`, which *does* use `mint_into`, still records its proof explicitly.
* `count_transfers` (`chain/runtime/src/transaction_extensions.rs:217-250`) does not list either wormhole call, and its own comment states it "must stay in sync with the events matched by `record_proofs_from_events_since`" — i.e. the extension is not expected to record anything here.

### The invariant this breaks — stated by the project itself

`chain/pallets/wormhole/src/lib.rs:325-329`, documenting `WormholeAccountId`:

> "The `AsRef<[u8]>`/`From<[u8; 32]>` bounds allow `record_transfer` to canonicalize recipients (reduce each 8-byte limb mod the Goldilocks prime) **before keying `TransferCount` and inserting ZK-tree leaves**."

`record_transfer` *is* the mechanism by which a wormhole-address balance becomes spendable. The invariant is therefore: **any credit to a wormhole address must be accompanied by a ZK-tree leaf.** The pallet upholds it on the exit path, `mining-rewards` upholds it for block rewards and transaction fees, and this one branch does not.

## Proof of Concept

Runnable on **unmodified production code**. The test drives the real extrinsic (`Wormhole::verify_private_batch`) with the repository's own checked-in proof fixture — no internals are called by hand and no production line is touched.

The test carries its own **non-vacuity oracle**: in the *same extrinsic*, the exit account's `TransferCount` increases — proving the leaf mechanism is live, reachable and working in this very call — while the block author's stays at `0`.

### Steps to reproduce

{% stepper %}
{% step %}

## Clone the repository at the frozen competition commit

```bash
git clone --branch audit-comp-ready \
  https://github.com/immunefi-team/audit-comp-quantus-chain.git
cd audit-comp-quantus-chain
git checkout 3b243b870a5442a6e5f443056f67bbba87d472f5
```

{% endstep %}

{% step %}

## Add the test

Open `pallets/wormhole/src/tests.rs` and paste the test from the next section **inside** `mod private_batch_proof_tests { … }` — for example immediately after that module's opening brace (line 568). This is the only file edited; no production file is modified.
{% endstep %}

{% step %}

## Run the test

```bash
cargo test -p pallet-wormhole poc_miner_volume_fee_credit_has_no_zk_tree_leaf
```

{% endstep %}
{% endstepper %}

{% hint style="info" %}
**Build note.** `pallets/wormhole/build.rs` generates the ZK circuit binaries at compile time and the first build takes roughly 12 minutes. Leave the circuit-size knobs at their defaults — do **not** set `QP_NUM_LEAF_PROOFS` / `QP_NUM_PRIVATE_BATCH_PROOFS` to smaller values to speed the build up, because the checked-in proof fixtures are generated at the defaults (7 leaves / 53 private batches) and will fail to deserialize against a smaller circuit. Toolchain: the repo's pinned `rust-toolchain` (1.93.0).
{% endhint %}

### Proof-of-concept source (complete, self-contained)

```rust
#[test]
fn poc_miner_volume_fee_credit_has_no_zk_tree_leaf() {
    use codec::Decode;

    new_test_ext().execute_with(|| {
        let proof = deserialize_test_proof();
        let inputs = parse_private_batch_public_inputs(&proof).expect("Should parse");

        let block_number = inputs.block_data.block_number as u64;
        let block_hash_bytes: [u8; 32] =
            inputs.block_data.block_hash.as_ref().try_into().unwrap();
        let block_hash = H256::from(block_hash_bytes);
        frame_system::BlockHash::<Test>::insert(block_number, block_hash);
        System::set_block_number(block_number + 10);

        let miner_preimage = [7u8; 32];
        set_miner_preimage_digest(miner_preimage);
        let expected_author = sp_core::crypto::AccountId32::from(
            qp_wormhole::derive_wormhole_address(miner_preimage)
                .expect("test preimage limbs are canonical"),
        );

        // The exit account from the fixture, used as the "leaf mechanism works" control.
        let exit_account = inputs
            .account_data
            .iter()
            .find(|a| a.summed_output_amount > 0)
            .map(|a| {
                <Test as frame_system::Config>::AccountId::decode(
                    &mut a.exit_account.as_ref(),
                )
                .expect("valid exit account encoding")
            })
            .expect("fixture must have a real exit");

        let exit_leaves_before = crate::TransferCount::<Test>::get(&exit_account);
        let author_leaves_before = crate::TransferCount::<Test>::get(&expected_author);
        assert_eq!(author_leaves_before, 0, "author must start with no leaves");

        let author_balance_before = Balances::free_balance(expected_author.clone());

        assert_ok!(Wormhole::verify_private_batch(
            RawOrigin::None.into(),
            get_test_proof_bytes()
        ));

        // The credit genuinely happened: the miner's balance went up.
        let author_balance_after = Balances::free_balance(expected_author.clone());
        assert!(
            author_balance_after > author_balance_before,
            "non-vacuity: the miner-fee credit must actually be non-zero for this fixture"
        );

        // Non-vacuity oracle: the SAME extrinsic DID record a leaf for the exit
        // account, proving `record_transfer` is reachable and working here.
        let exit_leaves_after = crate::TransferCount::<Test>::get(&exit_account);
        assert!(
            exit_leaves_after > exit_leaves_before,
            "control failed: exit credit should have recorded a ZK-tree leaf"
        );

        // The bug: the author's wormhole address received real value with zero
        // accompanying leaves, so there is no nullifier path that can ever prove
        // spending it.
        let author_leaves_after = crate::TransferCount::<Test>::get(&expected_author);
        assert_eq!(
            author_leaves_after, 0,
            "BUG: miner-fee credit created no ZK-tree leaf for a keyless wormhole address \
             — the funds are permanently unspendable"
        );
    });
}
```

### Output

```
test tests::private_batch_proof_tests::poc_miner_volume_fee_credit_has_no_zk_tree_leaf ... ok
test result: ok. 69 passed; 0 failed; 3 ignored; 0 measured; 0 filtered out
```

The whole pallet suite stays green with the test added: **69 passed, 0 failed**.

### The test asserts, in order

{% stepper %}
{% step %}

## Derive and inspect the block author

The block author is derived exactly as production derives it — from a QPoW pre-runtime digest preimage through `qp_wormhole::derive_wormhole_address` — and starts with **zero** ZK-tree leaves.
{% endstep %}

{% step %}

## Execute the real extrinsic

The real extrinsic `Wormhole::verify_private_batch` succeeds against the repository's own proof fixture. Nothing is stubbed.
{% endstep %}

{% step %}

## Confirm the credit occurred

**The credit really happened**: the author's free balance strictly increases. Without this the rest would be vacuous.
{% endstep %}

{% step %}

## Confirm the leaf mechanism is active

**Control — the leaf mechanism is live in this very call**: the exit account's `TransferCount` strictly increases in the same extrinsic. `record_transfer` is reachable, wired, and working here.
{% endstep %}

{% step %}

## Confirm the missing author leaf

**The bug**: the author's `TransferCount` is still `0`. Real value was credited to a key-less address with no leaf, so no nullifier path exists that could ever prove spending it.
{% endstep %}
{% endstepper %}

### Negative control

Injecting the minimal fix into the `Ok(_)` arm at `pallets/wormhole/src/lib.rs:960` — pairing the credit with a leaf exactly as the exit path does — makes the PoC fail at precisely the leaf assertion, and nowhere else:

```
test tests::private_batch_proof_tests::poc_miner_volume_fee_credit_has_no_zk_tree_leaf ... FAILED

assertion `left == right` failed: BUG: miner-fee credit created no ZK-tree leaf for a
keyless wormhole address — the funds are permanently unspendable
  left: 1
 right: 0

test result: FAILED. 0 passed; 1 failed
```

`left: 1` is the leaf the fix creates. This confirms the assertion is load-bearing and that the missing `record_transfer` is the sole cause. Reverting the fix restores an empty `git diff` and the pass (69 passed, 0 failed). No production line is modified in the positive run.

## Impact

At the shipped configuration — `VolumeFeeRateBps = 4`, `VolumeFeesBurnRate = 50%` (`chain/runtime/src/configs/mod.rs:734-736`) — the miner share is half of a 4 bps fee, so **0.02% of all wormhole exit volume is permanently destroyed**, on every exit, without bound as volume accrues.

Additionally:

* **Miners never receive volume-fee income at all**, removing the intended incentive to include exit settlements — while `MinerVolumeFeePaid` reports otherwise, so the failure is silent to operators and to any accounting built on chain events.
* **`TotalIssuance` permanently includes tokens that are provably unspendable**, so circulating-supply accounting drifts from reality with every exit, and the supply-driven block-reward calculation in `mining-rewards::on_finalize` is computed against an inflated `total_issuance()`.

The identical omission applies to the **aggregator rebate** credit at `chain/pallets/wormhole/src/lib.rs:919-933` whenever the aggregator supplies a wormhole-style address — that branch calls `increase_balance` with no `record_transfer` either.

## Mitigations, and why none of them save it

| Candidate mitigation                          | Why it fails                                                                                                                                                          |
| --------------------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| The miner signs for the address               | Impossible. The address is `hash(hash(salt + secret))`; colliding it with a Dilithium account address requires a Poseidon2 preimage break.                            |
| `WormholeProofRecorderExtension` records it   | It matches only `Transfer` / `Minted` / `TransferOnHold` / `ReserveRepatriated`. `increase_balance` → `write_balance` emits **no** event at all.                      |
| `Event::Minted` fires anyway                  | `Minted` is emitted only from `done_mint_into` (`pallets/balances/src/impl_fungible.rs:192-193`), on the `mint_into` path. `increase_balance` does not go through it. |
| `count_transfers` covers wormhole calls       | It does not list them, and its own comment says it is deliberately kept in sync with only the recorded events.                                                        |
| A later exit re-mints the value               | Exits mint *against a leaf*. With no leaf there is nothing to prove, so no exit can ever reference this balance.                                                      |
| The `Err` branch burns it, so it is accounted | The `Err` branch burns and is accounted. The **`Ok`** branch is the defect: the mint succeeds, issuance rises, and the value is unreachable.                          |
| Governance can fix it later                   | A runtime upgrade can fix *future* payments. It cannot create the historical leaves, and every exit until then destroys value.                                        |

## Novelty

* **Not one of the three published Immunefi Known Issues.** The declared quantization item concerns *mining rewards* falling below the circuit's quantized minimum; this report is about a missing leaf record, at any fee size.
* **Distinct from the prior review finding on this same line.** That finding concerned miner-fee **address derivation** — raw `decode` of the preimage, the missing `POW_ENGINE_ID` filter, and a silent burn on extraction failure. It was fixed: extraction is now correct, filtered, and fallible. The defect reported here is that a *correctly* derived author is credited **without a leaf**, and it survives that fix completely — the fix made the address right, not the accounting.
* **Still present on the project's `main`.** Verified at upstream `Quantus-Network/chain` HEAD (`c3f7f3b8`), ten commits past the frozen competition commit: `pallets/wormhole/src/lib.rs` is untouched and the `Ok(_)` arm at lines 959-965 still goes straight from `increase_balance` to `deposit_event` with no `record_transfer`. No branch in the repository carries a fix.

## Scope

`pallets/wormhole` is Quantus-authored — it has no upstream Substrate baseline — so the defect is unambiguously in in-scope first-party code. The single affected file is `chain/pallets/wormhole/src/lib.rs`, with the same shape at the aggregator-rebate branch in that file.

The bug requires **no attacker at all** and no privileged actor. It triggers on the normal, intended operation of the protocol: every wormhole exit that pays a non-zero miner fee destroys that fee. No special state, no race, no configuration change.

## Recommended fix

Pair the credit with a leaf, exactly as the exit path and `mining-rewards` already do:

```rust
Ok(_) => {
    let from_account: <T as Config>::WormholeAccountId = mint_account.clone().into();
    let to_account: <T as Config>::WormholeAccountId = author.clone().into();
    Self::record_transfer(
        T::AssetId::default(),
        &from_account,
        &to_account,
        miner_fee,
    );
    Self::deposit_event(Event::MinerVolumeFeePaid { miner: author, amount: miner_fee });
},
```

Apply the same treatment to the aggregator rebate at `lib.rs:919-933`.

{% hint style="warning" %}
`record_transfer` quantizes to the circuit's 2-decimal precision, so a fee below one quantum still cannot be represented as a leaf. Such residue should be added to the burn bucket rather than credited, so that issuance and spendability stay consistent — that is the same class of concern as the declared known issue on quantized mining rewards, and the two should be resolved together.
{% endhint %}

More generally, the safe invariant to enforce repository-wide is: **every credit to a `WormholeAccountId` must be accompanied by a `record_transfer`.** A single helper that performs both, used everywhere in place of a bare `increase_balance`/`mint_into` to a wormhole address, would close the class rather than this instance.
