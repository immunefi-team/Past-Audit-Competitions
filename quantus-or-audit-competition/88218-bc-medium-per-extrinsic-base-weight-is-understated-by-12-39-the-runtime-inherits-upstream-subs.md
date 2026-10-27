# 88218 bc medium per extrinsic base weight is understated by 12 39 the runtime inherits upstream substrate s sr25519 constant while verifying post quantum dilithium signatures

## #88218 \[BC-Medium] Per-extrinsic base weight is understated by 12.39×: the runtime inherits upstream Substrate's sr25519 constant while verifying post-quantum Dilithium signatures

**Submitted on Aug 12th 2026 at 09:45:08 UTC by @shadowHunter for** [**Audit Comp | Quantus**](https://immunefi.com/audit-competition/audit-comp-quantus)

* **Report ID:** #88218
* **Report Type:** Blockchain/DLT
* **Report severity:** Medium
* **Target:** <https://github.com/immunefi-team/audit-comp-quantus-chain>
* **Impacts:**
  * Causing network processing nodes to process transactions from the mempool beyond set parameters

### Description

### 1. Summary

`frame_system::Config::BlockWeights` sets a **base weight charged once per extrinsic**, intended to cover extrinsic decoding and signature verification. Quantus never overrides this value, so it inherits upstream Substrate's constant of **108,157 ns**.

That constant was measured on the stock Substrate development chain, which uses **sr25519** signatures verified through a **native host function**. Quantus does not use sr25519. It uses **Dilithium (ML-DSA-65/87)**, verified **inside the runtime WASM** with no host function, plus a Poseidon hash over the entire public key to derive the account.

Running the project's own `benchmark overhead` command against a release build of this chain produces:

| Constant               | Declared (inherited) | Measured (this chain) | Under-declaration |
| ---------------------- | -------------------: | --------------------: | ----------------: |
| `ExtrinsicBaseWeight`  |           108,157 ns |      **1,340,574 ns** |        **12.39×** |
| `BlockExecutionWeight` |           431,614 ns |      **1,223,182 ns** |         **2.83×** |

Because `BlockWeight` is the mechanism that bounds how long a block takes to execute, the limit is measuring the wrong quantity. A block that fills the normal-dispatch length budget records **58.5 ms** of weight while really taking **725.3 ms** to execute — **666.7 ms per block of completely unmetered CPU**.

A related consequence has no fee attached at all: the same verification cost is paid by every node for every **gossiped** transaction during pool validation, where no fee is charged and the sender need not be solvent.

### 2. Description

#### 2.1 What the base weight is for

Every extrinsic in a FRAME runtime is charged `base_extrinsic` in addition to its call weight. This charge exists to account for the work that happens *before* the call body runs — SCALE decoding and, critically, **signature verification**. It is added in `CheckWeight`:

```rust
// pallets/frame-system/src/extensions/check_weight.rs:163-166
let extrinsic_weight = info
    .total_weight()
    .saturating_add(maximum_weight.get(info.class).base_extrinsic)
    .saturating_add(Weight::from_parts(0, len as u64));
```

`info.total_weight()` is the call weight plus declared extension weights. None of those cover the signature check — the extension weights in `pallets/frame-system/src/extensions/weights.rs` are small, separately benchmarked values (`check_nonce()` ≈ 7.26 µs, `check_weight()` ≈ 3.0 µs) that never touch the signature.

So `base_extrinsic` is the **only** budget covering signature verification.

#### 2.2 Quantus never sets it

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

#### 2.3 The inherited constant describes a different chain

The constant's own auto-generated header records exactly how it was produced:

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

`RUNTIME: Development` on `substrate-node --chain=dev` is the stock Substrate node, whose signature type is `MultiSignature` and whose sr25519 verification is a **native host function call** (`sp_io::crypto::sr25519_verify`), costing on the order of tens of microseconds.

#### 2.4 What Quantus actually does per extrinsic

```rust
// runtime/src/lib.rs:112-113
// pub type Signature = MultiSignature;
pub type Signature = DilithiumSignatureScheme;
```

The upstream type is commented out directly above the replacement. Verification runs entirely in-runtime:

```rust
// primitives/dilithium-crypto/src/traits.rs:160-184
Self::Dilithium65(sig_public) => {
    let account = sig_public.public().clone().into_account();   // Poseidon over the FULL public key
    if account != *signer { return false; }
    crate::verify_ml_dsa_65(
        sig_public.public().as_ref(), msg.get(), sig_public.signature().as_ref())
}
```

Two expensive operations per extrinsic:

1. `into_account()` — `primitives/dilithium-crypto/src/traits.rs:197-198` — is `qp_poseidon_core::hash_bytes` over the **entire** public key (1,952 bytes for ML-DSA-65, 2,592 bytes for ML-DSA-87).
2. `verify_ml_dsa_*` — `primitives/dilithium-crypto/src/scheme_macro.rs:493-501` — is pure Rust lattice verification from `qp-rusty-crystals-dilithium`, with **no host function**, so it executes interpreted/JIT-compiled inside the WASM sandbox.

This runs on every extrinsic on every node during block execution, via `frame/executive/src/lib.rs:711`, and again for every gossiped transaction during pool validation via `frame/executive/src/lib.rs:990`.

### 3. Steps to reproduce

These are the plain steps. The complete, runnable PoC with full scripts and captured output is in Section 6.

{% stepper %}
{% step %}

### A user (or attacker) submits an ordinary signed transaction

Any signed extrinsic works; the simplest is a no-op remark. Nothing unusual is required — this is normal chain traffic:

```
System::remark { remark: vec![] }
```

signed with a Dilithium key. In the repo's own benchmarking harness this is built by `RemarkBuilder`, which signs with the dev account `dilithium_bob()`:

```rust
// node/src/benchmarking.rs:44-55
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

{% endstep %}

{% step %}

### The runtime charges the inherited base weight for it

`CheckWeight` adds `base_extrinsic` to the consumed block weight:

```rust
// pallets/frame-system/src/extensions/check_weight.rs:163-166
let extrinsic_weight = info
    .total_weight()
    .saturating_add(maximum_weight.get(info.class).base_extrinsic)
    .saturating_add(Weight::from_parts(0, len as u64));
```

Because nothing overrides it, `base_extrinsic` is **108,157 ns**.
{% endstep %}

{% step %}

### The node actually spends \~12× that verifying the signature

Before the call body runs, every node executes the Dilithium verification and the Poseidon account derivation from §2.4 inside the WASM runtime. Measured cost on this chain: **1,340,574 ns**.
{% endstep %}

{% step %}

### Fill a block with ordinary signed transactions

The normal-dispatch length budget is `0.75 × 5 MiB = 3,932,160 bytes` (`runtime/src/configs/mod.rs:95`). An ML-DSA-87 signed remark encodes to roughly 7,266 bytes, so a full block holds about 541 of them.

This figure does not need to be trusted — **the benchmark tool derived it independently** during the PoC run:

```
2026-08-11 11:30:24 Extrinsics per block: 541
```

{% endstep %}

{% step %}

### Observe the accounting gap

|                                    |        Per block (541 extrinsics) |
| ---------------------------------- | --------------------------------: |
| Weight recorded into `BlockWeight` |    541 × 108,157 ns = **58.5 ms** |
| Real execution time                | 541 × 1,340,574 ns = **725.3 ms** |
| **Unmetered**                      |                      **666.7 ms** |

The block reports consuming 1.3 % of the 4.5-second normal-class weight budget while actually costing 725 ms of CPU on every node that imports it. The `max_block` guard cannot see the difference, so it never engages.
{% endstep %}

{% step %}

### The unpriced variant: no fee at all

Steps 1–5 describe traffic that pays fees. The same verification is also performed during **pool validation** of gossiped transactions:

```rust
// frame/executive/src/lib.rs:983-991  (validate_transaction path)
let uxt = <Block::Extrinsic as codec::DecodeLimit>::decode_all_with_depth_limit(
    MAX_EXTRINSIC_DEPTH,
    &mut &encoded[..],
)
.map_err(|_| InvalidTransaction::Call)?;

let xt = within_span! { sp_tracing::Level::TRACE, "check";
    uxt.check(&Default::default())     // <-- full Dilithium verify + Poseidon derive
}?;
```

`uxt.check(..)` is the same `Verify::verify` path shown in §2.4 — the Dilithium lattice verification and the Poseidon hash over the full public key. It runs here **before** any nonce, solvency, or fee check.

Here no fee is charged and the sender is not required to be solvent. An attacker generates throwaway Dilithium keys, signs transactions with a nonce that will fail, and broadcasts them. Every node performs the full \~1.34 ms verification before discovering the transaction is invalid, and nothing is ever charged because the transaction never enters a block.
{% endstep %}
{% endstepper %}

### 4. Impact

**Primary: the block weight limit no longer bounds block execution time.**

`BlockWeight` exists so that a block cannot take longer to execute than the network can afford between blocks. With the per-extrinsic base understated 12.39×, that ledger is wrong by **666.7 ms on a full block** — against a 12-second target block time (`runtime/src/lib.rs:87`), roughly **5.6 % of the entire block interval** is consumed by work no one is metering.

This is not hypothetical or attacker-dependent: **ordinary, fully-paid, legitimate traffic already produces this gap.** A miner filling blocks with normal transactions produces blocks whose recorded weight says 58.5 ms and which take \~725 ms to re-execute on every importing node.

Consequences:

* Slower or more loaded nodes fall progressively behind on import, because the safety margin that was supposed to prevent this is measuring a number 12× too small.
* On a proof-of-work chain whose difficulty controller targets wall-clock block time, systematically longer import times shift the effective time budget in a direction the weight system cannot observe or throttle.
* Any future capacity planning based on `BlockWeight` is built on a figure that is wrong by more than an order of magnitude.

**Secondary: unpriced CPU consumption via the transaction pool.** As described in Step 6, gossiped-but-invalid transactions impose the full \~1.34 ms verification on every node with no fee and no solvency requirement.

**Also affected: `BlockExecutionWeight`**, the per-block constant, is likewise inherited and understated — 431,614 ns declared (`frame/support/src/weights/block_weights.rs:56`, from the same 2024-11-08 upstream `substrate-node --chain=dev` run) versus 1,223,182 ns measured, 2.83× — adding a further \~0.79 ms of unmetered time per block.

#### What this is *not*

Stated plainly so the severity is not overread: **this is not fee arbitrage.** `LengthToFeeMultiplier` charges 1e6 planck/byte (`runtime/src/configs/mod.rs:445`) and `WeightToFee = IdentityFee` charges 1 planck/ps (`runtime/src/configs/mod.rs:483`). A 7,266-byte extrinsic therefore pays roughly 7.27 ms-equivalent in length fees for \~1.34 ms of real base work — the signature blob's own length fee **over-covers** its verification by about 5×. An attacker cannot buy in-block compute below cost through this path.

The defect is in the **weight ledger** that protects block production and import time, and in the **pool path**, which has no fee at all.

### 5. Recommendation

#### 5.1 Generate the correct constants

The tooling already exists in the repository and is fully written — `node/src/benchmarking.rs:19-53` implements `ExtrinsicBuilder`, and `node/src/benchmarking.rs:101,154-159` already signs with `qp_dilithium_crypto::Dilithium87Pair` / `runtime::Signature::Dilithium87`. It has simply never been runnable (see §5.3).

```bash
quantus-node benchmark overhead \
  --chain=dev --wasm-execution=compiled \
  --weight-path=./runtime/src/weights/
```

#### 5.2 Stop inheriting the upstream defaults

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

Note that the correct value (1.341 ms) **exceeds Substrate's own sanity ceiling** of 1 ms — see §6.6. The generated `sane()` test will fail and the team will need to make a deliberate decision about that ceiling rather than silently deleting the test.

#### 5.3 Fix the benchmark executor so this can be measured at all

`node/src/service.rs` hardcodes the executor host functions with no `runtime-benchmarks` variant:

```rust
// node/src/service.rs:559
sc_executor::WasmExecutor<sp_io::SubstrateHostFunctions>
// node/src/service.rs:599
sc_service::new_wasm_executor::<sp_io::SubstrateHostFunctions>(&config.executor)
```

A runtime built with `runtime-benchmarks` imports `ext_benchmarking_*` host functions the node cannot provide, so **every** benchmark subcommand that constructs a client (`overhead`, `extrinsic`, `block`, `storage`) aborts at runtime instantiation. Only `benchmark pallet` works, because it uses `frame-benchmarking-cli`'s own executor — which is exactly why `scripts/regenerate_weights.sh` only ever invokes `benchmark pallet`.

The standard upstream wiring (also applied as instrumentation for this PoC, §6.4):

```rust
#[cfg(not(feature = "runtime-benchmarks"))]
pub type HostFunctions = sp_io::SubstrateHostFunctions;
#[cfg(feature = "runtime-benchmarks")]
pub type HostFunctions =
    (sp_io::SubstrateHostFunctions, frame_benchmarking::benchmarking::HostFunctions);
```

#### 5.4 Address the unpriced pool path separately

Correcting the weight constants does **not** fix Step 6. Pool validation performs full Dilithium verification for free. Consider per-peer rate limiting on transaction gossip, or a cheap pre-filter (for example, checking the nonce against state before verifying the signature) so an invalid transaction is rejected before the expensive operation.

## Proof of Concept

### 6. Proof of concept

Everything below is complete and self-contained. No external references, no downloads beyond the repository itself.

#### 6.1 Environment used

|             |                                                |
| ----------- | ---------------------------------------------- |
| OS          | Ubuntu 24.04.1 LTS (WSL2 on Windows)           |
| CPU         | AMD Ryzen 7 5800H                              |
| Cores / RAM | 12 cores, 9.9 GB                               |
| Rust        | 1.93.0 (pinned by the repo's `rust-toolchain`) |
| Targets     | `wasm32v1-none` (required by `rust-toolchain`) |

#### 6.2 Setup — obtain and pin the source

```bash
git clone https://github.com/immunefi-team/audit-comp-quantus-chain quantus-chain
cd quantus-chain
git checkout audit-comp-ready      # 3b243b870a5442a6e5f443056f67bbba87d472f5 (in-scope commit)
```

Install the toolchain and system dependencies:

```bash
rustup toolchain install 1.93.0
rustup target add wasm32v1-none --toolchain 1.93.0
apt-get install -y clang libclang-dev protobuf-compiler pkg-config libssl-dev cmake build-essential
```

**Required environment variable.** The build fails at `clang-sys v1.8.1` with `"couldn't find any valid shared libraries matching: ['libclang.so', ...]"` unless `libclang` is locatable. On Ubuntu 24.04 it lives outside the default search path:

```bash
export LIBCLANG_PATH=/usr/lib/llvm-18/lib
```

#### 6.3 Build

```bash
cargo build --release --features runtime-benchmarks -p quantus-node -j 6
```

The `runtime-benchmarks` feature is mandatory: the entire `Benchmark` subcommand is gated behind it at `node/src/command.rs:498`. Build time on the hardware above was **46 minutes**. Use `-j 6` rather than the full core count — `pallet-wormhole`'s build script peaks around 4.8 GB RSS during ZK circuit generation.

Verify the binary is built from the intended commit:

```
$ ./target/release/quantus-node --version
quantus-node 0.9.0-endless-sky-223d938923a
```

#### 6.4 Required instrumentation patch (and why it does not affect the result)

As described in §5.3, `benchmark overhead` **cannot run** on this repository as shipped. The following patch applies the standard upstream host-function wiring so the project's own command can start.

Save as `poc1_patch.diff` in the repository root and apply with `git apply poc1_patch.diff`:

```diff
diff --git a/node/Cargo.toml b/node/Cargo.toml
index f9b4c2c3..4ff79942 100644
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
index 61e8ae44..2fe90e07 100644
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

**Why this cannot influence the measurement.** The patch registers additional host functions with the WASM executor. Those functions (`ext_benchmarking_current_time`, `ext_benchmarking_commit_db`, `ext_benchmarking_read_write_count`, and so on) are called **only** from inside pallet benchmark code paths. `benchmark overhead` measures the execution of ordinary `System::remark` extrinsics and empty blocks, which never invoke any of them. The patch changes only whether the runtime can be *instantiated*, not what the timed code does.

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

Rebuild after applying:

```bash
cargo build --release --features runtime-benchmarks -p quantus-node -j 6
```

#### 6.5 The PoC script

Save the following as `poc1_run.sh` in the repository root, then `chmod +x poc1_run.sh && ./poc1_run.sh`. It captures the complete evidence chain — the declared constant, its provenance, proof of no override, the signature type, and finally the measurement.

```bash
#!/usr/bin/env bash
# PoC - Finding #1: ExtrinsicBaseWeight is upstream's sr25519 constant on a
# post-quantum chain. Runs the project's OWN `benchmark overhead` tool, whose
# RemarkBuilder signs with dilithium_bob() - i.e. a real Dilithium signature.
set -uo pipefail

CHAIN_DIR="$HOME/quantus-poc/chain"
NODE="$CHAIN_DIR/target/release/quantus-node"
OUT="$HOME/quantus-poc/poc1-basefee/out"
mkdir -p "$OUT"

echo "=============================================================="
echo " PoC #1  ExtrinsicBaseWeight under-declaration"
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
echo "### benchmark overhead --help (capture real flag names for this CLI version)"
"$NODE" benchmark overhead --help > "$OUT/overhead-help.txt" 2>&1
sed -n '1,70p' "$OUT/overhead-help.txt"

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
echo "### RUNNING benchmark overhead - measures the REAL per-extrinsic base"
"$NODE" benchmark overhead \
  --chain=dev \
  --wasm-execution=compiled \
  --weight-path="$OUT" \
  --warmup=10 \
  --repeat=100 \
  2>&1 | tee "$OUT/overhead-run.txt"
RC=${PIPESTATUS[0]}
echo "benchmark overhead exit code: $RC" | tee -a "$OUT/overhead-run.txt"

echo
echo "### GENERATED OUTPUT FILES"
ls -la "$OUT"
for f in "$OUT"/*.rs; do
  [ -e "$f" ] || continue
  echo "----- $f -----"
  cat "$f"
done

echo
echo "=============================================================="
echo " COMPARE: measured ExtrinsicBase above vs inherited 108_157 ns"
echo "=============================================================="
```

Adjust `CHAIN_DIR` to wherever the repository was cloned. `OUT` may be any writable directory.

#### 6.6 PoC output

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

### PROOF the runtime never overrides it
(no results - the constant is never overridden in runtime/)

### CONFIRM the signature scheme is Dilithium, not sr25519
112:// pub type Signature = MultiSignature;
113:pub type Signature = DilithiumSignatureScheme;
```

**The benchmark run:**

```
2026-08-11 11:30:20 Genesis: Set initial difficulty to 3d0900
2026-08-11 11:30:21 🔨 Initializing Genesis block/state (state: 0x9ee4…7a21, header-hash: 0x4417…9732)
2026-08-11 11:30:21 Initialized genesis block 0x44172100426ce53b25ce2a573d3c9918d76ab5f889862eeba9d6eecda34e9732 achieved work to 1
2026-08-11 11:30:21 Creating transaction pool txpool_type=SingleState ready=Limit { count: 8192, total_bytes: 20971520 } future=Limit { count: 512, total_bytes: 1048576 }
2026-08-11 11:30:21 Running 10 warmups...
2026-08-11 11:30:21 Executing block 100 times
2026-08-11 11:30:21 Per-block execution overhead [ns]:
Total: 122318294
Min: 1085528, Max: 2235841
Average: 1223182, Median: 1166226, Stddev: 229734.18
Percentiles 99th, 95th, 75th: 2194911, 1666796, 1192597
2026-08-11 11:30:22 Writing weights to "/root/quantus-poc/poc1-basefee/out/block_weights.rs"
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
2026-08-11 11:31:43 Writing weights to "/root/quantus-poc/poc1-basefee/out/extrinsic_weights.rs"
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
//! WEIGHT-PATH: `/root/quantus-poc/poc1-basefee/out`
//! WEIGHT-METRIC: `Average`, WEIGHT-MUL: `1.0`, WEIGHT-ADD: `0`

// Executed Command:
//   /root/quantus-poc/chain/target/release/quantus-node
//   benchmark
//   overhead
//   --chain=dev
//   --wasm-execution=compiled
//   --weight-path=/root/quantus-poc/poc1-basefee/out
//   --warmup=10
//   --repeat=100

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
	///
	/// Percentiles nanoseconds:
	///   99th: 1_889_523
	///   95th: 1_508_992
	///   75th: 1_345_917
	pub const ExtrinsicBaseWeight: Weight =
		Weight::from_parts(WEIGHT_REF_TIME_PER_NANOS.saturating_mul(1_340_574), 0);
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

**Generated `block_weights.rs`, complete and unmodified:**

```rust
//! THIS FILE WAS AUTO-GENERATED USING THE SUBSTRATE BENCHMARK CLI VERSION 53.0.0
//! DATE: 2026-08-11 (Y/M/D)
//! HOSTNAME: `LAPTOP-PGMUQ2ED`, CPU: `AMD Ryzen 7 5800H with Radeon Graphics`
//!
//! SHORT-NAME: `block`, LONG-NAME: `BlockExecution`, RUNTIME: `Quantus DevNet`
//! WARMUPS: `10`, REPEAT: `100`
//! WEIGHT-PATH: `/root/quantus-poc/poc1-basefee/out`
//! WEIGHT-METRIC: `Average`, WEIGHT-MUL: `1.0`, WEIGHT-ADD: `0`

// Executed Command:
//   /root/quantus-poc/chain/target/release/quantus-node
//   benchmark
//   overhead
//   --chain=dev
//   --wasm-execution=compiled
//   --weight-path=/root/quantus-poc/poc1-basefee/out
//   --warmup=10
//   --repeat=100

use sp_core::parameter_types;
use sp_weights::{constants::WEIGHT_REF_TIME_PER_NANOS, Weight};

parameter_types! {
	/// Weight of executing an empty block.
	/// Calculated by multiplying the *Average* with `1.0` and adding `0`.
	///
	/// Stats nanoseconds:
	///   Min, Max: 1_085_528, 2_235_841
	///   Average:  1_223_182
	///   Median:   1_166_226
	///   Std-Dev:  229734.18
	///
	/// Percentiles nanoseconds:
	///   99th: 2_194_911
	///   95th: 1_666_796
	///   75th: 1_192_597
	pub const BlockExecutionWeight: Weight =
		Weight::from_parts(WEIGHT_REF_TIME_PER_NANOS.saturating_mul(1_223_182), 0);
}

#[cfg(test)]
mod test_weights {
	use sp_weights::constants;

	/// Checks that the weight exists and is sane.
	// NOTE: If this test fails but you are sure that the generated values are fine,
	// you can delete it.
	#[test]
	fn sane() {
		let w = super::BlockExecutionWeight::get();

		// At least 100 µs.
		assert!(
			w.ref_time() >= 100u64 * constants::WEIGHT_REF_TIME_PER_MICROS,
			"Weight should be at least 100 µs."
		);
		// At most 50 ms.
		assert!(
			w.ref_time() <= 50u64 * constants::WEIGHT_REF_TIME_PER_MILLIS,
			"Weight should be at most 50 ms."
		);
	}
}
```

#### 6.7 Explanation of the output

**The headline numbers.**

| Constant               | Declared in the shipped runtime | Measured on this chain |      Ratio |
| ---------------------- | ------------------------------: | ---------------------: | ---------: |
| `ExtrinsicBaseWeight`  |                      108,157 ns |           1,340,574 ns | **12.39×** |
| `BlockExecutionWeight` |                      431,614 ns |           1,223,182 ns |  **2.83×** |

Shortfall per extrinsic: `1,340,574 − 108,157 = 1,232,417 ns`.

**The measurement is stable, not noise.** Across 100 repetitions after 10 warmups, the per-extrinsic figure ranged 1,261,221–1,897,952 ns with a standard deviation of 106,242 ns (about 8 % of the mean) and a median of 1,308,359 ns — close to the mean. Even the **minimum** observed value, 1,261,221 ns, is 11.7× the declared constant. There is no sampling regime in which the declared value is correct.

**The tool independently confirms the block-capacity arithmetic.** Section 3 Step 4 estimates \~541 ML-DSA-87 extrinsics per full block from the length budget. The benchmark derived the same number by actually building a block: `Extrinsics per block: 541`.

**The two headers, side by side, are the whole finding.** The shipped constant reads `RUNTIME: Development`, dated 2024-11-08, produced by `substrate-node` — the stock Substrate chain with sr25519 host-function verification. The generated constant reads `RUNTIME: Quantus DevNet`, produced by `quantus-node` with real Dilithium signatures. Same command, same parameters (`--chain=dev --wasm-execution=compiled --warmup=10 --repeat=100`), two different chains, a 12.39× difference — and the runtime is shipping the wrong one.

**The correct value violates Substrate's own sanity bound.** The generated file carries this assertion:

```rust
// At most 1 ms.
assert!(
    w.ref_time() <= constants::WEIGHT_REF_TIME_PER_MILLIS,
    "Weight should be at most 1 ms."
);
```

The measured 1,340,574 ns is **1.34 ms**, so the correct constant for this chain **fails** the framework's own upper bound on a plausible per-extrinsic base. `BlockExecutionWeight` at 1,223,182 ns stays within its 50 ms ceiling, so only the per-extrinsic constant breaches its bound.

**Hardware dependence, stated honestly.** Absolute nanosecond values depend on the machine (here an AMD Ryzen 7 5800H). What matters is the **ratio**: both the declared and measured constants describe the same workload — one no-op extrinsic's decode-and-verify overhead — so the 12.39× gap reflects the difference between sr25519-with-host-function and Dilithium-in-WASM, not between two CPUs. Re-running on the team's own reference hardware will produce different absolute figures and a similar ratio.
