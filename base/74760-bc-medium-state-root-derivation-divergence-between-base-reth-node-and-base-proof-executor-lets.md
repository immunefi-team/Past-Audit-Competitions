# 74760 bc medium state root derivation divergence between base reth node and base proof executor lets unprivileged blobbasefee transactions stall tee finality for affected l2 ranges

**Submitted on Apr 24th 2026 at 17:31:24 UTC by @remot3 for** [**Audit Comp | Base Azul**](https://immunefi.com/audit-competition/audit-comp-base-azul)

* **Report ID:** #74760
* **Report Type:** Blockchain/DLT
* **Report severity:** Medium
* **Target:** <https://github.com/base/base/tree/v0.8.0-rc.24>
* **Impacts:**
  * A bug in the respective layer 0/1/2 network code that results in unintended smart contract behavior with no concrete funds at direct risk

## Description

Hello team, this is my first report on immunefi and I tried to detail the issue as much as possible, hope I made it easily understandable but let me know if you need any additional information.

I am submitting this conservatively as High severity. I do not claim permanent bridge-fund freezing from outside the system, but Critical severity may apply only if Base confirms that no live ZK, operational, or dispute-game recovery path can finalize or recover affected ranges.

## Brief/Intro

On **base/base @ v0.8.0-rc.15,** two different execution paths construct the EVM `BlockEnv.blob_excess_gas_and_price` field from the same parent header with different semantics.

`base-reth-node` is the main EL path used by the sequencer, follower nodes, and eth\_call RPC hardcodes `BlobExcessGasAndPrice { excess_blob_gas: 0, blob_gasprice: 1 }` for every post-Cancun block.

`base-proof-executor` is the stateless executor used by the TEE Nitro enclave to re-derive state roots during proof generation. It computes the same field from the parent header via the EIP-4844 blob-gas formula.

This creates an execution discrepancy. The **BLOBBASEFEE** opcode reads `blob_gasprice` directly from `BlockEnv`. If an unprivileged transaction invoking `BLOBBASEFEE` lands in a block whose parent has sufficiently high `blob_gas_used` for the proof executor to compute `blob_gasprice > 1`, the sequencer/main-EL path and the TEE proof-executor path persist different contract storage for the same transaction.

In the captured Base Sepolia example, the parent block has:

`blob_gas_used = 9_517_140`

For this, the main EL returns `blob_gasprice = 1` while the proof executor computes `blob_gasprice = 5`

A 4-byte runtime therefore stores 1 under the main EL path and 5 under the proof-executor path. This changes contract storage, which changes the resulting state root.

When the TEE Nitro enclave re-executes the affected block range, it computes a different output root from the one claimed by the sequencer/main-EL path. `Epilogue::validate()` then returns `InvalidClaim` before any valid proposal or signature is produced.

The conservative impact is considered High as finality-liveness failure for affected L2 block ranges: the TEE prover cannot produce a valid proof for any range containing the triggering transaction until mitigation or an alternate fallback path is used.

## Vulnerability Details

### Main EL path

In `crates/execution/evm/src/lib.rs`, the main EL path hardcodes the blob-gas environment for post-Cancun blocks.

The relevant logic is:

```rust
let blob_excess_gas_and_price = spec
    .into_eth_spec()
    .is_enabled_in(SpecId::CANCUN)
    .then_some(BlobExcessGasAndPrice { excess_blob_gas: 0, blob_gasprice: 1 });
```

As a result, the sequencer, follower EL nodes, and eth\_call RPC path all use for post-Cancun blocks:

```
excess_blob_gas = 0
blob_gasprice   = 1
```

### Proof executor path

In `crates/proof/executor/src/builder/env.rs`, the proof executor computes the same field from the parent header:

```rust
let (params, fraction) = if spec_id.is_enabled_in(OpSpecId::ISTHMUS) {
    (Some(BlobParams::prague()), BLOB_BASE_FEE_UPDATE_FRACTION_PRAGUE)
} else if spec_id.is_enabled_in(OpSpecId::ECOTONE) {
    (Some(BlobParams::cancun()), BLOB_BASE_FEE_UPDATE_FRACTION_CANCUN)
} else {
    (None, 0)
};

let blob_excess_gas_and_price = parent_header
    .maybe_next_block_excess_blob_gas(params)
    .or_else(|| spec_id.is_enabled_in(OpSpecId::ECOTONE).then_some(0))
    .map(|excess| BlobExcessGasAndPrice::new(excess, fraction));
```

This means the proof executor can derive:

```
excess_blob_gas > 0
blob_gasprice   > 1
```

Now that we understand both paths there are two thresholds to distinguish:

1. **BlockEnv tuple divergence** — The proof executor’s `excess_blob_gas` can diverge from the main EL once the parent’s `blob_gas_used` exceeds the active blob target.
2. **State divergence through BLOBBASEFEE** — The `BLOBBASEFEE` opcode only observes `blob_gasprice`, not `excess_blob_gas`. Therefore, state divergence through the minimal PoC requires the proof-computed `blob_gasprice` to become greater than 1.

Under the post-Isthmus Prague parameters, the active blob target is:

`6 * 131_072 = 786_432`

The proof executor’s computed `blob_gasprice` reaches 2 at approximately the 5.8M parent-blob-gas range, and reaches 5 for the captured Base Sepolia parent:

`blob_gas_used = 9_517_140`

I implemented the PoC around this condition.

The next step is understanding **how the divergence reaches contract state**

`BLOBBASEFEE` is opcode `0x4a`. It pushes the current `blob_gasprice` from the EVM block environment onto the stack.

An attacker-controlled contract can persist that value with the following 4-byte runtime:

```
0x4a5f5500
```

| Byte | Opcode        | Effect                            |
| ---- | ------------- | --------------------------------- |
| `4a` | `BLOBBASEFEE` | Push `block.blob_gasprice()`      |
| `5f` | `PUSH0`       | Push storage slot `0`             |
| `55` | `SSTORE`      | Store `blob_gasprice` in slot `0` |
| `00` | `STOP`        | Halt                              |

For the same transaction and same initial state:

* Main EL stores `1`
* Proof executor stores `5`

This produces different post-transaction storage and therefore different state roots. The TEE Nitro enclave proof pipeline re-executes the L2 block range through the proof-executor path:

```
FaultProofDriver::execute_with_intermediates
→ BaseExecutor::execute_payload
→ StatelessL2Builder::build_block
→ prepare_block_env
```

The enclave computes an output root from the re-executed state. Then `Epilogue::validate()` runs before any proposal is constructed or signed and because the proof executor computes state root `S_2` while the sequencer/main-EL path claimed state root `S_1`, validation fails. `Epilogue::validate()` returns `FaultProofProgramError::InvalidClaim` and the proof pipeline surfaces an error. No valid TEE proposal or signature is produced for the affected range.

## Additional Notes

I observed that the live AggregateVerifier implementation wired as gameType 621 exposes `ZK_VERIFIER = address(0)` and has no deployed bytecode. I could not identify a live ZK fallback for this game type at the time of capture. If Base confirms that no alternate fallback path is available for affected ranges, the finalization stall persists until operational mitigation is applied, such as blacklisting the affected game, retiring the game type, or deploying a hotfix to the attested TEE binary with re-attestation and re-registration.

## Impact Details

`A bug in the respective layer 0/1/2 network code that results in unintended smart contract behavior with no concrete funds at direct risk.`

I selected this impact because the PoC demonstrates that the same EVM bytecode produces different persistent contract storage depending on whether execution occurs through the main EL path or the proof-executor path. The storage difference changes the resulting state root, causing the TEE proof executor to compute a different output root from the one claimed by the sequencer/main-EL path.

Affected L2 block ranges containing the triggering transaction cannot receive a valid TEE proof until mitigation or an alternate fallback path is used. This denies finality liveness for those ranges.

I do not claim permanent bridge-fund freezing from outside the system but **Critical severity may apply only if Base confirms that no live ZK, operational, or dispute-game recovery path can finalize or recover affected ranges**.

## References

**Code:** `base/base @ v0.8.0-rc.15`

Commit: `de349fc9e8bf61531ce36ca57572345b03b2b097`

**Relevant files:**

```
crates/execution/evm/src/lib.rs:81-84
crates/execution/evm/src/lib.rs:112-115
crates/execution/evm/src/lib.rs:292-295
```

Main EL hardcoded blob-gas environment.

`crates/proof/executor/src/builder/env.rs:104-115`

Proof-executor computed blob-gas environment.

`crates/proof/tee/nitro-enclave/src/server.rs:141-170`

TEE enclave proof pipeline.

`crates/proof/client/src/epilogue.rs:26`

Base Sepolia parent block capturd via eth\_getBlockByNumber on 2026-04-24.

Reference parent:

```
L2 block:        40_200_307
timestamp:       1_776_168_902
blob_gas_used:   9_517_140
excess_blob_gas: 0
base_fee_per_gas: 5_000_000
```

This parent is used because its `blob_gas_used` is high enough for the proof executor to compute `blob_gasprice = 5`, while the main EL path still exposes `blob_gasprice = 1`.

## Link to Proof of Concept

<https://gist.github.com/Nucklear/d49c79fdcdad9162049a6b885ae385f5>

## Proof of Concept

The attached poc ( <https://drive.google.com/file/d/1XwYc4Ue9ky\\_64cKS6H53pNT6bUSaennz/view?usp=drive\\_link> ) directory contains a self-contained Rust crate that reproduces the divergence and the end-to-end storage mismatch locally.

**Run instructions**

From the archive root:

```bash
cd poc
./run.sh
```

### PoC 1: Differential tests

The hermetic test suite compares the main EL semantics against the proof-executor semantics

```
poc/tests/blockenv_diff.rs - verifies that the two paths diverge across boundary and realistic parent-header values.
poc/tests/blobbasefee_state_divergence.rs - deploys the 4-byte runtime into a local CacheDB, executes one call under each BlockEnv, and asserts that storage slot 0 differs.
```

Expected output for the main test:

```
Main-EL  blob env = Some(BlobExcessGasAndPrice { excess_blob_gas: 0,       blob_gasprice: 1 })
PROOF    blob env = Some(BlobExcessGasAndPrice { excess_blob_gas: 8730708, blob_gasprice: 5 })
EL    storage[slot 0] = 1
PROOF storage[slot 0] = 5
```

The same transaction produces different persistent storage under the two execution paths.

### PoC 2: Local Anvil fork replay

Test:

```
poc/tests/anvil_fork_replay.rs
```

Expected output:

```
raw response:
{"jsonrpc":"2.0","id":1,"result":"0x0000000000000000000000000000000000000000000000000000000000000001"}

test read_blobbasefee_from_local_anvil_fork ... ok
```

This confirms that the main EL / forked RPC path exposes `BLOBBASEFEE = 1` for the block, while the proof-executor path computes `5` from the same parent data.

I am also attaching a Gist with a simplified Foundry version of the PoC (`Exploit.t.sol`). This is not the primary proof; it is included only as a friendly model of the same attack path. The Solidity version re-derives the proof-executor `blob_gasprice = 5` from the captured parent block values using the EIP-4844 `fake_exponential` formula, then models the two execution paths by persisting `1` for the main EL path and `5` for the proof-executor path. This shows, in a single-file Foundry test, why the same `BLOBBASEFEE to SSTORE` logic leads to different storage values and therefore different state roots. The primary PoC remains the Rust/revm crate, which executes the 4-byte runtime under the two `BlockEnv` configurations directly.
