# 74772 bc medium remote unauthenticated el crash via admin postunsafepayload and zero elasticity base fee

**Submitted on Apr 24th 2026 at 19:33:32 UTC by @Zeta for** [**Audit Comp | Base Azul**](https://immunefi.com/audit-competition/audit-comp-base-azul)

* **Report ID:** #74772
* **Report Type:** Blockchain/DLT
* **Report severity:** Medium
* **Target:** <https://github.com/base/base/tree/v0.8.0-rc.24>
* **Impacts:**
  * Increasing network processing node resource consumption by at least 30% without brute force actions, compared to the preceding 24 hours
  * Shutdown of greater than or equal to 30% of network processing nodes without brute force actions, but does not shut down the network

## Description

## Summary

A default-configured `base-consensus node` binds its JSON-RPC socket to `0.0.0.0:9545` and mounts `admin_postUnsafePayload` with no authentication. One POST of a payload whose `extra_data` encodes `elasticity_multiplier = 0` crashes the follower's reth EL via divide-by-zero in the txpool maintainer's base-fee refresh as a result of two bugs:

* `--rpc.enable-admin` is declared in `--help`, but never read at runtime. Admin methods are always available.
* `compute_jovian_base_fee` falls back to safe defaults only when `elasticity == 0 && denominator == 0`. Any mixed-zero pair divides by zero downstream, causing a panic and crashing the attacked node's execution layer.

## Preconditions

* Target is running the `base-consensus node` subcommand (the production path).
* The RPC server is not disabled via `--rpc.disabled` (RPC is enabled by default).
* Attacker has TCP reach to the target's consensus RPC port (default `:9545`).

## Vulnerability

### Admin API without authentication

The flag is declared with help text "Enable the admin API." (`crates/client/cli/src/rpc.rs:31-32`), stored in `RpcBuilder.enable_admin` (`crates/consensus/rpc/src/config.rs:13`), and never read outside two test fixtures (`crates/consensus/service/src/actors/rpc/actor.rs:185,201`).

The admin module is mounted whenever `network_admin.is_some()` (`crates/consensus/service/src/actors/rpc/actor.rs:119-123`):

```rust
if let Some(network_admin) = network_admin {
    modules.merge(AdminRpc::new(self.sequencer_admin_rpc_client, network_admin).into_rpc())?;
}
```

`network_admin` is unconditionally `Some(net_admin_rpc)` on the `base-consensus node` subcommand (`crates/consensus/service/src/service/node.rs:552`). The RPC middleware applies no authentication (`crates/consensus/service/src/actors/rpc/actor.rs:71-80`). The handler has no signer check (`crates/consensus/rpc/src/admin.rs:70-81`) and forwards to `engine_client.send_unsafe_block(payload)`, bypassing the `UnsafeBlockSigner` check that gossip-arriving payloads face (`crates/consensus/gossip/src/block_validity.rs:218-230`).

Default CLI values: `--rpc.addr 0.0.0.0`, `--port 9545`. In contrast, the `base-consensus follow` subcommand is unaffected as it passes `network_admin: None` (`crates/consensus/service/src/service/follow.rs:216`).

### Mixed-zero `extra_data` div-by-zero

On Jovian-active chains, `BaseChainSpec::next_block_base_fee` (`crates/execution/chainspec/src/spec.rs:188-196`) dispatches to `compute_jovian_base_fee`. This helper decodes `(elasticity, denominator)` from the parent's `extra_data`, builds a `BaseFeeParams`, and passes it to `alloy_eips::calc_next_block_base_fee` without checking whether elasticity is zero when the denominator is non-zero (`crates/execution/chainspec/src/basefee.rs:51-57`):

```rust
let (elasticity, denominator, min_base_fee) = JovianExtraData::decode(parent.extra_data())?;
let base_fee_params = if elasticity == 0 && denominator == 0 {
    chain_spec.base_fee_params_at_timestamp(timestamp)
} else {
    BaseFeeParams::new(denominator as u128, elasticity as u128)
};
```

`compute_jovian_base_fee` then calls `calc_next_block_base_fee(gas_used, parent.gas_limit(), …, base_fee_params)` (`crates/execution/chainspec/src/basefee.rs:63`). `(elasticity=0, denominator=1)` takes the `else` branch -> `BaseFeeParams::new(1, 0)` -> `calc_next_block_base_fee` computes `gas_target = gas_limit / base_fee_params.elasticity_multiplier as u64` (`alloy-eips-1.8.3/src/eip1559/helpers.rs:99`), triggering a divide-by-zero panic and crashing the process.

`decode_holocene_base_fee` has the same bug but is unreachable on Base mainnet because Jovian is active (`crates/execution/chainspec/src/basefee.rs:24-30`). The proof-side guard rejects `denominator == 0` but not `elasticity == 0` (`crates/proof/executor/src/util.rs:28,46`); the live-node path has neither guard.

## Execution Path

{% stepper %}
{% step %}

## The attacker reads the latest block

The attacker opens a TCP connection to `:9545` and reads the target's latest block via `debug_getRawBlock latest`.
{% endstep %}

{% step %}

## The attacker forges a payload

They clone the header byte-for-byte, overwrite `extra_data` with a 17-byte (version=1, denominator=1, elasticity=0, min\_base\_fee=0), and recompute `block_hash`.
{% endstep %}

{% step %}

## The unsafe payload is posted

`POST admin_postUnsafePayload(P1)`. The CL handler (`crates/consensus/rpc/src/admin.rs:70-81`) forwards to `engine_client.send_unsafe_block(payload)` (`crates/consensus/service/src/actors/network/actor.rs:205`), which is dispatched as an `InsertTask` that calls `new_payload_v4` on the EL (`crates/consensus/engine/src/task_queue/tasks/insert/task.rs:100`). P1 validates against its honest parent (extra\_data is the only changed field; its parent's extra\_data is clean) and imports.
{% endstep %}

{% step %}

## Synchronization canonicalizes the block

A `SynchronizeTask` follows, calling `fork_choice_updated_v3` with unsafe head = `P1.hash` (`crates/consensus/engine/src/task_queue/tasks/synchronize/task.rs:136`). reth canonicalizes P1.
{% endstep %}

{% step %}

## The transaction pool maintenance task observes the change

The canonicalization fires a `CanonStateNotification` that reth's `maintain_transaction_pool_future` task observes. To refresh the pool's notion of the next block's base fee, it calls `chain_spec.next_block_base_fee(new_tip.header(), new_tip.timestamp())` (reth upstream `crates/transaction-pool/src/maintain.rs:341-342`, pinned commit `d6324d6`).
{% endstep %}

{% step %}

## Jovian base fee computation starts

`BaseChainSpec::next_block_base_fee` (`crates/execution/chainspec/src/spec.rs:190`) sees Jovian is active at P1's timestamp and dispatches to `compute_jovian_base_fee(parent=P1, …)` (`crates/execution/chainspec/src/basefee.rs:63`).
{% endstep %}

{% step %}

## Divide-by-zero occurs

`compute_jovian_base_fee` calls `JovianExtraData::decode(P1.extra_data)` -> `(elasticity=0, denominator=1, min_base_fee=0)` -> `BaseFeeParams::new(1, 0)` -> `calc_next_block_base_fee(gas_used, gas_limit, base_fee, params)` (`alloy-eips-1.8.3/src/eip1559/helpers.rs:99`) -> `gas_limit / 0` -> panic.
{% endstep %}

{% step %}

## The process exits

reth classifies the maintainer as a critical task; the main thread unwraps the propagated error and the process exits with code 101 (`bin/node/src/main.rs:75`). The binary has no internal restart.

The PoC retries because the sequencer's gossip advances the unsafe head between the attacker's fetch and POST. On a 2-second block time the attack lands in 1–3 attempts.
{% endstep %}
{% endstepper %}

## Impact

Remote unauthenticated DoS against any node started with `base-consensus node` (the production subcommand) and reachable on its RPC port. Third-party Base followers are exposed when they accept the shipped defaults as the binary ships the port on `0.0.0.0` and the `--rpc.enable-admin` opt-out has no effect.

Port `:9545` also serves `healthz`, `optimism_syncStatus`, `optimism_rollupConfig`, `optimism_version`, and `opp2p_*`. Therefore, operators have legitimate reasons to expose it for monitoring and dashboards, and have no clean way to disable only admin without taking the rest down with `--rpc.disabled` or a reverse proxy.

Node operators would have to restart nodes (manually or in an automated way), after which the attacker could trivially crash them again, causing excessive compute consumption and potentially disrupting the network.

### Sequencer Halt

The missing admin gate also exposes the other mutator methods in `AdminApi`. Notably, `admin_stopSequencer` on sequencer-mode nodes, which halts block production until `admin_startSequencer(<head>)` is posted; the chain tip (observed via `eth_blockNumber` on any EL) stops advancing.

If the sequencer CL's RPC port were exposed, this could lead to a true chain halt.\
However, I would expect the sequencer node operated by Base to have a functioning firewall that prevents this attack.

## Suggested Fix

Consider gating the admin merge on `self.config.enable_admin` (`crates/consensus/service/src/actors/rpc/actor.rs:120`):

```rust
if let (true, Some(network_admin)) = (self.config.enable_admin, network_admin) {
    modules.merge(AdminRpc::new(self.sequencer_admin_rpc_client, network_admin).into_rpc())?;
}
```

Reject mixed-zero pairs in `JovianExtraData::decode` (`crates/common/consensus/src/extra/jovian.rs:26-41`), and apply the same guard to `HoloceneExtraData::decode` (`crates/common/consensus/src/extra/holocene.rs`):

```rust
if (denominator == 0) != (elasticity == 0) {
    return Err(EIP1559ParamError::InvalidParams);
}
```

Furthermore, consider defaulting `--rpc.addr` to `127.0.0.1` (`crates/client/cli/src/rpc.rs:25`).

## Link to Proof of Concept

<https://gist.github.com/0Zeta/ddc1308c536df47f2c68d1be72bbf3f9>

## Proof of Concept

Requirements: Docker + `docker compose`, `just` (`cargo install just`), Rust toolchain 1.93.1 (auto-installed from `attack/rust-toolchain.toml`), `curl`, `git`.

Execute the PoC script from the Gist in a terminal:

```bash
./run.sh
```

It should create the following directory structure and run the PoC:

```
base_poc/
├── run.sh
└── attack/
    ├── Cargo.toml
    ├── rust-toolchain.toml
    └── src/
        └── main.rs
```

`run.sh` clones `base/base@v0.8.0-rc.24`, runs `just devnet::up-single`, overrides the follower container's docker restart policy to `no` (so the binary's own lack of internal restart is visible rather than masked by docker's `unless-stopped` default), runs the attack, and verifies the container reaches `state: exited, exit: 101`. Expected tail:

```
base-client state: exited (exit code 101)
--- panic signature in base-client logs ---
thread 'tokio-rt' panicked at alloy-eips-1.8.3/src/eip1559/helpers.rs:99:22: attempt to divide by zero
  4: base_execution_chainspec::basefee::compute_jovian_base_fee
  5: <BaseChainSpec as EthChainSpec>::next_block_base_fee
  7: reth_transaction_pool::maintain
ERROR Critical task `txpool maintenance task` panicked
thread 'main' panicked at bin/node/src/main.rs:75:6
PoC succeeded: base-client exited with code 101 and is not running.
```

Teardown: `( cd base && just devnet::down )`
