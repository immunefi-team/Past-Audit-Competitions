# 88568 bc low missing collector storage weight in transactionfeescollector leads to a whole extrinsic undercharge of at least 16 7160&#x20;

**Submitted on Aug 13th 2026 at 21:03:40 UTC by @v4rvl for** [**Audit Comp | Quantus**](https://immunefi.com/audit-competition/audit-comp-quantus)

* **Report ID:** #88568
* **Report Type:** Blockchain/DLT
* **Report severity:** Low
* **Target:** <https://github.com/immunefi-team/audit-comp-quantus-chain>
* **Impacts:**
  * Inaccuracies to the weight (similar to gas) accounting system that under or overcharge by less than 50% of the correct amount (and are not intentional as indicated by the program's terms

## Description

## Summary

This is a Quantus integration defect, and both halves of the integration are Quantus-owned in-scope code. At `runtime/src/configs/mod.rs:477-490`, Quantus's own runtime configures transaction payment with a Quantus-own custom `TransactionFeesCollector` that updates the `CollectedFees` storage item at `pallets/mining-rewards/src/lib.rs:210-217` for every nonzero paid signed extrinsic, and in the same configuration block selects a generic, unmodified-upstream `WeightInfo` that was never benchmarked against that collector. The selected weight therefore omits the collector's additional unique database read and write. A production-runtime test at the frozen competition pin establishes at least a 16.7160% undercharge for the whole minimal signed remark using only the conservative database lower bound, within the program's paid Low weight-accounting row.

The unmodified upstream `pallets/transaction-payment/src/` files are cited in this report only as out-of-scope-upstream context. No defect is alleged in them, and the fix is entirely inside Quantus-owned code.

## Root Cause

The root cause is stated at two Quantus-owned in-scope sites: the runtime binding at `runtime/src/configs/mod.rs:477-490` (UPSTREAM\_MODIFIED, in scope) and the unmetered collector storage access at `pallets/mining-rewards/src/lib.rs:210-217` (QUANTUS\_OWN, in scope). Everything cited from `pallets/transaction-payment/src/` is unmodified upstream and is presented as out-of-scope-upstream context only, to show what the Quantus binding selected.

At frozen pin `3b243b870a5442a6e5f443056f67bbba87d472f5`, `runtime/src/configs/mod.rs:477-490` binds:

```rust
type OnChargeTransaction =
    FungibleAdapter<Balances, pallet_mining_rewards::TransactionFeesCollector<Runtime>>;
type WeightInfo = pallet_transaction_payment::weights::SubstrateWeight<Runtime>;
```

Both lines of that binding are chosen by Quantus. The first selects a stateful Quantus-own fee handler; the second selects a generic `WeightInfo` whose benchmark never executed that handler. The mismatch between the two is the defect, and it exists only because these two choices sit side by side in a Quantus-owned runtime file.

Out-of-scope-upstream context, unmodified, cited only to show what that selection resolves to: `pallets/transaction-payment/src/lib.rs:990-1001` charges only `T::WeightInfo::charge_transaction_payment()`, and the selected implementation at `pallets/transaction-payment/src/weights.rs:81-90` declares only `System::Account (r:1 w:1)` and returns `35,979,000 ps + one database read + one database write = 160,979,000 ps`. That upstream weight was generated on 21 February 2025 against `kitchensink_runtime.wasm`, not against Quantus's production runtime, and behaves exactly as its own benchmark intends. Quantus introduced the custom collector later in source revision `6149164fe235a1c3fbecf372fb8e5f022cddf579` without changing the `WeightInfo` selection at `runtime/src/configs/mod.rs:489`.

After fee correction, the unmodified upstream handoff at `pallets/transaction-payment/src/payment.rs:169-207` (out-of-scope-upstream context) passes the nonzero credit to the Quantus-configured handler. Back inside in-scope Quantus code, `pallets/mining-rewards/src/lib.rs:309-318` routes it to `collect_transaction_fees`, and `pallets/mining-rewards/src/lib.rs:210-217` mutates `CollectedFees`, reads its resulting value for the event, and emits `FeesCollected`. That storage work is Quantus-own, unconditional on every paid extrinsic, and unmetered.

The conservative omitted lower bound is one unique-key database read plus one write, or `125,000,000 ps` under the runtime database weights. FRAME's same-extrinsic storage tracker means the subsequent same-key `get` is not counted as a second database read. Its CPU work and event work are also omitted from the calculation, pending a regenerated benchmark.

The broken invariant is Quantus-owned: the `WeightInfo` a runtime selects at `runtime/src/configs/mod.rs:489` must account for the work that the `OnChargeTransaction` handler bound three lines above it unconditionally performs at `pallets/mining-rewards/src/lib.rs:210-217`.

## Attack Path

{% stepper %}
{% step %}

## Sign an ordinary remark

A funded user signs an ordinary `System::remark([])` using the production ML-DSA-65 signed-transaction format.
{% endstep %}

{% step %}

## Apply the extrinsic

`Executive::apply_extrinsic` runs the production `TxExtension` tuple, including `ChargeTransactionPayment`.
{% endstep %}

{% step %}

## Correct and collect the fee

The extension withdraws and corrects the nonzero fee, then invokes the runtime-configured `TransactionFeesCollector`.
{% endstep %}

{% step %}

## Update collector storage

The collector updates the `CollectedFees` storage key and emits `FeesCollected`.
{% endstep %}

{% step %}

## Charge the generic weight

The payer and block meter are charged using the generic `160,979,000 ps` payment-extension weight, without the collector's additional `125,000,000 ps` unique-key database lower bound.
{% endstep %}

{% step %}

## Repeat on paid extrinsics

The same undercharge recurs on every signed extrinsic whose corrected fee is nonzero.
{% endstep %}
{% endstepper %}

Any ordinary funded account can trigger the path. No special authority or unusual chain state is needed, and the reproduction uses only a local checkout.

## Impact

For the smallest production signed remark, the runtime declares `622,787,000 ps` for the whole extrinsic. Adding only the omitted unique-key collector lower bound gives a conservative correct lower bound of `747,787,000 ps`:

```
conservative whole-extrinsic lower bound = 125,000,000 / 747,787,000
                                         = 16.7160%
```

For reference, the isolated payment-extension component has this conservative lower bound:

```
conservative payment-component lower bound = 125,000,000 / (160,979,000 + 125,000,000)
                                           = 43.7095%
```

The conservative whole-extrinsic lower-bound percentage is the filing basis. The error is systemic rather than a self-only display issue: the block meter accepts paid transactions using an understated weight, and the identity `WeightToFee` mapping also converts the understated ref-time into the compute-fee component.

No higher-tier impact is claimed.

## Severity Justification

The exact Low row is:

> Inaccuracies to the weight (similar to gas) accounting system that under or overcharge by less than 50% of the correct amount (and are not intentional as indicated by the program's terms

The executed production-runtime path establishes at least a 16.7160% whole-extrinsic undercharge from database work alone. This conservative proven amount is below 50%; unmeasured CPU and event work are not used to claim an upper bound.

### Escalation attempted and terminally resolved with a numeric bound

Higher tiers were pursued first and are refuted by a numeric bound on the omitted quantity, not by measurement difficulty.

The omitted work is one unique-key `CollectedFees` read plus one write under `RocksDbWeight` (`runtime/src/configs/mod.rs:124`), which is a constant `25,000,000 + 100,000,000 = 125,000,000 ps` regardless of the extrinsic. The undercharge ratio is therefore `125,000,000 / (declared_whole + 125,000,000)`, which is strictly decreasing in the declared whole weight. The smallest production paid extrinsic maximizes it, so `16.7160%` is the maximum attained value over paid extrinsics, not a sample point.

The bound does not depend on enumerating calls. Every paid signed extrinsic pays a constant floor before any call weight is added: the `108,157,000 ps` extrinsic base weight plus the whole `TxExtension` tuple, which together are `620,787,000 ps` at this pin (the executed `622,787,000 ps` whole-extrinsic figure minus the `2,000,000 ps` `System::remark` call weight). Even a hypothetical zero-weight paid call therefore yields at most `125,000,000 / (620,787,000 + 125,000,000) = 16.7608%`, so `16.7608%` is a strict ceiling over all paid extrinsics that could ever exist on this runtime. Enumerating every `fn` in every `weights.rs` in the tree found no dispatchable base cheaper than `1,000,000 ps`, which would give `16.7384%`. Free and unsigned paths cannot raise it, because the collector is invoked only through `OnUnbalanced::on_nonzero_unbalanced` at `pallets/mining-rewards/src/lib.rs:309-318`, so a zero-fee extrinsic never reaches `CollectedFees` and performs none of the omitted work.

* Medium row "under or overcharge by more than 50% of the correct amount": unreachable. Exceeding 50% of the whole extrinsic would need the omission to exceed `622,787,000 ps`, which is `4.98x` the constant `125,000,000 ps` bound. Even on the isolated payment-extension denominator the figure is `43.7095%`, and reaching 50% there would need more than `160,979,000 ps` of omitted work, which the fixed one-read-one-write bound plus a single same-key `get` and one event cannot supply.
* Medium row "Increasing network processing node resource consumption by at least 30%": unreachable. The unmetered fraction of block capacity is bounded above by the same supremum `16.7160%`, which is `1.79x` short of 30%, and it falls for every heavier extrinsic because the denominator grows while the omission stays constant.
* Medium row "Block stuffing": unreachable for the same bound. Admission is still gated by `CheckWeight` against the declared weight; the attacker gains at most a `16.7160%` unmetered margin per transaction and cannot exceed the block limit, so there is no capacity-exhaustion endpoint.
* High row "Causing network processing nodes to process transactions from the mempool beyond set parameters": unreachable, and this is established by block-level measurement rather than by assertion, because the mechanism of the block-stuffing refutation above is exactly what this row describes. The count of paid extrinsics in one block is bound by the Normal LENGTH budget, not by weight: `3,932,160` bytes of Normal length capacity divided by the `7,339`-byte minimum ML-DSA signed extrinsic gives `535`, while the weight budget would allow `7,225`. Total unmetered real work per block is therefore capped at `535 x 125,000,000 = 66,875,000,000 ps`, which is `1.4861%` of the Normal budget and `1.1146%` of the `6 s` `max_block` parameter, and that cap is independent of call composition because the length bound holds however heavy the calls are. Filling declared weight all the way to the Normal `max_total` and then adding the full unmetered addition gives a worst real block cost of `4,566,875,000,000 ps`, which is `76.1146%` of `max_block` and `38.0573%` of the `12,000 ms` target block time. The absolute block parameter is never crossed, with `23.8854%` headroom remaining. The omitted work is also never registered anywhere, so `CheckWeight`'s own accounting stays internally consistent, and node-side mempool admission limits are byte and count based and are untouched.
* Remaining High and Critical rows: no funds, freezing, chain-split, or transaction-confirmation endpoint exists on this path. The defect changes only the metered amount of an extrinsic that is admitted and executed correctly either way. A row-by-row disposition for all 37 program impact rows is recorded at `.auditooor/escalation/fee_collector_row_sweep.json`.

The honest tier is therefore Low, held after the escalation attempt rather than chosen as a default.

The omission is not indicated as intentional by the program or code terms. `runtime/src/configs/mod.rs:481-486` expressly converts compute weight to fees using a 1:1 mapping. The current weight file describes only the generic kitchensink benchmark, and `scripts/regenerate_weights.sh:26-35` regenerates several Quantus pallets but omits transaction payment. No comment grants free execution to the custom collector. Quantus also meters custom transaction-extension work elsewhere, so the most direct reading is an integration weight that predates the collector rather than an intentional subsidy.

## Originality

Exact mechanism searches found no prior audit or public post-pin fix that identifies the missing `CollectedFees` database weight. The nearest prior report quotes the binding only while analyzing a different weight dimension, and the one-fix comparison below establishes separate endpoints and remediation.

### Distinction from post-audit-pin revisions

The frozen asset is `immunefi-team/audit-comp-quantus-chain@3b243b870a5442a6e5f443056f67bbba87d472f5`. The post-pin source range checked was `3b243b870a5442a6e5f443056f67bbba87d472f5..b9c8849064164824d494b39bf6556960fe8daed6`, together with GitHub pull-request searches for transaction-payment, collector, weight, fee, and `CollectedFees` terms.

No post-pin revision changes the relevant transaction-payment weight, collector implementation, or runtime binding. The frozen and post-pin `pallets/transaction-payment/src/weights.rs` files have the same SHA-256, `0aed2945ec764e1c1e6770b7e310bda4a0c315abd0700f8e7c3fb36fc59b8240`. Merged Quantus PR #650 regenerated multiple pallet weights but did not regenerate transaction payment. No public fix for this mechanism was found.

Changeset `3361cf7` has the near-name title `address tx fee ignoring storage`, but its change only adjusts `LengthToFeeMultiplier` and related comments. It does not change `TransactionFeesCollector`, `CollectedFees`, or `charge_transaction_payment`, and therefore does not meter the omitted collector work.

Public issue #417 concerns finalization overcharge and the burn-and-remint fee flow. Its endpoint and remediation are in finalization/economic handling, not the weight returned by `ChargeTransactionPayment`; resolving it cannot add the collector's missing extension weight.

PR #641 meters `CollectedFees` in the mining-rewards `on_finalize` reservation. It does not change `pallets/transaction-payment/src/weights.rs` or the weight declared for the per-extrinsic collector path, so that reservation cannot close this omission.

### Distinction from prior findings

All four official prior-audit sources and the official disclosed-issue set were searched for the collector, `CollectedFees`, transaction-payment weight, and equivalent mechanisms.

`EQ-QNT-RUNTIME-O-02` quotes the runtime configuration, including the collector binding, while discussing the deliberate global treatment of the proof-size dimension. It does not identify the collector's additional ref-time/database work.

* Q1, different files or operative endpoints: Yes. O-02 concerns the global proof-size limit and pricing policy in runtime configuration. This report concerns the executed `CollectedFees` access at `pallets/mining-rewards/src/lib.rs:210-217` and its absence from `pallets/transaction-payment/src/weights.rs:81-90`.
* Q2, would one fix close both: No. O-02's remediation is to document or revisit proof-size policy. This report requires regenerating or replacing the runtime-specific payment-extension weight to include the custom fee handler. Either fix can be applied without fixing the other.

The prior `EQ-QNT` opaque inner-call execution report concerns a call executed through a separate dispatch wrapper.

* Q1, different files or operative endpoints: Yes. That issue uses the wrapper's execute path. This issue occurs in the transaction-payment extension of every ordinary paid signed extrinsic.
* Q2, would one fix close both: No. The prior fix adds a maximum call-weight bound and post-dispatch accounting. This fix changes the runtime-specific payment-extension weight.

The prior transaction-fee treasury fallback observation concerns test coverage when no miner is found.

* Q1, different files or operative endpoints: Yes. That observation concerns mining-reward finalization and a missing test. This report concerns live signed-extrinsic weight charging.
* Q2, would one fix close both: No. Adding fallback coverage cannot change the payment-extension weight, and regenerating the payment weight cannot add the requested fallback test.

### Distinction from the project's documented accepted trade-off

PR description at the pin records one accepted weight trade-off, section "Accepted trade-off, now documented", item "3. Post-hoc, not-fee-charged event scan". Its stated rationale is that the cost "cannot be fee-charged: the scan cost depends on how many events earlier transactions in the same block emitted, unknowable at fee time". That rationale is specific to a block-cumulative, input-dependent quantity. The collector's omitted work is the opposite case: a fixed one unique-key read plus one write, identical for every paid extrinsic, statically knowable and directly benchmarkable. The documented acceptance therefore does not extend to this omission, and no comment at `runtime/src/configs/mod.rs:477-490`, `pallets/transaction-payment/src/weights.rs:81-90`, or `pallets/mining-rewards/src/lib.rs:210-217` grants the collector free execution.

The acceptance record is bound to one specific extension. The accepted mechanism lives in `runtime/src/transaction_extensions.rs`, inside `WormholeProofRecorderExtension` (declared at `runtime/src/transaction_extensions.rs:129`), and its uncharged work is registered at `runtime/src/transaction_extensions.rs:516` through `register_extra_weight_unchecked(extra, info.class)`. The in-source rationale sits directly above it at `runtime/src/transaction_extensions.rs:494-508`.

This report's mechanism shares none of those elements:

| Axis                           | Accepted post-dispatch event scan                                                       | This report                                                                                                                                           |
| ------------------------------ | --------------------------------------------------------------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------- |
| Extension                      | `WormholeProofRecorderExtension`, `runtime/src/transaction_extensions.rs:129`           | `ChargeTransactionPayment`, `pallets/transaction-payment/src/lib.rs:990`                                                                              |
| Weight mechanism               | Post-dispatch registration, `runtime/src/transaction_extensions.rs:516`                 | Static pre-dispatch declaration, `pallets/transaction-payment/src/lib.rs:999-1001` returning `pallets/transaction-payment/src/weights.rs:83-90`       |
| Uncharged quantity             | Block-cumulative event scan, varies with what earlier transactions in the block emitted | Fixed one unique-key read plus one write on `CollectedFees`, `pallets/mining-rewards/src/lib.rs:210-217`, identical for every paid extrinsic          |
| Why it is or is not chargeable | Stated unknowable at fee time, `runtime/src/transaction_extensions.rs:494-500`          | Statically knowable and directly benchmarkable; nothing in the code or program terms exempts it                                                       |
| Fix                            | None; accepted as designed                                                              | Regenerate `charge_transaction_payment` against the production runtime and select a runtime-specific `WeightInfo` at `runtime/src/configs/mod.rs:489` |

`register_extra_weight_unchecked` occurs only in `runtime/src/transaction_extensions.rs` (lines 227, 499, 516) and appears nowhere in `pallets/transaction-payment/src/` or `pallets/mining-rewards/src/lib.rs`. This report's path therefore never touches the accepted mechanism, is not a post-dispatch event scan, and does not rest on unpaid block capacity through that extension. The documented acceptance does not reach it.

No prior report discloses this exact unique-key storage omission. The prior-audit exclusion is therefore not triggered.

## Proof of Concept

Run the following from a fresh checkout of the listed asset and frozen pin:

```bash
git clone https://github.com/immunefi-team/audit-comp-quantus-chain.git
cd audit-comp-quantus-chain
git checkout 3b243b870a5442a6e5f443056f67bbba87d472f5
```

Apply this complete runnable regression patch:

```diff
diff --git a/runtime/tests/transactions/ml_dsa_65.rs b/runtime/tests/transactions/ml_dsa_65.rs
index e7f09f43e4..6e05f5ee55 100644
--- a/runtime/tests/transactions/ml_dsa_65.rs
+++ b/runtime/tests/transactions/ml_dsa_65.rs
@@ -5,7 +5,7 @@ use codec::Encode;
 use frame_support::{
 	assert_ok,
 	dispatch::{GetDispatchInfo, Pays, PostDispatchInfo},
-	traits::Currency,
+	traits::{Currency, Get},
 	weights::Weight,
 };
 use qp_dilithium_crypto::Dilithium65Pair;
@@ -12,7 +12,7 @@ use qp_dilithium_crypto::Dilithium65Pair;
 use quantus_runtime::{
 	transaction_extensions::{ReversibleTransactionExtension, WormholeProofRecorderExtension},
-	Balances, BalancesCall, Executive, Runtime, RuntimeCall, RuntimeEvent, Signature,
-	SignedPayload, System, TxExtension, UncheckedExtrinsic, UNIT, VERSION,
+	Balances, BalancesCall, Executive, MiningRewards, Runtime, RuntimeCall, RuntimeEvent,
+	Signature, SignedPayload, System, TxExtension, UncheckedExtrinsic, UNIT, VERSION,
 };
 use sp_core::Pair;
 use sp_runtime::{
@@ -42,9 +42,18 @@ fn signed_transfer(
 	value: u128,
 	nonce: u32,
 ) -> UncheckedExtrinsic {
-	let genesis_hash = System::block_hash(0);
 	let call: RuntimeCall =
 		BalancesCall::transfer_keep_alive { dest: MultiAddress::Id(dest), value }.into();
+	signed_call(pair, sender, call, nonce)
+}
+
+fn signed_call(
+	pair: &Dilithium65Pair,
+	sender: AccountId32,
+	call: RuntimeCall,
+	nonce: u32,
+) -> UncheckedExtrinsic {
+	let genesis_hash = System::block_hash(0);
 
 	let tx_ext: TxExtension = (
 		frame_system::CheckNonZeroSender::<Runtime>::new(),
@@ -89,6 +98,94 @@ fn signed_transfer(
 	)
 }
 
+#[test]
+fn transaction_fee_collector_storage_is_missing_from_declared_weight() {
+	let pair = Dilithium65Pair::from_seed_slice(&[42u8; 32]).expect("valid seed");
+	let account = pair.public().into_account();
+	let mut ext = test_ext(&account);
+
+	ext.execute_with(|| {
+		let _: core::marker::PhantomData<
+			<Runtime as pallet_transaction_payment::Config>::OnChargeTransaction,
+		> = core::marker::PhantomData::<
+			pallet_transaction_payment::FungibleAdapter<
+				Balances,
+				pallet_mining_rewards::TransactionFeesCollector<Runtime>,
+			>,
+		>;
+
+		let call = RuntimeCall::System(frame_system::Call::remark { remark: Vec::new() });
+		let xt = signed_call(&pair, account.clone(), call, 0);
+		let info = xt.get_dispatch_info();
+		let collected_before = MiningRewards::collected_fees();
+		System::reset_events();
+
+		let outcome = Executive::apply_extrinsic(xt).expect("paid extrinsic must be valid");
+		assert_ok!(outcome);
+
+		let collected_after = MiningRewards::collected_fees();
+		assert!(
+			collected_after > collected_before,
+			"the production fee handler must add this transaction's nonzero fee to CollectedFees",
+		);
+		let (event_amount, event_total) = System::events()
+			.into_iter()
+			.find_map(|record| match record.event {
+				RuntimeEvent::MiningRewards(pallet_mining_rewards::Event::FeesCollected {
+					amount,
+					total,
+				}) => Some((amount, total)),
+				_ => None,
+			})
+			.expect("the custom fee collector must emit FeesCollected");
+		assert_eq!(event_total, collected_after);
+		assert_eq!(event_amount, collected_after - collected_before);
+
+		let db = <Runtime as frame_system::Config>::DbWeight::get();
+		let declared_payment_weight =
+			<pallet_transaction_payment::weights::SubstrateWeight<Runtime> as
+				pallet_transaction_payment::weights::WeightInfo>::charge_transaction_payment();
+		let generic_payment_baseline = Weight::from_parts(35_979_000, 3_593)
+			.saturating_add(db.reads(1))
+			.saturating_add(db.writes(1));
+		assert_eq!(
+			declared_payment_weight, generic_payment_baseline,
+			"the vulnerable runtime reuses the generic 1R+1W payment weight despite its custom collector",
+		);
+
+		// Conservative lower bound only. The same-key CollectedFees mutate and
+		// subsequent get account for one unique DB read and one write; repeat-get
+		// CPU is left for benchmarking. Event storage is also excluded because it
+		// can be treated specially by benchmark conventions.
+		let omitted_collector_weight = db.reads(1).saturating_add(db.writes(1));
+		assert!(
+			declared_payment_weight
+				.saturating_add(omitted_collector_weight)
+				.ref_time() > declared_payment_weight.ref_time(),
+			"the runtime's declared payment weight must omit the collector's 1R+1W lower bound",
+		);
+
+		let base_extrinsic = <Runtime as frame_system::Config>::BlockWeights::get()
+			.get(info.class)
+			.base_extrinsic;
+		let declared_whole_weight = info.total_weight().saturating_add(base_extrinsic);
+		let corrected_whole_lower_bound =
+			declared_whole_weight.saturating_add(omitted_collector_weight);
+		let omitted = u128::from(omitted_collector_weight.ref_time());
+		let correct = u128::from(corrected_whole_lower_bound.ref_time());
+		let undercharge_bps = omitted.saturating_mul(10_000) / correct;
+		println!(
+			"payment_declared_ps={}, collector_omitted_ps={}, whole_declared_ps={}, conservative_undercharge_bps={}",
+			declared_payment_weight.ref_time(),
+			omitted_collector_weight.ref_time(),
+			declared_whole_weight.ref_time(),
+			undercharge_bps,
+		);
+		assert!(undercharge_bps > 1_000, "omission must exceed 10% of correct whole weight");
+		assert!(undercharge_bps < 5_000, "whole-extrinsic impact must remain honestly below 50%");
+	});
+}
+
 #[test]
 fn test_ml_dsa_65_extrinsic_accepted_by_runtime() {
 	let pair = Dilithium65Pair::from_seed_slice(&[42u8; 32]).expect("valid seed");
```

Run the focused production-runtime test:

```bash
SKIP_WASM_BUILD=1 CARGO_BUILD_JOBS=1 cargo test -p quantus-runtime --test mod transactions::ml_dsa_65::transaction_fee_collector_storage_is_missing_from_declared_weight -- --exact --nocapture
```

Executed baseline transcript:

```
process exit code: 0
running 1 test
payment_declared_ps=160979000
collector_omitted_ps=125000000
whole_declared_ps=622787000
conservative_undercharge_bps=1671
test result: ok. 1 passed; 0 failed
```

For the mutation control, apply exactly one additional unique-key read and write to the declared payment weight:

```diff
diff --git a/pallets/transaction-payment/src/weights.rs b/pallets/transaction-payment/src/weights.rs
--- a/pallets/transaction-payment/src/weights.rs
+++ b/pallets/transaction-payment/src/weights.rs
@@ -88,5 +88,6 @@ impl<T: frame_system::Config> WeightInfo for SubstrateWeight<T> {
 		Weight::from_parts(35_979_000, 3593)
 			.saturating_add(T::DbWeight::get().reads(1_u64))
 			.saturating_add(T::DbWeight::get().writes(1_u64))
+			.saturating_add(T::DbWeight::get().reads_writes(1_u64, 1_u64))
 	}
 }
```

The same focused command then fails at the vulnerable-weight equality assertion, as expected:

```
process exit code: 101
assertion failed: the vulnerable runtime reuses the generic 1R+1W payment weight despite its custom collector
left:  Weight { ref_time: 285979000, proof_size: 3593 }
right: Weight { ref_time: 160979000, proof_size: 3593 }
test result: FAILED. 0 passed; 1 failed
```

After removing only that one-line mutation, the identical baseline was rerun from restored production source:

```
process exit code: 0
running 1 test
payment_declared_ps=160979000
collector_omitted_ps=125000000
whole_declared_ps=622787000
conservative_undercharge_bps=1671
test result: ok. 1 passed; 0 failed
```

### What the tests prove

* The actual runtime type binds `FungibleAdapter<Balances, TransactionFeesCollector<Runtime>>`.
* A real signed paid extrinsic reaches the collector, increases `CollectedFees`, and emits `FeesCollected`.
* The production runtime still returns the generic payment weight containing only its original one read and one write.
* The additional collector key contributes a conservative lower bound of one distinct read and one write, or `125,000,000 ps`.
* The minimal whole signed remark is undercharged by at least `1671` basis points, or 16.7160% before integer truncation, using only the conservative database lower bound.
* Adding exactly the omitted lower-bound database weight changes the declared payment component from `160,979,000 ps` to `285,979,000 ps` and kills the vulnerable assertion.
* Removing the mutation restores the same passing baseline and metrics, excluding mutation contamination.

## Runtime Reachability

The ordinary paid-transaction call chain is:

```
signed extrinsic
-> Runtime Executive::apply_extrinsic
-> TxExtension tuple
-> ChargeTransactionPayment::weight
-> generic SubstrateWeight<Runtime>::charge_transaction_payment

post-dispatch fee correction
-> payment.rs::correct_and_deposit_fee
-> configured OnChargeTransaction
-> TransactionFeesCollector::on_nonzero_unbalanced
-> MiningRewards::collect_transaction_fees
-> CollectedFees mutate + FeesCollected event
```

Source hops at the frozen pin:

* `runtime/src/configs/mod.rs:477-490`: default runtime binding and generic `WeightInfo` selection.
* `pallets/transaction-payment/src/lib.rs:990-1001`: extension weight entrypoint.
* `pallets/transaction-payment/src/weights.rs:81-90`: declared generic storage set and weight.
* `pallets/transaction-payment/src/payment.rs:169-207`: corrected fee reaches `OnUnbalanced`.
* `pallets/mining-rewards/src/lib.rs:309-318`: production collector entrypoint.
* `pallets/mining-rewards/src/lib.rs:210-217`: omitted storage work.

This path is enabled by the default runtime configuration and is taken by an ordinary funded signer whenever the corrected fee is nonzero.

## Protocol-Owned Defenses Considered

`opposed_trace_coverage: not_applicable` to value-loss defenses because this report claims only weight-accounting inaccuracy. The relevant accounting controls were nevertheless checked:

| Control                         | Expected protection                             | Included in proof                                         | Result                                                                                  |
| ------------------------------- | ----------------------------------------------- | --------------------------------------------------------- | --------------------------------------------------------------------------------------- |
| Post-dispatch fee correction    | Correct fee amount after call execution         | Yes, through `Executive::apply_extrinsic`                 | It still invokes the same collector and does not alter the extension's declared weight. |
| Weight reclaim                  | Remove reported unused weight                   | Yes, as part of the production extension tuple            | It cannot add work absent from the declared extension weight.                           |
| Call and base-extrinsic weights | Account for call and fixed transaction overhead | Yes, both are included in the whole-extrinsic denominator | Neither includes the collector's separate storage key.                                  |

Outcome: an accounting error of at least 16.7160% remains after these controls.

## Suggested Fix

Regenerate `charge_transaction_payment` against the production Quantus runtime with `TransactionFeesCollector` configured, then use a Quantus runtime-specific `WeightInfo` implementation. The benchmark must execute a nonzero corrected fee and include the collector's `CollectedFees` access and any measured CPU or event work.

As an interim conservative guard, add at least one database read and one database write for the unique `CollectedFees` key to the runtime-selected payment weight. The hand-written `reads_writes(1, 1)` mutation above is only a causal lower-bound demonstration, not a substitute for benchmarking. Add the regression test to prevent future generic-weight reintroduction, and add transaction payment to `scripts/regenerate_weights.sh`.
