# 88571 bc high missing zk tree leaf on the wormhole exit miner fee credit permanently freezes all miner exit fee revenue keyless account no exit path&#x20;

**Submitted on Aug 13th 2026 at 21:31:17 UTC by @OxPrince for** [**Audit Comp | Quantus**](https://immunefi.com/audit-competition/audit-comp-quantus)

* **Report ID:** #88571
* **Report Type:** Blockchain/DLT
* **Report severity:** High
* **Target:** <https://github.com/immunefi-team/audit-comp-quantus-chain>
* **Impacts:**
  * Permanent freezing of unclaimed yield

## Description

### Brief/Intro

On Quantus, block authors (miners) are paid at a **wormhole address** — a *keyless* address derived as `hash(hash(salt + secret))`. Such an address has no private key, so it can never sign a transaction; the **only** way to spend value held there is a **wormhole exit**, which requires a **ZK-tree leaf** committing the amount. Every legitimate credit to a keyless address is therefore paired with a leaf (block rewards, ordinary transfers, and deposits all do this).

When a wormhole exit is processed, it charges a volume fee and pays a share to the block author. That payment is minted to the author's keyless address with an **event-free `increase_balance` and no leaf-recording call**, so **no leaf is ever created for it**. The miner thus accrues exit-fee revenue that can never be exited and never spent — it is permanently frozen. Under the production configuration this occurs on **every** wormhole exit, for the life of the chain. On mainnet this silently strands the miners' entire exit-fee income stream: the value is minted into existence at an address from which it can never be moved.

### Relationship to EQ-QNT-WORMHOLE-F-03 (prior audit)

The Eiger audit's medium finding EQ-QNT-WORMHOLE-F-03 already identified that miners were not receiving their volume fees: author extraction decoded the preimage directly (wrong), so extraction failed and the fee was **burned**. The remediation corrected the derivation (`extract_author_from_digest` → `derive_wormhole_address`), so the fee now reaches the miner's address. **This report shows that remediation is incomplete:** the address it now reaches is keyless, and no leaf is created, so the fee is **frozen** rather than burned — the audit's root complaint ("miners don't get their fees") is still not resolved. Notably, the audit cited `mining-rewards` as the "correct implementation" for the *derivation*; that same pallet also records a **leaf**, which the wormhole fix did not carry over. This report should be submitted as an *incomplete fix* of F-03, not as a novel discovery, and its acceptance turns on whether the program treats an incomplete remediation of a prior finding as in scope.

## Vulnerability Details

### Background: why a keyless credit needs a leaf

The wormhole implements privacy-preserving transfers using "unspendable" (keyless) addresses. A miner's reward address is derived from a secret preimage carried in the QPoW pre-runtime digest:

[primitives/wormhole/src/lib.rs:100-151](broken://pages/e02fce965cfd35e7289eebcec6d6773f196f7343#L100-L151):

```rust
/// The wormhole address is: address = hash(hash(salt + secret))
pub fn derive_wormhole_address(inner_digest: [u8; 32]) -> Result<[u8; 32], &'static str> { … }

/// Extract the block author (miner) account from a digest … derives the wormhole address …
pub fn extract_author_from_digest<…>(digest: I) -> Option<AccountId> { … }
```

Because the address is a hash, no key can sign for it. Funds there are realized only via a wormhole exit, which proves knowledge of `secret`, nullifies a **leaf**, and mints the amount to a fresh address. If no leaf exists for a credit, no exit proof can ever be constructed for it, so the funds are unreachable.

### The correct pattern (positive control): block rewards record a leaf

The mining-rewards pallet mints the block reward to the same keyless author **and records a leaf**, which is exactly what makes rewards spendable:

[pallets/mining-rewards/src/lib.rs:230-237](broken://pages/2e4748fd8965be977a2838e80d9774654801b650#L230-L237):

```rust
match T::Currency::mint_into(miner, reward) {
    Ok(_) => {
        T::ProofRecorder::record_transfer_proof(None, mint_account, miner.clone(), reward); // ← leaf
        Self::deposit_event(Event::MinerRewarded { miner: miner.clone(), reward });
    }
```

The exit path likewise records a leaf for each **exit-account** credit, immediately after an event-free mint:

[pallets/wormhole/src/lib.rs:828-859](broken://pages/afdee51ace40897042b5610b30abcc7d636a9a04#L828-L859):

```rust
// exit credit: event-free mint (deliberately, to avoid the proof-recorder scan double-counting)
<T::Currency as Unbalanced<_>>::increase_balance(exit_account, *exit_balance, Exact)?;
…
// … then a leaf is recorded so the recipient can exit again
Self::record_transfer(T::AssetId::default(), &from_account, &to_account, *exit_balance); // ← leaf
```

### The bug: the miner-fee credit mints without recording a leaf

A few lines later in the same function, the miner-fee credit mints the same way but **stops there — no `record_transfer`**:

[pallets/wormhole/src/lib.rs:944-965](broken://pages/afdee51ace40897042b5610b30abcc7d636a9a04#L944-L965):

```rust
if !miner_fee.is_zero() {
    let digest = frame_system::Pallet::<T>::digest();
    if let Some(author) = qp_wormhole::extract_author_from_digest::<…>(digest.logs.iter()) {
        match <T::Currency as Unbalanced<_>>::increase_balance(&author, miner_fee, Exact) {
            Ok(_) => {
                Self::deposit_event(Event::MinerVolumeFeePaid { miner: author, amount: miner_fee });
                // ← NO record_transfer here: no leaf is ever created for this credit
            }
            Err(_) => { /* burn the fee instead */ }
        }
    } else { /* no author: burn the fee */ }
}
```

The event-free mint is intentional — emitting `Balances::Minted` would make the runtime's proof-recorder scan double-count the credit (see the note at [lib.rs:822-827](broken://pages/afdee51ace40897042b5610b30abcc7d636a9a04#L822-L827)). That is correct for the scan, but it makes the explicit `record_transfer` the *only* way a leaf can be created for this credit — and it is missing. There is no other leaf source: leaves are created only inside `record_transfer`, and once at genesis in `on_initialize` for block 1 ([lib.rs:468-496](broken://pages/afdee51ace40897042b5610b30abcc7d636a9a04#L468-L496)). A credit that misses its leaf at credit time is unrecoverable.

The **aggregator-rebate** credit ([lib.rs:919-934](broken://pages/afdee51ace40897042b5610b30abcc7d636a9a04#L919-L934)) has the identical omission and freezes the same way whenever the aggregator address is a wormhole address.

### It fires on every exit under production config

[runtime/src/configs/mod.rs:734-738](broken://pages/52ebeb71568f190c80ca419f5f1e3ccd0e5b91f0#L734-L738): `VolumeFeeRateBps = 4` (0.04 %), `VolumeFeesBurnRate = 50 %`. The miner therefore receives `50 % × 0.04 % = 0.02 %` of every exit's volume, minted to a keyless address with no leaf. Every wormhole exit permanently freezes that miner fee.

### The existing test checks balance and event, but never a leaf

The pallet's own test asserts the fee lands in the author's raw balance and emits `MinerVolumeFeePaid`, but never checks whether a leaf was created — so the gap is unpinned:

[pallets/wormhole/src/tests.rs:1036-1092](broken://pages/e18317fc8853662b959616e82e4be86100ee1f9f#L1036-L1092):

```rust
System::assert_has_event(Event::MinerVolumeFeePaid { miner: expected_author, amount: expected_miner_fee }.into());
assert_eq!(Balances::free_balance(expected_author), author_balance_before + expected_miner_fee);
// (no assertion about ZkTree leaves or TransferCount)
```

## Impact Details

On every wormhole exit, the miner's share of the volume fee (0.02 % of exit volume under production config) is credited to the miner's keyless address without the leaf that is the sole means of ever spending it. The value is minted into existence but stranded: no user action can free it, because the address cannot sign and no exit proof can be built without a leaf. Over the chain's life this is a continuous, unbounded, irrecoverable loss of the miners' legitimately-earned exit-processing revenue. Recovery requires a runtime change (e.g. a migration minting leaves for the stranded balances); it cannot be fixed by any user.

**Magnitude and likelihood.** No attacker, no cost, no privileged access, no special preconditions — it is a standing protocol defect that strands value on *every* exit. Per-exit the amount is small (0.02 % of that exit's volume); in aggregate it is the entire miner-fee stream from the wormhole. Impact scales with wormhole exit volume, which on a privacy-first chain is the primary usage.

## Recommended Fix

In `process_exit_bundle`, after a successful miner-fee mint, record the leaf exactly as the exit-account credit does at [lib.rs:854](broken://pages/afdee51ace40897042b5610b30abcc7d636a9a04#L854):

```rust
Ok(_) => {
    Self::deposit_event(Event::MinerVolumeFeePaid { miner: author.clone(), amount: miner_fee });
    let from_account: <T as Config>::WormholeAccountId = mint_account.clone().into();
    let to_account:   <T as Config>::WormholeAccountId = author.clone().into();
    Self::record_transfer(T::AssetId::default(), &from_account, &to_account, miner_fee); // ← add
}
```

Apply the same to the aggregator-rebate credit ([lib.rs:919-934](broken://pages/afdee51ace40897042b5610b30abcc7d636a9a04#L919-L934)). Add regression tests asserting a leaf (and a `TransferCount` advance) for both credits, mirroring the exit-account leaf assertions; the PoC in this report inverts trivially into that regression test. Keep the fee output quantum-aligned (multiple of `SCALE_DOWN_FACTOR`) so no sub-quantum remainder is stranded.

## References

**Vulnerable / relevant code**

* [pallets/wormhole/src/lib.rs](broken://pages/afdee51ace40897042b5610b30abcc7d636a9a04) — miner-fee credit with no leaf (944-965), aggregator rebate (919-934), exit-account credit + leaf (828-859), event-free-mint rationale (822-827), genesis-only leaf snapshot (468-496)
* [pallets/mining-rewards/src/lib.rs:230-237](broken://pages/2e4748fd8965be977a2838e80d9774654801b650#L230-L237) — block reward records a leaf (positive control)
* [primitives/wormhole/src/lib.rs:100-151](broken://pages/e02fce965cfd35e7289eebcec6d6773f196f7343#L100-L151) — keyless wormhole-address derivation + author extraction
* [runtime/src/configs/mod.rs:734-738](broken://pages/52ebeb71568f190c80ca419f5f1e3ccd0e5b91f0#L734-L738) — volume-fee config (miner fee non-zero on every exit)

## Proof of Concept

{% stepper %}
{% step %}

### Add the test

This is a unit test that must live inside the wormhole pallet's test module, because it uses the pallet's mock (`crate::mock::*`), the real proof fixture (`deserialize_test_proof` / `get_test_proof_bytes`), and internal storage (`crate::TransferCount`).

Paste the `#[test] fn poc_miner_volume_fee_has_no_leaf_and_is_permanently_frozen` function below into `pallets/wormhole/src/tests.rs`, inside the existing `mod private_batch_proof_tests { ... }` block (e.g. right after `test_verify_private_batch_emits_miner_volume_fee_paid`). All imports it needs are already in scope there.

Alternatively, apply the companion patch:

```bash
git apply poc_wormhole_miner_fee_frozen.patch
```

{% endstep %}

{% step %}

### Run the test

The first build generates the ZK circuit binaries, \~8-9 min, then they are cached and reruns take \~20 s:

```bash
LIBCLANG_PATH=/usr/lib/llvm-18/lib \
  cargo test -p pallet-wormhole \
  poc_miner_volume_fee_has_no_leaf_and_is_permanently_frozen -- --nocapture
```

{% endstep %}

{% step %}

### Verify the expected output

```
running 1 test
test tests::private_batch_proof_tests::poc_miner_volume_fee_has_no_leaf_and_is_permanently_frozen ... ok
test result: ok. 1 passed; 0 failed; 0 ignored; 0 measured; 71 filtered out; ...
```

After the fix (add `Self::record_transfer(..)` for the miner-fee credit), assertion (2) flips: the tree grows by `num_exit_accounts + 1` and `TransferCount(author)` becomes 1 — i.e. this test then FAILS, which is the regression guard the fix should ship with (invert the two assertions).
{% endstep %}
{% endstepper %}

```rust
/// PoC (audit finding): the miner volume fee is credited to the block author's
/// KEYLESS wormhole address but NO ZK-tree leaf is inserted for it. A wormhole
/// address (`H(H("wormhole"||secret))`) has no signing key, so its only spend
/// path is a wormhole exit — which is impossible without a leaf. The fee is
/// therefore permanently frozen.
///
/// This proves the impact against the REAL `process_exit_bundle`:
///  (1) the fee is genuinely credited to the author's balance (value exists);
///  (2) the exit inserts one leaf per EXIT account but none for the fee
///      (positive control that the exit path does create leaves);
///  (3) the author has no `TransferCount` entry, so it can never build an exit
///      proof for the fee.
/// Contrast: block rewards to the same keyless address DO record a leaf
/// (pallets/mining-rewards/src/lib.rs:232), which is what makes them spendable.
#[test]
fn poc_miner_volume_fee_has_no_leaf_and_is_permanently_frozen() {
    new_test_ext().execute_with(|| {
        let proof = deserialize_test_proof();
        let inputs = parse_private_batch_public_inputs(&proof).expect("Should parse");

        let block_number = inputs.block_data.block_number as u64;
        let block_hash_bytes: [u8; 32] =
            inputs.block_data.block_hash.as_ref().try_into().unwrap();
        frame_system::BlockHash::<Test>::insert(block_number, H256::from(block_hash_bytes));
        System::set_block_number(block_number + 10);

        // Seed a block author (miner) via the pre-runtime digest, exactly like QPoW.
        // The derived address is a KEYLESS wormhole address.
        let miner_preimage = [7u8; 32];
        set_miner_preimage_digest(miner_preimage);
        let author = sp_core::crypto::AccountId32::from(
            qp_wormhole::derive_wormhole_address(miner_preimage)
                .expect("test preimage limbs are canonical"),
        );

        // Non-zero miner fee under the mock config (burn rate 50%).
        let exit_quanta: u128 = inputs
            .account_data
            .iter()
            .filter(|a| a.summed_output_amount > 0)
            .map(|a| a.summed_output_amount as u128)
            .sum();
        let total_fee = super::ceil_volume_fee(exit_quanta, VolumeFeeRateBps::get() as u128);
        let expected_miner_fee = total_fee - total_fee / 2;
        assert!(expected_miner_fee > 0, "fixture must produce a non-zero miner fee");

        // The exit path records exactly one leaf per real exit account.
        let num_exit_accounts = inputs
            .account_data
            .iter()
            .filter(|a| a.summed_output_amount > 0)
            .count() as u64;
        assert!(num_exit_accounts > 0, "fixture must credit >=1 exit (positive control)");

        let leaves_before = ZkTree::leaf_count();
        let author_count_before = crate::TransferCount::<Test>::get(&author);
        let author_balance_before = Balances::free_balance(author.clone());
        assert_eq!(author_count_before, 0, "author must start with no leaf");

        assert_ok!(Wormhole::verify_private_batch(
            RawOrigin::None.into(),
            get_test_proof_bytes()
        ));

        // (1) The fee really was credited to the author's balance — value exists.
        assert_eq!(
            Balances::free_balance(author.clone()),
            author_balance_before + expected_miner_fee,
            "miner fee must be credited to the author's balance"
        );

        // (2) THE BUG: the tree grew by exactly the number of EXIT-account leaves,
        //     with NO extra leaf for the miner-fee credit (a fee leaf would make the
        //     delta num_exit_accounts + 1). Leaves ARE created here — just not for
        //     the fee.
        let leaves_after = ZkTree::leaf_count();
        assert!(leaves_after > leaves_before, "positive control: exit inserts leaves");
        assert_eq!(
            leaves_after - leaves_before,
            num_exit_accounts,
            "exit inserted a leaf per exit account but NONE for the miner fee"
        );

        // (3) The keyless author has no TransferCount entry, so it can never build an
        //     exit proof for the fee: the credited `expected_miner_fee` is unspendable.
        assert_eq!(
            crate::TransferCount::<Test>::get(&author),
            0,
            "no TransferCount/leaf for the author => the miner fee is permanently frozen"
        );
    });
}
```
