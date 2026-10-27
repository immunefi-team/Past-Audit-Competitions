# 74673 bc medium proof executor uses non canonical blobbasefee after jovian base v1

> Submitted on Apr 24th 2026 at 06:52:32 UTC by @VulSight for [Audit Comp | Base Azul](https://immunefi.com/audit-competition/audit-comp-base-azul)

* **Report ID:** #74673
* **Report Type:** Blockchain/DLT
* **Report severity:** Medium
* **Target:** <https://github.com/base/base/releases/tag/v0.8.0-rc.15>
* **Impacts:**
  * Temporary freezing of network transactions by delaying one block by 500% or more of the average block time of the preceding 24 hours beyond standard difficulty adjustments

## Description

### Brief/Intro

After Jovian/Base V1, Base no longer uses the Ethereum blob gas fields in the normal way. `blob_gas_used` is reused to carry the DA footprint, and canonical Base execution keeps `BLOBBASEFEE` at `1`. The proof-side executor still feeds the parent header into Ethereum blob excess math, so it can derive a higher blob base fee for the child block. This makes proof execution disagree with canonical execution. In production, a valid canonical block can become unprovable by Base's own proof stack, which can block proof/finality for that target block.

### Vulnerability Details

The bug is in the proof executor environment builder.

In `crates/proof/executor/src/builder/env.rs`, the child block env is built from the parent header with `maybe_next_block_excess_blob_gas(...)`:

```rust
let blob_excess_gas_and_price = parent_header
    .maybe_next_block_excess_blob_gas(params)
    .or_else(|| spec_id.is_enabled_in(OpSpecId::ECOTONE).then_some(0))
    .map(|excess| BlobExcessGasAndPrice::new(excess, fraction));
```

That value is then used when the proof executor builds the EVM for the block in `crates/proof/executor/src/builder/core.rs`.

The problem is that on Base after Jovian, the parent header field is not Ethereum blob gas anymore. In `crates/proof/executor/src/builder/assemble.rs`, the proof-side block assembly writes:

```rust
let (blob_gas_used, excess_blob_gas) = if self.config.is_jovian_active(timestamp) {
    (Some(ex_result.blob_gas_used), Some(0))
}
```

And the consensus rules in `crates/execution/consensus/src/lib.rs` make it clear that after Jovian:

* `blob_gas_used` contains the current DA footprint
* `excess_blob_gas` must stay `0`

Canonical Base execution does not reuse Ethereum blob excess math here. In `crates/execution/evm/src/lib.rs`, the canonical env is hard-coded to:

```rust
BlobExcessGasAndPrice { excess_blob_gas: 0, blob_gasprice: 1 }
```

So there are now two different behaviors:

1. Canonical Base execution: `BLOBBASEFEE == 1`
2. Proof executor: `BLOBBASEFEE` is derived from parent `blob_gas_used`, even though that field is now DA footprint data

If a child block executes the `BLOBBASEFEE` opcode after a parent with large enough DA footprint, the proof path and canonical path will execute different state transitions.

The repo PoC shows this end to end. The steps are:

1. Start a local chain with Jovian/Base V1 active
2. Deploy a contract that stores `BLOBBASEFEE` into storage
3. Send a large calldata transaction so the parent block gets a high DA footprint
4. Call the contract in the child block

The PoC output shows:

* parent `blobGasUsed=5670400`, which is above the threshold `4257519`
* canonical builder stored `BLOBBASEFEE=1`
* canonical safe chain also stored `BLOBBASEFEE=1`
* canonical child output root was `0x913766291aad91849c9827c7998adeb36522ed3acd6ed337b1cef5f4c9192b08`
* proof host computed `0x4e1364cf7f433138a7051a6e6b57912103a4f274429be67bd3755c9be3dc6568`

The proof host then rejected the canonical block with:

```
invalid claim: computed 0x4e1364cf7f433138a7051a6e6b57912103a4f274429be67bd3755c9be3dc6568, claimed 0x913766291aad91849c9827c7998adeb36522ed3acd6ed337b1cef5f4c9192b08
```

The second PoC test shows the same issue in the proposer/finality path. The proposer reached the poisoned target block, attempted proving, got `invalid claim`, and did not submit/finalize a game for that target:

```
RESULT   | proposer attempted poisoned_target=7 and received invalid_claim_count=2
RESULT   | no finalized/submitted game reached poisoned_target; max_game_l2_block=None, final_game_count=0
```

This is not just a local simulation mismatch. The proof side really computes a different output root from the one produced by canonical Base execution.

### Impact Details

High: temporary freezing of network transactions by delaying one block by 500% or more of normal block time.

More specifically, this bug can make a valid canonical L2 block unprovable by Base's own proof stack. The sequencer can still build blocks, but once the proof system reaches a poisoned target block, the proof host computes a different output root and rejects the canonical claim as `invalid claim`. The proposer then cannot move that target block through the normal proving/finality flow.

So the practical impact is a proof/finality outage for affected blocks. Users may still see transactions land on the canonical L2 chain, but those blocks cannot be confirmed/finalized through the proof pipeline. That is a real liveness issue for finality and it can delay confirmation far beyond normal block time.

### References

* `crates/proof/executor/src/builder/env.rs`
* `crates/proof/executor/src/builder/core.rs`
* `crates/proof/executor/src/builder/assemble.rs`
* `crates/execution/consensus/src/lib.rs`
* `crates/execution/evm/src/lib.rs`
* `crates/proof/client/src/epilogue.rs`
* `crates/proof/host/src/host.rs`
* `crates/proof/tee/nitro-enclave/src/server.rs`
* `crates/proof/proposer/src/pipeline.rs`
* `devnet/tests/proof_finality_blobbasefee.rs`
* `run_blobbasefee_poc.sh`

### Link to Proof of Concept

<https://gist.github.com/ZeroCipher002/7513e15b41db0eb551242c385b508e6b>

## Proof of Concept

This PoC is a local end-to-end repro. It does not patch the vulnerable logic itself. The vulnerable proof executor, canonical execution code, proposer logic, and output-root validation code are left unchanged. The patch only adds the test harness and local devnet support needed to exercise the real proof path against a canonical chain that is already in the post-Jovian/Base V1 state.

The cleanest way to run it from scratch is to start with three things placed next to each other:

```
~/Base/base-0.8.0-rc.15
~/Base/contracts
~/Base/blobbasefee-finality-poc.patch
```

The .patch file is provided as a gist due to the POC section character limit here.

The `contracts` repo is needed because the second test deploys the local multiproof stack through Foundry. The PoC runner expects that repo at `../contracts` by default, but it also accepts `BASE_CONTRACTS_ROOT=/absolute/path/to/contracts`.

Use the latest refreshed patch artifact. The current patch includes the quiet audit-style runner output and fixes an earlier export issue that could make `git apply` fail on a fresh checkout.

To apply the patch from a clean Base checkout:

```bash
cd ~/Base/base-0.8.0-rc.15
git apply --check ../blobbasefee-finality-poc.patch
git apply ../blobbasefee-finality-poc.patch
```

If the tree is not a git checkout and is just an unpacked source archive, use:

```bash
cd ~/Base/base-0.8.0-rc.15
patch -p1 < ../blobbasefee-finality-poc.patch
```

Before running the PoC, make sure the machine has the required tools:

```bash
docker ps
cargo --version
forge --version
make --version
ld.lld --version
```

After the patch is applied, run the PoC from the Base repo root:

```bash
cd ~/Base/base-0.8.0-rc.15
./run_blobbasefee_poc.sh
```

If the contracts repo is not at `../contracts`, run:

```bash
cd ~/Base/base-0.8.0-rc.15
BASE_CONTRACTS_ROOT=/absolute/path/to/contracts ./run_blobbasefee_poc.sh
```

The helper script is only a wrapper. The real PoC is the Rust test file `devnet/tests/proof_finality_blobbasefee.rs`. The script just checks the environment, installs missing Solidity deps if needed, builds the unrelated fixture contract if needed, and then runs:

```bash
cargo test -p devnet \
  --config 'target.x86_64-unknown-linux-gnu.rustflags=["-C","link-arg=-fuse-ld=lld"]' \
  --test proof_finality_blobbasefee -- --test-threads=1 --nocapture
```

The reproduction flow is:

1. Start a local chain with Jovian/Base V1 active at genesis.
2. Force the local batcher to post calldata instead of EIP-4844 blobs.
3. Deploy a small contract whose runtime stores `BLOBBASEFEE` into storage slot `0`.
4. Send a large ordinary calldata transaction so the parent block gets a high DA footprint in `blob_gas_used`.
5. Send a child transaction that executes `BLOBBASEFEE`.
6. Confirm that canonical execution stores `1`.
7. Ask the real proof host to prove the canonical child output root and confirm it rejects it as `invalid claim`.
8. Start the real proposer pipeline and confirm it reaches the poisoned target, gets `invalid claim`, and does not submit/finalize a game for that block.

On a successful run, the output should show the following facts:

* the parent block has `blobGasUsed` above the threshold
* canonical execution stored `BLOBBASEFEE=1`
* the proof host computed a different output root from the canonical one
* the proof host rejected the canonical output as `invalid claim`
* the proposer attempted the poisoned block but no game was submitted/finalized for it
* the test suite finished with `2 passed; 0 failed`

The patch changes these files and each one has a specific purpose:

* `BLOBBASEFEE_POC_README.md`: This is only documentation for the overlay. It explains what the PoC is, how to run it, and what the expected result looks like. It is not part of the exploit path, but it is useful so the overlay can be handed to another person without extra context.
* `Cargo.lock`: This updates the dependency lockfile after adding new devnet dependencies for the E2E proof harness. It is needed so the overlay builds reproducibly instead of depending on whatever Cargo might resolve later.
* `devnet/Cargo.toml`: This adds the crates needed by the new E2E test. The test needs proof-side crates, proposer crates, tx manager support, the proofs-history extension, JSON-RPC server support, and a few extra alloy/reth features. Without this file change, the new PoC test would not compile.
* `devnet/src/l2/in_process_batcher.rs`: This adds a new DA mode selector, `InProcessBatcherDaType`, and passes that setting into the batcher encoder. This is necessary because the PoC needs the local batcher to use calldata instead of blobs. The bug is triggered by the post-Jovian/Base V1 DA footprint being written into `blob_gas_used`, so the test must be able to generate that footprint with ordinary calldata.
* `devnet/src/l2/in_process_client.rs`: This enables proof-history storage and debug/proof RPC support on the local client node. It adds a proof window, turns on the `ProofsHistoryExtension`, initializes the proofs-history database from genesis, and exposes the RPC modules needed by the proof host. This is necessary because the proof host has to fetch historical headers and proofs from the canonical client during the repro.
* `devnet/src/l2/mod.rs`: This is only a plumbing change that re-exports the new `InProcessBatcherDaType`. It is needed so the higher-level devnet builder can select calldata mode.
* `devnet/src/l2/stack.rs`: This threads the batcher DA mode through the full L2 stack, delays batcher startup until after the builder has sealed a real post-genesis block, and adds a small wait helper. This is necessary so the test can reliably start the chain, choose calldata DA, and avoid the batcher starting too early on an empty genesis state.
* `devnet/src/setup/container.rs`: This passes a new environment variable into the L2 setup container so the test harness can activate the latest L2 hardforks at genesis. This is necessary because the bug is specifically post-Jovian/Base V1 and the PoC needs to start directly inside that fork state instead of waiting for a later timestamp.
* `devnet/src/smoke.rs`: This adds a new `batcher_da_type` field to `DevnetBuilder` and a helper method `with_calldata_l2_batcher()`. This is necessary because the PoC test needs a simple way to tell the local devnet to use calldata DA.
* `devnet/tests/proof_finality_blobbasefee.rs`: This is the actual PoC. It contains the full attack flow and the two E2E checks. The first test proves the core bug by showing that the proof host rejects a valid canonical child output root. The second test proves real impact by showing the proposer/finality pipeline stalls on the poisoned target. It also includes the minimal attacker contract, helper functions, a local Nitro prover server, and a temporary Foundry deployment script for the local multiproof contracts.
* `etc/scripts/devnet/setup-l2.sh`: This teaches the devnet L2 genesis generator how to activate all relevant L2 hardforks at genesis through `L2_ACTIVATE_LATEST_HARDFORKS_AT_GENESIS=1`. It also makes the generated artifacts readable by the host test process. This is necessary because the PoC needs Jovian and Base V1 active from block `0`, and the host-side test process must be able to read the generated files.
* `run_blobbasefee_poc.sh`: This is the one-command runner. It checks that `cargo` and `forge` exist, finds the contracts repo, installs missing contracts dependencies with `make deps` if needed, builds the unrelated fixture contract if needed, and then runs the devnet test. This is necessary so another person can reproduce the PoC from a clean machine without manually figuring out the setup order.

It is also important to say what the patch does not do. It does not modify:

* `crates/proof/executor/src/builder/env.rs`
* `crates/proof/executor/src/builder/core.rs`
* `crates/execution/evm/src/lib.rs`
* `crates/execution/consensus/src/lib.rs`
* `crates/proof/client/src/epilogue.rs`
* `crates/proof/proposer/src/pipeline.rs`

That matters because it means the PoC is not creating the divergence by patching the vulnerable code. It only creates the local environment needed to hit the existing bug with real proof/proposer components.
