# 75597 bc insight lossy ingress metering fanout lets later lower fee transactions outrank earlier higher fee transactions under builder metering wait mode

**Submitted on Apr 30th 2026 at 00:49:06 UTC by @kaiserlimp0 for** [**Audit Comp | Base Azul**](https://immunefi.com/audit-competition/audit-comp-base-azul)

* **Report ID:** #75597
* **Report Type:** Blockchain/DLT
* **Report severity:** Insight
* **Target:** <https://github.com/base/base/tree/v0.8.0-rc.28>
* **Impacts:**
  * A bug in the respective layer 0/1/2 network code that results in unintended smart contract behavior with no concrete funds at direct risk

## Description

## Brief/Intro

Under builder metering wait mode, Base Azul's offchain ingress/builder path can lose fairness-critical metering updates. A public raw transaction burst can make the ingress BuilderConnector lag, causing an earlier victim transaction's metering update to be dropped while later attacker metering updates still reach the builder. The builder then skips the victim as metering\_data\_pending and includes a later lower-fee attacker transaction first. The local PoC proves this as a deterministic FirstWins state-race inversion.

Severity note: this is submitted as Medium because the deployment oracle is not publicly confirmable by the researcher. If the sponsor confirms that the Base Azul competition/public transaction deployment uses the metering wait path described below, the demonstrated impact should be re-evaluated as a concrete inclusion fairness / transaction starvation issue.

## Vulnerability Details

ingress-rpc forwards MeterBundleResponse values to builder RPCs through a bounded tokio::broadcast channel. Bounded broadcast semantics drop older messages when a receiver lags. BuilderConnector handles RecvError::Lagged(n) by logging and continuing, with no replay, retry, durable queue, tx-hash repair, or fail-safe behavior.

Relevant code:

* `crates/infra/ingress-rpc/src/lib.rs`
* `Config.max_buffered_meter_bundle_responses`
* `Config.send_to_builder`
* `BuilderConnector::connect(...)`
* `RecvError::Lagged(n) => warn and continue`

The builder then treats missing metering data as a temporary exclusion condition for fresh transactions:

* `crates/builder/core/src/flashblocks/context.rs`
* if metering is enabled
* and tx metering data is missing
* and tx age is below metering\_wait\_duration
* then the tx is skipped as pending metering data

This composition makes metering data inclusion-critical, but the delivery path is lossy.

Attack flow:

* victim high-tip tx enters first
* victim metering event is broadcast
* attacker public tx burst makes BuilderConnector lag
* victim metering update is lost from the bounded fanout
* later attacker metering updates still reach the builder
* builder skips victim as metering\_data\_pending
* lower-tip/later attacker tx is included first
* FirstWins.winner = attacker

## Why This Is Not Normal Sequencing Discretion

This report does not claim that the builder must always provide absolute fee ordering. The vulnerability is narrower: unrelated attacker traffic can selectively remove fairness-critical metering data for earlier honest transactions while later attacker metering data survives. The builder then skips the earlier transaction as metering\_data\_pending and includes the later attacker transaction.

The PoC controls for alternative explanations:

* the victim transaction is valid
* the victim has sufficient balance and correct nonce
* the attack block has enough remaining gas for the victim
* the victim remains pending after being skipped
* disabling metering\_wait\_duration prevents the inversion
* the same state race is won by the victim when metering data is present

Therefore the changed contract outcome is caused by lossy metering delivery, not by normal transaction ordering discretion, invalid transaction state, or block capacity.

## Deployment Oracle Requested

Please confirm, at a yes/no level, whether the Base Azul competition/public transaction deployment satisfies:

* `TIPS_INGRESS_SEND_TO_BUILDER=true`
* AND `TIPS_INGRESS_BUILDER_RPCS` is non-empty
* AND builder-side resource or execution metering is enabled
* AND `builder.metering-wait-duration-ms > 0`

The attacker does not need privileged builder RPC access and does not call base\_setMeteringInformation directly. The attacker-controlled input is normal public raw transaction submission. The sponsor-confirmable condition is only whether that public ingress path is connected to a builder running nonzero metering wait mode.

## Impact Details

This creates a deterministic priority inversion in the builder's own local inclusion outcome: under identical transaction validity and fee setup, the higher-tip earlier victim wins when metering is present, but loses solely because its metering update was dropped by attacker-induced fanout lag.

Observed PoC result:

* Baseline:
  * `victim_tip = 10 gwei`
  * `attacker_tip = 1 gwei`
  * `victim metering = present`
  * `attacker metering = present`
  * `FirstWins.winner = victim`
* Attack:
  * `victim_tip = 10 gwei`
  * `attacker_tip = 1 gwei`
  * `victim metering = missing`
  * `attacker metering = present`
  * `FirstWins.winner = attacker`

The PoC also proves recovery can be too late: once the lower-tip attacker transaction wins `FirstWins.claim()`, later victim recovery cannot undo the state-race result.

## Why The Selected Impact Applies

The changed behavior is caused by L2/offchain network code, not by the FirstWins contract. The same contract and same transaction setup produce victim victory in the baseline, but attacker victory when the ingress metering fanout drops the victim metering update. Therefore the L2 network code causes unintended smart contract behavior, even though the local PoC does not demonstrate direct funds at risk.

This matches the selected impact: a bug in the L2 network code that results in unintended smart contract behavior with no concrete funds at direct risk.

## References

* `crates/infra/ingress-rpc/src/lib.rs`
* `crates/infra/ingress-rpc/src/service.rs`
* `crates/builder/core/src/flashblocks/context.rs`
* `crates/builder/core/tests/smoke.rs`

## Link to Proof of Concept

<https://gist.github.com/kaiserlimp0/3339af704720cefb83864b00e01e6baf>

## Proof of Concept

Target: base/base v0.8.0-rc.28\
Commit: e3467a2048881213b56739a54a876efb9c6ea103

PoC patch:

* PoC patch Gist:\
  <https://gist.github.com/kaiserlimp0/3339af704720cefb83864b00e01e6baf>
* Patch raw URL:\
  <https://gist.githubusercontent.com/kaiserlimp0/3339af704720cefb83864b00e01e6baf/raw/600f0780f6dd9922c0b9cc516cc706b4cdf0cdf1/INGRESS\\_STARVATION\\_POC.patch>
* Patch sha256:\
  `4e584bae7d2d1395bfbec618dd553d926093fb7f9de286b91bcca04466aa223b`

The PoC consists of six local tests.

{% stepper %}
{% step %}

## Running BuilderConnector drops victim but forwards attacker tail

**Test:**\
`crates/infra/ingress-rpc/src/lib.rs`\
`test_running_builder_connector_lag_drops_victim_but_forwards_attacker_tail`

This starts BuilderConnector normally and blocks it on a slow builder RPC response. While blocked, the test broadcasts one victim metering event followed by attacker metering events over a bounded channel.

**Assertions:**

* builder RPC received blocker tx hash
* builder RPC did not receive victim tx hash
* builder RPC did receive at least one later attacker tx hash
  {% endstep %}

{% step %}

## Public raw transaction path triggers the same loss

**Test:**\
`crates/infra/ingress-rpc/src/service.rs`\
`test_public_raw_tx_burst_can_drop_victim_metering_update`

This uses `IngressService::send_raw_transaction`, not direct internal broadcast writes. A dynamic base\_meterBundle mock returns `MeterBundleResponse` for the actual raw transaction hash. `send_to_builder=true` is enabled and the builder fanout buffer is bounded.

**Assertions:**

* public blocker raw tx metering update reaches builder
* public victim raw tx metering update is lost under fanout lag
* later public attacker raw tx metering updates reach builder
  {% endstep %}

{% step %}

## Builder impact is priority inversion

**Test:**\
`crates/builder/core/tests/smoke.rs`\
`metering_starvation_causes_lower_tip_first_wins_priority_inversion`

**Baseline:**

* victim priority fee = 10 gwei
* attacker priority fee = 1 gwei
* both have metering data
* victim included before attacker
* `FirstWins.winner = victim`

**Attack:**

* victim priority fee = 10 gwei
* attacker priority fee = 1 gwei
* victim metering data missing
* attacker metering data present
* victim skipped as pending metering data
* attacker included first
* `FirstWins.winner = attacker`

The test also asserts that the attack block still has enough remaining gas for the victim transaction and that the skipped victim remains in the txpool. This rules out "block was full" or "victim was invalid" as the explanation.
{% endstep %}

{% step %}

## Recovery is too late for FirstWins state races

**Test:**\
`crates/builder/core/tests/smoke.rs`\
`metering_starvation_causes_irreversible_first_wins_loss_after_recovery`

**Attack block:**

* victim metering data missing
* attacker metering data present
* victim skipped as pending metering data
* attacker included first
* `FirstWins.winner = attacker`

**Recovery block:**

* victim metering data arrives
* victim transaction is processed
* `FirstWins.winner` remains attacker

This proves the impact is not only temporary delay. For ordering-sensitive state races, recovery after the attacker has already won is too late.
{% endstep %}

{% step %}

## Wait-mode negative control

**Test:**\
`crates/builder/core/tests/smoke.rs`\
`missing_metering_does_not_invert_order_when_wait_mode_disabled`

This keeps victim metering absent but disables `metering_wait_duration`.

**Result:**

* victim included before attacker
* `FirstWins.winner = victim`

This isolates the deployment oracle: missing metering becomes exploitable only when builder metering wait mode is active.
{% endstep %}

{% step %}

## Default buffer threshold

**Ignored slow test:**\
`crates/infra/ingress-rpc/src/lib.rs`\
`ignored_metering_starvation_reproduces_with_default_buffer_100`

**Observed output:**

* `buffer=100 rpc_delay_ms=25 attacker_txs=250 victim_lost=true attacker_tail_forwarded=true`
* `buffer=100 rpc_delay_ms=10 attacker_txs=250 victim_lost=true attacker_tail_forwarded=true`
* `buffer=100 rpc_delay_ms=50 attacker_txs=150 victim_lost=true attacker_tail_forwarded=true`

This shows the primitive is not limited to the deterministic tiny-buffer demo.
{% endstep %}
{% endstepper %}

Reproduction commands:

```bash
cargo test -p ingress-rpc-lib -- --nocapture

cargo test -p ingress-rpc-lib ignored_metering_starvation_reproduces_with_default_buffer_100 -- --ignored --nocapture

cargo test -p base-builder-core --test smoke metering_starvation_causes_lower_tip_first_wins_priority_inversion -- --nocapture

cargo test -p base-builder-core --test smoke metering_starvation_causes_irreversible_first_wins_loss_after_recovery -- --nocapture

cargo test -p base-builder-core --test smoke missing_metering_does_not_invert_order_when_wait_mode_disabled -- --nocapture
```

Expected result:

* `ingress-rpc-lib`:
  * 19 passed, 1 ignored
* ignored buffer=100 threshold:
  * 1 passed
* base-builder-core focused smoke tests:
  * priority inversion passed
  * irreversible recovery passed
  * wait-mode negative control passed

Local verification performed:

* `cargo test -p ingress-rpc-lib -- --nocapture`
  * PASS: 19 passed, 1 ignored
* `cargo test -p ingress-rpc-lib ignored_metering_starvation_reproduces_with_default_buffer_100 -- --ignored --nocapture`
  * PASS: 1 passed
* `cargo test -p base-builder-core --test smoke metering_starvation_causes_lower_tip_first_wins_priority_inversion -- --nocapture`
  * PASS
* `cargo test -p base-builder-core --test smoke metering_starvation_causes_irreversible_first_wins_loss_after_recovery -- --nocapture`
  * PASS
* `cargo test -p base-builder-core --test smoke missing_metering_does_not_invert_order_when_wait_mode_disabled -- --nocapture`
  * PASS
* `RUNS=1 scripts/run_metering_starvation_poc_repeated.sh`
  * PASS: repeatability\_summary runs=1 ingress\_passes=1 builder\_passes=1
* `cargo fmt --all --check`
  * PASS, only existing stable-rust warnings for nightly rustfmt options
* `git diff --check`
  * PASS
