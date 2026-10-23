# 76082 bc low checkpoint witness accumulation can trigger oom in nitro prover

## #76082 \[BC-Low] Checkpoint witness accumulation can trigger OOM in Nitro prover

Submitted on May 2nd 2026 at 15:48:28 UTC by @kirillt for [Audit Comp | Base Azul](https://immunefi.com/audit-competition/audit-comp-base-azul)

* **Report ID:** #76082
* **Report Type:** Blockchain/DLT
* **Report severity:** Low
* **Target:** <https://github.com/base/base/tree/v0.8.0-rc.28>
* **Impacts:**
  * Shutdown of greater than 10% or equal to but less than 30% of network processing nodes without brute force actions, but does not shut down the network

### Description

## Checkpoint witness accumulation can trigger OOM in Nitro prover

A sustained sequence of checkpoint-fresh state accesses, optionally reinforced with blob-heavy DA activity, can make the host assemble medium-heavy but honest `Prove` payloads that are large enough to exhaust Nitro enclave memory once several jobs overlap in the same process.

### Key locations in the codebase

1. `base/crates/proof/tee/nitro-enclave/src/transport.rs:76-80` (no validation or memory check before allocation):

```rust
let len = reader.read_u32().await? as usize;

debug!(payload_bytes = len, "frame read start");

let mut payload = vec![0u8; len];
```

2. `base/crates/proof/tee/nitro-enclave/src/transport.rs:83` (memory consumption is roughly double of the preimage size):

```rust
let (value, _) = bincode::serde::decode_from_slice(&payload, bincode::config::standard())
    .map_err(|e| TransportError::Codec(e.to_string()))?;
```

3. `base/crates/proof/tee/nitro-enclave/src/runtime.rs` (multiple requests are handled concurrently):

```rust
loop {
    let (stream, peer) = listener.accept().await?;
    debug!(cid = peer.cid(), port = peer.port(), "accepted connection");

    let enclave = self.clone();
    tokio::spawn(async move {
        if let Err(e) = enclave.handle_connection(stream).await {
            warn!(
                error = %e,
                cid = peer.cid(),
                port = peer.port(),
                "connection failed"
            );
        }
    });
}
```

4. `base/crates/proof/tee/nitro-host/src/backend.rs:38` (preimages are used to construct the witness):

```rust
async fn prove(&self, witness: Oracle) -> Result<ProofResult, NitroHostError> {
    let preimages = witness.into_preimages().map_err(NitroHostError::Enclave)?;
    self.transport.prove(preimages).await
}
```

5. `etc/docker/nitro-enclave/allocator.yaml` (out of scope, an example of realistic configuration - the enclave runs with a finite shared memory pool, and multiple concurrent proof jobs consume that same pool):

```yaml
memory_mib: 8192
cpu_count: 2
```

### Impact

An OOM or process abort can drop multiple in-flight proving jobs at once. The aggregate wasted compute scales with both the number of concurrent jobs and duration of each job. The caller side will also observe failures or timeouts, so the harm is not limited to local wasted work.

### Likelihood

Large checkpoints are dangerous because raw witness bytes understate peak enclave memory: the `Prove` payload exists once in the frame buffer and again after decode before proving starts.

Concurrent jobs worsen this because Nitro handles multiple vsock connections in the same enclave process, so these duplicated buffers and proving working sets stack within one shared memory pool (reference configuration is 8 GB).

Cost of attack and specific attack scenario are elaborated after the "Recommendations" section. The issue requires irregular spike of preimages usages during several checkpoints, or low-funded attack during several checkpoints.

### Recommendations

#### Recommendation 1

Use staged synchronization with explicit admission control for memory-intensive phases. In particular, allow only a bounded number of jobs to enter witness-heavy, serialization, decode, and proving sections concurrently, instead of treating all in-flight jobs the same. This is preferable to a flat concurrency limit because jobs blocked on slow external RPC or other I/O should not permanently occupy the scarce permits needed for the highest-memory sections.

#### Recommendation 2

As a lower-effort defense, track a shared memory budget and block new allocations or phase transitions when there is not enough headroom. If this approach is used, the memory-availability check must itself be synchronized or reserve capacity atomically; otherwise two jobs can both observe "enough free memory" and race into the same allocation, defeating the check.

#### Recommendation 3

Add explicit limit-based validation at admission time:

* a hard maximum frame size
* a hard maximum proof span
* a hard cap on total witness bytes or preimage count

These checks should fail early, before expensive witness generation, serialization, or enclave transfer begins.

### Attack scenario

The attack aims to cause not a single giant witness spike, but a cheap sustained production of medium-heavy checkpoints that causes OOM condition in the TEE enclave. This requires avoiding explosive fee growth. The OOM condition is produced not by one huge witness, but by several concurrent prover jobs converting the malicious checkpoints into large witnesses.

Tools of the attack are (1) state-proof accesses, and (2) DA-heavy blob activity added on top whenever it increases checkpoint witness size cheaply.

The core of the attack is a sequence of compact, ordinary-looking transactions that spend most of their gas on new state accesses that trigger distinct account and storage proofs. The attacker minimizes calldata and fixed transaction overhead and instead iterates over many fresh targets (accounts, storage slots, contracts, state trie paths) internally. It may require cheap preparation phase to bootstrap the targets, but every target can be re-used in every new checkpoint (not within same checkpoint because witness entries deduplicate). This keeps the current block byte size moderate while forcing the witness to accumulate historic trie material.

On top of that state-proof pressure, the attacker adds DA-heavy blob activity in the same general execution-fee regime. After Jovian, fees are driven by `max(gasUsed, blobGasUsed)` and not by their sum. As a result, once the attacker is already paying for a state-proof-heavy block near the target execution load, adding blob-heavy DA pressure can increase witness size without paying proportional extra cost. This makes the combined strategy more reliable per dollar than a pure state-proof-only strategy.

The Nitro runtime accepts multiple vsock connections and handles them concurrently in the same enclave process.

So the attacker does not need one checkpoint to contain the full fatal witness size by itself. Several medium-heavy checkpoints, especially overlapping ones, are enough to create the same operational effect.

#### Cost estimate

**Roughly $25-$49 per GB of preimages, when the fee-neutral strategy is used. 1 GB of preimages consumes about 2 GB of memory in the TEE enclave.**

If the attacker tries to force one intense checkpoint, it targets about 3 GB of raw witness in a single 512-block checkpoint and costs roughly $350-$500, because concentrating that much witness into one checkpoint pushes metered usage above the target and causes fee drift after several blocks.

The fee-neutral low-profile approach accepts less witness per checkpoint, stays near the normal fee regime, and reaches the same practical pressure only through repetition.

From `base/crates/common/chains/src/config.rs:180`:

```rust
eip1559_elasticity: 6,
```

Assume:

* `2,700..3,000` gas per fresh proof-inducing state touch
* average encoded node size `250..400` bytes
* `gas_price_gwei` is `0.005`
* `eth_price_usd` is `2354`

**Conservative gas limit**

`block_gas_limit`: = 30,000,000\
`target_gas`: 30,000,000 / 6 = 5,000,000\
`cost_usd`: 512 x 5,000,000 x 0.005e-9 x 2354 \~ $30

`~1,700..1,850` fresh touches\
`~5,500` unique nodes\
`preimages_size_block`: 1.5..2.3 MB\
`preimages_size_checkpoint`: 512 x `preimages_size_block` = 0.7..1.2 GB

**Maximum configured gas limit**

This is unlikely to be reached, but worth calculating to understand the range.

`block_gas_limit`: 105,000,000\
`target_gas`: 105,000,000 / 6 = 17,500,000\
`cost_usd`: 512 x 17,500,000 x 0.005e-9 x 2354 \~ $105

`~5,800..6,500` fresh touches\
`~24,000..28,000` unique nodes\
`preimages_size_block`: 4.2..7.7 MB\
`preimages_size_checkpoint`: 512 x `preimages_size_block` = 2.2..3.9 GB

### Proof of Concept

See "Attack scenario" and "Cost estimate" sections in the description for high-level overview. Runnable unit-test like PoC is prepared and can be submitted upon request (uses synthetic preimages to submit them into TEE and observe the crash).

#### Current conditions

* The proving span is 512 blocks.
* The Nitro enclave processes multiple proof jobs in the same process.
* Each `Prove` request is received as one length-prefixed payload, fully buffered, then decoded into owned values.

#### Assumptions

* The attacker can deploy ordinary helper contracts on L2.
* Base’s proving workflow automatically generates proof jobs for checkpoints in the normal course of operation.

#### Step-by-step attack

{% stepper %}
{% step %}

### Deploy many ordinary `SlotBank` contracts on L2

Each `SlotBank` stores data in a large mapping, for example `mapping(uint256 => bytes32) slots`, and exposes:

* `prime(start, count)` to write nonzero values into many keys
* `readRange(start, count)` to loop over and read many keys
  {% endstep %}

{% step %}

### Deploy one `CheckpointDriver` contract

It should expose a function such as `run(address[] banks, uint256[] starts, uint256 readsPerBank)` that loops over many `SlotBank` contracts and `staticcall`s their `readRange(...)` functions.
{% endstep %}

{% step %}

### Deploy one trivial `DataSink` contract for DA-heavy transactions

It can expose a function such as `consume(bytes calldata data) external {}` and do little or no work.
{% endstep %}

{% step %}

### Before the target checkpoint begins, bootstrap the state

Call `prime(...)` across many `SlotBank` contracts.

This creates a large pool of readable storage keys that later blocks can consume.
{% endstep %}

{% step %}

### During each target block, submit one or a few compact `CheckpointDriver.run(...)` transactions

These transactions should:

* touch many `SlotBank` contracts not yet touched in that checkpoint
* read slot ranges not yet touched in that checkpoint
* keep calldata small by passing only contract lists, range starts, and a common batch size
  {% endstep %}

{% step %}

### In the same blocks, submit separate DA-heavy transactions to `DataSink.consume(...)`

Their calldata should be large and poorly compressible so that the sequencer’s L1 publication for those blocks consumes more blob-backed DA.
{% endstep %}

{% step %}

### Repeat steps 5 and 6 across many consecutive blocks

Do this so that the full 512-block checkpoint is both:

* state-proof-heavy from fresh account/storage accesses; and
* DA-heavy from large batch data footprint.
  {% endstep %}

{% step %}

### Keep each block near the fee-neutral target instead of maximizing a single block

The goal is not one extreme block, but a sustained sequence of medium-heavy blocks that remains comparatively cheap.
{% endstep %}

{% step %}

### Continue the same pattern across later checkpoints as well

Because proof jobs are generated automatically, Base’s normal proving workflow will enqueue proof jobs for these witness-heavy checkpoints without any attacker interaction with the prover RPC.
{% endstep %}

{% step %}

### Once several such proof jobs overlap in time, the enclave OOMs or aborts

Each job brings a large honest `Prove` payload into the same enclave process.

The enclave buffers each full payload, decodes owned values from it, and then proves it, so the memory usage of overlapping jobs stacks until the enclave OOMs or aborts.
{% endstep %}
{% endstepper %}

#### Expected result

* Each attacker-shaped checkpoint produces a large but honest witness.
* Several overlapping proof jobs stack memory in the same Nitro enclave process.
* The prover eventually OOMs or aborts, dropping in-flight jobs.
