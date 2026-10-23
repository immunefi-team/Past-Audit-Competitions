# 88300 bc high missing wormhole zk leaf permanently freezes all miner exit fee yield

**Submitted on Aug 12th 2026 at 17:11:50 UTC by @Porcupine17126 for** [**Audit Comp | Quantus**](https://immunefi.com/audit-competition/audit-comp-quantus)

* **Report ID:** #88300
* **Report Type:** Blockchain/DLT
* **Report severity:** High
* **Target:** <https://github.com/immunefi-team/audit-comp-quantus-chain>
* **Impacts:**
  * Permanent freezing of unclaimed yield

## Description

## Brief / Introduction

Every successful Wormhole exit allocates part of its volume fee to the current block author. The runtime increases the author's native balance and emits `MinerVolumeFeePaid`, but it does not call `Wormhole::record_transfer`. Production block-author reward accounts are keyless Wormhole addresses, so a committed ZK-tree leaf is the only usable claim to the credited value. The miner therefore receives an on-chain balance that can never be spent or withdrawn.

The production parameters charge 4 bps on Wormhole exit volume and allocate 50% to the miner. Approximately 2 bps of every successful Wormhole exit is permanently frozen (with the circuit's one-quantum minimum affecting small exits).

Scoped commit: `3b243b870a5442a6e5f443056f67bbba87d472f5`

## Vulnerability Details

### The block author is a Wormhole address

Production validators must provide `--rewards-inner-hash`. The node validates it as a canonical Poseidon digest and stores that inner digest in the QPoW pre-runtime log. The runtime derives the author account as:

```
author = Poseidon(inner_digest)
       = Poseidon(Poseidon(salt || secret))
```

This is a Wormhole address, not an ordinary ML-DSA signing account:

* [node/src/command.rs#L590-L662](https://github.com/Quantus-Network/chain/blob/3b243b870a5442a6e5f443056f67bbba87d472f5/node/src/command.rs#L590-L662)
* [primitives/wormhole/src/lib.rs#L97-L110](https://github.com/Quantus-Network/chain/blob/3b243b870a5442a6e5f443056f67bbba87d472f5/primitives/wormhole/src/lib.rs#L97-L110)
* [primitives/wormhole/src/lib.rs#L113-L148](https://github.com/Quantus-Network/chain/blob/3b243b870a5442a6e5f443056f67bbba87d472f5/primitives/wormhole/src/lib.rs#L113-L148)

### Settlement credits a balance but omits the leaf

The exit settlement credits the miner with event-free `Unbalanced::increase_balance` and emits `MinerVolumeFeePaid`:

* [pallets/wormhole/src/lib.rs#L937-L975](https://github.com/Quantus-Network/chain/blob/3b243b870a5442a6e5f443056f67bbba87d472f5/pallets/wormhole/src/lib.rs#L937-L975)

The success branch never invokes `Self::record_transfer`. No other mechanism repairs the omission:

1. `increase_balance` emits no `Balances::Minted` or transfer event for `WormholeProofRecorderExtension` to scan.
2. `verify_private_batch` and `verify_public_batch` are bare unsigned extrinsics, so the signed-transaction extension does not run for settlement.

The adjacent production paths demonstrate the required invariant:

* Each successfully minted Wormhole exit explicitly calls `Self::record_transfer`: [pallets/wormhole/src/lib.rs#L816-L859](https://github.com/Quantus-Network/chain/blob/3b243b870a5442a6e5f443056f67bbba87d472f5/pallets/wormhole/src/lib.rs#L816-L859).
* Each block mining reward explicitly calls `ProofRecorder::record_transfer_proof`: [pallets/mining-rewards/src/lib.rs#L220-L240](https://github.com/Quantus-Network/chain/blob/3b243b870a5442a6e5f443056f67bbba87d472f5/pallets/mining-rewards/src/lib.rs#L220-L240).

`record_transfer` increments `TransferCount`, inserts `(recipient, transfer_count, asset_id, amount)` into the Poseidon Merkle tree, and exposes the leaf index needed for a proof:

* [pallets/wormhole/src/lib.rs#L1273-L1328](https://github.com/Quantus-Network/chain/blob/3b243b870a5442a6e5f443056f67bbba87d472f5/pallets/wormhole/src/lib.rs#L1273-L1328)

Knowing the Wormhole secret cannot compensate for a missing leaf. The secret proves ownership of a leaf already committed to a historical root; it cannot synthesize an omitted leaf or witness.

## Impact Details

Any valid private- or public-batch Wormhole exit included by a normal production validator reaches this path. No malicious validator, privilege, malformed proof, Sybil behavior, or race is required.

The post-state is inconsistent:

```
Balances::free_balance(author) += miner_fee
MinerVolumeFeePaid(author, miner_fee) is emitted
Wormhole::TransferCount[author] is unchanged
No ZK leaf for (author, miner_fee) exists
```

The miner cannot sign from the derived Wormhole address and cannot construct a Wormhole withdrawal proof without the omitted leaf. The entire miner share is permanently frozen as unclaimed mining yield.

At the scoped runtime parameters:

```
volume fee = exit amount * 4 bps
miner share = volume fee * 50%
frozen miner yield ~= exit amount * 2 bps
```

See [runtime/src/configs/mod.rs#L728-L752](https://github.com/Quantus-Network/chain/blob/3b243b870a5442a6e5f443056f67bbba87d472f5/runtime/src/configs/mod.rs#L728-L752).

Later correctly recorded mining rewards do not repair the missing credit: those leaves commit only their own amounts and transfer counts, not the stranded earlier fee.

## Suggested Remediation

After successful miner `increase_balance`, explicitly record the native-token credit before emitting `MinerVolumeFeePaid`:

```rust
let from: T::WormholeAccountId = T::MintingAccount::get().into();
let to: T::WormholeAccountId = author.clone().into();
Self::record_transfer(T::AssetId::default(), &from, &to, miner_fee);
```

Update settlement weights for the additional ZK-tree insertion. Retaining event-free `increase_balance` is appropriate; changing only to `mint_into` would rely on an event-scanning extension that does not execute for these bare unsigned settlement extrinsics.

## Note on commit 60ec991

Commit `60ec991` added `MinerVolumeFeePaid` because the event-free credit was ambiguous to indexers. That change improves visibility but does not create a ZK-tree leaf. It consequently reports the keyless address as paid while the value remains unclaimable; it does not fix or document this permanent-freeze impact.

## Proof of Concept

This PoC extends the repository's existing valid-private-batch test. The original test already loads a real valid proof fixture, sets the matching block hash and QPoW author digest, executes `verify_private_batch`, and verifies both the nonzero fee balance increase and `MinerVolumeFeePaid` event.

The added invariant captures `Wormhole::transfer_count(author)` before settlement and proves it remains unchanged afterward, even though the balance was credited.

{% stepper %}
{% step %}

## Clone the scoped repository

Clone the scoped chain repository and check out the exact scoped commit:

```bash
git clone https://github.com/immunefi-team/audit-comp-quantus-chain
cd audit-comp-quantus-chain
git checkout 3b243b870a5442a6e5f443056f67bbba87d472f5
```

{% endstep %}

{% step %}

## Apply the patch

```diff
diff --git a/pallets/wormhole/src/tests.rs b/pallets/wormhole/src/tests.rs
index 4473b70..fde2908 100644
--- a/pallets/wormhole/src/tests.rs
+++ b/pallets/wormhole/src/tests.rs
@@ -1072,6 +1072,7 @@ mod private_batch_proof_tests {
             assert!(expected_miner_fee > 0, "fixture should produce a non-zero miner fee");

             let author_balance_before = Balances::free_balance(expected_author.clone());
+            let author_transfer_count_before = Wormhole::transfer_count(expected_author.clone());

             assert_ok!(Wormhole::verify_private_batch(
                 RawOrigin::None.into(),
@@ -1086,9 +1087,14 @@ mod private_batch_proof_tests {
                 .into(),
             );
             assert_eq!(
-                Balances::free_balance(expected_author),
+                Balances::free_balance(expected_author.clone()),
                 author_balance_before + expected_miner_fee
             );
+            assert_eq!(
+                Wormhole::transfer_count(expected_author),
+                author_transfer_count_before,
+                "the miner fee was credited to a keyless wormhole address but no spendable ZK leaf was recorded"
+            );
         });
     }
```

{% endstep %}

{% step %}

## Run the focused PoC

```bash
cargo test -p pallet-wormhole test_verify_private_batch_emits_miner_volume_fee_paid -- --nocapture
```

Observed result:

```
running 1 test
test tests::private_batch_proof_tests::test_verify_private_batch_emits_miner_volume_fee_paid ... ok

test result: ok. 1 passed; 0 failed; 0 ignored; 0 measured; 70 filtered out
```

{% endstep %}

{% step %}

## Run the full pallet regression suite

```bash
cargo test -p pallet-wormhole
```

Observed result:

```
test result: ok. 68 passed; 0 failed; 3 ignored
```

{% endstep %}
{% endstepper %}

### What the PoC proves

The test verifies all of the following in one successful valid-proof execution:

* `expected_miner_fee > 0`.
* The QPoW-derived author's free balance increases by exactly that fee.
* `MinerVolumeFeePaid` reports the exact author and fee.
* The same author's `Wormhole::TransferCount` does not advance.

Since `record_transfer` is the only path that advances the counter and inserts the spendable ZK leaf, the credited fee has no witness in the committed tree and is permanently unclaimable.
