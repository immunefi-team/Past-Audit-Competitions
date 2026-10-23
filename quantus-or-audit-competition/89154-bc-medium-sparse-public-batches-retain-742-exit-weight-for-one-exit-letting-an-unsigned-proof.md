# 89154 bc medium sparse public batches retain 742 exit weight for one exit letting an unsigned proof stuff 83 of the normal quota

**Submitted on Aug 17th 2026 at 14:32:19 UTC by @IronsideSec for** [**Audit Comp | Quantus**](https://immunefi.com/audit-competition/audit-comp-quantus)

* **Report ID:** #89154
* **Report Type:** Blockchain/DLT
* **Report severity:** Medium
* **Target:** <https://github.com/immunefi-team/audit-comp-quantus-chain>
* **Impacts:**
  * Block stuffing

## Description

## Brief/Intro

The level-2 Wormhole circuit deliberately permits a public batch containing one real private-batch proof and 52 reusable dummy proofs. The chain correctly skips those 52 inert segments and can settle exactly one real exit. However, `verify_public_batch` always declares storage and ZK-tree work for the fully populated maximum of `2 * 7 * 53 = 742` exits, and every successful sparse execution returns `actual_weight: None`. The all-742-exit declaration is therefore never reduced to the work actually performed.

The resulting unsigned extrinsic carries a fully valid 223,616-byte proof payload and is `Pays::No`, yet declares and retains **3.744227 seconds of call weight, or 83.205% of the 4.5-second Normal block quota**, for one real exit. The same one-exit settlement through the private-batch path reserves 87.137 ms, so wrapping it in the supported public format gives an unprivileged attacker a **42.97x weight-reservation amplifier**. One fresh sparse proof per block reduces available same-priority Wormhole exit capacity from roughly 24 to 8 without malformed proofs, verifier bypasses, a colluding miner, or brute-force traffic.

## Vulnerability Details

### Sparse level-2 batches are an intended, valid proof shape

The frozen production circuit is built with seven leaf proofs per private batch and 53 private batches per public batch. [`build.rs`](https://github.com/immunefi-team/audit-comp-quantus-chain/blob/3b243b870a5442a6e5f443056f67bbba87d472f5/pallets/wormhole/build.rs#L39-L52) sets those defaults:

```rust
let num_leaf_proofs: usize = env::var("QP_NUM_LEAF_PROOFS")
    .unwrap_or_else(|_| "7".to_string())
    .parse()
    .expect("QP_NUM_LEAF_PROOFS must be a valid usize");

let num_private_batch_proofs: usize = env::var("QP_NUM_PRIVATE_BATCH_PROOFS")
    .unwrap_or_else(|_| "53".to_string())
    .parse()
    .expect("QP_NUM_PRIVATE_BATCH_PROOFS must be a valid usize");
```

The corresponding scoped ZK circuit explicitly supports partial public batches. [`circuit_logic.rs:143-156`](https://github.com/immunefi-team/audit-comp-quantus-qp-zk-circuits/blob/2a6914ed0184fbc0aaf621801ac8b8b7e3b01f6e/wormhole/aggregator/src/public_batch/circuit/circuit_logic.rs#L143-L156) says dummy inner proofs are supported for padding, exempt from consistency checks, and have their exits and nullifiers zeroed so the on-chain verifier can skip them. [`circuit_logic.rs:280-313`](https://github.com/immunefi-team/audit-comp-quantus-qp-zk-circuits/blob/2a6914ed0184fbc0aaf621801ac8b8b7e3b01f6e/wormhole/aggregator/src/public_batch/circuit/circuit_logic.rs#L280-L313) enforces the zeroing while retaining the fixed structural layout.

The pallet implements the same design. [`segment_is_inert`](https://github.com/immunefi-team/audit-comp-quantus-chain/blob/3b243b870a5442a6e5f443056f67bbba87d472f5/pallets/wormhole/src/lib.rs#L649-L654) classifies all-zero-nullifier segments as inert. [`process_exit_bundle`](https://github.com/immunefi-team/audit-comp-quantus-chain/blob/3b243b870a5442a6e5f443056f67bbba87d472f5/pallets/wormhole/src/lib.rs#L751-L788) skips every inert segment and zero exit slot:

```rust
for (seg_idx, (segment, valid)) in
    bundle.segments.iter().zip(validity.iter()).enumerate()
{
    if Self::segment_is_inert(segment) {
        continue;
    }
    // ...
    if exit_account_bytes == [0u8; 32] || account_data.summed_output_amount == 0 {
        continue;
    }
}
```

A one-real-exit public batch is consequently normal supported behavior, not a malformed or invalid proof.

### The declaration always charges the fully populated 742-exit tail

The public call uses one fixed declaration:

```rust
#[pallet::weight(<T as Config>::WeightInfo::verify_public_batch())]
pub fn verify_public_batch(
    origin: OriginFor<T>,
    proof_bytes: Vec<u8>,
) -> DispatchResultWithPostInfo {
    ensure_none(origin)?;
    // ...
    Self::process_exit_bundle(bundle)
}
```

See [`pallets/wormhole/src/lib.rs:613-645`](https://github.com/immunefi-team/audit-comp-quantus-chain/blob/3b243b870a5442a6e5f443056f67bbba87d472f5/pallets/wormhole/src/lib.rs#L613-L645).

[`public_batch_max_exits`](https://github.com/immunefi-team/audit-comp-quantus-chain/blob/3b243b870a5442a6e5f443056f67bbba87d472f5/pallets/wormhole/src/weights.rs#L86-L98) returns the structural maximum:

```rust
fn private_batch_max_exits() -> u64 {
    EXITS_PER_LEAF.saturating_mul(circuit_config::NUM_LEAF_PROOFS as u64)
}

fn public_batch_max_exits() -> u64 {
    private_batch_max_exits()
        .saturating_mul(circuit_config::NUM_PRIVATE_BATCH_PROOFS as u64)
}
```

`verify_public_batch()` then scales the storage tail and ZK-tree hash charge by all 742 possible slots, irrespective of how many segments or accounts are real. See [`weights.rs:154-196`](https://github.com/immunefi-team/audit-comp-quantus-chain/blob/3b243b870a5442a6e5f443056f67bbba87d472f5/pallets/wormhole/src/weights.rs#L154-L196):

```rust
fn verify_public_batch() -> Weight {
    let tree_ops = pallet_zk_tree::INSERT_LEAF_DB_OPS;
    let (reads, writes, proof_size) =
        storage_tail(public_batch_max_exits(), true, tree_ops);
    let hash_time = public_batch_max_exits()
        .saturating_mul(pallet_zk_tree::INSERT_LEAF_HASH_REF_TIME_PS);
    Weight::from_parts(
        PUBLIC_BATCH_ZK_VERIFY_REF_TIME_PS.saturating_add(hash_time),
        proof_size,
    )
    .saturating_add(T::DbWeight::get().reads(reads))
    .saturating_add(T::DbWeight::get().writes(writes))
    .saturating_add(Self::pre_validate_public_batch_proof())
    .saturating_add(Self::pre_validate_public_batch_proof())
}
```

Using this as an upfront maximum is safe. The defect is that a successful sparse dispatch never refunds the unexecuted storage/tree tail. The common success return at [`lib.rs:997-998`](https://github.com/immunefi-team/audit-comp-quantus-chain/blob/3b243b870a5442a6e5f443056f67bbba87d472f5/pallets/wormhole/src/lib.rs#L997-L998) is:

```rust
Ok(PostDispatchInfo { actual_weight: None, pays_fee: Pays::No })
```

`actual_weight: None` tells FRAME to retain the entire declared weight. The call is unsigned (`ensure_none`) and `Pays::No`. Pool admission also assigns every Wormhole exit the fixed priority `1`; see [`UNSIGNED_EXIT_PRIORITY`](https://github.com/immunefi-team/audit-comp-quantus-chain/blob/3b243b870a5442a6e5f443056f67bbba87d472f5/pallets/wormhole/src/lib.rs#L19-L28) and the [`verify_public_batch` unsigned-validity branch](https://github.com/immunefi-team/audit-comp-quantus-chain/blob/3b243b870a5442a6e5f443056f67bbba87d472f5/pallets/wormhole/src/lib.rs#L1028-L1039).

### Exact frozen-runtime charge

The frozen runtime permits six seconds of block compute and assigns 75% to Normal dispatches, giving a 4.5-second Normal quota. See [`runtime/src/configs/mod.rs:72-95`](https://github.com/immunefi-team/audit-comp-quantus-chain/blob/3b243b870a5442a6e5f443056f67bbba87d472f5/runtime/src/configs/mod.rs#L72-L95). The call is in the default Normal dispatch class. FRAME derives a Normal `max_extrinsic` of `3,899,460,229,000` ps from this configuration, so the `3,744,227,000,000`-ps call fits as a single extrinsic.

Applying the frozen `WeightInfo` and RocksDB constants gives:

```
maximum exit slots                 2 * 7 * 53 = 742
storage-tail reads                 floor(200*742/32) + 742*52 + 1 = 43,222
storage-tail writes                floor(170*742/32) + 742*21 + 1 = 19,524
public ZK verification             21,077,000,000 ps
742 ZK-tree hash charges           667,800,000,000 ps
DB reads and writes                3,032,950,000,000 ps
2 public pre-validations           22,400,000,000 ps
--------------------------------------------------------------------------
verify_public_batch call weight    3,744,227,000,000 ps = 3.744227 s
Normal max_extrinsic               3,899,460,229,000 ps
Normal quota                       4,500,000,000,000 ps = 4.5 s
Normal quota reserved              83.205044%
verify_private_batch               87,137,000,000 ps = 87.137 ms
public/private reservation ratio   42.9694x
```

The PoC obtains these values by calling the real frozen `WeightInfo` functions rather than reimplementing the arithmetic.

### End-to-end attack path

{% stepper %}
{% step %}

## Create a valid sparse public batch

The attacker deposits to an ordinary Wormhole address they control and creates one valid private-batch proof.

They invoke the supported `PublicBatchAggregator` with that one proof. The remaining 52 slots are filled with the circuit's pinned dummy proof.
{% endstep %}

{% step %}

## Submit the unsigned public-batch extrinsic

They submit the resulting unsigned `verify_public_batch` extrinsic, carrying the 223,616-byte proof payload, before other priority-1 Wormhole exits.
{% endstep %}

{% step %}

## Pass verification and settle one exit

`pre_dispatch` performs full ZK verification. Dispatch then successfully settles the attacker's one exit and skips 52 inert segments.
{% endstep %}

{% step %}

## Retain the maximum declared weight

The success result supplies no `actual_weight`, so the block retains 3.744227 seconds instead of reclaiming the unexecuted storage and tree-insertion tail.
{% endstep %}

{% step %}

## Repeat per target block

The attacker repeats with one fresh leaf/proof in each target block.
{% endstep %}
{% endstepper %}

The scoped aggregator documents public proving at approximately 16 seconds for 53 slots on Apple-Silicon-class hardware; see [`aggregator.rs:14-20`](https://github.com/immunefi-team/audit-comp-quantus-qp-zk-circuits/blob/2a6914ed0184fbc0aaf621801ac8b8b7e3b01f6e/wormhole/aggregator/src/aggregator.rs#L14-L20). Two public-aggregation workers can therefore outpace the 12-second target cadence once the private proofs are available; the preceding proving stages are independently parallelizable, and complete proofs can be prepared ahead of time within the runtime's 4,096-block hash-retention window. Bandwidth is approximately 224 KB per block, or only about 19 KB/s at the target cadence. This is not a brute-force packet or transaction flood.

A smallest-path attacker locks two `0.01 QUAN` input quanta, receives a one-quantum exit, and can direct the configured public-aggregator rebate to themselves. The net volume-fee loss is on the order of one quantum, plus the ordinary deposit transaction fee. For larger exits the configured fee approaches four basis points. Nothing charges the attacker in proportion to the 83% block reservation.

### Why this is distinct from the published Eiger findings

**`EQ-QNT-WORMHOLE-F-02` used invalid proof admission.** It concerned arbitrary/invalid unsigned proof bytes that entered the pool and failed during verification. The frozen code added cheap pre-validation, full verification in `pre_dispatch`, and early-failure post-dispatch corrections. This report's proof is genuine, passes the full ZK inclusion gate, and settles successfully. Invalid-proof filtering and early-failure refunds therefore cannot mitigate it.

**`EQ-QNT-WORMHOLE-R-09` reviewed the old `verify_aggregated_proof` placeholder.** That finding centered on a hand-written `8x` verifier estimate in the pre-level-2 endpoint and did generally suggest scaling its hard-coded storage components with actual exits/nullifiers. The `verify_public_batch` endpoint, 53-inner-proof circuit, inert-segment settlement design, and successful sparse execution path did not yet exist; they were introduced later in July 2026, after the Eiger audit was integrated. The current public ZK-verification base was separately benchmarked, and this report does not dispute that base. The new reachable regression is that this post-audit level-2 endpoint charges the maximum 742-exit **storage/tree tail** after successfully executing only one exit and provides no sparse success refund.

A later pre-freeze hardening commit, [`ca3978fd`](https://github.com/Quantus-Network/chain/commit/ca3978fdce99e230befbd6bef5ec30b7d2d15d8a), also publicly notes that the fixture has one real segment plus 52 dummies and changes the **pre-validation compute floor** to the all-valid worst case. That fix does not add a successful `actual_weight`, does not make the public batch require 53 real segments, and does not address the dominant 742-exit storage/tree tail demonstrated here. The report therefore treats this as a success-refund bypass left after the all-valid admission-metering hardening, not as a claim that worst-case upfront declaration is itself wrong.

This is a distinct new endpoint and successful attack path rather than a resubmission of malformed-proof admission or the deprecated placeholder timing. The general R-09 recommendation and the closest sparse-fixture hardening are both disclosed here so triage can assess the distinction on the complete public record.

### Recommended remediation

Keep the all-742-exit declaration as the safe pre-dispatch maximum, but calculate successful weight from work actually performed:

{% stepper %}
{% step %}

## Retain a fixed base

Retain a fixed base for deserialization, public-input parsing, ZK verification, and fixed-layout processing.
{% endstep %}

{% step %}

## Count actual work

Count the non-inert valid segments, real exit accounts, nullifiers, and ZK-tree inserts actually processed.
{% endstep %}

{% step %}

## Return successful actual weight

Return `PostDispatchInfo { actual_weight: Some(base + actual_tail), pays_fee: Pays::No }` on success.
{% endstep %}

{% step %}

## Add a sparse-fixture regression

Add the existing one-real-leaf fixture as a regression and require its successful weight to be far below the all-742-exit maximum.
{% endstep %}
{% endstepper %}

Alternatively, enforce a minimum fill ratio for the public path and direct sparse users to the existing private-batch endpoint.

## Impact Details

### Selected in-scope impact

**Medium — “Block stuffing.”** This exact impact appears in the Quantus Audit Competition's [Impacts in Scope](https://immunefi.com/audit-competition/audit-comp-quantus/scope/).

The checked-in public fixture is 223,616 bytes and contains exactly one real exit. A private fixture is 157,500 bytes. The Normal length allowance is 75% of 5 MiB, so an otherwise empty block can carry approximately 24 one-exit private proofs by length; compute would allow more. After a sparse public proof reserves 3.744227 seconds, only 755.773 ms remains, enough for at most eight 87.137-ms private proofs. Available same-priority Wormhole capacity therefore falls from roughly 24 to 8, a **66.7% reduction**. Including the attacker's own exit, the block processes about nine exits instead of 24.

One sparse public proof fits; two cannot fit the Normal weight quota. Once selected early in a block with sufficient Normal headroom, its displacement is deterministic rather than dependent on saturating a network interface or racing hundreds of transactions. Each accepted proof occupies most of the block's Normal compute accounting even though its 52 dummy segments perform none of the priced storage writes or tree insertions.

Ordinary signed transactions normally have payment-derived priority greater than the Wormhole priority of 1. This report therefore does **not** claim total network shutdown, a 500%-delayed block, or categorical censorship of higher-priority signed calls. The directly demonstrated impact is fee-free block stuffing and severe displacement of other same-priority Wormhole exits.

The same result also demonstrates a greater-than-50% successful-path weight overcharge, but “Block stuffing” is selected because it captures the concrete adversarial effect without depending on an estimate of the exact sparse-path benchmark.

## References

* [Quantus competition scope and impacts](https://immunefi.com/audit-competition/audit-comp-quantus/scope/)
* [Scoped Quantus chain asset](https://github.com/immunefi-team/audit-comp-quantus-chain)
* [Frozen chain commit](https://github.com/immunefi-team/audit-comp-quantus-chain/tree/3b243b870a5442a6e5f443056f67bbba87d472f5)
* [Frozen ZK-circuit commit](https://github.com/immunefi-team/audit-comp-quantus-qp-zk-circuits/tree/2a6914ed0184fbc0aaf621801ac8b8b7e3b01f6e)
* [Production circuit sizes](https://github.com/immunefi-team/audit-comp-quantus-chain/blob/3b243b870a5442a6e5f443056f67bbba87d472f5/pallets/wormhole/build.rs#L39-L52)
* [Public-batch dummy-padding constraints](https://github.com/immunefi-team/audit-comp-quantus-qp-zk-circuits/blob/2a6914ed0184fbc0aaf621801ac8b8b7e3b01f6e/wormhole/aggregator/src/public_batch/circuit/circuit_logic.rs#L143-L156)
* [Public-batch output zeroing](https://github.com/immunefi-team/audit-comp-quantus-qp-zk-circuits/blob/2a6914ed0184fbc0aaf621801ac8b8b7e3b01f6e/wormhole/aggregator/src/public_batch/circuit/circuit_logic.rs#L280-L313)
* [Public-batch call](https://github.com/immunefi-team/audit-comp-quantus-chain/blob/3b243b870a5442a6e5f443056f67bbba87d472f5/pallets/wormhole/src/lib.rs#L613-L645)
* [Inert-segment classification](https://github.com/immunefi-team/audit-comp-quantus-chain/blob/3b243b870a5442a6e5f443056f67bbba87d472f5/pallets/wormhole/src/lib.rs#L649-L654)
* [Inert-segment and zero-account dispatch skips](https://github.com/immunefi-team/audit-comp-quantus-chain/blob/3b243b870a5442a6e5f443056f67bbba87d472f5/pallets/wormhole/src/lib.rs#L751-L788)
* [Maximum-exit and storage-tail formula](https://github.com/immunefi-team/audit-comp-quantus-chain/blob/3b243b870a5442a6e5f443056f67bbba87d472f5/pallets/wormhole/src/weights.rs#L86-L130)
* [Fixed public-batch weight](https://github.com/immunefi-team/audit-comp-quantus-chain/blob/3b243b870a5442a6e5f443056f67bbba87d472f5/pallets/wormhole/src/weights.rs#L154-L196)
* [No successful actual-weight correction](https://github.com/immunefi-team/audit-comp-quantus-chain/blob/3b243b870a5442a6e5f443056f67bbba87d472f5/pallets/wormhole/src/lib.rs#L997-L998)
* [Normal block limits](https://github.com/immunefi-team/audit-comp-quantus-chain/blob/3b243b870a5442a6e5f443056f67bbba87d472f5/runtime/src/configs/mod.rs#L72-L95)
* [Checked-in sparse fixture assertions](https://github.com/immunefi-team/audit-comp-quantus-chain/blob/3b243b870a5442a6e5f443056f67bbba87d472f5/pallets/wormhole/src/tests.rs#L1888-L1977)
* [Checked-in sparse fixture success](https://github.com/immunefi-team/audit-comp-quantus-chain/blob/3b243b870a5442a6e5f443056f67bbba87d472f5/pallets/wormhole/src/tests.rs#L2000-L2021)
* [Public-batch proving cost](https://github.com/immunefi-team/audit-comp-quantus-qp-zk-circuits/blob/2a6914ed0184fbc0aaf621801ac8b8b7e3b01f6e/wormhole/aggregator/src/aggregator.rs#L14-L20)

## Proof of Concept

### What the PoC proves

The scoped repository already commits a real Plonky2 public-batch proof whose test documentation states that it contains one real private batch, itself containing one real leaf, plus dummy private-batch padding. Existing assertions establish 742 structural exit slots, exactly one real exit, and only one non-inert segment. The existing success test settles that fixture.

The single-file regression below executes the same authentic proof and additionally proves that:

* its serialized proof is 223,616 bytes and its public inputs contain exactly one real exit;
* it passes permissionless external transaction-pool validation at the fixed priority `1` and is propagated;
* it passes `ValidateUnsigned::pre_dispatch`, which invokes the full public-batch ZK verifier;
* dispatch then settles exactly one ZK-tree exit leaf with `actual_weight: None` and `Pays::No`;
* the frozen RocksDB weight implementation charges exactly `3,744,227,000,000` ps;
* this reserves more than 83% of the production Normal quota; and
* it is more than 42 times the private-batch reservation.

The patch adds only one pallet-externalities test to `pallets/wormhole/src/tests.rs`. It does not modify production logic, mock or replace the verifier, or alter the committed proof fixture. It injects the fixture's referenced block hash rather than running a live node; the block-capacity calculation comes directly from the frozen production weight and block-limit implementations.

### Reproduction — one patch file against the scoped repository

{% stepper %}
{% step %}

## Clone the scoped asset

Clone the scoped asset and check out the exact frozen commit:

```bash
git clone https://github.com/immunefi-team/audit-comp-quantus-chain.git
cd audit-comp-quantus-chain
git checkout --detach 3b243b870a5442a6e5f443056f67bbba87d472f5
```

{% endstep %}

{% step %}

## Create the patch file

Create the following file in the repository root and paste the complete unified diff from the end of this section into it:

```
poc-medium-sparse-public-batch-block-stuffing.patch
```

{% endstep %}

{% step %}

## Apply and run the PoC

Confirm that the patch applies cleanly, apply it, and run only the PoC test:

```bash
git apply --check poc-medium-sparse-public-batch-block-stuffing.patch
git apply poc-medium-sparse-public-batch-block-stuffing.patch
cargo test -p pallet-wormhole --lib \
  poc_sparse_public_batch_stuffs_normal_block_weight -- \
  --nocapture --test-threads=1
```

{% endstep %}
{% endstepper %}

`--test-threads=1` only makes the printed evidence deterministic. It does not change runtime state or any assertion path. The repository's pinned Rust toolchain is selected automatically; the first build may also generate the pinned circuits.

### Expected result

The significant output is:

```
running 1 test
test tests::public_batch_proof_tests::poc_sparse_public_batch_stuffs_normal_block_weight ... real_exits=1 proof_bytes=223616 pool_priority=1 full_zk_gate=ok settled_leaf_delta=1 charged_ref_time=3744227000000 normal_quota=4500000000000 percent=83.20504444444444 private_ref_time=87137000000 amplifier_x=42.969427453320634
ok

test result: ok. 1 passed; 0 failed; 0 ignored; 0 measured; 71 filtered out
```

The output and assertions establish that the committed one-exit proof is admitted through the unsigned pool path, passes the full ZK inclusion gate, and settles exactly one exit leaf without a fee or successful-path weight correction. The production RocksDB formula consequently retains 83.205% of the Normal quota even though 52 inner segments are inert.

### Complete single-file PoC — `poc-medium-sparse-public-batch-block-stuffing.patch`

Copy this file exactly as shown:

```diff
diff --git a/pallets/wormhole/src/tests.rs b/pallets/wormhole/src/tests.rs
index 4473b706..f6c56923 100644
--- a/pallets/wormhole/src/tests.rs
+++ b/pallets/wormhole/src/tests.rs
@@ -1977,6 +1977,63 @@ mod public_batch_proof_tests {
 		);
 	}
 
+	/// PoC: the supported one-real-exit public proof passes pool and inclusion gates,
+	/// but successful dispatch gets no sparse refund and reserves >83% of Normal weight.
+	#[test]
+	fn poc_sparse_public_batch_stuffs_normal_block_weight() {
+		use frame_support::pallet_prelude::ValidateUnsigned;
+		use sp_runtime::transaction_validity::TransactionSource;
+
+		new_test_ext().execute_with(|| {
+			let inputs = parse_test_inputs();
+			setup_matching_block_state(&inputs);
+
+			let real_slots =
+				inputs.account_data.iter().filter(|a| a.summed_output_amount > 0).count();
+			assert_eq!(real_slots, 1, "fixture must settle exactly one real exit");
+
+			let proof_bytes = get_test_proof_bytes();
+			assert_eq!(proof_bytes.len(), 223_616);
+			let call = crate::Call::<Test>::verify_public_batch { proof_bytes: proof_bytes.clone() };
+
+			let validity = <Wormhole as ValidateUnsigned>::validate_unsigned(
+				TransactionSource::External,
+				&call,
+			)
+			.expect("the sparse proof must enter the unsigned pool");
+			assert_eq!(validity.priority, crate::UNSIGNED_EXIT_PRIORITY);
+			assert!(validity.propagate);
+			<Wormhole as ValidateUnsigned>::pre_dispatch(&call)
+				.expect("the genuine proof must pass the full ZK inclusion gate");
+
+			let leaves_before = ZkTree::leaf_count();
+			let post = Wormhole::verify_public_batch(RawOrigin::None.into(), proof_bytes)
+				.expect("the sparse public batch must settle successfully");
+			assert_eq!(post.actual_weight, None, "success reclaims no sparse storage tail");
+			assert_eq!(post.pays_fee, frame_support::dispatch::Pays::No);
+			let leaves_after = ZkTree::leaf_count();
+			assert_eq!(leaves_after, leaves_before + 1, "exactly one exit leaf must settle");
+
+			let charged = <() as crate::weights::WeightInfo>::verify_public_batch();
+			let private = <() as crate::weights::WeightInfo>::verify_private_batch();
+			let normal_quota = 4_500_000_000_000u64;
+			println!(
+				"real_exits={real_slots} proof_bytes={} pool_priority={} full_zk_gate=ok settled_leaf_delta={} charged_ref_time={} normal_quota={} percent={} private_ref_time={} amplifier_x={}",
+				223_616,
+				validity.priority,
+				leaves_after - leaves_before,
+				charged.ref_time(),
+				normal_quota,
+				charged.ref_time() as f64 * 100.0 / normal_quota as f64,
+				private.ref_time(),
+				charged.ref_time() as f64 / private.ref_time() as f64,
+			);
+			assert_eq!(charged.ref_time(), 3_744_227_000_000);
+			assert!(charged.ref_time() * 100 > normal_quota * 83);
+			assert!(charged.ref_time() > private.ref_time() * 42);
+		});
+	}
+
 	#[test]
 	fn test_verify_public_batch_fails_with_wrong_origin() {
 		new_test_ext().execute_with(|| {
```
