# 90377 bc low stale transaction payment benchmark omits quantus fee collector storage undercharging the extension by 48 12 in the measured run

**Submitted on Aug 24th 2026 at 18:21:54 UTC by @Pig46940 for** [**Audit Comp | Quantus**](https://immunefi.com/audit-competition/audit-comp-quantus)

* **Report ID:** #90377
* **Report Type:** Blockchain/DLT
* **Report severity:** Low
* **Target:** <https://github.com/immunefi-team/audit-comp-quantus-chain>
* **Impacts:**
  * Inaccuracies to the weight (similar to gas) accounting system that under or overcharge by less than 50% of the correct amount (and are not intentional as indicated by the program's terms

## Description

## Brief/Intro

Quantus configures `TransactionFeesCollector` as the fee handler used by `ChargeTransactionPayment`, but continues to use a transaction-payment weight generated against the upstream Kitchensink runtime. That checked-in weight does not include the `MiningRewards::CollectedFees` storage accessed by the Quantus fee collector. A benchmark-enabled Quantus runtime built from the pinned production configuration recorded this additional storage and measured a transaction-payment extension weight of `310,318,000` ref-time, while the runtime declares only `160,979,000`. This is a `48.1245%` undercharge in the preserved benchmark run.

## Vulnerability Details

A normal fee-paying signed extrinsic reaches the Quantus collector through the following path:

```
ChargeTransactionPayment::post_dispatch_details
→ OnChargeTransaction::correct_and_deposit_fee
→ TransactionFeesCollector::on_nonzero_unbalanced
→ collect_transaction_fees
→ CollectedFees::mutate
→ FeesCollected event
```

The production configuration selects the Quantus collector while continuing to use the checked-in transaction-payment `WeightInfo`:

```rust
type OnChargeTransaction =
    FungibleAdapter<Balances, pallet_mining_rewards::TransactionFeesCollector<Runtime>>;

type WeightInfo =
    pallet_transaction_payment::weights::SubstrateWeight<Runtime>;
```

The checked-in transaction-payment weight says it was generated using `kitchensink_runtime.wasm`. Its storage ledger contains only:

```
System::Account (r:1 w:1)
```

Under Quantus’s configured `RocksDbWeight`, the checked-in ref-time is:

```
35,979,000 + reads(1) + writes(1)
= 35,979,000 + 25,000,000 + 100,000,000
= 160,979,000
```

I rebuilt the pinned Quantus runtime with the `runtime-benchmarks` feature and ran the dedicated transaction-payment benchmark using compiled-Wasm execution, 50 steps, and 20 repeats.

The preserved benchmark run generated this storage ledger:

```
System::Account                    (r:1 w:1)
MiningRewards::CollectedFees       (r:1 w:1)
```

Its generated ref-time formula was:

```
60,318,000 + reads(2) + writes(2)
= 60,318,000 + 50,000,000 + 200,000,000
= 310,318,000
```

The additional `MiningRewards::CollectedFees` read and write account for `125,000,000` ref-time under the configured database schedule. The generated benchmark also measured additional computation beyond those storage terms.

The difference observed in the preserved benchmark run is:

```
absolute undercharge = 310,318,000 - 160,979,000
                     = 149,339,000

percentage undercharge = 149,339,000 / 310,318,000 × 100
                       = 48.124504540504%
```

The precise benchmark computation term can vary with the benchmark host. The reproducible structural result is that the Quantus benchmark adds `MiningRewards::CollectedFees (r:1 w:1)`, `reads(2)`, and `writes(2)`, while the checked-in formula contains only `System::Account (r:1 w:1)`, `reads(1)`, and `writes(1)`.

`ChargeTransactionPayment::weight()` returns this checked-in transaction-payment weight as the extension’s declared weight. During post-dispatch, the same extension invokes the configured `OnChargeTransaction`, which reaches the omitted Quantus collector storage.

A separate lifecycle test confirms that:

* A normal signed `System::remark` can reach the production fee collector.
* The extension declares `160,979,000` ref-time.
* `MiningRewards::CollectedFees` increases.
* The `FeesCollected` and `TransactionFeePaid` events are emitted.
* Post-dispatch returns zero unspent weight.

That lifecycle test directly invokes the extension lifecycle and does not execute the extrinsic through FRAME Executive. It is therefore used only to establish production callback reachability and state mutation. The numerical discrepancy is established by the dedicated compiled-Wasm benchmark and its generated storage ledger.

The appropriate fix is to regenerate `pallet_transaction_payment` weights using Quantus’s configured fee collector, or to compose the collector’s weight into `ChargeTransactionPayment` exactly once.

## Impact Details

The selected Blockchain/DLT impact is:

> Inaccuracies to the weight (similar to gas) accounting system that under or overcharge by less than 50% of the correct amount (and are not intentional as indicated by the program’s terms).

Severity: **Low**

In the preserved benchmark run, the transaction-payment extension’s declared ref-time is `160,979,000`, while the benchmark-generated ref-time is `310,318,000`. The resulting discrepancy is `48.124504540504%`, which is below the program’s 50% Medium threshold.

Any funded account can reach the affected callback by submitting an ordinary fee-paying signed extrinsic. The omitted work is therefore executed repeatedly in normal production transaction processing.

This report does not claim network shutdown, liveness failure, theft, asset loss, or Medium severity.

Report `#88529` concerns ML-DSA verification omitted from `ExtrinsicBaseWeight`. This report concerns a different accounting component: the weight returned by `ChargeTransactionPayment::weight()` and the Quantus-specific fee-collector storage executed during its post-dispatch lifecycle.

A reviewer may nevertheless decide to group both reports as stale fixed-cost benchmarking issues or calculate the discrepancy against a larger complete-extrinsic denominator. This report therefore limits its claim to the dedicated transaction-payment extension formula measured by the provided benchmark.

## References

Production transaction-payment configuration, lines 451–464:

<https://github.com/immunefi-team/audit-comp-quantus-chain/blob/3b243b870a5442a6e5f443056f67bbba87d472f5/runtime/src/configs/mod.rs#L451-L464>

Transaction-payment benchmark, lines 45–90:

<https://github.com/immunefi-team/audit-comp-quantus-chain/blob/3b243b870a5442a6e5f443056f67bbba87d472f5/pallets/transaction-payment/src/benchmarking.rs#L45-L90>

Transaction-payment extension lifecycle and weight consumer, lines 990–1080:

<https://github.com/immunefi-team/audit-comp-quantus-chain/blob/3b243b870a5442a6e5f443056f67bbba87d472f5/pallets/transaction-payment/src/lib.rs#L990-L1080>

Checked-in Kitchensink-generated weight, lines 35–91:

<https://github.com/immunefi-team/audit-comp-quantus-chain/blob/3b243b870a5442a6e5f443056f67bbba87d472f5/pallets/transaction-payment/src/weights.rs#L35-L91>

`CollectedFees` storage, lines 35–38:

<https://github.com/immunefi-team/audit-comp-quantus-chain/blob/3b243b870a5442a6e5f443056f67bbba87d472f5/pallets/mining-rewards/src/lib.rs#L35-L38>

`collect_transaction_fees`, lines 195–203:

<https://github.com/immunefi-team/audit-comp-quantus-chain/blob/3b243b870a5442a6e5f443056f67bbba87d472f5/pallets/mining-rewards/src/lib.rs#L195-L203>

`TransactionFeesCollector::on_nonzero_unbalanced`, lines 292–301:

<https://github.com/immunefi-team/audit-comp-quantus-chain/blob/3b243b870a5442a6e5f443056f67bbba87d472f5/pallets/mining-rewards/src/lib.rs#L292-L301>

Configured RocksDB weight schedule, lines 23–29:

<https://github.com/immunefi-team/audit-comp-quantus-chain/blob/3b243b870a5442a6e5f443056f67bbba87d472f5/frame/support/src/weights/rocksdb\\_weights.rs#L23-L29>

## Link to Proof of Concept

<https://gist.github.com/HiroyukiNaito/39deea6b07a58a607148d58b80155835>

## Proof of Concept

Minimal evidence Gist:

```
https://gist.github.com/HiroyukiNaito/39deea6b07a58a607148d58b80155835
```

{% stepper %}
{% step %}

## Clone the audited repository

```bash
git clone https://github.com/immunefi-team/audit-comp-quantus-chain.git
cd audit-comp-quantus-chain
git checkout 3b243b870a5442a6e5f443056f67bbba87d472f5
git rev-parse HEAD
```

Expected revision:

```
3b243b870a5442a6e5f443056f67bbba87d472f5
```

{% endstep %}

{% step %}

## Build the benchmark-enabled node

```bash
CARGO_BUILD_JOBS=1 cargo build \
  --release \
  -p quantus-node \
  --features runtime-benchmarks
```

{% endstep %}

{% step %}

## Run the transaction-payment benchmark

```bash
mkdir -p outputs/transaction-fees-collector-benchmark

target/release/quantus-node benchmark pallet \
  --runtime target/release/wbuild/quantus-runtime/quantus_runtime.wasm \
  --genesis-builder none \
  --pallets pallet_transaction_payment \
  --extrinsic charge_transaction_payment \
  --steps 50 \
  --repeat 20 \
  --wasm-execution compiled \
  --heap-pages 4096 \
  --no-storage-info \
  --no-median-slopes \
  --no-min-squares \
  --output outputs/transaction-fees-collector-benchmark/run1-weights.rs \
  --json-file outputs/transaction-fees-collector-benchmark/run1-raw.json
```

{% endstep %}

{% step %}

## Inspect the generated weight

```bash
sed -n '45,70p' \
  outputs/transaction-fees-collector-benchmark/run1-weights.rs

sed -n '35,92p' \
  pallets/transaction-payment/src/weights.rs
```

The preserved benchmark run produced:

```
/// Storage: `System::Account` (r:1 w:1)
/// Storage: `MiningRewards::CollectedFees` (r:1 w:1)

Weight::from_parts(60_318_000, 0)
    .saturating_add(Weight::from_parts(0, 3593))
    .saturating_add(T::DbWeight::get().reads(2))
    .saturating_add(T::DbWeight::get().writes(2))
```

A reproduction should independently confirm:

```
MiningRewards::CollectedFees (r:1 w:1)
reads(2)
writes(2)
```

The exact measured computation term can vary with the benchmark host.

The checked-in weight contains only:

```
/// Storage: `System::Account` (r:1 w:1)

Weight::from_parts(35_979_000, 3593)
    .saturating_add(T::DbWeight::get().reads(1))
    .saturating_add(T::DbWeight::get().writes(1))
```

{% endstep %}

{% step %}

## Download and verify the lifecycle-test patch

```bash
curl -fL \
  'https://gist.githubusercontent.com/HiroyukiNaito/39deea6b07a58a607148d58b80155835/raw/697ab2864ed1796b4235a569e4daf389b4148443/poc.patch' \
  -o poc.patch

sha256sum poc.patch
```

Expected SHA-256:

```
416ffc8a2c8803e2742c35f67c7787f49409d8199dca5ec2c6dd6aab59430ad6  poc.patch
```

Apply the patch:

```bash
git apply poc.patch
```

{% endstep %}

{% step %}

## Run the lifecycle test

First confirm the test name:

```bash
cargo test -p quantus-runtime -- --list | \
  grep -F 'researcher_poc_transaction_fee_collector'
```

Run only the researcher test:

```bash
cargo test -p quantus-runtime \
  transaction_extensions::tests::researcher_poc_transaction_fee_collector_executes_unbenchmarked_storage_work \
  -- --exact --nocapture
```

The preserved successful run included:

```
running 1 test
DECLARED_TRANSACTION_PAYMENT_WEIGHT: Weight { ref_time: 160979000, proof_size: 3593 }
COLLECTED_FEES_BEFORE: 0
COLLECTED_FEES_AFTER: 269136001
FEES_COLLECTED_EVENT_EMITTED: true
TRANSACTION_FEE_PAID_EVENT_EMITTED: true
RETURNED_UNSPENT_WEIGHT: Weight { ref_time: 0, proof_size: 0 }
test transaction_extensions::tests::researcher_poc_transaction_fee_collector_executes_unbenchmarked_storage_work ... ok
test result: ok. 1 passed; 0 failed; 0 ignored; 0 measured; 50 filtered out
```

The lifecycle test establishes that the production transaction-payment configuration reaches `TransactionFeesCollector` and mutates `CollectedFees` during post-dispatch.

The benchmark establishes that this storage appears in the generated transaction-payment weight but is absent from the checked-in weight consumed by the runtime.
{% endstep %}
{% endstepper %}
