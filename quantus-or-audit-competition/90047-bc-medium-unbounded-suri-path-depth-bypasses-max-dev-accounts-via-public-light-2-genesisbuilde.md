# 90047 bc medium unbounded suri path depth bypasses max dev accounts via public light 2 genesisbuilder build state consuming 190 cpu seconds per 34 kb unauthenticated p2p request

**Submitted on Aug 22nd 2026 at 17:52:57 UTC by @Jornason for** [**Audit Comp | Quantus**](https://immunefi.com/audit-competition/audit-comp-quantus)

* **Report ID:** #90047
* **Report Type:** Blockchain/DLT
* **Report severity:** Medium
* **Target:** <https://github.com/immunefi-team/audit-comp-quantus-chain>
* **Impacts:**
  * Increasing network processing node resource consumption by at least 30% without brute force actions, compared to the preceding 24 hours
  * Griefing (e.g. no profit motive for an attacker, but damage to the users or the protocol)

## Description

### Brief/Intro

An ordinary, unauthenticated P2P peer can call the production runtime method GenesisBuilder\_build\_state on a running Quantus full node through the default //light/2 light-client proof protocol, which forwards peer-selected method names and bytes to execution\_proof with no method allowlist. Quantus publicly fixed this externally reachable workload in PR #616 by capping balances.devAccounts at MAX\_DEV\_ACCOUNTS = 1,000 — but the fix bounds only the account count, not the attacker-controlled SURI template inside each derivation. A valid 32,767-byte template containing 10,922 hard HDKD junctions, combined with the permitted 1,000 accounts, makes one 34,313-byte protobuf request execute 10,922,000 hard sr25519 derivation steps. Measured against an unmodified node built from the frozen competition commit, that single request kept the victim's serial light-proof handler busy for 202.5 seconds and consumed 189.95 victim CPU-seconds (93.7% of one core); an independent re-run reproduced 190.00 CPU-seconds. One such request every \~3.4 minutes (\~169 request bytes/second, far below any traffic-flood threshold) keeps the global light-proof service permanently occupied, and queued requests from other peers time out and are still executed afterward. This matches the program's Medium impacts: increasing network processing node resource consumption by at least 30% without brute force actions, and griefing. Per the program's Code Freeze Assurance, a bypass of a public fix is a new, valid bug.

### Vulnerability Details

All references are to audit-comp-quantus-chain at the frozen commit 3b243b870a5442a6e5f443056f67bbba87d472f5. Registry crates are named with their exact published versions.

{% stepper %}
{% step %}

## The remote entrypoint is real, default, and requires nothing from the attacker

Litep2p is the default network backend (client/cli/src/params/network\_params.rs:174-186), and every full node registers an inbound light-request handler (client/service/src/builder.rs:1183-1194). The handler serves the chain-specific protocol //light/2 with a 1 MiB request limit and a 15-second protocol timeout (sc-network-light-0.54.0/src/light\_client\_requests.rs:31-64), holds one global 20-slot queue (light\_client\_requests/handler.rs:46-48), drains it serially in a single task (handler.rs:85-90), and for remote-call requests passes the peer-selected block hash, method name, and data bytes directly to client.execution\_proof with no runtime-method allowlist (handler.rs:168-188), which executes the method against retained state in an execution-proof overlay.

The attacker needs one ordinary TCP P2P connection and any retained block hash (public chain information). No account, key, funds, fee, PoW, validator or governance role, Sybil identity, or control over the victim's genesis configuration is required.
{% endstep %}

{% step %}

## Quantus exports the setup-only method in the production runtime and documents a trust model that the entrypoint falsifies

runtime/src/apis.rs:323-332 implements sp\_genesis\_builder::GenesisBuilder for the production Runtime. runtime/src/genesis\_config\_presets.rs:462-480 deliberately omits input-size limits on the stated premise that this API "is not reachable by network peers or on a running chain."

Step 1 shows that premise is false on the default configuration.
{% endstep %}

{% step %}

## The in-scope root cause: Quantus's count-only mitigation does not bound the actual work

After PR #616, pallets/balances/src/lib.rs:200-208 documents that dev\_accounts is attacker-controlled through the externally reachable GenesisBuilder::build\_state API and caps it at MAX\_DEV\_ACCOUNTS = 1\_000 because each derived account performs an expensive sr25519 derivation.

derive\_dev\_account (pallets/balances/src/lib.rs:1360-1386) validates only:

* num\_accounts > 0
* num\_accounts <= 1,000
* balance >= existential deposit
* derivation contains "{}"

There is no bound on template bytes, parsed path components, or the product num\_accounts \* junctions\_per\_account. The loop at lines 1388-1416 then repeats, for every permitted account: a full-string replace of "{}", a full SURI parse via SrPair::from\_string, and one sr25519 hard derivation per path junction.

The upstream parser (sp-core-39.0.0/src/address\_uri.rs:155-191) accepts path components until input exhaustion with no component cap, and sr25519 derivation (sp-core-39.0.0/src/sr25519.rs:239-256) folds over every junction with a schnorrkel hard derivation.

The exact request is balances.devAccounts = \[1000, EXISTENTIAL\_DEPOSIT, ("//a" repeated 10,921 times) + "//{}"] inside an otherwise complete, publicly constructible development RuntimeGenesisConfig JSON; after "{}" replacement each account's string contains 10,922 hard junctions, so 1,000 \* 10,922 = 10,922,000 hard derivation folds are requested by a 34,313-byte protobuf message.
{% endstep %}

{% step %}

## Relation to public fixes (duplicate boundary, disclosed prominently)

PR #616 publicly fixed this resource class with the count cap (public body says 10,000; the merged commit db94e2a1 lowered it to 1,000 because 10,000 single derivations still took seconds). This report does not resurrect counts above the cap: the count stays at exactly 1,000 and the bypass comes from a second attacker-controlled dimension the fix never inspects — per-account path depth.

The program's Code Freeze Assurance states that a bypass of a public fix is considered a new, valid bug. PR #617 repeats the count-cap claim with no path bound; PR #639 changes dev-account issuance/collision/error behavior without constraining path depth; PR #647 rejected an unbounded genesis-member finding on the premise that GenesisBuilder input is operator-only, which step 1 removes; PR #648 explicitly records "no input-size bound ... in the genesis builder" as an accepted limitation because the API is allegedly unreachable from peers or on a running chain — the PoC below falsifies that reachability premise on the default node.

The generic light dispatcher and SURI parser are upstream amplifiers and are not claimed as independently in-scope defects; the in-scope root is Quantus's production runtime export, its documented trust model, and its incomplete count-only bound in the modified vendored balances pallet.
{% endstep %}
{% endstepper %}

### Impact Details

Selected impacts (both Medium): Increasing network processing node resource consumption by at least 30% without brute force actions, compared to the preceding 24 hours; and Griefing.

Measured against an unmodified quantus-node built from the frozen commit, attacked over the real //light/2 protocol on loopback (full step-by-step in the PoC section):

| Case (accounts x hard junctions) | Protobuf request bytes | Direct Wasm call median | Real /light/2 median |
| -------------------------------- | ---------------------: | ----------------------: | -------------------: |
| 1x1                              |                  1,543 |                6.791 ms |            10.060 ms |
| 1000x1                           |                  1,546 |            2,714.143 ms |         2,772.529 ms |
| 1x10922                          |                 34,310 |              169.886 ms |           202.075 ms |
| 1000x10922                       |                 34,313 |          177,103.291 ms |       202,543.163 ms |

The two one-dimensional controls confirm that account count and path depth each scale the work independently, and the product case exercises the missing combined bound: a 22.24x wire-size increase over the 1x1 case produces a 26,079x runtime increase.

Process impact of the single heavy request (cumulative victim CPU time correlated with request timestamps and the handler log): sampled wall 202.665 s, victim CPU 189.950 s = 93.73% of one core, versus a stable-idle control of 0.87%; maximum RSS rose from 88,320 KiB (idle) to 123,808 KiB. An independent re-verification on the same commit and same binaries reproduced the heavy case end-to-end with 190.00 victim CPU-seconds and the server-side completion signature below. The victim moves from near-idle to sustained single-core saturation after one 34 KB request; at the measured service time, one request every \~3.4 minutes (≈169 request bytes/second averaged) keeps the handler continuously busy — the amplification comes from per-request cost asymmetry, not traffic volume, so it is not brute force.

Griefing / service denial against other users: the light-proof handler is node-global and serial. While one heavy request was executing, a normal 1x1 request from a second, independent peer — 10.060 ms at idle — timed out after 15,007 ms and only reached handler completion 187,847 ms after submission; the victim executed it 172,840 ms after its own caller had already timed out, proving queued work from other peers is retained and delayed by minutes. Canceling the request stream at 106 ms, dropping the whole attacker peer at 110 ms, or the 15-second protocol timeout did not stop server-side execution: the heavy request ran to completion 176,860 ms after the client-side timeout, with the node logging the terminal runtime assertion at the predicted source line.

Server-side completion is directly observable in the victim log at the exact predicted source location, after the full derivation loop:

```
ERROR runtime: panicked at pallets/balances/src/lib.rs:628:5: duplicate balances in genesis: endowed account collides with a dev account.
```

The assertion fires only after all 1,000 account derivations complete (the derivation loop at lines 609-622 runs before the endowed-account collision check at lines 623-630), and the 1000x1 real-P2P median (2,772.529 ms) closely tracks the direct Wasm median (2,714.143 ms), independently confirming the full loop executes on the live path. Execution happens in a discarded proof overlay; the attack needs CPU consumption and handler occupancy, not a successful state commit, so execution failure does not negate the impact.

Boundary of the claim (no overstatement): the node did not crash; all 786 RPC samples per method succeeded during the heavy window (system\_syncState p50/p99/max moved from 0.760/2.002/2.746 ms idle to 1.540/15.538/30.512 ms under attack — measurable degradation, no outage); a real one-block loopback sync completed during a separate heavy run in 81.868 ms versus 43.532 ms idle. No High/Critical impact (consensus halt, transaction freeze, 500% block delay, fund loss) is claimed, and the exact preceding-24-hour production baseline in the Medium wording was not measured on a deployed fleet; the raw matched-control measurements are reported as collected on the frozen commit.

### References

* client/cli/src/params/network\_params.rs:174-186; node/src/command.rs:672-683; client/service/src/builder.rs:1183-1194 — default litep2p backend and registration of the inbound light-request handler
* sc-network-light-0.54.0/src/light\_client\_requests.rs:31-64 and light\_client\_requests/handler.rs:46-48, 85-90, 168-188 — //light/2 protocol, 1 MiB request, 15 s timeout, one serial global queue, peer-selected method forwarded to execution\_proof without an allowlist
* runtime/src/apis.rs:323-332 — production GenesisBuilder::build\_state export
* runtime/src/genesis\_config\_presets.rs:462-480 — documented trust model ("not reachable by network peers or on a running chain")
* pallets/balances/src/lib.rs:200-208 (MAX\_DEV\_ACCOUNTS = 1\_000 and its attacker-controlled rationale), 1360-1386 (count-only validation), 1388-1416 (per-account SURI parse + derivation loop), 609-630 (derivation loop runs before the collision assertion at line 628)
* sp-core-39.0.0/src/address\_uri.rs:155-191 (uncapped path components) and sp-core-39.0.0/src/sr25519.rs:239-256 (hard derivation fold per junction) — upstream amplifiers
* Official PRs in the competition repository: #616 (public count-only fix that this report bypasses on a new input dimension), #617, #639, #647 (finding 181340 rejected on the operator-only-input premise), #648 (accepted limitation premised on peer unreachability, falsified here)

## Link to Proof of Concept

<https://gist.github.com/Jornason/a78b804de541b960ff86307b64d2348e>

## Proof of Concept

For Audit Competitions, step-by-step explanations are provided below; the complete reproducer source is in the linked secret Gist. Everything runs on loopback against a locally built node; no public network, bootnode, telemetry, or project infrastructure is contacted. Host used: macOS, Apple M1 Max, rustc 1.93.0; node and helpers are debug host builds, while the executed runtime code is the embedded optimized Wasm artifact, which is the production execution path for remote proof requests.

{% stepper %}
{% step %}

## Build the frozen node and the three reproducer files

```bash
git clone https://github.com/immunefi-team/audit-comp-quantus-chain
cd audit-comp-quantus-chain
git checkout 3b243b870a5442a6e5f443056f67bbba87d472f5
# copy the three files from the Gist into the tree:
#   poc_runtime.rs        -> node/src/bin/poc_runtime.rs
#   poc_emit_payload.rs   -> node/src/bin/poc_emit_payload.rs
#   poc_light_peer.rs     -> client/network/examples/poc_light_peer.rs
cargo build -p quantus-node --bin quantus-node --bin poc_runtime --bin poc_emit_payload
cargo build -p sc-network --example poc_light_peer
```

poc\_runtime builds the exact complete RuntimeGenesisConfig from Quantus's own development preset, changes only balances.devAccounts, and calls the embedded optimized Wasm GenesisBuilder\_build\_state directly (a differential control that isolates runtime work). poc\_emit\_payload writes the four exact request payloads as JSON. poc\_light\_peer independently encodes the protobuf light-client schema and speaks the production //light/2 protocol as an ordinary peer; it uses no internal node helper.
{% endstep %}

{% step %}

## Emit the four exact payloads

```bash
for case in 1x1 1000x1 1x10922 1000x10922; do
  target/debug/poc_emit_payload "$case" "payload_$case.json"
done
```

Expected sizes: 1x1 = 1,473 JSON bytes; 1000x1 = 1,476; 1x10922 = 34,236; 1000x10922 = 34,239. The only difference from the stock development preset is balances.devAccounts = \[count, existential\_deposit, template], with the template ("//a" repeated 10,921 times) + "//{}" in the deep-path cases.
{% endstep %}

{% step %}

## Direct Wasm control (isolates the runtime work)

```bash
/usr/bin/time -l target/debug/poc_runtime wasm 1000x10922   # and: 1x1, 1000x1, 1x10922
```

Measured medians (3 runs each for the small cases; the heavy case run once): 1x1 = 6.791 ms; 1000x1 = 2,714.143 ms; 1x10922 = 169.886 ms; 1000x10922 = 177,103.291 ms with hard\_derivations\_requested=10922000 printed by the harness, all returning result=ok. An independent re-run on the same commit measured the heavy case at 163,444 ms (172.5 s process real time).
{% endstep %}

{% step %}

## Start the loopback-only victim

```bash
target/debug/quantus-node \
  --chain dev --base-path /tmp/victim-base --name Victim \
  --listen-addr /ip4/127.0.0.1/tcp/30444 --rpc-port 19944 --rpc-methods unsafe \
  --reserved-only --allow-private-ip --no-mdns --no-telemetry --no-prometheus \
  -l 'info,light-client-request-handler=trace'
```

Then read the genesis hash:

```bash
curl -s http://127.0.0.1:19944 -H 'content-type: application/json' --data '{"jsonrpc":"2.0","id":1,"method":"chain_getBlockHash","params":[0]}'
```

Read the victim peer id from the "Local node identity is:" log line. On the tested commit the dev-chain genesis was 0x547b3f26f7b6b3c85bd7c17ca83354a6f8427e66e6d6d8e04cbe7abceb27d9c3.
{% endstep %}

{% step %}

## Send the real P2P requests as an ordinary peer

```bash
VICTIM=/ip4/127.0.0.1/tcp/30444/p2p/<victim-peer-id>
GENESIS=0x547b3f26f7b6b3c85bd7c17ca83354a6f8427e66e6d6d8e04cbe7abceb27d9c3
target/debug/examples/poc_light_peer "$VICTIM" "$GENESIS" "$GENESIS" payload_1x1.json await 30 0 0
target/debug/examples/poc_light_peer "$VICTIM" "$GENESIS" "$GENESIS" payload_1000x1.json await 30 0 0
target/debug/examples/poc_light_peer "$VICTIM" "$GENESIS" "$GENESIS" payload_1x10922.json await 30 0 0
target/debug/examples/poc_light_peer "$VICTIM" "$GENESIS" "$GENESIS" payload_1000x10922.json await 300 0 0
```

The helper prints the exact wire shape: protocol=//light/2, method GenesisBuilder\_build\_state, data = SCALE(Vec(complete genesis JSON)); the heavy protobuf request is 34,313 bytes.

Measured real-P2P medians (4 runs each for the small cases): 1x1 = 10.060 ms; 1000x1 = 2,772.529 ms; 1x10922 = 202.075 ms — each returns a response with proof=None because the live-state build ends in the expected terminal assertion after the derivation loop. The heavy case occupied the victim handler for 202,543.163 ms; the client observes the connection close only around handler completion.
{% endstep %}

{% step %}

## Measure the victim CPU cost of the heavy request

Sample the victim once per second while the heavy request runs:

```bash
VICTIM_PID=$(pgrep -f 'quantus-node.*30444')
( while kill -0 $VICTIM_PID 2>/dev/null; do ps -o time=,rss= -p $VICTIM_PID; sleep 1; done ) > victim_samples.txt
```

The cumulative CPU-time delta across the request window is the load-independent cost. Measured: 189.950 victim CPU-seconds over the 202.665 s sampled window (93.73% of one core; stable-idle control 0.87%), maximum RSS 123,808 KiB versus 88,320 KiB idle. The independent re-verification reproduced 190.00 CPU-seconds for the same request.

The victim log simultaneously shows the terminal assertion at the predicted source line, proving the full 1,000-account loop executed:

```
ERROR runtime: panicked at pallets/balances/src/lib.rs:628:5: duplicate balances in genesis: endowed account collides with a dev account.
```

{% endstep %}

{% step %}

## Show that cancel, disconnect, and timeout do not stop the work

```bash
# cancel only the request stream 100 ms after sending:
target/debug/examples/poc_light_peer "$VICTIM" "$GENESIS" "$GENESIS" payload_1000x1.json cancel-stream 300 100 5
# drop the whole attacker peer 100 ms after sending:
target/debug/examples/poc_light_peer "$VICTIM" "$GENESIS" "$GENESIS" payload_1000x1.json drop-peer 300 100 0
# heavy request with the 15 s protocol timeout, then keep observing for 210 s:
target/debug/examples/poc_light_peer "$VICTIM" "$GENESIS" "$GENESIS" payload_1000x10922.json timeout 15 0 210
```

Observed with the handler trace enabled: the 1000x1 requests ran to handler completion 2,699-2,698 ms after the cancel/drop actions; the heavy request completed 176,860 ms after its 15-second client-side timeout. Additionally, a benign 1x1 request submitted by a second peer during a heavy request timed out at 15,007 ms yet was still executed by the victim, reaching handler completion 187,847 ms after submission — the global serial queue retains and executes work whose caller is already gone.
{% endstep %}
{% endstepper %}

### Expected results summary

| Measurement                                 | Result                                                |
| ------------------------------------------- | ----------------------------------------------------- |
| Heavy request size                          | 34,313 protobuf bytes                                 |
| Hard sr25519 derivations requested          | 10,922,000 (1,000 accounts x 10,922 junctions)        |
| Handler occupancy per request               | 202.543 s (direct Wasm control 177.103 s)             |
| Victim CPU per request                      | 189.95 CPU-seconds (independent re-run: 190.00)       |
| Idle vs attacked CPU                        | 0.87% -> 93.73% of one core                           |
| Attack rate for continuous occupancy        | 1 request / \~3.4 min (\~169 bytes/s)                 |
| Benign second-peer 1x1 during heavy request | 15 s client timeout; handler completion at 187,847 ms |
| Cancel / drop / timeout effects             | work continues to completion in all cases             |
| Node/RPC/consensus failure                  | none observed and none claimed                        |

### Suggested fix

Parse the SURI template once before the account loop and reject templates over small byte and junction-count limits; enforce a checked combined budget such as num\_accounts \* (root\_cost + junction\_count) <= MAX\_DEV\_DERIVATION\_WORK; preferably remove template freedom entirely and use a fixed development derivation prefix with only the numeric index variable. At the proof boundary, deny setup/benchmark/try-runtime exports in remote proof calls via an explicit allowlist of proof-safe methods; skip queued requests whose response receiver is already closed and add an execution budget for in-progress calls; add per-peer admission/rate limits with a reputation consequence for repeated rejected high-cost methods.
