# 89091 bc medium quantus adopts an sr25519 calibrated base weight for ml dsa signatures undercharging signed extrinsics by 79 58&#x20;

**Submitted on Aug 17th 2026 at 04:52:09 UTC by @Ailenia for** [**Audit Comp | Quantus**](https://immunefi.com/audit-competition/audit-comp-quantus)

* **Report ID:** #89091
* **Report Type:** Blockchain/DLT
* **Report severity:** Medium
* **Target:** <https://github.com/immunefi-team/audit-comp-quantus-chain>
* **Impacts:**
  * Inaccuracies to the weight (similar to gas) accounting system that under or overcharge by more than 50% of the correct amount (and are not intentional as indicated by the program's terms)

## Description

### Brief/Intro

The Quantus runtime uses compiled-Wasm ML-DSA signature verification but retains Substrate's default `ExtrinsicBaseWeight`, which was measured for a conventional development runtime. Quantus's own canonical overhead benchmark constructs a signed empty `System::remark` with ML-DSA-87. Executing that benchmark against the frozen runtime measures 2,534,781 ns of per-extrinsic base overhead while the shipped runtime charges only 108,157 ns. Even crediting every other declared weight term in full, the complete signed remark is undercharged by 79.58%.

### Vulnerability Details

I looked at the frozen commit `3b243b870a5442a6e5f443056f67bbba87d472f5`:

* `runtime/src/configs/mod.rs:86-89` constructs `RuntimeBlockWeights` with `BlockWeights::with_sensible_defaults`.
* The derived `frame_system::Config` then adopts that value as its block-weight configuration.
* `with_sensible_defaults` uses the inherited `ExtrinsicBaseWeight`, whose checked-in calibration is 108,157 ns:

```rust
pub RuntimeBlockWeights: BlockWeights = BlockWeights::with_sensible_defaults(
    Weight::from_parts(6u64 * WEIGHT_REF_TIME_PER_SECOND, u64::MAX),
    NORMAL_DISPATCH_RATIO,
);
```

Quantus changed the signed-extrinsic cryptography without replacing this calibration.

The runtime accepts both ML-DSA-65 and ML-DSA-87, and ML-DSA verification executes inside the Wasm runtime rather than through a separately charged host function.

An attacker can use an ordinary ML-DSA-87 account and submit valid signed calls; no privilege or invalid transaction is required.

The repository already contains the correct canonical measurement path:

* `node/src/benchmarking.rs:22-59` defines `RemarkBuilder`.
* Uses `dilithium_bob()`.
* Constructs an empty `System::remark`.
* Signs the complete frozen `UncheckedExtrinsic` as `Signature::Dilithium87`.

Running `quantus-node benchmark overhead` with compiled Wasm, 10 warmups, and 100 repeats produced:

```
Extrinsics per block: 541
Per-extrinsic execution overhead [ns]:
Min: 2284458, Max: 3789661
Average: 2534781, Median: 2480678, Stddev: 202055.78
```

The generated replacement is 2,534,781 ns, or 23.436 times the shipped 108,157 ns.

The cheapest signed empty remark otherwise declares 622,787 ns, comprising the old 108,157-ns base plus 514,630 ns of call and transaction-extension weight.

Replacing only the miscalibrated base yields a conservative correct total of 3,049,411 ns.

So, the shipped 622,787-ns total undercharges the complete extrinsic by `(3,049,411 - 622,787) / 3,049,411 = 79.58%`.

The measurement uses the frozen runtime's real signed extrinsic, transaction extensions, ML-DSA implementation, Poseidon hashing, and compiled-Wasm execution. The attached measurement-only patch merely registers the benchmark host functions that the frozen node's existing benchmark command omitted; it does not change the runtime, signature builder, extrinsic, weight constant, or measured code path.

### Impact Details

This matches the very specific Medium impact "Inaccuracies to the weight (similar to gas) accounting system that under or overcharge by more than 50% of the correct amount (and are not intentional)."

A valid ML-DSA-87 signed extrinsic is charged 79.58% less than its conservative (and correct) complete-extrinsic weight.

This allows ordinary users to consume substantially more block-validation work than the runtime's weight accounting records. I unfortunately cannot claim the High 500%-block-delay impact because the 5-MiB block-length limit independently bounds the number of large ML-DSA-87 extrinsics.

### Remediation

Generate Quantus-specific block and extrinsic overhead weights with the production signature schemes and compiled-Wasm execution, replace the inherited base constants with those generated values, and include `benchmark overhead` in every runtime weight-regeneration release gate.

Benchmark both ML-DSA variants and charge the worst supported scheme unless the signature variant has an explicit separate weight.

Update the generated sanity test's legacy one-millisecond ceiling so it does not reject the measured production value.

### References

* [Quantus runtime block-weight configuration](https://github.com/immunefi-team/audit-comp-quantus-chain/blob/3b243b870a5442a6e5f443056f67bbba87d472f5/runtime/src/configs/mod.rs#L75-L102)
* [Dilithium-87 benchmark extrinsic builder](https://github.com/immunefi-team/audit-comp-quantus-chain/blob/3b243b870a5442a6e5f443056f67bbba87d472f5/node/src/benchmarking.rs#L22-L59)
* [signed-extrinsic construction](https://github.com/immunefi-team/audit-comp-quantus-chain/blob/3b243b870a5442a6e5f443056f67bbba87d472f5/node/src/benchmarking.rs#L96-L164)
* [inherited base-weight calibration](https://github.com/immunefi-team/audit-comp-quantus-chain/blob/3b243b870a5442a6e5f443056f67bbba87d472f5/frame/support/src/weights/extrinsic_weights.rs#L40-L57)

## Link to Proof of Concept

<https://gist.github.com/niafreu/12227631669dc94f6e99c706b6438a10>

## Proof of Concept

The attached PoC clones the frozen competition revision, applies a measurement-only host-function registration, builds the existing runtime-benchmark node, runs its existing Dilithium-87 overhead benchmark under compiled Wasm, and fails unless the measured base exceeds the shipped value by more than 50% of the correct value.

Requirements: Git, Rust/Cargo 1.93.0, Clang/libclang, the normal Quantus native build dependencies, and a local checkout at commit `3b243b870a5442a6e5f443056f67bbba87d472f5`. A cold build can take approximately 15 minutes because the Wormhole circuit artifacts are generated.

{% stepper %}
{% step %}

## Run the proof of concept

```bash
cd poc
./run.sh /path/to/audit-comp-quantus-chain
```

{% endstep %}

{% step %}

## Review the retained run output

The retained full run used 10 warmups and 100 repeats and produced:

```
shipped_base_ns=108157
measured_base_ns=2534781
base_ratio=23.436125
base_undercharge_percent=95.7331
declared_complete_ns=622787
conservative_correct_complete_ns=3049411
complete_undercharge_percent=79.5768
```

`execution.log` preserves the decisive output and `extrinsic_weights.rs` is the benchmark-generated raw weight file.
{% endstep %}
{% endstepper %}

## Code

### run.sh

```bash
#!/usr/bin/env bash
set -euo pipefail

source_repo=${1:?usage: ./run.sh /path/to/audit-comp-quantus-chain}
expected_head=3b243b870a5442a6e5f443056f67bbba87d472f5
bundle_dir=$(CDPATH= cd -- "$(dirname -- "$0")" && pwd)
cargo_bin=${CARGO_BIN:-cargo}
rust_toolchain=${RUSTUP_TOOLCHAIN:-1.93.0}
warmup=${WARMUP:-10}
repeat=${REPEAT:-100}

actual_head=$(git -C "$source_repo" rev-parse HEAD)
if [[ "$actual_head" != "$expected_head" ]]; then
    echo "unexpected source HEAD: $actual_head (expected $expected_head)" >&2
    exit 1
fi

scratch=$(mktemp -d "${TMPDIR:-/tmp}/quantus-qlf-poc.XXXXXX")
cleanup() {
    rm -rf -- "$scratch"
}
trap cleanup EXIT

git clone --quiet --no-hardlinks "$source_repo" "$scratch/chain"
git -C "$scratch/chain" checkout --quiet --detach "$expected_head"
git -C "$scratch/chain" apply --check "$bundle_dir/benchmark-host-functions.patch"
git -C "$scratch/chain" apply "$bundle_dir/benchmark-host-functions.patch"

target_dir=${CARGO_TARGET_DIR:-$scratch/target}
RUSTUP_TOOLCHAIN="$rust_toolchain" CARGO_TARGET_DIR="$target_dir" LIBCLANG_PATH="${LIBCLANG_PATH:-/usr/lib/llvm-18/lib}" "$cargo_bin" build --release --manifest-path "$scratch/chain/Cargo.toml" -p quantus-node --features runtime-benchmarks

out="$scratch/weights"
mkdir -p "$out"
"$target_dir/release/quantus-node" benchmark overhead --chain dev --wasm-execution compiled --warmup "$warmup" --repeat "$repeat" --weight-path "$out"

declared=$(sed -n 's/.*saturating_mul(\([0-9_][0-9_]*\)).*/\1/p' "$scratch/chain/frame/support/src/weights/extrinsic_weights.rs" | head -n 1 | tr -d _)
measured=$(sed -n 's/.*saturating_mul(\([0-9_][0-9_]*\)).*/\1/p' "$out/extrinsic_weights.rs" | head -n 1 | tr -d _)
if [[ -z "$declared" || -z "$measured" ]]; then
    echo "could not parse base weights" >&2
    exit 1
fi
if (( declared * 2 >= measured )); then
    echo "measured undercharge does not exceed 50% of correct base" >&2
    exit 1
fi

awk -v declared="$declared" -v measured="$measured" 'BEGIN { declared_total=622787; corrected=declared_total-declared+measured; printf "shipped_base_ns=%d\nmeasured_base_ns=%d\nbase_ratio=%.6f\nbase_undercharge_percent=%.4f\ndeclared_complete_ns=%d\nconservative_correct_complete_ns=%d\ncomplete_undercharge_percent=%.4f\n", declared, measured, measured/declared, (measured-declared)*100/measured, declared_total, corrected, (corrected-declared_total)*100/corrected }'
```

### benchmark-host-functions.patch

```diff
diff --git a/node/Cargo.toml b/node/Cargo.toml
--- a/node/Cargo.toml
+++ b/node/Cargo.toml
@@ -17,6 +17,7 @@ targets = ["x86_64-unknown-linux-gnu"]
 [dependencies]
 clap = { features = ["derive"], workspace = true }
 codec.workspace = true
+frame-benchmarking = { workspace = true, default-features = true, optional = true }
 frame-benchmarking-cli = { workspace = true, default-features = true, optional = true }
 frame-metadata-hash-extension.default-features = true
 frame-metadata-hash-extension.workspace = true
@@ -131,5 +132,6 @@ tx-logging = [] # Enable transaction pool logging for debugging
 # Dependencies that are only required if runtime benchmarking should be build.
 runtime-benchmarks = [
+	"dep:frame-benchmarking",
 	"dep:frame-benchmarking-cli",
 	"frame-benchmarking-cli?/runtime-benchmarks",
 	"frame-system/runtime-benchmarks",
diff --git a/node/src/service.rs b/node/src/service.rs
--- a/node/src/service.rs
+++ b/node/src/service.rs
@@ -553,10 +553,16 @@ fn spawn_pow_miner<NB, C, IQ, SC>(
 // Type Definitions
 // ============================================================================
 
+#[cfg(feature = "runtime-benchmarks")]
+type HostFunctions =
+	(sp_io::SubstrateHostFunctions, frame_benchmarking::benchmarking::HostFunctions);
+#[cfg(not(feature = "runtime-benchmarks"))]
+type HostFunctions = sp_io::SubstrateHostFunctions;
+
 pub(crate) type FullClient = sc_service::TFullClient<
 	Block,
 	RuntimeApi,
-	sc_executor::WasmExecutor<sp_io::SubstrateHostFunctions>,
+	sc_executor::WasmExecutor<HostFunctions>,
 >;
 type FullBackend = sc_service::TFullBackend<Block>;
 pub type PowBlockImport = sc_consensus_qpow::PowBlockImport<
@@ -596,7 +602,7 @@ pub fn new_partial(config: &Configuration) -> Result<Service, ServiceError> {
 		})
 		.transpose()?;
 
-	let executor = sc_service::new_wasm_executor::<sp_io::SubstrateHostFunctions>(&config.executor);
+	let executor = sc_service::new_wasm_executor::<HostFunctions>(&config.executor);
 	let (client, backend, keystore_container, task_manager) =
 		sc_service::new_full_parts::<Block, RuntimeApi, _>(
 			config,
```

### extrinsic\_weights.rs

```rust
//! THIS FILE WAS AUTO-GENERATED USING THE SUBSTRATE BENCHMARK CLI VERSION 53.0.0
//! DATE: 2026-08-17 (Y/M/D)
//! HOSTNAME: `vmi3255853`, CPU: `AMD EPYC Processor (with IBPB)`
//!
//! SHORT-NAME: `extrinsic`, LONG-NAME: `ExtrinsicBase`, RUNTIME: `Quantus DevNet`
//! WARMUPS: `10`, REPEAT: `100`
//! WEIGHT-METRIC: `Average`, WEIGHT-MUL: `1.0`, WEIGHT-ADD: `0`

use sp_core::parameter_types;
use sp_weights::{constants::WEIGHT_REF_TIME_PER_NANOS, Weight};

parameter_types! {
	/// Weight of executing a NO-OP extrinsic, for example `System::remark`.
	/// Calculated by multiplying the *Average* with `1.0` and adding `0`.
	///
	/// Stats nanoseconds:
	///   Min, Max: 2_284_458, 3_789_661
	///   Average:  2_534_781
	///   Median:   2_480_678
	///   Std-Dev:  202055.78
	///
	/// Percentiles nanoseconds:
	///   99th: 3_242_380
	///   95th: 2_834_503
	///   75th: 2_574_609
	pub const ExtrinsicBaseWeight: Weight =
		Weight::from_parts(WEIGHT_REF_TIME_PER_NANOS.saturating_mul(2_534_781), 0);
}

#[cfg(test)]
mod test_weights {
	use sp_weights::constants;

	/// Checks that the weight exists and is sane.
	#[test]
	fn sane() {
		let w = super::ExtrinsicBaseWeight::get();
		assert!(w.ref_time() >= 10u64 * constants::WEIGHT_REF_TIME_PER_MICROS);
		assert!(w.ref_time() <= constants::WEIGHT_REF_TIME_PER_MILLIS);
	}
}
```
