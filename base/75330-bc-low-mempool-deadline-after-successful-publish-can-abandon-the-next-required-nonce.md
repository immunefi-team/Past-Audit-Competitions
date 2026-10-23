# 75330 bc low mempool deadline after successful publish can abandon the next required nonce

Submitted on Apr 28th 2026 at 14:45:24 UTC by @y4y for [Audit Comp | Base Azul](https://immunefi.com/audit-competition/audit-comp-base-azul)

* **Report ID:** #75330
* **Report Type:** Blockchain/DLT
* **Report severity:** Low
* **Target:** <https://github.com/base/base/tree/v0.8.0-rc.28>
* **Impacts:**
  * Temporary freezing of network transactions by delaying one block by 500% or more of the average block time of the preceding 24 hours beyond standard difficulty adjustments

## Description

## Brief/Intro

The tx manager sets `tx_not_in_mempool_timeout` before the first publish attempt, records a successful publish when `send_raw_transaction` returns, but still treats the expired mempool deadline as a terminal send failure if no receipt was observed in time. In the `send_async()` path that nonce was already pre-reserved and is not returned once publication succeeded. The next send therefore allocates the next nonce even though the lower nonce may still be live in the txpool or later mineable.

In the batcher path this becomes an L1 publication stall:

1. a batch publication tx is accepted with nonce `N`;
2. the tx manager returns `MempoolDeadlineExpired`;
3. the batcher requeues the same work and retries it;
4. `send_async()` pre-reserves nonce `N+1`;
5. the lower nonce `N` is still unresolved on L1;
6. `unsafe_l2` can continue advancing, but the client safe view remains stale until the lower L1 nonces are finally resolved.

## Vulnerability Details

`send_tx()` arms the mempool deadline before the transaction is ever published:

```rust
// base/crates/utilities/tx-manager/src/manager.rs
async fn send_tx(&self, candidate: TxCandidate, nonce_override: Option<u64>) -> SendResponse {
    ...
    let send_state = Arc::new(SendState::new(self.config.safe_abort_nonce_too_low_count)?);

    if !self.config.tx_not_in_mempool_timeout.is_zero() {
        send_state.set_mempool_deadline(Instant::now() + self.config.tx_not_in_mempool_timeout);
    }
    ...
}
```

The initial send path publishes first, then enters the event loop:

```rust
// base/crates/utilities/tx-manager/src/manager.rs
let prepared =
    self.prepare_with_initial_caps(candidate, None, None, nonce_override, None).await?;
let tx_hash = self.publish_tx(send_state, &prepared.raw_tx, None).await?;
let mut bump = BumpState::from_prepared(prepared, tx_hash);
```

`publish_tx()` records a successful publish:

```rust
// base/crates/utilities/tx-manager/src/manager.rs
match result {
    Ok(Ok(pending)) => {
        let tx_hash = *pending.tx_hash();
        send_state.record_successful_publish();
        info!(tx_hash = %tx_hash, "transaction published");
        Ok(tx_hash)
    }
    ...
}
```

But `critical_error()` still aborts on the mempool deadline even after `has_published = true`:

```rust
// base/crates/utilities/tx-manager/src/send_state.rs
pub fn critical_error(&self) -> Option<TxManagerError> {
    ...
    if let Some(deadline) = inner.mempool_deadline
        && now >= deadline
    {
        return Some(TxManagerError::MempoolDeadlineExpired);
    }

    None
}

pub fn record_successful_publish(&self) {
    let mut inner = self.inner.lock().expect("SendState mutex poisoned");
    inner.has_published = true;
}
```

When the send started through `send_async()`, the nonce was already consumed before the task was spawned:

```rust
// base/crates/utilities/tx-manager/src/manager.rs
async fn send_async(&self, candidate: TxCandidate) -> SendHandle {
    let (tx, rx) = oneshot::channel();

    let nonce = match self.nonce_manager.reserve_nonce().await {
        Ok(n) => Some(n),
        Err(e) => {
            let _ = tx.send(Err(e));
            return SendHandle::new(rx);
        }
    };

    let manager = self.clone();
    tokio::spawn(async move {
        let result = manager.send_tx(candidate, nonce).await;
        let _ = tx.send(result);
    });
    ...
}
```

After a failed `send_async()`, the reserved nonce is returned only if no transaction was ever published:

```rust
// base/crates/utilities/tx-manager/src/manager.rs
if let Some(n) = nonce_override
    && Self::should_return_reserved_nonce(&result, &send_state)
{
    self.nonce_manager.return_reserved_nonce(n).await;
}

fn should_return_reserved_nonce<T>(
    result: &TxManagerResult<T>,
    send_state: &SendState,
) -> bool {
    match result {
        Ok(_) | Err(TxManagerError::NonceTooHigh | TxManagerError::NonceTooLow) => false,
        Err(_) => !send_state.has_published(),
    }
}
```

Once the batcher sees that error, it treats it as a normal failed submission, requeues the frames, and tries again:

```rust
// base/crates/batcher/core/src/submissions.rs
let handle = self.tx_manager.send_async(candidate).await;
let outcome = match handle.await {
    Ok(receipt) => TxOutcome::Confirmed { l1_block },
    Err(TxManagerError::AlreadyReserved) => TxOutcome::TxpoolBlocked,
    Err(e) => {
        warn!(error = %e, "submission failed");
        TxOutcome::Failed
    }
};

match outcome {
    TxOutcome::Failed => {
        for id in ids {
            pipeline.requeue(id);
        }
        warn!(submissions = %count, "submission failed, requeued for retry");
    }
    ...
}
```

Concrete runtime example from the verified compose-backed PoC:

1. the batcher published L1 nonces `11` and `12`;
2. the chain nonce was still `11`, but the pending nonce had already advanced to `13`;
3. `txpool_content` still contained live batcher entries at nonces `11` and `12`;
4. `unsafe_l2` advanced from `23` to `24` while `safe_l2` stayed at `0`;
5. the previously abandoned lower nonces later mined in L1 block `28`;
6. only after those lower nonces resolved did the client safe head advance to `10`.

The logical mismatch is:

```
published nonce N -> local mempool timeout -> reserve nonce N+1 anyway
```

even though nonce `N` is still unresolved and still gates later nonce execution.

**Pre-conditions**

* `tx_not_in_mempool_timeout` is non-zero. The default config is `120s`.
* The send path uses `send_async()` or another caller that pre-reserves the nonce before the send task runs.
* The transaction is accepted by RPC and recorded as published.
* No receipt is observed before the mempool deadline expires.
* The caller retries the work after the failed send, as the batcher does.

## Impact Details

The compose-backed PoC demonstrates the protocol consequence directly:

* the batcher sender had live L1 txs at nonces `11` and `12`;
* the batcher pending nonce had already advanced to `13`;
* `unsafe_l2` continued to advance;
* the L1-backed `safe_l2` remained stale until those lower L1 nonces later mined.

The system can continue producing unsafe L2 blocks while the L1-backed safe view lags behind because publication is stranded behind abandoned lower nonces. The PoC prints this directly as:

```
proof: unsafe_l2=23 -> 24 while l1_backed_safe_l2=0 -> 0
```

Operational recovery is possible by eventual inclusion, explicit nonce repair, or restart, but forward safe progress is still lost once this path is hit.

## References

* base/crates/utilities/tx-manager/src/manager.rs:906-968, 1010-1022, 1606-1629
* base/crates/utilities/tx-manager/src/send\_state.rs:138-177
* base/crates/batcher/core/src/submissions.rs:145-220
* base/crates/utilities/tx-manager/src/config.rs:128-142

## Link to Proof of Concept

<https://gist.github.com/brandonshiyay/0d683896ee228a3269b9e78a3b392f35>

## Proof of Concept

Add the following file under `devnet/examples/stale_l1_publications.rs`:

```rust
//! End-to-end PoC for the post-publish mempool-deadline nonce-abandonment issue
//! against the Docker devnet.
//!
//! This attaches to an already-running compose devnet whose batcher was
//! started with a short mempool deadline. It demonstrates:
//! 1. live L2 traffic continues to produce unsafe blocks;
//! 2. the batcher abandons a lower L1 nonce and allocates higher ones;
//! 3. the client's L1-backed safe head stays stale during that window; and
//! 4. the supposedly failed lower-nonce L1 txs later mine anyway.

use std::{
    collections::BTreeMap,
    time::{Duration, Instant},
};

use alloy_consensus::SignableTransaction;
use alloy_eips::eip2718::Encodable2718;
use alloy_network::{Ethereum, TransactionBuilder};
use alloy_primitives::{Address, Bytes, TxHash, U256};
use alloy_provider::{Provider, RootProvider};
use alloy_signer::SignerSync;
use alloy_signer_local::PrivateKeySigner;
use base_common_network::Base;
use base_common_rpc_types::BaseTransactionRequest;
use devnet::{
    config::{ANVIL_ACCOUNT_1, BATCHER},
    rpc::DevnetRpcClient,
};
use eyre::{ContextCompat, Result, WrapErr, ensure};
use jsonrpsee::{
    core::client::ClientT,
    http_client::{HttpClient, HttpClientBuilder},
    rpc_params,
};
use serde_json::Value;
use tokio::{
    task::JoinHandle,
    time::{sleep, timeout},
};

const DEFAULT_L1_RPC_URL: &str = "http://127.0.0.1:4545";
const DEFAULT_L2_BUILDER_RPC_URL: &str = "http://127.0.0.1:7545";
const DEFAULT_L2_CLIENT_RPC_URL: &str = "http://127.0.0.1:8545";
const DEFAULT_L2_BUILDER_OP_RPC_URL: &str = "http://127.0.0.1:7549";
const DEFAULT_L2_CLIENT_OP_RPC_URL: &str = "http://127.0.0.1:8549";
const L2_CHAIN_ID: u64 = 84538453;
const TRAFFIC_TX_COUNT: u64 = 32;
const TRAFFIC_SPACING_MS: u64 = 250;
const WAIT_FOR_DEVNET_SECS: u64 = 120;
const MONITOR_TIMEOUT_SECS: u64 = 90;
const SAFE_STALL_WINDOW_SECS: u64 = 2;
const CATCHUP_TIMEOUT_SECS: u64 = 60;

#[derive(Debug, Clone)]
struct BatcherTxpoolSnapshot {
    latest_nonce: u64,
    pending_nonce: u64,
    entries: Vec<TxpoolEntry>,
}

#[derive(Debug, Clone)]
struct TxpoolEntry {
    subpool: String,
    nonce: u64,
    hash: TxHash,
}

#[derive(Debug, Clone)]
struct SyncSnapshot {
    l1_block: u64,
    builder_block: u64,
    client_block: u64,
    unsafe_l2: u64,
    safe_l2: u64,
    finalized_l2: u64,
}

#[derive(Debug, Clone)]
struct GapObservation {
    sync: SyncSnapshot,
    txpool: BatcherTxpoolSnapshot,
}

#[derive(Debug, Clone)]
struct StaleObservation {
    start: GapObservation,
    end: SyncSnapshot,
}

#[derive(Debug, Clone)]
struct MinedObservation {
    confirmed_nonces: Vec<ConfirmedNonce>,
    safe_l2_after_catchup: u64,
    finalized_l2_after_catchup: u64,
}

#[derive(Debug, Clone)]
struct ConfirmedNonce {
    nonce: u64,
    hash: TxHash,
    block: u64,
}

#[tokio::main]
async fn main() -> Result<()> {
    let l1_rpc_url = std::env::var("H04_L1_RPC_URL")
        .unwrap_or_else(|_| DEFAULT_L1_RPC_URL.to_string());
    let l2_builder_rpc_url = std::env::var("H04_L2_BUILDER_RPC_URL")
        .unwrap_or_else(|_| DEFAULT_L2_BUILDER_RPC_URL.to_string());
    let l2_client_rpc_url = std::env::var("H04_L2_CLIENT_RPC_URL")
        .unwrap_or_else(|_| DEFAULT_L2_CLIENT_RPC_URL.to_string());
    let l2_builder_op_rpc_url = std::env::var("H04_L2_BUILDER_OP_RPC_URL")
        .unwrap_or_else(|_| DEFAULT_L2_BUILDER_OP_RPC_URL.to_string());
    let l2_client_op_rpc_url = std::env::var("H04_L2_CLIENT_OP_RPC_URL")
        .unwrap_or_else(|_| DEFAULT_L2_CLIENT_OP_RPC_URL.to_string());

    println!("step1: attach to the running devnet");
    println!("l1 rpc: {l1_rpc_url}");
    println!("l2 builder rpc: {l2_builder_rpc_url}");
    println!("l2 client rpc: {l2_client_rpc_url}");
    println!("l2 builder op rpc: {l2_builder_op_rpc_url}");
    println!("l2 client op rpc: {l2_client_op_rpc_url}");

    let rpc = DevnetRpcClient::new(
        &l1_rpc_url,
        &l2_builder_rpc_url,
        &l2_client_rpc_url,
        &l2_builder_op_rpc_url,
        &l2_client_op_rpc_url,
    )?;
    let l1_provider = RootProvider::<Ethereum>::new_http(l1_rpc_url.parse()?);
    let l2_builder_provider = RootProvider::<Base>::new_http(l2_builder_rpc_url.parse()?);
    let l1_rpc = HttpClientBuilder::default()
        .build(&l1_rpc_url)
        .wrap_err("failed to create L1 JSON-RPC client")?;

    println!("step2: wait for all RPC endpoints to become live");
    wait_for_devnet(&rpc, &l2_builder_provider).await?;

    println!("step3: send ordinary L2 transactions while the batcher is using the short mempool deadline");
    let traffic_task = spawn_l2_traffic(l2_builder_provider.clone());

    println!("step4: watch for an abandoned lower batcher nonce on L1");
    let gap_observation =
        wait_for_nonce_gap(&rpc, &l1_provider, &l1_rpc, BATCHER.address).await?;
    println!(
        "observed abandoned lower nonce: l1_block={} builder_block={} client_block={} unsafe_l2={} safe_l2={} finalized_l2={} latest_nonce={} pending_nonce={} txpool_nonces={:?}",
        gap_observation.sync.l1_block,
        gap_observation.sync.builder_block,
        gap_observation.sync.client_block,
        gap_observation.sync.unsafe_l2,
        gap_observation.sync.safe_l2,
        gap_observation.sync.finalized_l2,
        gap_observation.txpool.latest_nonce,
        gap_observation.txpool.pending_nonce,
        gap_observation
            .txpool
            .entries
            .iter()
            .map(|entry| entry.nonce)
            .collect::<Vec<_>>(),
    );

    println!("step5: confirm that unsafe L2 keeps advancing while the L1-backed safe L2 head stays stale");
    let stale_observation = wait_for_stale_safe_head(&rpc, &gap_observation).await?;
    println!(
        "unsafe L2 advanced while the L1-backed safe head stayed stale: start(unsafe={}, safe={}) -> end(builder_block={}, client_block={}, unsafe={}, safe={}, finalized={})",
        stale_observation.start.sync.unsafe_l2,
        stale_observation.start.sync.safe_l2,
        stale_observation.end.builder_block,
        stale_observation.end.client_block,
        stale_observation.end.unsafe_l2,
        stale_observation.end.safe_l2,
        stale_observation.end.finalized_l2,
    );

    println!("step6: wait for the earlier 'failed' lower-nonce batcher txs to mine later on L1");
    let observed_nonces: Vec<u64> =
        gap_observation.txpool.entries.iter().map(|entry| entry.nonce).collect();
    let mined_observation = wait_for_later_inclusion(
        &rpc,
        &l1_provider,
        &l1_rpc,
        BATCHER.address,
        gap_observation.sync.l1_block,
        &observed_nonces,
        gap_observation.sync.safe_l2,
    )
    .await?;
    println!(
        "later inclusion observed: confirmed_nonces={:?} safe_l2_after_catchup={} finalized_l2_after_catchup={}",
        mined_observation
            .confirmed_nonces
            .iter()
            .map(|entry| format!("{}@{}:{:#x}", entry.nonce, entry.block, entry.hash))
            .collect::<Vec<_>>(),
        mined_observation.safe_l2_after_catchup,
        mined_observation.finalized_l2_after_catchup,
    );

    traffic_task.await.wrap_err("traffic task panicked")??;

    println!("result:");
    println!(
        "- Unsafe L2 kept moving: builder block {} -> {}, client unsafe {} -> {}.",
        gap_observation.sync.builder_block,
        stale_observation.end.builder_block,
        gap_observation.sync.unsafe_l2,
        stale_observation.end.unsafe_l2,
    );
    println!(
        "- Safe L2 stayed stale during that window: safe head remained {}.",
        gap_observation.sync.safe_l2,
    );
    println!(
        "- The batcher had already advanced its pending nonce from {} to {} while live txpool entries still existed at nonces {:?}.",
        gap_observation.txpool.latest_nonce,
        gap_observation.txpool.pending_nonce,
        gap_observation
            .txpool
            .entries
            .iter()
            .map(|entry| entry.nonce)
            .collect::<Vec<_>>(),
    );
    println!(
        "- The earlier abandoned nonces later mined on L1 as {:?}.",
        mined_observation
            .confirmed_nonces
            .iter()
            .map(|entry| format!("nonce {} in block {} ({:#x})", entry.nonce, entry.block, entry.hash))
            .collect::<Vec<_>>(),
    );
    println!(
        "- After those delayed inclusions, the client's safe head resumed and advanced to {}.",
        mined_observation.safe_l2_after_catchup,
    );
    println!(
        "proof: unsafe_l2={} -> {} while l1_backed_safe_l2={} -> {}",
        stale_observation.start.sync.unsafe_l2,
        stale_observation.end.unsafe_l2,
        stale_observation.start.sync.safe_l2,
        stale_observation.end.safe_l2,
    );

    Ok(())
}

fn spawn_l2_traffic(provider: RootProvider<Base>) -> JoinHandle<Result<()>> {
    tokio::spawn(async move {
        let signer = PrivateKeySigner::from_bytes(&ANVIL_ACCOUNT_1.private_key)
            .expect("ANVIL_ACCOUNT_1 private key must be valid");
        let sender = signer.address();
        let recipient: Address = "0x000000000000000000000000000000000000dEaD"
            .parse()
            .expect("recipient literal must be valid");
        let mut nonce = provider
            .get_transaction_count(sender)
            .pending()
            .await
            .wrap_err("failed to fetch initial sender nonce")?;

        for idx in 0..TRAFFIC_TX_COUNT {
            let request = BaseTransactionRequest::default()
                .from(sender)
                .to(recipient)
                .value(U256::from(1 + idx))
                .transaction_type(2)
                .with_gas_limit(21_000)
                .with_max_fee_per_gas(1_000_000_000)
                .with_max_priority_fee_per_gas(0)
                .with_chain_id(L2_CHAIN_ID)
                .with_nonce(nonce);
            nonce += 1;

            let tx = request
                .build_typed_tx()
                .map_err(|_| eyre::eyre!("invalid traffic transaction request"))?;
            let signature = signer.sign_hash_sync(&tx.signature_hash())?;
            let signed_tx = tx.into_signed(signature);
            let raw_tx: Bytes = signed_tx.encoded_2718().into();
            let expected_hash = *signed_tx.hash();

            let pending = provider
                .send_raw_transaction(&raw_tx)
                .await
                .wrap_err("failed to submit traffic tx")?;
            ensure!(
                *pending.tx_hash() == expected_hash,
                "traffic tx hash mismatch: expected {expected_hash:#x}, got {:#x}",
                pending.tx_hash(),
            );
            sleep(Duration::from_millis(TRAFFIC_SPACING_MS)).await;
        }

        Ok(())
    })
}

async fn wait_for_devnet(
    rpc: &DevnetRpcClient,
    l2_builder_provider: &RootProvider<Base>,
) -> Result<()> {
    timeout(Duration::from_secs(WAIT_FOR_DEVNET_SECS), async {
        loop {
            let builder_block = l2_builder_provider.get_block_number().await;
            let l1_block = rpc.l1_block_number().await;
            let builder_sync = rpc.l2_builder_sync_status().await;
            let client_sync = rpc.l2_client_sync_status().await;

            if let (Ok(builder_block), Ok(l1_block), Ok(builder_sync), Ok(client_sync)) =
                (builder_block, l1_block, builder_sync, client_sync)
                && builder_block > 0
                && l1_block > 0
                && builder_sync.unsafe_l2.block_info.number > 0
                && client_sync.unsafe_l2.block_info.number > 0
            {
                return Ok::<_, eyre::Error>(());
            }

            sleep(Duration::from_millis(500)).await;
        }
    })
    .await
    .wrap_err("timed out waiting for devnet RPCs")??;
    Ok(())
}

async fn wait_for_nonce_gap(
    rpc: &DevnetRpcClient,
    l1_provider: &RootProvider<Ethereum>,
    l1_rpc: &HttpClient,
    batcher_address: Address,
) -> Result<GapObservation> {
    let deadline = Instant::now() + Duration::from_secs(MONITOR_TIMEOUT_SECS);
    loop {
        ensure!(Instant::now() < deadline, "timed out waiting for an abandoned lower nonce");
        let sync = collect_sync_snapshot(rpc).await?;
        let txpool = batcher_txpool_snapshot(l1_provider, l1_rpc, batcher_address).await?;
        print_status_line("monitor", &sync, &txpool);

        if txpool.pending_nonce >= txpool.latest_nonce + 2 && txpool.entries.len() >= 2 {
            return Ok(GapObservation { sync, txpool });
        }

        sleep(Duration::from_millis(200)).await;
    }
}

async fn wait_for_stale_safe_head(
    rpc: &DevnetRpcClient,
    start: &GapObservation,
) -> Result<StaleObservation> {
    let deadline = Instant::now() + Duration::from_secs(MONITOR_TIMEOUT_SECS);
    let stall_deadline = Instant::now() + Duration::from_secs(SAFE_STALL_WINDOW_SECS);

    loop {
        ensure!(Instant::now() < deadline, "timed out waiting for stale safe-head divergence");
        let sync = collect_sync_snapshot(rpc).await?;
        if sync.unsafe_l2 > start.sync.unsafe_l2 && sync.safe_l2 == start.sync.safe_l2
            && Instant::now() >= stall_deadline
        {
            return Ok(StaleObservation { start: start.clone(), end: sync });
        }
        sleep(Duration::from_millis(200)).await;
    }
}

async fn wait_for_later_inclusion(
    rpc: &DevnetRpcClient,
    l1_provider: &RootProvider<Ethereum>,
    l1_rpc: &HttpClient,
    batcher_address: Address,
    start_l1_block: u64,
    observed_nonces: &[u64],
    stale_safe_head: u64,
) -> Result<MinedObservation> {
    let deadline = Instant::now() + Duration::from_secs(CATCHUP_TIMEOUT_SECS);
    loop {
        ensure!(Instant::now() < deadline, "timed out waiting for delayed L1 inclusion");
        let confirmed_nonces = find_confirmed_batcher_nonces(
            l1_provider,
            l1_rpc,
            batcher_address,
            start_l1_block,
            observed_nonces,
        )
        .await?;

        let sync = rpc.l2_client_sync_status().await?;
        if confirmed_nonces.len() == observed_nonces.len() && sync.safe_l2.block_info.number > stale_safe_head {
            return Ok(MinedObservation {
                confirmed_nonces,
                safe_l2_after_catchup: sync.safe_l2.block_info.number,
                finalized_l2_after_catchup: sync.finalized_l2.block_info.number,
            });
        }

        sleep(Duration::from_millis(500)).await;
    }
}

async fn find_confirmed_batcher_nonces(
    l1_provider: &RootProvider<Ethereum>,
    l1_rpc: &HttpClient,
    batcher_address: Address,
    start_l1_block: u64,
    observed_nonces: &[u64],
) -> Result<Vec<ConfirmedNonce>> {
    let target_nonces = observed_nonces.iter().copied().collect::<std::collections::BTreeSet<_>>();
    let latest_l1_block = l1_provider
        .get_block_number()
        .await
        .wrap_err("failed to query latest L1 block")?;
    let batcher_hex = format!("{batcher_address:#x}");
    let mut found = BTreeMap::new();

    for block_number in start_l1_block..=latest_l1_block {
        let block_tag = format!("0x{block_number:x}");
        let block: Value = ClientT::request(
            l1_rpc,
            "eth_getBlockByNumber",
            rpc_params![block_tag, true],
        )
        .await
        .wrap_err("eth_getBlockByNumber")?;
        let Some(transactions) = block.get("transactions").and_then(Value::as_array) else {
            continue;
        };
        for tx in transactions {
            let Some(from) = tx.get("from").and_then(Value::as_str) else {
                continue;
            };
            if !from.eq_ignore_ascii_case(&batcher_hex) {
                continue;
            }
            let Some(nonce_value) = tx.get("nonce").and_then(Value::as_str) else {
                continue;
            };
            let nonce = parse_quantity_u64(nonce_value)?;
            if !target_nonces.contains(&nonce) {
                continue;
            }
            let hash = tx
                .get("hash")
                .and_then(Value::as_str)
                .context("block transaction missing hash")?
                .parse::<TxHash>()
                .wrap_err("failed to parse block tx hash")?;
            found.insert(nonce, ConfirmedNonce { nonce, hash, block: block_number });
        }
    }

    Ok(found.into_values().collect())
}

async fn collect_sync_snapshot(rpc: &DevnetRpcClient) -> Result<SyncSnapshot> {
    let client_sync = rpc.l2_client_sync_status().await?;
    Ok(SyncSnapshot {
        l1_block: rpc.l1_block_number().await?,
        builder_block: rpc.l2_builder_block_number().await?,
        client_block: rpc.l2_client_block_number().await?,
        unsafe_l2: client_sync.unsafe_l2.block_info.number,
        safe_l2: client_sync.safe_l2.block_info.number,
        finalized_l2: client_sync.finalized_l2.block_info.number,
    })
}

async fn batcher_txpool_snapshot(
    l1_provider: &RootProvider<Ethereum>,
    l1_rpc: &HttpClient,
    batcher_address: Address,
) -> Result<BatcherTxpoolSnapshot> {
    let latest_nonce: u64 = l1_provider
        .get_transaction_count(batcher_address)
        .await
        .wrap_err("failed to query latest batcher nonce")?;
    let pending_nonce: u64 = l1_provider
        .get_transaction_count(batcher_address)
        .pending()
        .await
        .wrap_err("failed to query pending batcher nonce")?;
    let content: Value =
        ClientT::request(l1_rpc, "txpool_content", rpc_params![]).await.wrap_err("txpool_content")?;

    let batcher_hex = format!("{batcher_address:#x}");
    let mut nonces: BTreeMap<u64, TxpoolEntry> = BTreeMap::new();
    for subpool in ["pending", "queued"] {
        let Some(addresses) = content.get(subpool).and_then(Value::as_object) else {
            continue;
        };
        for (address, nonce_map) in addresses {
            if !address.eq_ignore_ascii_case(&batcher_hex) {
                continue;
            }
            let Some(nonce_map) = nonce_map.as_object() else {
                continue;
            };
            for (nonce_key, tx_value) in nonce_map {
                let hash = tx_value
                    .get("hash")
                    .and_then(Value::as_str)
                    .context("batcher txpool entry missing hash")?
                    .parse::<TxHash>()
                    .wrap_err("failed to parse txpool hash")?;
                let nonce = parse_quantity_u64(nonce_key)?;
                nonces.insert(
                    nonce,
                    TxpoolEntry { subpool: subpool.to_string(), nonce, hash },
                );
            }
        }
    }

    Ok(BatcherTxpoolSnapshot { latest_nonce, pending_nonce, entries: nonces.into_values().collect() })
}

fn parse_quantity_u64(value: &str) -> Result<u64> {
    if let Some(trimmed) = value.strip_prefix("0x") {
        if trimmed.is_empty() {
            return Ok(0);
        }
        return u64::from_str_radix(trimmed, 16)
            .wrap_err_with(|| format!("failed to parse quantity {value} as hex u64"));
    }

    value
        .parse::<u64>()
        .wrap_err_with(|| format!("failed to parse quantity {value} as decimal u64"))
}

fn print_status_line(prefix: &str, sync: &SyncSnapshot, txpool: &BatcherTxpoolSnapshot) {
    let entries = txpool
        .entries
        .iter()
        .map(|entry| format!("{}:{}:{:#x}", entry.subpool, entry.nonce, entry.hash))
        .collect::<Vec<_>>()
        .join(", ");
    println!(
        "{prefix}: l1_block={} builder_block={} client_block={} unsafe_l2={} safe_l2={} finalized_l2={} latest_nonce={} pending_nonce={} txpool=[{}]",
        sync.l1_block,
        sync.builder_block,
        sync.client_block,
        sync.unsafe_l2,
        sync.safe_l2,
        sync.finalized_l2,
        txpool.latest_nonce,
        txpool.pending_nonce,
        entries,
    );
}

```

For more instruction/reproduction steps, please see attached gist.
