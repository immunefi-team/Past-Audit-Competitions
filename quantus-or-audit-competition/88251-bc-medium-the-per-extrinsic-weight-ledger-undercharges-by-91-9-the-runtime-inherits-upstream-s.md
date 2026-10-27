# 88251 bc medium the per extrinsic weight ledger undercharges by 91 9 the runtime inherits upstream substrate s sr25519 constant while verifying post quantum dilithium signatures in wasm

**Submitted on Aug 12th 2026 at 12:00:20 UTC by @shadowHunter for** [**Audit Comp | Quantus**](https://immunefi.com/audit-competition/audit-comp-quantus)

* **Report ID:** #88251
* **Report Type:** Blockchain/DLT
* **Report severity:** Medium
* **Target:** <https://github.com/immunefi-team/audit-comp-quantus-chain>
* **Impacts:**
  * Inaccuracies to the weight (similar to gas) accounting system that under or overcharge by more than 50% of the correct amount (and are not intentional as indicated by the program's terms)

## Description

## 1. Summary

`frame_system::Config::BlockWeights` sets `base_extrinsic`, a weight charged once per extrinsic to account for the work performed *before* the call body runs: SCALE decoding and, principally, **signature verification**.

Quantus never overrides it, so it inherits upstream Substrate's constant of **108,157 ns** — a figure upstream measured on the stock Substrate development chain, which uses **sr25519** verified by a **native host function**.

Quantus does not use sr25519. It uses **Dilithium (ML-DSA-65/87)**, verified **inside the runtime WASM with no host function**, plus a Poseidon hash over the entire public key to derive the account.

Running the project's own `benchmark overhead` command against a release build of this chain:

| Constant               | Declared (inherited) | Measured (this chain) | Undercharge, as % of correct |
| ---------------------- | -------------------: | --------------------: | ---------------------------: |
| `ExtrinsicBaseWeight`  |           108,157 ns |      **1,340,574 ns** |                    **91.9%** |
| `BlockExecutionWeight` |           431,614 ns |      **1,223,182 ns** |                    **64.7%** |

Both exceed the 50% threshold. The per-extrinsic figure is the primary claim; the per-block constant is the same defect from the same inherited file and is included for completeness.

**The result does not depend on averaging.** Across 100 repetitions, the *fastest single observation* was 1,261,221 ns — still a **91.4%** undercharge. There is no sampling regime in which the declared value is within 50% of correct.

***

## 2. Description

### 2.1 What `base_extrinsic` is for

It is added to every extrinsic's consumed weight in `CheckWeight`:

```rust
// pallets/frame-system/src/extensions/check_weight.rs:163-166
let extrinsic_weight = info
    .total_weight()
    .saturating_add(maximum_weight.get(info.class).base_extrinsic)
    .saturating_add(Weight::from_parts(0, len as u64));
```

`info.total_weight()` is the call weight plus declared extension weights. None of those cover the signature check — the extension weights in `pallets/frame-system/src/extensions/weights.rs` are small, separately benchmarked values (`check_nonce()` ≈ 7.26 µs, `check_weight()` ≈ 3.0 µs) that never touch the signature.

`base_extrinsic` is therefore the **only** budget line covering signature verification.

### 2.2 Quantus never sets it

```rust
// runtime/src/configs/mod.rs:86-89
pub RuntimeBlockWeights: BlockWeights = BlockWeights::with_sensible_defaults(
    Weight::from_parts(6u64 * WEIGHT_REF_TIME_PER_SECOND, u64::MAX),
    NORMAL_DISPATCH_RATIO,
);
```

`with_sensible_defaults` funnels into the builder, which applies the inherited constant to every dispatch class:

```rust
// pallets/frame-system/src/limits.rs:350
base_extrinsic: constants::ExtrinsicBaseWeight::get(),
```

```rust
// frame/support/src/weights/extrinsic_weights.rs:54-56
pub const ExtrinsicBaseWeight: Weight =
    Weight::from_parts(WEIGHT_REF_TIME_PER_NANOS.saturating_mul(108_157), 0);
```

A repository-wide search confirms no override exists anywhere in the runtime:

```
$ grep -rn "ExtrinsicBaseWeight\|base_extrinsic\|BlockExecutionWeight" runtime/
(no results)
```

### 2.3 The inherited constant describes a different chain

The constant's own generated header records how it was produced:

```
// frame/support/src/weights/extrinsic_weights.rs:18-36
//! THIS FILE WAS AUTO-GENERATED USING THE SUBSTRATE BENCHMARK CLI VERSION 32.0.0
//! DATE: 2024-11-08 (Y/M/D)
//! HOSTNAME: `runner-wiukf8gn-project-674-concurrent-0`, CPU: `Intel(R) Xeon(R) CPU @ 2.60GHz`
//!
//! SHORT-NAME: `extrinsic`, LONG-NAME: `ExtrinsicBase`, RUNTIME: `Development`
//! WARMUPS: `10`, REPEAT: `100`
//
// Executed Command:
//   ./target/production/substrate-node
//   benchmark
//   overhead
//   --chain=dev
//   --wasm-execution=compiled
```

`RUNTIME: Development` on `substrate-node --chain=dev` is the stock Substrate node, whose signature type is `MultiSignature` and whose sr25519 verification is a **native host function** (`sp_io::crypto::sr25519_verify`), costing on the order of tens of microseconds.

### 2.4 What Quantus actually does per extrinsic

```rust
// runtime/src/lib.rs:112-113
// pub type Signature = MultiSignature;
pub type Signature = DilithiumSignatureScheme;
```

The upstream type is commented out directly above the replacement. Verification runs entirely in-runtime:

```rust
// primitives/dilithium-crypto/src/traits.rs:172-182
Self::Dilithium65(sig_public) => {
    let account = sig_public.public().clone().into_account();   // Poseidon over the FULL public key
    if account != *signer {
        return false;
    }
    crate::verify_ml_dsa_65(
        sig_public.public().as_ref(),
        msg.get(),
        sig_public.signature().as_ref(),
    )
},
```

Two expensive operations per extrinsic, neither reflected in the 108,157 ns:

1. `into_account()` — `primitives/dilithium-crypto/src/traits.rs:197-198` — `qp_poseidon_core::hash_bytes` over the **entire** public key (1,952 bytes for ML-DSA-65, 2,592 bytes for ML-DSA-87).
2. `verify_ml_dsa_*` — `primitives/dilithium-crypto/src/scheme_macro.rs:493-501` — pure Rust lattice verification from `qp-rusty-crystals-dilithium`, with **no host function**, executing inside the WASM sandbox.

This is the delta the inherited constant cannot describe: upstream measured a native host-function call; this chain performs lattice verification in WASM.

***

## 3. Steps to reproduce

The complete runnable PoC, with scripts and captured output, is in §7. In outline:

{% stepper %}
{% step %}

## Build the node

Build the node with `--features runtime-benchmarks` (the `Benchmark` subcommand is gated at `node/src/command.rs:498-499`).
{% endstep %}

{% step %}

## Apply the host-function patch

Apply the small host-function patch in §7.4 — without it `benchmark overhead` cannot instantiate the runtime at all (§6.3).
{% endstep %}

{% step %}

## Run the overhead benchmark

Run the project's own command:

```
quantus-node benchmark overhead --chain=dev --wasm-execution=compiled --warmup=10 --repeat=100
```

{% endstep %}

{% step %}

## Compare generated and shipped weights

Compare the generated `extrinsic_weights.rs` against the shipped `frame/support/src/weights/extrinsic_weights.rs:54-56`.

Expected: generated ≈ 1.34 ms versus shipped 108,157 ns.
{% endstep %}
{% endstepper %}

The benchmark's `RemarkBuilder` signs with a **real Dilithium key**, so the measurement exercises the actual verification path:

```rust
// node/src/benchmarking.rs:42-54
fn build(&self, nonce: u32) -> std::result::Result<OpaqueExtrinsic, &'static str> {
    let acc = dilithium_bob();
    let extrinsic: OpaqueExtrinsic = create_benchmark_extrinsic(
        self.client.as_ref(),
        acc,
        SystemCall::remark { remark: vec![] }.into(),
        nonce,
    )
    .into();

    Ok(extrinsic)
}
```

***

## 4. Impact

### 4.1 The claim, measured against the threshold

The impact selected is graded on how far the accounting deviates from the correct amount:

|                                  | `ExtrinsicBaseWeight` | `BlockExecutionWeight` |
| -------------------------------- | --------------------: | ---------------------: |
| Correct (measured on this chain) |          1,340,574 ns |           1,223,182 ns |
| Charged (inherited)              |            108,157 ns |             431,614 ns |
| Shortfall                        |          1,232,417 ns |             791,568 ns |
| **Undercharge as % of correct**  |             **91.9%** |              **64.7%** |
| Threshold for Medium             |                   50% |                    50% |

The declared per-extrinsic value is **8.1%** of the correct one — the ledger records about one twelfth of the work it is supposed to account for.

**Robustness.** The measurement is 100 repetitions after 10 warmups: range 1,261,221–1,897,952 ns, median 1,308,359 ns, standard deviation 106,242 ns (≈8% of the mean). Taking the **most conservative single sample** — the fastest of the 100 — the undercharge is still **91.4%**. The conclusion does not depend on the mean, the median, or outlier handling.

***

## 6. Recommendation

### 6.1 Generate the correct constants

The tooling already exists and is fully written — `node/src/benchmarking.rs` implements `ExtrinsicBuilder` and already signs with Dilithium.

```bash
quantus-node benchmark overhead \
  --chain=dev --wasm-execution=compiled \
  --weight-path=./runtime/src/weights/
```

### 6.2 Stop inheriting the upstream defaults

Replace `with_sensible_defaults` with an explicit builder so both constants come from this chain:

```rust
// runtime/src/configs/mod.rs — replace the with_sensible_defaults call
pub RuntimeBlockWeights: BlockWeights = BlockWeights::builder()
    .base_block(crate::weights::block_weights::BlockExecutionWeight::get())
    .for_class(DispatchClass::all(), |weights| {
        weights.base_extrinsic = crate::weights::extrinsic_weights::ExtrinsicBaseWeight::get();
    })
    .for_class(DispatchClass::Normal, |weights| {
        weights.max_total = Some(NORMAL_DISPATCH_RATIO * MAXIMUM_BLOCK_WEIGHT);
    })
    .for_class(DispatchClass::Operational, |weights| {
        weights.max_total = Some(MAXIMUM_BLOCK_WEIGHT);
        weights.reserved = Some(MAXIMUM_BLOCK_WEIGHT - NORMAL_DISPATCH_RATIO * MAXIMUM_BLOCK_WEIGHT);
    })
    .avg_block_initialization(Perbill::from_percent(10))
    .build_or_panic();
```

{% hint style="warning" %}
The correct value (1.341 ms) **exceeds Substrate's own sanity ceiling** of 1 ms, so the generated `sane()` test will fail (§7.7). That is a deliberate decision for the team to make — the ceiling encodes an assumption about signature cost that a post-quantum chain breaks — rather than something to silently delete.
{% endhint %}

### 6.3 Fix the benchmark executor so this can be measured at all

`node/src/service.rs` hardcodes the executor host functions with no `runtime-benchmarks` variant:

```rust
// node/src/service.rs:559
sc_executor::WasmExecutor<sp_io::SubstrateHostFunctions>
// node/src/service.rs:599
sc_service::new_wasm_executor::<sp_io::SubstrateHostFunctions>(&config.executor)
```

A runtime built with `runtime-benchmarks` imports `ext_benchmarking_*` host functions the node cannot provide, so **every** benchmark subcommand that constructs a client (`overhead`, `extrinsic`, `block`, `storage`) aborts at runtime instantiation. Only `benchmark pallet` works, because it uses `frame-benchmarking-cli`'s own executor — which is exactly why `scripts/regenerate_weights.sh` only ever invokes `benchmark pallet`.

The standard upstream wiring:

```rust
#[cfg(not(feature = "runtime-benchmarks"))]
pub type HostFunctions = sp_io::SubstrateHostFunctions;
#[cfg(feature = "runtime-benchmarks")]
pub type HostFunctions =
    (sp_io::SubstrateHostFunctions, frame_benchmarking::benchmarking::HostFunctions);
```

This is arguably the more consequential of the two fixes: without it the project cannot measure its own overhead constants at all.

## Proof of Concept

## 7. Proof of concept

Everything below is complete and self-contained. No external references, no downloads beyond the repository itself.

### 7.1 Environment

|             |                                                |
| ----------- | ---------------------------------------------- |
| OS          | Ubuntu 24.04.1 LTS (WSL2 on Windows)           |
| CPU         | AMD Ryzen 7 5800H                              |
| Cores / RAM | 12 cores, 9.9 GB                               |
| Rust        | 1.93.0 (pinned by the repo's `rust-toolchain`) |
| Targets     | `wasm32v1-none`                                |

### 7.2 Setup

```bash
git clone https://github.com/immunefi-team/audit-comp-quantus-chain quantus-chain
cd quantus-chain
git checkout audit-comp-ready      # 3b243b870a5442a6e5f443056f67bbba87d472f5

rustup toolchain install 1.93.0
rustup target add wasm32v1-none --toolchain 1.93.0
apt-get install -y clang libclang-dev protobuf-compiler pkg-config libssl-dev cmake build-essential
export LIBCLANG_PATH=/usr/lib/llvm-18/lib   # Ubuntu 24.04: libclang is off the default path
```

### 7.3 Build

```bash
cargo build --release --features runtime-benchmarks -p quantus-node -j 6
```

The `runtime-benchmarks` feature is mandatory: the entire `Benchmark` subcommand is gated at `node/src/command.rs:498-499`. Build time on the hardware above was 46 minutes. Use `-j 6` rather than the full core count — `pallet-wormhole`'s build script peaks around 4.8 GB RSS during ZK circuit generation.

### 7.4 Required instrumentation patch (and why it cannot affect the result)

As described in §6.3, `benchmark overhead` cannot run as shipped. Save as `poc1_patch.diff` and apply with `git apply poc1_patch.diff`:

```diff
diff --git a/node/Cargo.toml b/node/Cargo.toml
--- a/node/Cargo.toml
+++ b/node/Cargo.toml
@@ -18,6 +18,7 @@ targets = ["x86_64-unknown-linux-gnu"]
 clap = { features = ["derive"], workspace = true }
 codec.workspace = true
 frame-benchmarking-cli = { workspace = true, default-features = true, optional = true }
+frame-benchmarking = { workspace = true, default-features = true, optional = true }
 frame-metadata-hash-extension.default-features = true
 frame-metadata-hash-extension.workspace = true
 frame-system.default-features = true
@@ -132,6 +133,7 @@ tx-logging = [] # Enable transaction pool logging for debugging
 # Dependencies that are only required if runtime benchmarking should be build.
 runtime-benchmarks = [
 	"dep:frame-benchmarking-cli",
+	"dep:frame-benchmarking",
 	"frame-benchmarking-cli?/runtime-benchmarks",
 	"frame-system/runtime-benchmarks",
 	"pallet-transaction-payment/runtime-benchmarks",
diff --git a/node/src/service.rs b/node/src/service.rs
--- a/node/src/service.rs
+++ b/node/src/service.rs
@@ -553,10 +553,21 @@ fn spawn_authority_tasks(
 // Type Definitions
 // ============================================================================
 
+/// PoC INSTRUMENTATION: host functions for the wasm executor.
+/// Upstream Substrate nodes add the benchmarking host functions under the
+/// `runtime-benchmarks` feature; this repo did not, which makes every
+/// benchmark subcommand that builds a client (overhead/extrinsic/block/storage)
+/// fail with "function imports which are not present on the host".
+#[cfg(not(feature = "runtime-benchmarks"))]
+pub type PocHostFunctions = sp_io::SubstrateHostFunctions;
+#[cfg(feature = "runtime-benchmarks")]
+pub type PocHostFunctions =
+	(sp_io::SubstrateHostFunctions, frame_benchmarking::benchmarking::HostFunctions);
+
 pub(crate) type FullClient = sc_service::TFullClient<
 	Block,
 	RuntimeApi,
-	sc_executor::WasmExecutor<sp_io::SubstrateHostFunctions>,
+	sc_executor::WasmExecutor<PocHostFunctions>,
 >;
 type FullBackend = sc_service::TFullBackend<Block>;
 pub type PowBlockImport = sc_consensus_qpow::PowBlockImport<
@@ -596,7 +607,7 @@ pub fn new_partial(config: &Configuration) -> Result<Service, ServiceError> {
 		})
 		.transpose()?;
 
-	let executor = sc_service::new_wasm_executor::<sp_io::SubstrateHostFunctions>(&config.executor);
+	let executor = sc_service::new_wasm_executor::<PocHostFunctions>(&config.executor);
 	let (client, backend, keystore_container, task_manager) =
 		sc_service::new_full_parts::<Block, RuntimeApi, _>(
 			config,
```

**Why this cannot influence the measurement.** The patch registers additional host functions with the WASM executor. Those functions (`ext_benchmarking_current_time`, `ext_benchmarking_commit_db`, and so on) are called **only** from inside pallet benchmark code paths. `benchmark overhead` measures the execution of ordinary `System::remark` extrinsics and empty blocks, which never invoke any of them. The patch changes only whether the runtime can be *instantiated*, not what the timed code does.

Without the patch the command aborts before measuring anything:

```
Error: Service(Client(VersionInvalid("Other error happened while constructing the runtime:
runtime requires function imports which are not present on the host:
'env:ext_benchmarking_set_whitelist_version_1', 'env:ext_benchmarking_proof_size_version_1',
'env:ext_benchmarking_read_write_count_version_1',
'env:ext_benchmarking_get_read_and_written_keys_version_1',
'env:ext_benchmarking_reset_read_write_count_version_1',
'env:ext_benchmarking_current_time_version_1', 'env:ext_benchmarking_wipe_db_version_1',
'env:ext_benchmarking_commit_db_version_1', 'env:ext_benchmarking_add_to_whitelist_version_1'")))
```

Rebuild after applying.

### 7.5 The PoC script

Save as `poc1_run.sh`, `chmod +x`, run. It captures the whole evidence chain: the declared constant, its provenance, proof of no override, the signature type, and the measurement.

```bash
#!/usr/bin/env bash
# PoC - ExtrinsicBaseWeight is upstream's sr25519 constant on a post-quantum
# chain. Runs the project's OWN `benchmark overhead` tool, whose RemarkBuilder
# signs with dilithium_bob() - i.e. a real Dilithium signature.
set -uo pipefail

CHAIN_DIR="${CHAIN_DIR:-$PWD}"          # run from the repository root
NODE="$CHAIN_DIR/target/release/quantus-node"
OUT="${OUT:-$PWD/poc1-out}"
mkdir -p "$OUT"

echo "=============================================================="
echo " PoC  ExtrinsicBaseWeight under-declaration"
echo " node : $NODE"
echo " date : $(date -u)"
echo "=============================================================="

if [ ! -x "$NODE" ]; then
  echo "FATAL: node binary not found at $NODE"
  exit 1
fi

echo
echo "### node version"
"$NODE" --version 2>&1 | tee "$OUT/version.txt"

echo
echo "### INHERITED CONSTANT - what the runtime actually charges per extrinsic"
grep -n "saturating_mul" "$CHAIN_DIR/frame/support/src/weights/extrinsic_weights.rs" \
  | tee "$OUT/inherited-constant.txt"

echo
echo "--- provenance of that constant, from its own generated header ---"
sed -n '18,36p' "$CHAIN_DIR/frame/support/src/weights/extrinsic_weights.rs" \
  | tee "$OUT/inherited-provenance.txt"

echo
echo "### PROOF the runtime never overrides it"
grep -rn "ExtrinsicBaseWeight\|base_extrinsic\|BlockExecutionWeight" "$CHAIN_DIR/runtime/" \
  > "$OUT/no-override.txt" 2>&1
if [ -s "$OUT/no-override.txt" ]; then
  cat "$OUT/no-override.txt"
else
  echo "(no results - the constant is never overridden in runtime/)" | tee "$OUT/no-override.txt"
fi

echo
echo "### CONFIRM the signature scheme is Dilithium, not sr25519"
grep -n "pub type Signature" "$CHAIN_DIR/runtime/src/lib.rs" | tee "$OUT/signature-type.txt"

echo
echo "### PROOF the project's own weight tooling never runs 'benchmark overhead'"
grep -n "benchmark" "$CHAIN_DIR/scripts/regenerate_weights.sh" | tee "$OUT/regen-tooling.txt"

echo
echo "### RUNNING benchmark overhead - measures the REAL per-extrinsic base"
"$NODE" benchmark overhead \
  --chain=dev \
  --wasm-execution=compiled \
  --weight-path="$OUT" \
  --warmup=10 \
  --repeat=100 \
  2>&1 | tee "$OUT/overhead-run.txt"
echo "benchmark overhead exit code: ${PIPESTATUS[0]}" | tee -a "$OUT/overhead-run.txt"

echo
echo "### GENERATED OUTPUT FILES"
for f in "$OUT"/*.rs; do
  [ -e "$f" ] || continue
  echo "----- $f -----"; cat "$f"
done

echo
echo "=============================================================="
echo " COMPARE: measured ExtrinsicBase above vs inherited 108_157 ns"
echo "=============================================================="
```

### 7.6 PoC output

**Static evidence collected before the measurement:**

```
### node version
quantus-node 0.9.0-endless-sky-223d938923a

### INHERITED CONSTANT - what the runtime actually charges per extrinsic
56:		Weight::from_parts(WEIGHT_REF_TIME_PER_NANOS.saturating_mul(108_157), 0);

--- provenance of that constant, from its own generated header ---
//! THIS FILE WAS AUTO-GENERATED USING THE SUBSTRATE BENCHMARK CLI VERSION 32.0.0
//! DATE: 2024-11-08 (Y/M/D)
//! HOSTNAME: `runner-wiukf8gn-project-674-concurrent-0`, CPU: `Intel(R) Xeon(R) CPU @ 2.60GHz`
//!
//! SHORT-NAME: `extrinsic`, LONG-NAME: `ExtrinsicBase`, RUNTIME: `Development`
//! WARMUPS: `10`, REPEAT: `100`

// Executed Command:
//   ./target/production/substrate-node
//   benchmark
//   overhead
//   --chain=dev
//   --wasm-execution=compiled

### PROOF the runtime never overrides it
(no results - the constant is never overridden in runtime/)

### CONFIRM the signature scheme is Dilithium, not sr25519
112:// pub type Signature = MultiSignature;
113:pub type Signature = DilithiumSignatureScheme;
```

**The benchmark run:**

```
2026-08-11 11:30:21 Running 10 warmups...
2026-08-11 11:30:21 Executing block 100 times
2026-08-11 11:30:21 Per-block execution overhead [ns]:
Total: 122318294
Min: 1085528, Max: 2235841
Average: 1223182, Median: 1166226, Stddev: 229734.18
Percentiles 99th, 95th, 75th: 2194911, 1666796, 1192597
2026-08-11 11:30:22 Writing weights to ".../block_weights.rs"
2026-08-11 11:30:22 Running 10 warmups...
2026-08-11 11:30:22 Executing block 100 times
2026-08-11 11:30:22 Building block, this takes some time...
2026-08-11 11:30:24 Extrinsics per block: 541
2026-08-11 11:30:24 Running 10 warmups...
2026-08-11 11:30:31 Executing block 100 times
2026-08-11 11:31:43 Per-extrinsic execution overhead [ns]:
Total: 134057440
Min: 1261221, Max: 1897952
Average: 1340574, Median: 1308359, Stddev: 106241.98
Percentiles 99th, 95th, 75th: 1889523, 1508992, 1345917
2026-08-11 11:31:43 Writing weights to ".../extrinsic_weights.rs"
benchmark overhead exit code: 0
```

**Generated `extrinsic_weights.rs`, complete and unmodified:**

```rust
//! THIS FILE WAS AUTO-GENERATED USING THE SUBSTRATE BENCHMARK CLI VERSION 53.0.0
//! DATE: 2026-08-11 (Y/M/D)
//! HOSTNAME: `LAPTOP-PGMUQ2ED`, CPU: `AMD Ryzen 7 5800H with Radeon Graphics`
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
	///   Min, Max: 1_261_221, 1_897_952
	///   Average:  1_340_574
	///   Median:   1_308_359
	///   Std-Dev:  106241.98
	pub const ExtrinsicBaseWeight: Weight =
		Weight::from_parts(WEIGHT_REF_TIME_PER_NANOS.saturating_mul(1_340_574), 0);
}

#[cfg(test)]
mod test_weights {
	use sp_weights::constants;

	/// Checks that the weight exists and is sane.
	#[test]
	fn sane() {
		let w = super::ExtrinsicBaseWeight::get();

		// At least 10 µs.
		assert!(
			w.ref_time() >= 10u64 * constants::WEIGHT_REF_TIME_PER_MICROS,
			"Weight should be at least 10 µs."
		);
		// At most 1 ms.
		assert!(
			w.ref_time() <= constants::WEIGHT_REF_TIME_PER_MILLIS,
			"Weight should be at most 1 ms."
		);
	}
}
```

**Generated `block_weights.rs`** produces `BlockExecutionWeight = 1_223_182` ns against the shipped 431,614 ns (`frame/support/src/weights/block_weights.rs:56`).

### 7.7 Explanation of the output

**The two headers, side by side, are the whole finding.** The shipped constant reads `RUNTIME: Development`, dated 2024-11-08, produced by `substrate-node` — the stock Substrate chain with sr25519 host-function verification. The generated constant reads `RUNTIME: Quantus DevNet`, produced by `quantus-node` with real Dilithium signatures. **Same command, same parameters** (`--chain=dev --wasm-execution=compiled --warmup=10 --repeat=100`), two different chains, a 12.39× difference — and the runtime ships the wrong one.

**The measurement is stable, not noise.** 100 repetitions after 10 warmups: standard deviation 106,242 ns (≈8% of the mean), median 1,308,359 ns close to the mean 1,340,574 ns. Even the fastest single sample, 1,261,221 ns, represents a 91.4% undercharge.

**The correct value violates Substrate's own sanity bound.** The generated file asserts `w.ref_time() <= WEIGHT_REF_TIME_PER_MILLIS` ("At most 1 ms"). The measured 1,340,574 ns is 1.34 ms, so the correct constant for this chain **fails** the framework's own upper bound on a plausible per-extrinsic base — a direct indication that a post-quantum signature scheme breaks an assumption baked into the inherited tooling. `BlockExecutionWeight` at 1,223,182 ns stays within its 50 ms ceiling.

**Hardware dependence, stated honestly.** Absolute nanosecond values depend on the machine. What matters is the **ratio**: both constants describe the same workload — one no-op extrinsic's decode-and-verify overhead — so the 12.39× gap reflects sr25519-with-host-function versus Dilithium-in-WASM, not two CPUs. Re-running on the team's reference hardware will produce different absolute figures and a similar ratio, and the 50% threshold is cleared by a factor of nearly two even at the most conservative sample.
