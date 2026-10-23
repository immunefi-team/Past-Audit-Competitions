# 74569 bc medium base consensus client process exits on safe invalid payloads because is deposits only misclassifies every derived block and bypasses holocene fallback

Submitted on Apr 23rd 2026 at 14:14:07 UTC by @coffee\_boi for [Audit Comp | Base Azul](https://immunefi.com/audit-competition/audit-comp-base-azul)

* **Report ID:** #74569
* **Report Type:** Blockchain/DLT
* **Report severity:** Medium
* **Target:** <https://github.com/base/base/releases/tag/v0.8.0-rc.15>
* **Impacts:**
  * Network not being able to confirm new transactions (total network shutdown)

## Description

### Summary

`AttributesWithParent::is_deposits_only` (`crates/consensus/protocol/src/attributes.rs:65-70`) calls `.iter()` on an `Option<Vec<Bytes>>`, iterating the `Option` wrapper instead of the inner `Vec`. The closure then checks only `transactions[0]`; every tx after the first is ignored.

Every derived L2 block begins with the L1-info deposit (`crates/consensus/derive/src/attributes/stateful.rs:184-186`), so the predicate returns `true` for every normal payload including payloads carrying user transactions.

The predicate gates a match arm at `crates/consensus/engine/src/task_queue/tasks/seal/task.rs:160-192` that precedes the Holocene deposits-only fallback. When the EL returns `PayloadStatusEnum::Invalid`, the fatal arm is always taken, the fallback never runs, and `SealTaskError::DepositOnlyPayloadFailed` propagates to `std::process::exit(1)`. The node does not degrade and the process terminates.

In a homogeneous Base fleet, the same L1-derived input drives every node through the same deterministic branch. The result is consensus-layer process exit across the fleet until operators patch and redeploy. In mixed-client environments, spec-conformant peers continue via the deposits-only fallback while Base Rust halts, creating chain-split conditions.

### Root cause

```rust
// crates/consensus/protocol/src/attributes.rs:65-70
pub fn is_deposits_only(&self) -> bool {
    self.attributes
        .transactions
        .iter()                                     // Option<Vec<Bytes>>::iter — yields &Vec<Bytes> once
        .all(|tx| tx.first().is_some_and(|tx| tx[0] == OpTxType::Deposit as u8))
        //        ^^^^^^^^^^^ tx is &Vec<Bytes>; tx.first() is the first Bytes only
}
```

Effective check: *"`transactions` is `None`, or `transactions[0][0] == Deposit`."* The only case where this disagrees with a correct implementation is `[deposit, user_tx, …]`, the shape of every normal L2 block.

### Fatal-reachability chain (process exit, not degradation)

| # | File:line                                                                | Effect                                                             |
| - | ------------------------------------------------------------------------ | ------------------------------------------------------------------ |
| 1 | `derive/src/attributes/stateful.rs:184-186`                              | Derivation always produces `[L1-info deposit, …]`.                 |
| 2 | `protocol/src/attributes.rs:65-70`                                       | `is_deposits_only()` returns `true` on mixed batches.              |
| 3 | `engine/src/task_queue/tasks/seal/task.rs:160-192`                       | Match arm A fires before Holocene-fallback arm B — arm B is dead.  |
| 4 | `engine/src/task_queue/tasks/seal/error.rs:81-85, 97-101`                | `DepositOnlyPayloadFailed` → `Critical`, `is_fatal = true`.        |
| 5 | `engine/src/task_queue/tasks/task.rs:265-268`                            | Critical severity returns from the engine driver.                  |
| 6 | `service/src/actors/engine/engine_request_processor.rs:164-173, 514-516` | Processor task completes with error.                               |
| 7 | `service/src/actors/engine/actor.rs:62-86`                               | Error escapes the actor boundary.                                  |
| 8 | `service/src/service/util.rs:46-72`                                      | `spawn_and_wait!` cancels all sibling actors.                      |
| 9 | `service/src/service/node.rs:285-290`, `bin/consensus/src/main.rs:14-16` | `RollupNode::start()` returns error → **`std::process::exit(1)`**. |

There is no in-process recovery: the Critical branch returns rather than looping. A supervisor that respawns the binary re-hits the same L1-derived input and re-triggers the same exit. Every Base node in a homogeneous fleet evaluates the predicate identically and exits simultaneously.

### Impact explanation

* **Critical - " Network not being able to confirm new transactions (total network shutdown)."** One natural L1-derived input halts the entire Base fleet. Recovery requires patching the binary and coordinating a fleet-wide redeploy.
* The Holocene spec mandates deposits-only replacement on `INVALID`; op-node Go conforms, Base Rust does not. Any spec-conformant peer advances while Base halts. Holocene is live on every production Base network and has been for 15+ months.

### Likelihood explanation

The EL returns `Invalid` for derivation payloads in exactly the scenarios Holocene was introduced to handle:

* L1 reorg shifts an L2 block's origin, making a previously-valid user tx invalid under re-derivation (nonce/balance/storage drift).
* Post-Holocene Interop dependency invalidation.
* Sequencer/verifier nondeterminism or hardfork-boundary edge cases.

Base's own source documents the deposits-only replacement as the designed remediation at `service/src/actors/engine/engine_request_processor.rs:180-183`:

> `// This error is encountered when the payload is marked INVALID by the engine api. Post-holocene, the payload is replaced by a "deposits-only" block and re-executed.`

The bug disables the remediation the authors themselves documented as load-bearing.

### Recommendation

```rust
pub fn is_deposits_only(&self) -> bool {
    self.attributes
        .transactions
        .iter()
        .flatten()
        .all(|tx| tx.first().copied() == Some(OpTxType::Deposit as u8))
}
```

Plus:

1. Audit other `Option<Vec<_>>::iter()` sites; clippy misses this idiom.
2. Reconsider `DepositOnlyPayloadFailed` at `Critical + is_fatal`. Gating `std::process::exit(1)` on a single predicate is fragile even with the predicate correct.

### Additional Notes

The fatal path is not gated on privileged or authenticated input. Any sequencer-built or batcher-supplied block whose user txs re-execute invalidly under derivation reaches it. Upstream input validation does not limit blast radius - the bug is in the response logic.

## Proof of Concept

### A. Predicate (4 tests, `crates/consensus/protocol/src/attributes.rs`)

```rust
#[test]
fn is_deposits_only_bug_returns_true_on_realistic_mixed_batch() {
    let attributes = BasePayloadAttributes {
        transactions: Some(vec![
            vec![OpTxType::Deposit as u8, 0x00, 0x01].into(), // L1-info deposit
            vec![OpTxType::Eip1559 as u8, 0x00, 0x02].into(), // user tx
        ]),
        ..BasePayloadAttributes::default()
    };
    let awp = AttributesWithParent::new(attributes, L2BlockInfo::default(), None, true);

    assert!(awp.is_deposits_only(), "buggy behavior — this PASSES on broken code");
    assert!(!correct_is_deposits_only(&awp.attributes.transactions));
}

fn correct_is_deposits_only(txs: &Option<Vec<alloy_primitives::Bytes>>) -> bool {
    txs.iter().flatten()
        .all(|tx| tx.first().copied() == Some(OpTxType::Deposit as u8))
}
```

Three additional tests cover: multiple deposits followed by one user tx, all-deposits agreement with correct impl, and first-tx-is-user agreement with correct impl.

### B. Fatal-arm routing (`crates/consensus/engine/src/task_queue/tasks/seal/task_test.rs`)

Mirrors the match at `seal/task.rs:160-192` using the real `is_deposits_only`, real `is_holocene_active`, and real `SealTaskError::severity()`. Arm B (Holocene fallback) is wired to `panic!` — any fix immediately fails the test:

```rust
let sealed: Result<(), SealTaskError> = match insert_result {
    Err(InsertTaskError::UnexpectedPayloadStatus(_)) if awp.is_deposits_only() => {
        Err(SealTaskError::DepositOnlyPayloadFailed)
    }
    Err(InsertTaskError::UnexpectedPayloadStatus(_))
        if cfg.is_holocene_active(awp.attributes.timestamp) =>
    {
        panic!("Holocene fallback reached — bug is fixed or predicate lies");
    }
    _ => unreachable!(),
};

let err = sealed.unwrap_err();
assert!(matches!(err, SealTaskError::DepositOnlyPayloadFailed));
assert_eq!(err.severity(), EngineTaskErrorSeverity::Critical);
assert!(err.is_fatal());
```

Run:

```bash
cargo test -p base-protocol --lib is_deposits_only
cargo test -p base-consensus-engine --lib task_queue::tasks::seal::task_test
```

Both suites pass on current `main`. Tier A + B prove steps 1–4 of the fatal chain. Steps 5–9 are direct source citations.

### C. Live devnet verifier exit (end-to-end)

The deterministic live PoC is to force the EL to return `PayloadStatusEnum::Invalid` **only when the verifier is importing a safe mixed payload**. This does **not** create the bug; it only makes the natural Holocene trigger deterministic. The bug is the control flow *after* `Invalid`: Base takes the fatal `is_deposits_only()` branch instead of the deposits-only fallback.

#### Minimal PoC patch

Add a temporary env-gated injector in `crates/consensus/engine/src/task_queue/tasks/insert/task.rs`:

```rust
const INVALID_MIXED_PAYLOAD_POC_ENV: &str = "BASE_POC_INJECT_INVALID_MIXED_PAYLOAD";

fn maybe_inject_invalid_mixed_payload(&self) -> Option<PayloadStatus> {
    if !self.is_payload_safe
        || std::env::var_os(INVALID_MIXED_PAYLOAD_POC_ENV).is_none()
        || !Self::is_mixed_deposit_payload(&self.envelope.execution_payload)
    {
        return None;
    }

    warn!(
        target: "engine",
        block_number = self.envelope.execution_payload.block_number(),
        tx_count = self.envelope.execution_payload.transactions().len(),
        "Injecting PoC invalid payload status for safe mixed deposit payload"
    );

    Some(PayloadStatus {
        status: PayloadStatusEnum::Invalid {
            validation_error: "PoC injected invalid payload status".to_owned(),
        },
        latest_valid_hash: None,
    })
}
```

Pass the env only to `base-client-cl`, and disable restart so Docker preserves the non-zero exit:

```yaml
base-client-cl:
  environment:
    - BASE_POC_INJECT_INVALID_MIXED_PAYLOAD
  restart: ${BASE_POC_RESTART_POLICY:-unless-stopped}
```

#### Reproduction steps

{% stepper %}
{% step %}

## Start the single-node devnet

```bash
just devnet up-single
```

{% endstep %}

{% step %}

## Create a PoC env file

```bash
cp etc/docker/devnet-env /tmp/devnet-env-poc
cat >> /tmp/devnet-env-poc <<'EOF'
BASE_POC_INJECT_INVALID_MIXED_PAYLOAD=1
BASE_POC_RESTART_POLICY=no
EOF
```

{% endstep %}

{% step %}

## Rebuild only the verifier consensus container with the PoC env

```bash
docker compose --env-file /tmp/devnet-env-poc \
  -f etc/docker/docker-compose.yml \
  up -d --no-deps --build base-client-cl
```

> Note: the injector at `crates/consensus/engine/src/task_queue/tasks/insert/task.rs` gates on `std::env::var_os(INVALID_MIXED_PAYLOAD_POC_ENV)` at **runtime**, not build time. The compose snippet above passes the variable into the container via `environment: - BASE_POC_INJECT_INVALID_MIXED_PAYLOAD` (value interpolated from `--env-file`). If a reviewer reproduces these steps from a shell that does not load `/tmp/devnet-env-poc`, or drops the `--env-file` flag, the variable will not reach the container runtime and the injector will no-op — the verifier will keep running and the PoC will appear to fail. The `--env-file` flag must be present on every `docker compose` invocation that (re)creates `base-client-cl`.
> {% endstep %}

{% step %}

## Disconnect the verifier from its only unsafe CL peer

```bash
PEER_ID=$(curl -s http://127.0.0.1:8549 \
  -H 'content-type: application/json' \
  --data '{"jsonrpc":"2.0","id":1,"method":"opp2p_peers","params":[true]}' \
  | jq -r '.result.peers | keys[0]')

curl -s http://127.0.0.1:8549 \
  -H 'content-type: application/json' \
  --data "{\"jsonrpc\":\"2.0\",\"id\":1,\"method\":\"opp2p_blockPeer\",\"params\":[\"$PEER_ID\"]}"
```

Sanity check:

```bash
curl -s http://127.0.0.1:8549 \
  -H 'content-type: application/json' \
  --data '{"jsonrpc":"2.0","id":1,"method":"opp2p_peerStats","params":[]}' \
  | jq '.result'
```

Expected result: `connected = 0`, `banned = 1`.
{% endstep %}

{% step %}

## Send user transactions to the builder

```bash
cast send --rpc-url http://127.0.0.1:7545 \
  --private-key 0x59c6995e998f97a5a0044966f0945389dc9e86dae88c7a8412f4603b6b78690d \
  0x3C44CdDdB6a900fa2b585dd299e03d12FA4293BC \
  --value 1000000000000000

cast send --rpc-url http://127.0.0.1:7545 \
  --private-key 0x5de4111afa1a4b94908f83103eb1f1706367c2e68ca870fc3fb9a804cdab365a \
  0x90F79bf6EB2c4f870365E785982E1f101E93b906 \
  --value 1000000000000001
```

{% endstep %}

{% step %}

## Watch the verifier logs

```bash
docker compose --env-file /tmp/devnet-env-poc \
  -f etc/docker/docker-compose.yml \
  logs -f base-client-cl
```

{% endstep %}

{% step %}

## Confirm the verifier exits non-zero

```bash
docker inspect -f 'status={{.State.Status}} exit={{.State.ExitCode}} finished={{.State.FinishedAt}} restarted={{.RestartCount}}' base-client-cl
docker compose --env-file /tmp/devnet-env-poc -f etc/docker/docker-compose.yml ps -a base-client-cl
```

{% endstep %}
{% endstepper %}

#### Observed result on 2026-04-23

Safe derivation eventually produced a mixed payload:

```
generated attributes in payload queue txs=1 timestamp=1776951751
```

That `txs=1` user-batch became a 2-transaction execution payload once the mandatory L1-info deposit was prepended (`tx_count=2` in the next log). The PoC injector then forced `Invalid` **only** for that safe mixed payload:

```
2026-04-23T13:43:43.871951Z WARN  engine: Injecting PoC invalid payload status for safe mixed deposit payload block_number=1617 tx_count=2
```

Base immediately took the fatal branch instead of the Holocene fallback:

```
2026-04-23T13:43:43.872024Z ERROR engine: Critical: Deposit-only payload import failed error=Invalid { validation_error: "PoC injected invalid payload status" }
2026-04-23T13:43:43.872101Z ERROR engine: Critical engine error error=Deposit-only payload failed to import
2026-04-23T13:43:43.872131Z ERROR engine: Critical error draining engine tasks err=Consolidate(SealTaskFailed(DepositOnlyPayloadFailed))
2026-04-23T13:43:43.872245Z ERROR engine: Failed to drain engine tasks err=EngineTask(Consolidate(SealTaskFailed(DepositOnlyPayloadFailed)))
```

The verifier process then exited non-zero and stayed down because restart was disabled:

```
status=exited exit=1 finished=2026-04-23T13:43:58.601936462Z restarted=0

NAME             IMAGE                  COMMAND                  SERVICE          CREATED          STATUS
base-client-cl   base-consensus:local   "/app/base-consensus…"   base-client-cl   19 minutes ago   Exited (1)
```

#### Why this is sufficient

* The live run reaches the exact buggy control-flow decision under a real devnet verifier.
* The only artificial input is the deterministic `Invalid` response, which merely substitutes for the natural EL rejection Holocene was explicitly designed to handle.
* The bug itself is unchanged: a safe mixed payload still routes to `DepositOnlyPayloadFailed` instead of the deposits-only fallback.
* The terminal effect is demonstrated, not inferred: the verifier container exits with code `1`.
