# 74823 bc medium min base fee consolidation bypass below floor tx inclusion proof divergence

**Submitted on Apr 25th 2026 at 04:14:49 UTC by @nord0x for** [**Audit Comp | Base Azul**](https://immunefi.com/audit-competition/audit-comp-base-azul)

* **Report ID:** #74823
* **Report Type:** Blockchain/DLT
* **Report severity:** Medium
* **Target:** <https://github.com/base/base/tree/v0.8.0-rc.24>
* **Impacts:**
  * Causing network processing nodes to process transactions from the mempool beyond set parameters

## Description

### Bug Description

The Jovian consolidation check in `attributes.rs:214` silently discards `min_base_fee` when comparing a sequencer-produced block against L1-derived attributes. The `_` wildcard drops the field, so it's never compared. Two consequences:

1. The base fee floor set by L1 SystemConfig governance is removed from subsequent block execution. The public RPC txpool then accepts and the block builder includes user transactions whose `max_fee_per_gas` is below the intended floor. This maps to the program's High impact: *"Causing network processing nodes to process transactions from the mempool beyond set parameters."*
2. The proof executor re-seals blocks using the correct `min_base_fee` from L1 SystemConfig, producing different `extra_data` → different block hash → different `OutputRoot`. The canonical checkpoint becomes unprovable.

The poc demonstrates both impacts end-to-end against a real base reth node

### Root Cause 1 — Consolidation discards `min_base_fee`

```rust
// crates/consensus/engine/src/attributes.rs:213-214
let extra_data_decoded = if config.is_jovian_active(block.header.timestamp) {
    JovianExtraData::decode(&block.header.extra_data).map(|(be, bd, _)| (be, bd))
    //                                                               ^^^ discarded
```

Lines 244-246 compare only `(elasticity, denominator)` against the L1-derived values. `min_base_fee` is decoded but thrown away. A block with any arbitrary `min_base_fee` in its `extra_data` passes consolidation.

### Root Cause 2 — Proof executor uses L1 `min_base_fee`

```rust
// crates/proof/executor/src/util.rs:87-97
pub(crate) fn encode_jovian_eip_1559_params(
    config: &RollupConfig,
    attributes: &BasePayloadAttributes,
) -> ExecutorResult<Bytes> {
    Ok(JovianExtraData::encode(
        attributes.eip_1559_params.ok_or(ExecutorError::MissingEIP1559Params)?,
        config.chain_op_config.post_canyon_params(),
        attributes.min_base_fee.ok_or(ExecutorError::InvalidExtraData(  // ← from L1
            Eip1559ValidationError::Decode(EIP1559ParamError::MinBaseFeeNotSet),
        ))?,
    )?)
}
```

The proof executor's `seal_block` (`assemble.rs:79`) builds `extra_data` from L1-derived `BasePayloadAttributes` — it never copies the canonical block's header. So when the canonical chain accepted `min_base_fee = 0` via the consolidation gap, the proof executor re-seals with `min_base_fee = 8 gwei` → different encoding → different hash → different `OutputRoot`.

### Root Cause 3 — Gossip doesn't check `extra_data`

I checked `crates/consensus/gossip/src/block_validity.rs` and it validates timestamp, block hash, signature (must come from `unsafe_block_signer`), and structural fields. It does not look at `extra_data` contents at all — no check for `min_base_fee`, no comparison against L1 SystemConfig. A poisoned block signed by the sequencer propagates to every follower node via p2p gossip, and each node accepts it through the same consolidation path.

So this isn't isolated to one node — every node in the network ends up processing transactions below the governance floor.

## Impact

### Txpool inclusion below governance floor (proven)

After a poisoned block lands:

* `compute_jovian_base_fee` (`basefee.rs:51`) reads `min_base_fee = 0` from the poisoned parent
* The floor disappears — `base_fee_per_gas` drops below 8 gwei on the next block
* Users can submit txs with fees below the L1-configured floor and they get mined

The PoC shows this: after 8 poisoned blocks the base fee drops to \~2.75 gwei, and a tx with `max_fee_per_gas = 3 gwei` gets accepted and mined — 62.5% below the 8 gwei governance floor.

### Proof-system divergence (proven)

Because `extra_data` in the canonical header is not what the proof executor produces, the block hashes differ, and therefore the `OutputRoot` (which includes the block hash) differs. The poc computes both using `base_protocol::OutputRoot::from_parts` and shows they diverge.

What this hits in the proof pipeline:

* `proof/client/src/epilogue.rs:25-30` — rejects the canonical claim with `InvalidClaim { computed, claimed }`
* `proof/tee/nitro-enclave/src/server.rs:165-166` — TEE validates the epilogue before signing so it can't attest to the canonical root
* `proof/proposer/src/pipeline.rs:1063-1078` — re-fetches canonical root and refuses submission on `RootMismatch`
* `proof/challenge/src/validator.rs:244-255` — marks games with the proof-exec root invalid against canonical L2 state

No valid `AggregateVerifier` game can be created for the affected checkpoint range until the code is fixed or the chain is administratively repaired. The bug is fully deterministic — no race, no timing, no probability.

### Who triggers this

The consolidation check exists to validate sequencer-produced blocks against L1-derived parameters. That's the fault-proof threat model — catch a misbehaving sequencer. The trigger is a sequencer/engine path that stamps `min_base_fee` differently from what the L1 SystemConfig says. This could be:

* A compromised sequencer key
* A software bug in the sequencer that produces wrong `min_base_fee`
* A stale SystemConfig cache on the sequencer side

In all three cases, consolidation should catch it. It doesn't, because of the `_` wildcard.

### Limitations

* The PoC demonstrates txpool/block-inclusion and the proof failure predicates locally. It doesn't deploy L1 `AggregateVerifier`/portal contracts or prove an end-to-end withdrawal failure.
* The bug only fires when L1 SystemConfig `min_base_fee` and the stamped header value differ. If both are zero or equal, the path is latent.
* Manual intervention can repair future operation, but can't make an already-poisoned canonical checkpoint provable without code or config changes.

## Link to Proof of Concept

<https://gist.github.com/drawrowfly/833abd506e618724b36cee7b074c72d3>

## Proof of Concept

I did my best to create runnable poc that shows the problem: <https://gist.github.com/drawrowfly/833abd506e618724b36cee7b074c72d3> , there is a shell script that does all the job for convenience

### Setup

```bash
git clone --branch v0.8.0-rc.15 https://github.com/base/base.git
cd base

# Rust 1.82+
rustup update stable
```

### Run

```bash
cargo build -p min-basefee-poc --release
cargo run -p min-basefee-poc --release
```

Or use the provided `run_poc.sh` script which handles setup automatically.

### What the poc does

{% stepper %}
{% step %}
Launches a real Base-reth node with Jovian active at genesis (IPC engine API, full reth, MDBX — no mocks)
{% endstep %}

{% step %}
Mines 3 honest blocks with `min_base_fee = 8 gwei`
{% endstep %}

{% step %}
Injects poisoned block #4: builds a valid payload via engine API, rewrites `extra_data` to set `min_base_fee = 0`, recomputes hash, submits via `engine_newPayloadV4` + `engine_forkchoiceUpdatedV3`
{% endstep %}

{% step %}
Verifies the node accepts it (consolidation bypass)
{% endstep %}

{% step %}
Mines block #5 with poisoned parent to push base fee below the governance floor
{% endstep %}

{% step %}
Submits a signed EIP-1559 tx through public RPC with `max_fee_per_gas = 3 gwei` (62.5% below the 8 gwei floor)
{% endstep %}

{% step %}
Mines the next block from txpool and verifies the below-floor tx receipt
{% endstep %}

{% step %}
Computes proof-executor's re-sealed header using L1 SystemConfig `min_base_fee` — same path as `assemble.rs:79` → `encode_jovian_eip_1559_params()`
{% endstep %}

{% step %}
Computes real Base `OutputRoot` via `base_protocol::OutputRoot::from_parts`
{% endstep %}

{% step %}
Asserts `extra_data`, block hash, and `OutputRoot` all diverge
{% endstep %}
{% endstepper %}

### Output

```
--- min_base_fee consolidation bypass PoC ---

[phase 1] Mining 3 honest blocks (min_base_fee = 8000000000)
    block #1       honest    base_fee=0             extra.min_base_fee=8000000000
    block #2       honest    base_fee=8000000000    extra.min_base_fee=8000000000
    block #3       honest    base_fee=8000000000    extra.min_base_fee=8000000000

[phase 2] Injecting poisoned block #4 (attacker rewrites min_base_fee → 0)
  ⚠ block #4       POISONED  base_fee=8000000000    extra.min_base_fee=0

[phase 3] Mining poisoned continuation blocks (min_base_fee = 0, cascading base fee down)
    block #5       POISON    base_fee=7001000400    extra.min_base_fee=0
    block #6       POISON    base_fee=6126750826    extra.min_base_fee=0
    block #7       POISON    base_fee=5361673123    extra.min_base_fee=0
    block #8       POISON    base_fee=4692134460    extra.min_base_fee=0
    block #9       POISON    base_fee=4106204404    extra.min_base_fee=0
    block #10      POISON    base_fee=3593442335    extra.min_base_fee=0
    block #11      POISON    base_fee=3144711404    extra.min_base_fee=0
    block #12      POISON    base_fee=2752015725    extra.min_base_fee=0

[phase 4] Submitting below-floor EIP-1559 tx (3 gwei vs 8 gwei floor)
  tx max_fee_per_gas     : 3000000000 wei
  honest min_base_fee    : 8000000000 wei
  poisoned head base_fee : 2752015725 wei

[phase 5] Mining block #13 from txpool
    block #13      txpool    base_fee=2408357899    extra.min_base_fee=8000000000

✅ CONFIRMED — cascade observed
   node accepted block with min_base_fee=0, base fee dropped below
   governance floor, txpool included tx below the intended min_base_fee.

--- proof-executor divergence on block #4 ---
  canonical extra_data (node accepted):  0x0100000008000000080000000000000000
  proof-exec extra_data (re-sealed):     0x01000000080000000800000001dcd65000
  min_base_fee in canonical:  0
  min_base_fee in proof-exec: 8000000000
  ✅ extra_data DIVERGES

  canonical block hash:   0x184d846a…5f60e767
  proof-exec block hash:  0x38337ea9…0f7bee9a
  ✅ block hashes DIVERGE

  canonical OutputRoot:   0xfa912956…fdd39358
  proof-exec OutputRoot:  0x1a5e65fd…c1707846
  ✅ real OutputRoots DIVERGE

--- done ---
root cause: attributes.rs:214 discards min_base_fee during consolidation
proof divergence: util.rs:87-96 re-seals with L1 min_base_fee → different hash/root
```

## Verified on `main`

Both root causes checked on `main`:

* `attributes.rs:214` — `_` wildcard still discards `min_base_fee`
* `util.rs:87-96` — proof executor still uses `attributes.min_base_fee`
* `block_validity.rs` — zero checks for `extra_data` contents
