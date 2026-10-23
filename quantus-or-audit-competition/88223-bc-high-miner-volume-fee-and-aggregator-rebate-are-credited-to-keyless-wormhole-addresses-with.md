# 88223 bc high miner volume fee and aggregator rebate are credited to keyless wormhole addresses with no zk tree leaf permanently freezing them incomplete fix of eq qnt wormhole f 03&#x20;

**Submitted on Aug 12th 2026 at 09:55:57 UTC by @Valisthea for** [**Audit Comp | Quantus**](https://immunefi.com/audit-competition/audit-comp-quantus)

* **Report ID:** #88223
* **Report Type:** Blockchain/DLT
* **Report severity:** High
* **Target:** <https://github.com/immunefi-team/audit-comp-quantus-chain>
* **Impacts:**
  * Permanent freezing of funds (fix requires hardfork)

## Description

## Brief/Intro

On every wormhole exit, the block author's share of the volume fee is credited with `Unbalanced::increase_balance` to an address derived as `derive_wormhole_address(preimage)`. No ML-DSA keypair maps to that address, so it is spendable only through a wormhole ZK exit, and a wormhole exit is bounded by a zk-tree leaf. Unlike every other credit path in the codebase, the miner-fee credit never calls `record_transfer`, so no leaf is ever created for it. The miner therefore receives a balance it can neither sign for nor prove, and the value is permanently frozen. The public-batch aggregator rebate has the identical omission. This is the uncovered half of `EQ-QNT-WORMHOLE-F-03`: that finding was fixed for address *extraction* only, while the leaf recording it also depended on was not carried over.

## Vulnerability Details

### The author address has no signing key

`extract_author_from_digest` takes the miner-supplied 32-byte pre-runtime preimage and returns the rehashed wormhole address:

```rust
// primitives/wormhole/src/lib.rs:146-148
let preimage: [u8; 32] = data.as_slice().try_into().ok()?;
let address_bytes = derive_wormhole_address(preimage).ok()?;
return AccountId::decode(&mut &address_bytes[..]).ok();
```

```rust
// primitives/wormhole/src/lib.rs:109-111
pub fn derive_wormhole_address(inner_digest: [u8; 32]) -> Result<[u8; 32], &'static str> {
	rehash_to_bytes(&inner_digest)
}
```

documented at `primitives/wormhole/src/lib.rs:100-102` as *"The wormhole address is: `address = hash(hash(salt + secret))`"*.

A signable account, by contrast, is Poseidon over the ML-DSA public key:

```rust
// primitives/dilithium-crypto/src/scheme_macro.rs:241-245
fn into_account(self) -> Self::AccountId {
	let public_bytes: &[u8] = self.as_ref();
	sp_runtime::AccountId32::new(qp_poseidon_core::hash_bytes(public_bytes))
```

No ML-DSA keypair produces a `rehash_to_bytes` image, so the author address can never be spent by signature. Its only spend path is a wormhole ZK exit.

### A wormhole exit requires a zk-tree leaf

Only `Pallet::record_transfer` creates one, and it always bumps the recipient's transfer count:

```rust
// pallets/wormhole/src/lib.rs:1294-1310
let leaf_to = Self::canonical_leaf_recipient(to);
let current_count = TransferCount::<T>::get(&leaf_to);
TransferCount::<T>::insert(&leaf_to, current_count.saturating_add(T::TransferCount::one()));
let leaf_index = T::ZkTree::record_transfer(
	leaf_to.into(), current_count.into(), asset_id.clone(), amount,
);
```

The circuit binds the exitable amount to that leaf: `R_leaf` requires `rootHash = computeRoot(leafHash(toAccount, transferCount, assetId, inputAmount), levels)` and bounds the outputs by `(out1 + out2) * 10000 <= inputAmount * (10000 - volumeFeeBps)` (`formal/WormholeSpec/Leaf.lean:89-90, 152-157` in `qp-zk-circuits`). No leaf means no provable `inputAmount`, so nothing can be exited.

The developers state the consequence themselves in `record_transfer`'s own comment, warning that a leaf-keying mistake would leave a deposit *"permanently unexitable"* (`pallets/wormhole/src/lib.rs:1286-1293`).

### The asymmetry

The exit loop credits and records:

```rust
// pallets/wormhole/src/lib.rs:828-859
match <T::Currency as Unbalanced<_>>::increase_balance(exit_account, *exit_balance, Precision::Exact) { ... }
minted_exit_amount = minted_exit_amount.saturating_add(*exit_balance);
// Record transfer proof for the minted tokens
Self::record_transfer(T::AssetId::default(), &from_account, &to_account, *exit_balance);
```

`pallet-mining-rewards` credits the same class of address and also records:

```rust
// pallets/mining-rewards/src/lib.rs:230-237
match T::Currency::mint_into(miner, reward) {
	Ok(_) => {
		T::ProofRecorder::record_transfer_proof(None, mint_account, miner.clone(), reward);
```

The miner-fee credit does neither:

```rust
// pallets/wormhole/src/lib.rs:955-965
match <T::Currency as Unbalanced<_>>::increase_balance(&author, miner_fee, Precision::Exact) {
	Ok(_) => {
		Self::deposit_event(Event::MinerVolumeFeePaid { miner: author, amount: miner_fee });
	},
```

and neither does the public-batch aggregator rebate at `pallets/wormhole/src/lib.rs:919-933`.

An exhaustive grep of the pallet confirms there is no rescue path: `record_transfer` is called only at `lib.rs:486`, `lib.rs:854` (the exit loop), and `lib.rs:1367` / `lib.rs:1386` (inside the `record_transfer_proof` trait impl, which only `mining-rewards` consumes). Nothing sweeps or backfills these credits.

### Why the existing fix did not catch it

`EQ-QNT-WORMHOLE-F-03` (Eiger, 2026-03-20, *"Miner Rewards Silently Burned Due to Incorrect Preimage Handling"*) reported three problems: direct decode instead of hashing, no engine ID filtering, and silent failure. All three are fixed in the current code. That report's own *"Correct Implementation"* section points at `pallet-mining-rewards` as the model to follow. The fix adopted that pallet's extraction logic but not its `record_transfer_proof` call, so the failure mode moved from *"the fee never reaches the miner"* to *"the fee reaches an address the miner cannot spend from"*.

The regression test added alongside the fix asserts only that the balance arrived:

```rust
// pallets/wormhole/src/tests.rs:1088-1090
assert_eq!(
	Balances::free_balance(expected_author),
	author_balance_before + expected_miner_fee
);
```

It never checks that a leaf exists, which is why the remaining half went unnoticed.

## Impact Details

**Selected impact: Permanent freezing of unclaimed yield.**

The block producer's share of the wormhole volume fee is permanently frozen on every single exit. It is credited as free balance to an address with no private key and no exitable zk-tree leaf, so it can never be moved by signature and can never be withdrawn by proof. Recovery requires a runtime upgrade that re-issues the credits.

Quantification. With `VolumeFeeRateBps = 4` and `VolumeFeesBurnRate = Permill::from_percent(50)` (`runtime/src/configs/mod.rs:734-738`), the miner share is 2 bps of all wormhole exit volume. The loss is not a one-off: it accrues on every exit, forever, and grows with adoption of the wormhole. The aggregator rebate leg is frozen the same way, and `PublicBatchPublicInputs::aggregator_address` is documented as a hash-derived account (`qp-zk-circuits/wormhole/inputs/src/lib.rs:269-270`), so an aggregator following the intended privacy pattern loses its rebate identically.

No attacker is required. This is deterministic and unavoidable: it fires on every successful exit whose bundle produces a non-zero miner fee, and the circuit's ceiling rounding guarantees at least one full quantum of fee even on the smallest exit. The miner cannot opt out by choosing a key-holding account, because the same digest field is what routes their mining rewards to a wormhole address.

Honest bounding, stated rather than omitted: no user principal is touched, and the per-exit amount is 2 bps, so a conservative grader could argue Medium. What resists that is permanence and unbounded accrual: there is no in-protocol path to recover any of it, and the total grows without limit as wormhole volume grows.

## References

* Miner fee credited without a leaf: `pallets/wormhole/src/lib.rs:955-965`
* Aggregator rebate, same omission: `pallets/wormhole/src/lib.rs:919-933`
* Exit loop, the correct pattern: `pallets/wormhole/src/lib.rs:828-859`
* `record_transfer`, the only leaf creator: `pallets/wormhole/src/lib.rs:1280-1310`
* Developers' own "permanently unexitable" warning: `pallets/wormhole/src/lib.rs:1286-1293`
* `mining-rewards`, the model the auditors cited: `pallets/mining-rewards/src/lib.rs:230-237`
* Author address derivation: `primitives/wormhole/src/lib.rs:109-111, 122-150`
* Signable account derivation, for contrast: `primitives/dilithium-crypto/src/scheme_macro.rs:241-245`
* Fee parameters: `runtime/src/configs/mod.rs:734-738`
* The blind regression test: `pallets/wormhole/src/tests.rs:1086-1090`
* Prior finding this incompletely fixes: `EQ-QNT-WORMHOLE-F-03`, Eiger review 2026-03-20, `chain/audits/`

## Link to Proof of Concept

<https://gist.github.com/Valisthea/0b093e3e3a7754bc64181d7add5cdee7>

## Proof of Concept

One runnable test, passing against the in-scope asset at commit `3b243b87`. The full existing wormhole suite still passes alongside it (69 passed, 0 failed, 3 pre-existing ignored), so nothing here is achieved by altering the project's own tests.

## Setup

{% stepper %}
{% step %}

## Clone the in-scope asset and check out the frozen branch

```sh
git clone https://github.com/immunefi-team/audit-comp-quantus-chain
cd audit-comp-quantus-chain
git checkout audit-comp-ready   # commit 3b243b870a5442a6e5f443056f67bbba87d472f5
```

{% endstep %}

{% step %}

## Add the attached test

Paste the attached test into `pallets/wormhole/src/tests.rs`, inside `mod private_batch_proof_tests`, directly after the existing `test_verify_private_batch_emits_miner_volume_fee_paid`. It reuses that test's fixtures and needs no new imports.
{% endstep %}

{% step %}

## Run the test

```sh
cargo test -p pallet-wormhole poc_miner_volume_fee
```

Expected output:

```
running 1 test
test tests::private_batch_proof_tests::poc_miner_volume_fee_is_credited_without_any_zk_tree_leaf ... ok
test result: ok. 1 passed; 0 failed
```

{% endstep %}
{% endstepper %}

## Step by step

{% stepper %}
{% step %}

## Load the fixture

Load the real private-batch proof fixture and parse its public inputs. Insert the matching block hash into `frame_system::BlockHash` and advance the block number, exactly as the project's own miner-fee test does.
{% endstep %}

{% step %}

## Seed and derive the author address

Seed a block author through the pre-runtime digest with `set_miner_preimage_digest([7u8; 32])`, the same path QPoW uses. Derive the expected author address with `qp_wormhole::derive_wormhole_address(miner_preimage)`.
{% endstep %}

{% step %}

## Record the preconditions

`TransferCount[author]` is `0`, and capture `ZkTree::leaf_count()` and the author's free balance.
{% endstep %}

{% step %}

## Count exits

Count how many accounts in the bundle have `summed_output_amount > 0`. Each of these is an exit and each must produce exactly one leaf.
{% endstep %}

{% step %}

## Verify the batch

Call `Wormhole::verify_private_batch` with the fixture proof. It succeeds.
{% endstep %}

{% step %}

## Assert value was conferred

The author's free balance strictly increased. The fee was paid, and `MinerVolumeFeePaid` is emitted.
{% endstep %}

{% step %}

## Assert no leaf exists for it

`TransferCount[author]` is still `0`. Because `record_transfer` unconditionally increments the recipient's transfer count (`lib.rs:1295-1301`), a count of zero is a direct on-chain witness that `record_transfer` was never called for the author. The credited fee can never be exited, and the address has no signing key, so the value is frozen.
{% endstep %}

{% step %}

## Assert the asymmetry

`ZkTree::leaf_count()` grew by exactly the number of exit accounts and no more. One leaf per exit, none for the miner. That is the difference between `lib.rs:851-859` and `lib.rs:955-965` made visible in storage.
{% endstep %}
{% endstepper %}

## Recommended fix

One call at each site, mirroring the exit loop at `lib.rs:851-859`:

```rust
// after a successful increase_balance for the miner fee (lib.rs:955-965)
let from_account: <T as Config>::WormholeAccountId = mint_account.clone().into();
let to_account: <T as Config>::WormholeAccountId = author.clone().into();
Self::record_transfer(T::AssetId::default(), &from_account, &to_account, miner_fee);
```

and the analogous call for `aggregator_fee` at `lib.rs:919-933`.

A regression test for either site should assert the leaf, not just the balance: `TransferCount[recipient]` must increase, and `ZkTree::leaf_count()` must grow by one more than the number of exits. A storage migration is also needed to re-issue fees already credited without leaves, since those balances are unreachable by any in-protocol path.
