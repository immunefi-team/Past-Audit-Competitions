# 88249 bc medium extrinsicbaseweight is still upstream s sr25519 measurement after the switch to ml dsa so every signed extrinsic is charged 12 of what it costs

**Submitted on Aug 12th 2026 at 11:32:06 UTC by @skydev for** [**Audit Comp | Quantus**](https://immunefi.com/audit-competition/audit-comp-quantus)

* **Report ID:** #88249
* **Report Type:** Blockchain/DLT
* **Report severity:** Medium
* **Target:** <https://github.com/immunefi-team/audit-comp-quantus-chain>
* **Impacts:**
  * Inaccuracies to the weight (similar to gas) accounting system that under or overcharge by more than 50% of the correct amount (and are not intentional as indicated by the program's terms)

## Description

## Brief/Intro

Quantus replaced the signature scheme wholesale — `runtime/src/lib.rs:113`, `pub type Signature = DilithiumSignatureScheme` — so every signed extrinsic now performs an ML-DSA verification inside WASM. The constant that prices per-extrinsic work, `ExtrinsicBaseWeight`, was never re-derived: it is byte-identical to upstream, where it was benchmarked on a node that signs with sr25519. Running the project's own `benchmark overhead` against its own runtime measures **871,694 ns per extrinsic against the 108,157 ns the runtime declares**. The runtime charges **12.4% of the correct amount — an undercharge of 87.6%**, against the programme's 50% threshold for this impact class. That is the smallest of three runs on three machines; the other two give 91.6% and 94.2%.

## Vulnerability Details

### The constant prices sr25519 work

`runtime/src/configs/mod.rs:86-89` builds the block weights from the FRAME default:

```rust
pub RuntimeBlockWeights: BlockWeights = BlockWeights::with_sensible_defaults(
    Weight::from_parts(6u64 * WEIGHT_REF_TIME_PER_SECOND, u64::MAX),
    NORMAL_DISPATCH_RATIO,
);
```

`with_sensible_defaults` (`pallets/frame-system/src/limits.rs:323-335`) delegates to `BlockWeights::builder()`, which sets the per-class base at `limits.rs:350`:

```rust
base_extrinsic: constants::ExtrinsicBaseWeight::get(),
```

so the constant is added to the weight of every extrinsic in every dispatch class.

Its generated header records what it measured and where:

```
//! THIS FILE WAS AUTO-GENERATED USING THE SUBSTRATE BENCHMARK CLI VERSION 32.0.0
//! DATE: 2024-11-08 (Y/M/D)
//! HOSTNAME: `runner-wiukf8gn-project-674-concurrent-0`, CPU: `Intel(R) Xeon(R) CPU @ 2.60GHz`
//!
//! SHORT-NAME: `extrinsic`, LONG-NAME: `ExtrinsicBase`, RUNTIME: `Development`
//! WARMUPS: `10`, REPEAT: `100`
//! WEIGHT-PATH: `./substrate/frame/support/src/weights/`
//! WEIGHT-METRIC: `Average`, WEIGHT-MUL: `1.0`, WEIGHT-ADD: `0`

// Executed Command:
//   ./target/production/substrate-node
//   benchmark
//   overhead
//   --chain=dev
//   --wasm-execution=compiled
//   --weight-path=./substrate/frame/support/src/weights/
//   --header=./substrate/HEADER-APACHE2
//   --warmup=10
//   --repeat=100
```

`frame/support/src/weights/extrinsic_weights.rs` diffs **empty** against `polkadot-stable2512-2`, and there is no `runtime/src/weights/` directory anywhere in the repository. The constant was inherited and never regenerated.

The published Eiger review of 2026-05-13 quotes this same `parameter_types!` block, at `runtime/src/configs/mod.rs:69-82`, under `EQ-QNT-RUNTIME-O-02` — a finding about the proof-size dimension being left uncapped and unpriced. That finding assesses the limits the block sets. The per-extrinsic base constant those limits meter against is not part of it, and `ExtrinsicBaseWeight` appears nowhere in that review or in the other three published audits.

### The same command signs with a different scheme on each side

`benchmark overhead` measures a no-op **signed** extrinsic, so the signature check is inside the number. What changed is which signature.

Upstream, `substrate/bin/node/cli/src/benchmarking.rs`:

```rust
:31   use sp_keyring::Sr25519Keyring;
:60   let acc = Sr25519Keyring::Bob.pair();
```

Quantus, `node/src/benchmarking.rs`:

```rust
:15   use qp_dilithium_crypto::{self, dilithium_bob};
:43   let acc = dilithium_bob();
:101  sender: qp_dilithium_crypto::Dilithium87Pair,
:154  let signature = raw_payload.using_encoded(|e| sender.sign(e));
:159  runtime::Signature::Dilithium87(signature),
```

Same tool, same subcommand, same `--chain=dev`: upstream's 108,157 ns is the cost of verifying an sr25519 signature, and it is the number the runtime still declares. Quantus's own harness signs ML-DSA-87. The constant was carried across that change unmodified.

`DilithiumSignatureScheme` is a two-variant enum (`primitives/dilithium-crypto/src/types.rs:71-74`): the runtime accepts both ML-DSA-87 and ML-DSA-65. The harness signs the 87 variant, so that is what the measurement below prices, and it is what a regenerated constant would encode. A single `base_extrinsic` cannot price both variants correctly in any case.

The tool's own output corroborates the size of what it signed. It reports `Extrinsics per block: 541`, and the Normal-class length limit is 3,932,160 bytes (`runtime/src/configs/mod.rs:95`, 5 MiB at 75%) — 7,268 bytes per extrinsic. An ML-DSA-87 signature is 4,627 bytes and its public key 2,592, so the extrinsics it packed carried a full post-quantum signature envelope, not a 64-byte sr25519 one.

For scale, on the same machine as the runs below: an ML-DSA-87 verification measures 314.5 µs and the Poseidon2 hash over the 2,592-byte public key used for account derivation about 100 µs, both natively (criterion point estimates) — against a declared budget of 108.157 µs for the whole extrinsic. Those come from the criterion benches shipped in `qp-rusty-crystals-dilithium` 4.1.0 and `qp-poseidon-core` 3.1.0, run from their own repositories. They are context for the magnitude, not the measurement this report rests on.

### `benchmark overhead` does not run as shipped

`node/Cargo.toml:133-141` propagates `runtime-benchmarks` into `quantus-runtime`, so the runtime WASM built with that feature imports `ext_benchmarking_*` host functions — nine of them are named in the error below. Three of the four `benchmark` subcommands build a client through `service::new_partial` — `Block` (`node/src/command.rs:511`), `Storage` (`:517`) and `Overhead` (`:523`) — and that executor registers only `sp_io::SubstrateHostFunctions` (`node/src/service.rs:559`, `:599`). All three abort while constructing the runtime:

```
Error: Service(Client(VersionInvalid("... runtime requires function imports which are not
present on the host: 'env:ext_benchmarking_read_write_count_version_1', ... ")))
```

`BenchmarkCmd::Pallet` is unaffected: `node/src/command.rs:506-509` routes it to `run_with_spec`, which builds its own executor. Per-pallet benchmarking works; `overhead`, the subcommand that generates this constant, does not.

This is why the constant could not have been regenerated in place: the process at `scripts/regenerate_weights.sh` uses `benchmark pallet`, which works, and the constant is produced by `benchmark overhead`, which does not. Reproducing the measurement requires a three-line executor fix, attached as `enable-benchmarks.patch`.

### Measured

```sh
./target/release/quantus-node benchmark overhead --chain=dev \
    --base-path /tmp/qbench --weight-path /tmp/qbench-out \
    --warmup=10 --repeat=100 --wasm-execution=compiled
```

Run on three machines. Each generated file carries its own hostname and CPU in the header the tool stamps in; all three pairs are attached.

| Host                                       | profile        |     measured | undercharge |    Std-Dev |
| ------------------------------------------ | -------------- | -----------: | ----------: | ---------: |
| Xeon E5-2687W v4 @ 3.00 GHz (2016, server) | release        | 1,865,545 ns |   **94.2%** |  66,830 ns |
| Ryzen 7 5800X (2020, desktop)              | release        | 1,288,206 ns |   **91.6%** | 156,497 ns |
| Ryzen 7 5800X, same machine                | **production** | 1,205,508 ns |   **91.0%** |   7,770 ns |
| Core i9-14900K (2023, desktop)             | release        |   871,694 ns |   **87.6%** |  13,961 ns |
| *declared by the runtime*                  | *production*   | *108,157 ns* |           — |          — |

`undercharge` is the impact class's own metric — the shortfall as a fraction of the correct amount, `(measured − declared) / measured`. On the last row that is `(871,694 − 108,157) / 871,694 = 87.6%`, i.e. the runtime charges 12.4% of what the work costs. The threshold is 50%, and by construction the figure cannot exceed 100%, which would mean charging nothing.

Upstream generated 108,157 ns from `./target/production/substrate-node`, and `[profile.production]` in this workspace adds `lto = true` and `codegen-units = 1` over `release`. The third row is that like-for-like control, on the same machine as the second: the profile accounts for **6.4%**, and it also produces a far tighter measurement — standard deviation 7,770 ns against 156,497. The two rows measured on `release` are therefore marginally overstated in the same direction, and remain far above the threshold either way.

The declared constant is fixed while the measurement is not, so the ratio is a property of the pair, not of the runtime alone: it varies 2.1x across these three machines. What does not vary is the sign and the floor — the smallest gap measured, on the newest desktop part, is still 87.6%. The three order monotonically by single-thread performance. The figures quoted in this report are the smallest of the three; the Xeon is the closest match to the `Intel(R) Xeon(R) CPU @ 2.60GHz` in the inherited constant's header and measures 94.2%.

### What the measured runtime is

The attached patch changes the node only. The runtime WASM it executes is nevertheless built with `runtime-benchmarks`, because `node/Cargo.toml:138` propagates the feature and the `benchmark` subcommands are gated behind it. Two things bound what that can explain:

* The measured hot path is unchanged by the feature: `grep -rn 'runtime-benchmarks' pallets/frame-system/src/extensions/` returns nothing, so `CheckWeight` and the extension pipeline compile identically.
* The only `--chain=dev` genesis difference under the feature (`runtime/src/genesis_config_presets.rs:366` versus `:415`) is one additional `initial_high_security_accounts` entry.

Neither accounts for an eight-fold gap, and the direction of the finding does not depend on the exact figure: the constant prices an sr25519 signature and the runtime verifies ML-DSA.

### Target revision

Measured on `Quantus-Network/chain` at `v0.9.0-endless-sky`, one automated version-bump commit after the listed asset revision `3b243b870a5442a6e5f443056f67bbba87d472f5` (*"ci: Automate version bump to v0.9.0-endless-sky (#658)"*). `git diff 3b243b87 v0.9.0-endless-sky --stat` returns four paths and five changed lines: `Cargo.lock`, `node/Cargo.toml` and `runtime/Cargo.toml` (package version `0.8.0-exodus` → `0.9.0-endless-sky`), and `runtime/src/lib.rs` (`spec_version: 143` → `144`). `frame/support/src/weights/extrinsic_weights.rs`, `runtime/src/configs/mod.rs`, `runtime/src/lib.rs:113`, `node/src/benchmarking.rs` and `node/src/service.rs` are byte-identical between the two, so every line reference and every figure here holds on the asset revision.

## Impact Details

**Impact (Medium): "Inaccuracies to the weight (similar to gas) accounting system that under or overcharge by more than 50% of the correct amount (and are not intentional as indicated by the program's terms)".**

The runtime charges 108,157 ns where the measurement gives 871,694 ns — **12.4% of the correct amount, an undercharge of 87.6%** on the most favourable of three hosts and 94.2% on the least. The threshold is 50%.

* **The error applies to every extrinsic, not one call.** `base_extrinsic` is added to each extrinsic's weight by the block-weights configuration. It is a fixed 763,537 ns shortfall per extrinsic, so it dominates cheap calls and is proportionally smaller against heavy ones.
* **Block accounting is wrong in the same proportion.** `benchmark overhead` built blocks of 541 extrinsics; at the declared constant those book 59 ms against 472 ms of measured per-extrinsic overhead.
* **Fees are wrong in the same proportion**, since the length-and-weight fee model prices `base_extrinsic` into every transaction.
* **Nothing in the repository reports the discrepancy.** There is no `runtime/src/weights/` directory, and `benchmark overhead` does not run without the executor fix.

### Not intentional per the programme's terms

The impact class excludes inaccuracies "intentional as indicated by the program's terms". The project does document weight decisions where it makes them, and does regenerate weights — which is what places this constant outside that set.

`runtime/src/configs/mod.rs:78-84` carries a rationale for the block-weight limits it sets deliberately, `ref_time` at 6 s and `proof_size` uncapped. It says nothing about the per-extrinsic base.

`scripts/regenerate_weights.sh`, checked into the asset, is the project's weight-regeneration process. It runs `benchmark pallet` (`:50`) over eight named pallets and writes their `weights.rs`. It never runs `benchmark overhead`, and the strings `overhead`, `ExtrinsicBaseWeight` and `extrinsic_weights` do not appear in it. PR #647 states that "Official weights must be regenerated on the reference benchmark machine before release" — that is the output of this script, which does not include the base constants.

So the constant is not covered by the documented rationale, not covered by the regeneration process, and the subcommand that would produce it does not run. The file carries upstream's generated header verbatim, naming upstream's host, upstream's runtime and upstream's `substrate-node` command.

**Remediation:** apply the executor fix so `benchmark overhead` runs, then regenerate the constant and check the result in under `runtime/src/weights/`, as FRAME expects.

## References

* `chain/runtime/src/lib.rs:113` — `Signature = DilithiumSignatureScheme`
* `chain/runtime/src/configs/mod.rs:86-89` — `with_sensible_defaults`
* `chain/pallets/frame-system/src/limits.rs:323-335` and `:350` — `base_extrinsic` taken from `ExtrinsicBaseWeight`
* `chain/frame/support/src/weights/extrinsic_weights.rs` — the inherited constant and its upstream header
* `chain/node/src/benchmarking.rs:15,43,101,154,159` — the harness signs ML-DSA-87
* Upstream `polkadot-stable2512-2`, `substrate/bin/node/cli/src/benchmarking.rs:31,60` — the same harness signs sr25519
* `chain/node/src/command.rs:506-509` (`Pallet`, unaffected) and `:511,517,523` (`Block`, `Storage`, `Overhead`)
* `chain/node/src/service.rs:559,599` — executor registers only `SubstrateHostFunctions`
* `chain/node/Cargo.toml:133-141`, propagation at `:138` — `runtime-benchmarks` into the runtime
* `chain/scripts/regenerate_weights.sh:50` — the project's regeneration process runs `benchmark pallet` only
* `chain/runtime/src/configs/mod.rs:78-84` — documented rationale, covering `ref_time` and `proof_size`
* `chain/runtime/src/configs/mod.rs:95` — Normal-class length limit, 5 MiB at 75%

## Link to Proof of Concept

<https://gist.github.com/Skydev0h/4143db37bc4e4857397024f6564089d9>

## Proof of Concept

The measurement is made by the project's own benchmarking command, `quantus-node benchmark overhead`, against the project's own runtime, and compared against the constant that same runtime declares.

### One prerequisite: the benchmarking tooling does not run as shipped

`node/Cargo.toml:133-141` propagates `runtime-benchmarks` into `quantus-runtime`, so the runtime WASM imports `ext_benchmarking_*` host functions, but `node/src/service.rs:559,599` register only `sp_io::SubstrateHostFunctions`. The three subcommands that build a client through `service::new_partial` — `Block`, `Storage` and `Overhead` — abort while constructing the runtime (`BenchmarkCmd::Pallet` is routed to `run_with_spec` at `node/src/command.rs:506-509` and is unaffected):

```
Error: Service(Client(VersionInvalid("Other error happened while constructing the runtime:
runtime requires function imports which are not present on the host:
'env:ext_benchmarking_read_write_count_version_1', 'env:ext_benchmarking_proof_size_version_1', ... ")))
```

`enable-benchmarks.patch` (attached, 40 lines) adds `frame_benchmarking::benchmarking::HostFunctions` to the executor and the corresponding dependency. It touches `node/src/service.rs` and `node/Cargo.toml` only. The runtime WASM it then executes is built with `runtime-benchmarks`, because that feature is what gates the `benchmark` subcommands; see the Description for why that cannot account for the gap.

`overhead` is the subcommand that generates this constant.

### Steps

{% stepper %}
{% step %}

## Clone the target repository

```sh
git clone https://github.com/immunefi-team/audit-comp-quantus-chain
cd audit-comp-quantus-chain
git -c advice.detachedHead=false checkout 3b243b870a5442a6e5f443056f67bbba87d472f5   # listed asset revision
```

{% endstep %}

{% step %}

## Clone the proof-of-concept files

```sh
git clone https://gist.github.com/Skydev0h/4143db37bc4e4857397024f6564089d9 w01
```

{% endstep %}

{% step %}

## Configure `LIBCLANG_PATH`

On Ubuntu clang-sys cannot find libclang on its own: `libclang-dev` is a meta package and the library lands in a versioned dir. The project's own CI does the same probe at `.github/actions/ubuntu/action.yml:24-47`.

```sh
for v in 18 17 16 15 14; do
    d=/usr/lib/llvm-$v/lib
    [ -d "$d" ] && ls "$d"/libclang*.so* >/dev/null 2>&1 && { export LIBCLANG_PATH=$d; break; }
done
```

{% endstep %}

{% step %}

## Apply the executor patch

```sh
git apply w01/enable-benchmarks.patch      # node executor only; see above
```

{% endstep %}

{% step %}

## Build the node

```sh
cargo build --release --features runtime-benchmarks -p quantus-node
```

{% endstep %}

{% step %}

## Run the overhead benchmark

```sh
mkdir -p /tmp/qbench-out      # --weight-path must be an existing directory

./target/release/quantus-node benchmark overhead \
    --chain=dev --base-path /tmp/qbench --weight-path /tmp/qbench-out \
    --warmup=10 --repeat=100 --wasm-execution=compiled
```

{% endstep %}
{% endstepper %}

The key file is provided in the following section and in the linked gist:

<details>

<summary><code>enable-benchmarks.patch</code></summary>

```diff
diff --git a/node/Cargo.toml b/node/Cargo.toml
index f9b4c2c3..d65a1eff 100644
--- a/node/Cargo.toml
+++ b/node/Cargo.toml
@@ -17,6 +17,7 @@ targets = ["x86_64-unknown-linux-gnu"]
 [dependencies]
 clap = { features = ["derive"], workspace = true }
 codec.workspace = true
+frame-benchmarking = { workspace = true, default-features = false }
 frame-benchmarking-cli = { workspace = true, default-features = true, optional = true }
 frame-metadata-hash-extension.default-features = true
 frame-metadata-hash-extension.workspace = true
diff --git a/node/src/service.rs b/node/src/service.rs
index 61e8ae44..ad732732 100644
--- a/node/src/service.rs
+++ b/node/src/service.rs
@@ -556,7 +556,10 @@ fn spawn_authority_tasks(
 pub(crate) type FullClient = sc_service::TFullClient<
 	Block,
 	RuntimeApi,
-	sc_executor::WasmExecutor<sp_io::SubstrateHostFunctions>,
+	sc_executor::WasmExecutor<(
+		sp_io::SubstrateHostFunctions,
+		frame_benchmarking::benchmarking::HostFunctions,
+	)>,
 >;
 type FullBackend = sc_service::TFullBackend<Block>;
 pub type PowBlockImport = sc_consensus_qpow::PowBlockImport<
@@ -596,7 +599,10 @@ pub fn new_partial(config: &Configuration) -> Result<Service, ServiceError> {
 		})
 		.transpose()?;
 
-	let executor = sc_service::new_wasm_executor::<sp_io::SubstrateHostFunctions>(&config.executor);
+	let executor = sc_service::new_wasm_executor::<(
+		sp_io::SubstrateHostFunctions,
+		frame_benchmarking::benchmarking::HostFunctions,
+	)>(&config.executor);
 	let (client, backend, keystore_container, task_manager) =
 		sc_service::new_full_parts::<Block, RuntimeApi, _>(
 			config,
```

</details>

`run.sh` resolves the patch relative to its own location, so the gist can be cloned anywhere.

<details>

<summary><code>run.sh</code></summary>

```sh
#!/usr/bin/env bash
# The build-and-measure half of the Steps section, for convenience: it sets
# LIBCLANG_PATH, applies the executor patch, builds, and runs the benchmark.
#
# Assumes you have already done, from the Steps block:
#     git clone https://github.com/immunefi-team/audit-comp-quantus-chain
#     cd audit-comp-quantus-chain
#     git checkout 3b243b870a5442a6e5f443056f67bbba87d472f5
#     git clone <the URL of this gist> w01
#
# Then, from the root of that checkout:  bash w01/run.sh
#
# The patch is resolved relative to this script, so it is found wherever the
# gist was cloned to.
set -euo pipefail

HERE="$(cd "$(dirname "${BASH_SOURCE[0]}")" && pwd)"

# On Ubuntu clang-sys cannot find libclang on its own: `libclang-dev` is a meta
# package and the library lands in a versioned directory that is not on the
# default search path. The project's own CI runs the same probe, see
# .github/actions/ubuntu/action.yml:24-47.
for v in 18 17 16 15 14; do
    d=/usr/lib/llvm-$v/lib
    [ -d "$d" ] && ls "$d"/libclang*.so* >/dev/null 2>&1 && { export LIBCLANG_PATH=$d; break; }
done
echo "LIBCLANG_PATH=${LIBCLANG_PATH:-<unset, build may fail>}"

# The benchmarking tooling does not run as shipped: the runtime built with
# `runtime-benchmarks` imports nine ext_benchmarking_* host functions that
# node/src/service.rs never registers. This patch touches ONLY the node's
# executor and its dependency list. The runtime is not modified.
PATCH="$HERE/enable-benchmarks.patch"
if git apply --check "$PATCH" 2>/dev/null; then
    git apply "$PATCH"
    echo "executor patch applied"
elif git apply --reverse --check "$PATCH" 2>/dev/null; then
    echo "executor patch already applied, skipping"
else
    echo "ERROR: $PATCH does not apply cleanly and is not already applied." >&2
    echo "       Are you at 3b243b870a5442a6e5f443056f67bbba87d472f5 ?" >&2
    exit 1
fi

cargo build --release --features runtime-benchmarks -p quantus-node

# --weight-path must be an existing directory
mkdir -p /tmp/qbench-out

./target/release/quantus-node benchmark overhead \
    --chain=dev \
    --base-path /tmp/qbench \
    --weight-path /tmp/qbench-out \
    --warmup=10 --repeat=100 \
    --wasm-execution=compiled

# /tmp/qbench-out/extrinsic_weights.rs and block_weights.rs now hold the
# measured values for this runtime. Compare them against the checked-in
# frame/support/src/weights/{extrinsic,block}_weights.rs.
```

</details>

### Observed output

```
Running 10 warmups...
Executing block 100 times
Per-block execution overhead [ns]:
Total: 108499301
Min: 1061726, Max: 1608179
Average: 1084993, Median: 1075596, Stddev: 68315.71
Percentiles 99th, 95th, 75th: 1506748, 1090976, 1081526
Writing weights to "/tmp/qbench-out/block_weights.rs"

Building block, this takes some time...
Extrinsics per block: 541
Running 10 warmups...
Executing block 100 times
Per-extrinsic execution overhead [ns]:
Total: 128820669
Min: 1246975, Max: 2360579
Average: 1288206, Median: 1256626, Stddev: 156497.43
Percentiles 99th, 95th, 75th: 2166001, 1293275, 1266272
Writing weights to "/tmp/qbench-out/extrinsic_weights.rs"
```

### The comparison

| Host                            | profile    |   declared |     measured | undercharge |
| ------------------------------- | ---------- | ---------: | -----------: | ----------: |
| Core i9-14900K (2023, desktop)  | release    | 108,157 ns |   871,694 ns |   **87.6%** |
| Ryzen 7 5800X (2020, desktop)   | release    | 108,157 ns | 1,288,206 ns |   **91.6%** |
| Ryzen 7 5800X, same machine     | production | 108,157 ns | 1,205,508 ns |   **91.0%** |
| Xeon E5-2687W v4 (2016, server) | release    | 108,157 ns | 1,865,545 ns |   **94.2%** |

`undercharge` = `(measured − declared) / measured`, the shortfall as a fraction of the correct amount, matching the impact class wording. The threshold is 50%.

The console block above is the Ryzen `release` run — the second row. The third is the same machine rebuilt with `--profile production`, matching the profile upstream used for 108,157 ns; it isolates the build profile at 6.4%. The other two are in the attached generated files, each stamped by the tool with its own hostname and CPU. The percentages are of the correct amount, matching the impact class wording; the threshold is 50%.

Diff the generated files against the checked-in `frame/support/src/weights/extrinsic_weights.rs`, which is byte-identical to `polkadot-stable2512-2` and carries upstream's header naming upstream's host, runtime and `substrate-node` command.

### Where the time goes

On the same host, the criterion benches shipped with the two crypto crates give an ML-DSA-87 verification at 314.5 µs (min 200.8, max 453.0) and a Poseidon2 hash at 79.5 µs for 2,048 bytes and 163.2 µs for 4,096, bracketing the 2,592-byte public key used for account derivation at roughly 100 µs. Both were run from the crates' own repositories at the versions this runtime consumes — `qp-rusty-crystals-dilithium` 4.1.0 and `qp-poseidon-core` 3.1.0 — not from this checkout, where they are registry dependencies. Verification alone is about three times the entire 108.157 µs the runtime budgets for an extrinsic, natively, before the WASM penalty.

### Scope check

The constant itself is unmodified upstream: `frame/support/src/weights/extrinsic_weights.rs` diffs empty against `polkadot-stable2512-2`. The defect is not in that file.

The line that pulls it into this runtime is Quantus's own, and they edited it. Upstream's solochain template reads:

```rust
BlockWeights::with_sensible_defaults(
    Weight::from_parts(2u64 * WEIGHT_REF_TIME_PER_SECOND, u64::MAX),
```

`runtime/src/configs/mod.rs:86-89` reads `6u64`, changed in commit `3a65e8b9` ("Config Update + SigHandler (#133)"). So the block-weight configuration was tuned for this chain — the compute budget was tripled — while the per-extrinsic base it seeds was left at upstream's sr25519 measurement, alongside `runtime/src/lib.rs:113` replacing the signature scheme entirely. The composition of those two decisions is Quantus's.

### Additional supplementary files

The tool wrote these itself via `--weight-path`. The same command also emits a `block_weights.rs` per run; those are in the gist for completeness and this report makes no claim about `BlockExecutionWeight`. Each carries the header its generator stamps in — CLI version, hostname and CPU, runtime name, warmup and repeat counts, and the exact command — so the provenance of the measurement is machine-written, and lines up directly against the header on the checked-in `frame/support/src/weights/{extrinsic,block}_weights.rs`, which names upstream's host, upstream's runtime and upstream's `substrate-node` command. They also carry the full distribution rather than just the average quoted above: median, standard deviation, and the 99th/95th/75th percentiles. Being in the canonical generated format, they are the remediation as well — checking them in under `runtime/src/weights/` is the fix.

<details>

<summary><code>measured_extrinsic_weights.rs</code> — Ryzen 7 5800X, release</summary>

```rust
//! THIS FILE WAS AUTO-GENERATED USING THE SUBSTRATE BENCHMARK CLI VERSION 53.0.0
//! DATE: 2026-08-12 (Y/M/D)
//! HOSTNAME: `soliton`, CPU: `AMD Ryzen 7 5800X 8-Core Processor`
//!
//! SHORT-NAME: `extrinsic`, LONG-NAME: `ExtrinsicBase`, RUNTIME: `Quantus DevNet`
//! WARMUPS: `10`, REPEAT: `100`
//! WEIGHT-PATH: `/tmp/bench/out`
//! WEIGHT-METRIC: `Average`, WEIGHT-MUL: `1.0`, WEIGHT-ADD: `0`

// Executed Command:
//   ./target/release/quantus-node
//   benchmark
//   overhead
//   --chain=dev
//   --base-path
//   /tmp/bench/ov
//   --weight-path
//   /tmp/bench/out
//   --warmup=10
//   --repeat=100
//   --wasm-execution=compiled

use sp_core::parameter_types;
use sp_weights::{constants::WEIGHT_REF_TIME_PER_NANOS, Weight};

parameter_types! {
	/// Weight of executing a NO-OP extrinsic, for example `System::remark`.
	/// Calculated by multiplying the *Average* with `1.0` and adding `0`.
	///
	/// Stats nanoseconds:
	///   Min, Max: 1_246_975, 2_360_579
	///   Average:  1_288_206
	///   Median:   1_256_626
	///   Std-Dev:  156497.43
	///
	/// Percentiles nanoseconds:
	///   99th: 2_166_001
	///   95th: 1_293_275
	///   75th: 1_266_272
	pub const ExtrinsicBaseWeight: Weight =
		Weight::from_parts(WEIGHT_REF_TIME_PER_NANOS.saturating_mul(1_288_206), 0);
}

#[cfg(test)]
mod test_weights {
	use sp_weights::constants;

	/// Checks that the weight exists and is sane.
	// NOTE: If this test fails but you are sure that the generated values are fine,
	// you can delete it.
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

</details>

<details>

<summary><code>measured_extrinsic_weights_xeon.rs</code> — Xeon E5-2687W v4, release</summary>

```rust
//! THIS FILE WAS AUTO-GENERATED USING THE SUBSTRATE BENCHMARK CLI VERSION 53.0.0
//! DATE: 2026-08-12 (Y/M/D)
//! HOSTNAME: `xxlab`, CPU: `Intel(R) Xeon(R) CPU E5-2687W v4 @ 3.00GHz`
//!
//! SHORT-NAME: `extrinsic`, LONG-NAME: `ExtrinsicBase`, RUNTIME: `Quantus DevNet`
//! WARMUPS: `10`, REPEAT: `100`
//! WEIGHT-PATH: `/tmp/qbench-out`
//! WEIGHT-METRIC: `Average`, WEIGHT-MUL: `1.0`, WEIGHT-ADD: `0`

// Executed Command:
//   ./target/release/quantus-node
//   benchmark
//   overhead
//   --chain=dev
//   --base-path
//   /tmp/qbench
//   --weight-path
//   /tmp/qbench-out
//   --warmup=10
//   --repeat=100
//   --wasm-execution=compiled

use sp_core::parameter_types;
use sp_weights::{constants::WEIGHT_REF_TIME_PER_NANOS, Weight};

parameter_types! {
	/// Weight of executing a NO-OP extrinsic, for example `System::remark`.
	/// Calculated by multiplying the *Average* with `1.0` and adding `0`.
	///
	/// Stats nanoseconds:
	///   Min, Max: 1_825_861, 2_368_518
	///   Average:  1_865_545
	///   Median:   1_851_598
	///   Std-Dev:  66830.23
	///
	/// Percentiles nanoseconds:
	///   99th: 2_178_914
	///   95th: 1_927_760
	///   75th: 1_860_926
	pub const ExtrinsicBaseWeight: Weight =
		Weight::from_parts(WEIGHT_REF_TIME_PER_NANOS.saturating_mul(1_865_545), 0);
}

#[cfg(test)]
mod test_weights {
	use sp_weights::constants;

	/// Checks that the weight exists and is sane.
	// NOTE: If this test fails but you are sure that the generated values are fine,
	// you can delete it.
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

</details>

<details>

<summary><code>measured_extrinsic_weights_i9.rs</code> — Core i9-14900K, release</summary>

```rust
//! THIS FILE WAS AUTO-GENERATED USING THE SUBSTRATE BENCHMARK CLI VERSION 53.0.0
//! DATE: 2026-08-12 (Y/M/D)
//! HOSTNAME: `SkyFlare`, CPU: `Intel(R) Core(TM) i9-14900K`
//!
//! SHORT-NAME: `extrinsic`, LONG-NAME: `ExtrinsicBase`, RUNTIME: `Quantus DevNet`
//! WARMUPS: `10`, REPEAT: `100`
//! WEIGHT-PATH: `/tmp/qbench-out`
//! WEIGHT-METRIC: `Average`, WEIGHT-MUL: `1.0`, WEIGHT-ADD: `0`

// Executed Command:
//   ./target/release/quantus-node
//   benchmark
//   overhead
//   --chain=dev
//   --base-path
//   /tmp/qbench
//   --weight-path
//   /tmp/qbench-out
//   --warmup=10
//   --repeat=100
//   --wasm-execution=compiled

use sp_core::parameter_types;
use sp_weights::{constants::WEIGHT_REF_TIME_PER_NANOS, Weight};

parameter_types! {
	/// Weight of executing a NO-OP extrinsic, for example `System::remark`.
	/// Calculated by multiplying the *Average* with `1.0` and adding `0`.
	///
	/// Stats nanoseconds:
	///   Min, Max: 845_313, 921_813
	///   Average:  871_694
	///   Median:   870_110
	///   Std-Dev:  13961.59
	///
	/// Percentiles nanoseconds:
	///   99th: 914_153
	///   95th: 896_061
	///   75th: 878_017
	pub const ExtrinsicBaseWeight: Weight =
		Weight::from_parts(WEIGHT_REF_TIME_PER_NANOS.saturating_mul(871_694), 0);
}

#[cfg(test)]
mod test_weights {
	use sp_weights::constants;

	/// Checks that the weight exists and is sane.
	// NOTE: If this test fails but you are sure that the generated values are fine,
	// you can delete it.
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

</details>

<details>

<summary><code>measured_extrinsic_weights_production.rs</code> — Ryzen 7 5800X, production</summary>

```rust
//! THIS FILE WAS AUTO-GENERATED USING THE SUBSTRATE BENCHMARK CLI VERSION 53.0.0
//! DATE: 2026-08-12 (Y/M/D)
//! HOSTNAME: `soliton`, CPU: `AMD Ryzen 7 5800X 8-Core Processor`
//!
//! SHORT-NAME: `extrinsic`, LONG-NAME: `ExtrinsicBase`, RUNTIME: `Quantus DevNet`
//! WARMUPS: `10`, REPEAT: `100`
//! WEIGHT-PATH: `/tmp/bench/outp`
//! WEIGHT-METRIC: `Average`, WEIGHT-MUL: `1.0`, WEIGHT-ADD: `0`

// Executed Command:
//   ./target/production/quantus-node
//   benchmark
//   overhead
//   --chain=dev
//   --base-path
//   /tmp/bench/ovp5
//   --weight-path
//   /tmp/bench/outp
//   --warmup=10
//   --repeat=100
//   --wasm-execution=compiled

use sp_core::parameter_types;
use sp_weights::{constants::WEIGHT_REF_TIME_PER_NANOS, Weight};

parameter_types! {
	/// Weight of executing a NO-OP extrinsic, for example `System::remark`.
	/// Calculated by multiplying the *Average* with `1.0` and adding `0`.
	///
	/// Stats nanoseconds:
	///   Min, Max: 1_197_217, 1_232_395
	///   Average:  1_205_508
	///   Median:   1_202_581
	///   Std-Dev:  7770.36
	///
	/// Percentiles nanoseconds:
	///   99th: 1_228_036
	///   95th: 1_222_328
	///   75th: 1_207_899
	pub const ExtrinsicBaseWeight: Weight =
		Weight::from_parts(WEIGHT_REF_TIME_PER_NANOS.saturating_mul(1_205_508), 0);
}

#[cfg(test)]
mod test_weights {
	use sp_weights::constants;

	/// Checks that the weight exists and is sane.
	// NOTE: If this test fails but you are sure that the generated values are fine,
	// you can delete it.
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

</details>
