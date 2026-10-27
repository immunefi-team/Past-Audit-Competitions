# 76368 bc low challenger awaitingproof phase has no timeout

**Submitted on May 4th 2026 at 02:54:55 UTC by @Venator for** [**Audit Comp | Base Azul**](https://immunefi.com/audit-competition/audit-comp-base-azul)

* **Report ID:** #76368
* **Report Type:** Blockchain/DLT
* **Report severity:** Low
* **Target:** <https://github.com/base/base/tree/v0.8.0-rc.28>
* **Impacts:**
  * A bug in the respective layer 0/1/2 network code that results in unintended smart contract behavior with no concrete funds at direct risk

## Description

**Program:** Base Azul Audit Competition\
**Platform:** Immunefi\
**Severity:** **Medium** -- ISIS *"A bug in the respective layer 0/1/2 network code that results in unintended smart contract behavior with no concrete funds at direct risk."* A single deterministic trigger (a future ZK-prover proto status-code addition) simultaneously stalls every challenger instance running the pre-bump Rust client, leaving any invalid proposal during the rebuild window non-challengeable fleet-wide. Chains to Critical bridge-drain via the same mechanism as `IMMUNEFI_BASE_AZUL_CHALLENGER_STALE_INTERVAL_CACHE` (already filed).

**Asset:** `github.com/base/base` -- `crates/proof/challenge/`, `crates/proof/zk/client/`\
**Commit:** `main` HEAD on 2026-04-22

***

## Summary

`PendingProofs::poll` (`crates/proof/challenge/src/pending.rs:241-298`) advances each in-flight ZK-proof session by querying the ZK-prover gRPC service and dispatching on `GetProofResponse.status`. The dispatch has two terminal arms (`Succeeded → Ready`, `Failed → NeedsRetry`) and a single `_ => ProofUpdate::Pending` catch-all:

```rust
let status = ProofJobStatus::try_from(response.status).unwrap_or_else(|_| {
    warn!(raw_status = response.status, game = %game, "unrecognized proof job status");
    ProofJobStatus::Unspecified
});
// ...
let update = match status {
    ProofJobStatus::Succeeded => { /* ... */ ProofUpdate::Ready }
    ProofJobStatus::Failed    => { pending.retry_count += 1; /* ... */ ProofUpdate::NeedsRetry }
    _                         => ProofUpdate::Pending,
};
```

`ProofUpdate::Pending` is a terminal state for the driver: `poll_or_submit` (`driver.rs:699-707`) matches on `Some(ProofUpdate::Pending)` and returns `Ok(())` early, **without advancing `retry_count`, without transitioning the `ProofPhase`, and without dropping the entry**. There is no timeout on `AwaitingProof`.

Because `process_candidate` (`driver.rs:265-268`) short-circuits any game whose proxy is already in `pending_proofs`:

```rust
if self.pending_proofs.contains_key(&game_address) {
    debug!(game = %game_address, "skipping game with pending proof session");
    return Ok(());
}
```

a game whose ZK-prover session persistently returns the `Pending`-catch-all arm is **permanently excluded from the validation/submission pipeline until the challenger process restarts**. The candidate game is non-challengeable for that challenger instance for the duration of the process lifetime.

## Scope of the `_ => ProofUpdate::Pending` arm

From `crates/proof/zk/client/proto/zk_prover.proto:53-66`:

```proto
message GetProofResponse {
  enum Status {
    STATUS_UNSPECIFIED = 0;
    STATUS_CREATED     = 1;
    STATUS_PENDING     = 2;
    STATUS_RUNNING     = 3;
    STATUS_SUCCEEDED   = 4;
    STATUS_FAILED      = 5;
  }
  Status status = 1;
  bytes receipt = 2;
  optional string error_message = 3;
}
```

The `_` arm at `pending.rs:294` catches:

1. **`STATUS_UNSPECIFIED = 0`** -- the protobuf default. A response that omits the `status` field, a server bug that forgets to set it, a session-unknown condition the server encodes as `0`, or a deliberately crafted response from a compromised ZK-prover service.
2. **`STATUS_CREATED` / `STATUS_PENDING` / `STATUS_RUNNING`** -- the three legitimate intermediate states.
3. **Any future status code** (e.g. a hypothetical `CANCELLED`, `TIMEOUT`, `EXPIRED`) added to the proto before the Rust client is rebuilt. `try_from(response.status)` returns `Err(_)`, which is re-mapped to `Unspecified` by the `unwrap_or_else` fallback at `pending.rs:265-268`, and then falls into the same `_` arm.

Only states (4) and (5) ever exit `AwaitingProof`.

## Related gap: gRPC `get_proof` errors also do not advance `retry_count`

`pending.rs:264`:

```rust
let response = zk_prover.get_proof(request).await?;
```

The `?` propagates transport-/gRPC-level errors up through `poll_or_submit`. At the driver level (`driver.rs:244-255`) these errors are logged and swallowed, with the pending entry left untouched. So a server that *rejects* the `get_proof` call -- for a session the server has forgotten or during a sustained upstream outage -- has the same effect as the `UNSPECIFIED` case: the entry stays in `AwaitingProof` indefinitely, the candidate game stays in `pending_proofs`, and `process_candidate` keeps short-circuiting on every scan tick.

## Exploit scenarios

### 5.1 Benign operator-side trigger

ZK-prover services are remote and may crash, lose their session store, or be restarted by infrastructure (deployment, OOM kill, etc.). When the session is forgotten, the natural response to `GetProof(session_id)` is one of: a gRPC error, a response with `status = STATUS_UNSPECIFIED`, or a response with `status` unset (protobuf default = `UNSPECIFIED`). All three map to `ProofUpdate::Pending` in the challenger. The challenger stalls on that game until restart. Any invalid proposal scanned during the same session is non-challengeable.

### 5.2 Adversarial trigger (compromised or shared ZK-prover service)

If an attacker controls or compromises the ZK prover service the challenger connects to (multi-tenant service, DNS hijack of the configured `zk-rpc-url`, compromised mTLS certificate, etc.), the attacker can deterministically return `status = STATUS_UNSPECIFIED` for every `get_proof` query by an attacker-selected challenger, nullifying that challenger's ability to process any invalid proposal. This fits the program scope impact *"Circumventing the dispute/challenge mechanism to prevent correction of an invalid proposal before finalization."*

### 5.3 Forward-compatibility trigger

When the ZK-prover protocol is extended (e.g. adding `STATUS_CANCELLED` or `STATUS_TIMEOUT` to the proto), every challenger built before the Rust client is updated will map the new code to `Unspecified → Pending` via the `try_from(...).unwrap_or_else(...)` fallback at `pending.rs:265-268`. A fleet-wide stall is possible on the next protocol upgrade unless every challenger instance is rebuilt and redeployed simultaneously, which is not the default operational posture on a live chain.

## Severity rationale -- Medium

The severity claim rests on the fleet-correlated forward-compatibility stall described in §5.3.

### Fleet-correlated stall via protocol evolution (§5.3)

* **Uniform staleness at a single deterministic trigger.** The day the ZK-prover gRPC service deploys a new status code (a hypothetical `STATUS_CANCELLED = 6` or `STATUS_TIMEOUT = 7`; protocol evolution is a standard upgrade event), every challenger instance running a pre-bump Rust client simultaneously maps that code to `Unspecified → Pending`.
* **Single binary semantics.** Every challenger in the fleet runs the same compiled code with the same `match` arms. There is no instance-level variation that breaks the correlation.
* **Zero graceful-failure response.** Normal software versioning would produce either a surfaced deserialization error (caught upstream by a retry path) or a bounded-timeout fallback. This code path does neither: the deserialization error is silently re-mapped to `Unspecified`, and the timeout path does not exist.
* **Impact mirrors `CHALLENGER_STALE_INTERVAL_CACHE`.** The resulting failure mode is an indefinite stall on every affected game on every fleet instance simultaneously. This is exactly the failure-correlation pattern the optimistic protocol design assumes away. A single invalid proposal during the rebuild window from any authorized prover is non-challengeable fleet-wide.

This maps to ISIS *"A bug in the respective layer 0/1/2 network code that results in unintended smart contract behavior with no concrete funds at direct risk"* (**Medium**) and chains to the Critical bridge-drain impact through the same mechanism as `IMMUNEFI_BASE_AZUL_CHALLENGER_STALE_INTERVAL_CACHE`.

### Why the fleet-tolerance rebuttal does not apply

The redundant-challenger-fleet argument ("one challenger stalling on a bad ZK session is absorbed by the other instances") is valid for §5.1 (benign operator-side trigger) and §5.2 (adversarial ZK service), both of which are per-challenger, per-session events. It does not hold for §5.3: the trigger is a single deterministic event (a proto version bump) that hits every instance simultaneously because they all run the same compiled binary with the same `match` arms. The fleet design cannot absorb a fleet-wide correlated stall.

### Why High is not claimed

The §5.3 trigger is a ZK-prover protocol bump, an off-chain coordinated action under the operator's control. The preconditions for exploitation (an attacker submits an invalid proposal during the exact window between proto bump and challenger rebuild) are narrower, and the defense (pin the ZK-prover proto version across deployments and rebuild challengers as part of the same release window) is straightforward operational practice.

### Program-specific downgrade rules considered

* *"Downgrade valid reports if the attack requires compromising Base-operated infrastructure (not discoverable via code review alone)."* The defect is discoverable via code review alone. The Medium claim (§5.3) does not require infrastructure compromise -- it requires only a scheduled protocol-evolution event.
* *"Any report that assumes a service/program will not be manually restarted with possibly different configurations will be downgraded."* Does not apply. Restarting the challenger is what *clears* the stall; the report does not assume the challenger will not be restarted. The defect is that the stall is triggered fleet-wide simultaneously by a proto bump, affecting every instance regardless of restart cadence.
* *"Best practice recommendations"* (out of scope). Does not apply: a fleet-correlated stall on a planned protocol-evolution event is a concrete failure mode, not a recommendation to adopt better practices.

## Supporting artifacts

* **Secret GitHub gist:** <https://gist.github.com/MattHintz/1d826ff02cc19eeee87b38325a1d93c1>
  * `POC_STALLED_PROOF.md` -- step-by-step PoC walkthrough with code references
  * `stalled_proof.rs` -- runnable single-file Rust PoC, stdlib only
  * `stalled_proof_transcript.txt` -- captured run output
  * `README.md` -- vulnerability summary, build instructions, scenario table

## Source references

All repo-relative to `github.com/base/base`, commit `main` HEAD on 2026-04-22:

* `crates/proof/challenge/src/pending.rs:241-298` -- `PendingProofs::poll`
* `crates/proof/challenge/src/pending.rs:265-268` -- `try_from(...).unwrap_or_else(... Unspecified)` fallback
* `crates/proof/challenge/src/pending.rs:294` -- the `_ => ProofUpdate::Pending` catch-all
* `crates/proof/challenge/src/driver.rs:265-268` -- `process_candidate` skip-if-pending
* `crates/proof/challenge/src/driver.rs:244-255` -- gRPC error log-and-swallow path
* `crates/proof/challenge/src/driver.rs:699-707` -- `poll_or_submit` short-circuit on `Pending`
* `crates/proof/zk/client/proto/zk_prover.proto:53-66` -- the `GetProofResponse.Status` enum

## Suggested fix

One of (or, ideally, both):

1. **Add a wall-clock timeout to `AwaitingProof`.** Store the monotonic time at which the proof was initiated; in `poll_or_submit`, treat any `AwaitingProof` entry older than `2 × zk_request_timeout × MAX_PROOF_RETRIES` (or an explicit `max_proof_wall_time` config knob) as `NeedsRetry`. This bounds the stall without making any assumption about the ZK-prover protocol.
2. **Classify `UNSPECIFIED` explicitly and remove the unknown-code re-map.** Replace `_ => ProofUpdate::Pending` with an explicit enumeration:

   ```rust
   let update = match status {
       ProofJobStatus::Succeeded => { /* ... */ }
       ProofJobStatus::Failed    => { /* ... */ }
       ProofJobStatus::Created | ProofJobStatus::Pending | ProofJobStatus::Running
           => ProofUpdate::Pending,
       ProofJobStatus::Unspecified => {
           // Treat UNSPECIFIED as a retryable failure: it is the protobuf
           // default and should never be a legitimate terminal state.
           pending.retry_count += 1;
           pending.phase = ProofPhase::NeedsRetry;
           ProofUpdate::NeedsRetry
       }
   };
   ```

   Combined with (1) so a cooperative server that reports an intermediate status but never progresses (`CREATED` forever) also gets bounded, and a future unknown status code surfaces the deserialization error to the retry-and-eventually-drop path instead of being silently re-mapped to `Unspecified`.

## Disclosure

Discovered 2026-04-22 during the Immunefi Base Azul Audit Competition. Disclosed via this submission. No challenger has been operated against any Base-operated production endpoint as part of this report; the PoC is a standalone code-equivalent harness that exercises the same dispatch logic in isolation.

## Link to Proof of Concept

<https://gist.github.com/MattHintz/1d826ff02cc19eeee87b38325a1d93c1>

## Proof of Concept

**PoC: Base Azul Challenger `AwaitingProof` Timeout-less Stall**

### What this proves

The challenger's `PendingProofs::poll` status dispatch treats any non-`SUCCEEDED`/non-`FAILED` ZK-prover gRPC response as `ProofUpdate::Pending`, a state that never advances `retry_count`, never transitions the proof phase, and never drops the entry. Combined with the absence of a wall-clock timeout on `AwaitingProof` and the `process_candidate` skip-if-pending short-circuit, the affected game is permanently non-challengeable for the lifetime of the challenger process.

### Boundary statement

Calling `poll_dispatch(wire_status)` with any wire value other than 4 (`SUCCEEDED`) or 5 (`FAILED`) returns `ProofUpdate::Pending`. The driver matches on `Pending` and returns `Ok(())` early without advancing `retry_count`. There is no timeout. The entry persists in `pending_proofs` indefinitely, and `process_candidate` skips the game on every subsequent scan tick.

### Target

* **Repo:** `github.com/base/base`
* **Commit:** `main` HEAD on 2026-04-22
* **Files:**
  * `crates/proof/challenge/src/pending.rs:241-298` -- `PendingProofs::poll`
  * `crates/proof/challenge/src/pending.rs:265-268` -- `try_from(...).unwrap_or_else(... Unspecified)` fallback
  * `crates/proof/challenge/src/pending.rs:294` -- `_ => ProofUpdate::Pending` catch-all
  * `crates/proof/challenge/src/driver.rs:265-268` -- `process_candidate` skip-if-pending
  * `crates/proof/challenge/src/driver.rs:699-707` -- `poll_or_submit` short-circuit on `Pending`

### Step-by-step attack walkthrough

{% stepper %}
{% step %}

## Step 1 -- ZK-prover returns a non-terminal status

The ZK-prover gRPC service returns `GetProofResponse` with a `status` field that is not `STATUS_SUCCEEDED (4)` or `STATUS_FAILED (5)`. This includes:

* `STATUS_UNSPECIFIED (0)` -- the protobuf default when the field is omitted, the server forgets to set it, or the session is unknown.
* `STATUS_CREATED (1)`, `STATUS_PENDING (2)`, `STATUS_RUNNING (3)` -- legitimate intermediate states that never transition.
* Any future status code (e.g. `CANCELLED = 6`, `TIMEOUT = 7`) added to the proto before the Rust client is rebuilt; `try_from()` returns `Err(_)`, re-mapped to `Unspecified` at `pending.rs:265-268`.
  {% endstep %}

{% step %}

## Step 2 -- Dispatch maps to `ProofUpdate::Pending`

At `pending.rs:276-295`:

```rust
let update = match status {
    ProofJobStatus::Succeeded => { /* ... */ ProofUpdate::Ready }
    ProofJobStatus::Failed    => { pending.retry_count += 1; /* ... */ ProofUpdate::NeedsRetry }
    _                         => ProofUpdate::Pending,   // <-- HERE
};
```

{% endstep %}

{% step %}

## Step 3 -- Driver short-circuits without advancing retry

At `driver.rs:699-707`, the driver matches `Some(ProofUpdate::Pending)` and returns `Ok(())` immediately:

* `retry_count` is NOT incremented.
* `ProofPhase` is NOT transitioned.
* The entry is NOT removed from `pending_proofs`.
* There is NO timeout check.
  {% endstep %}

{% step %}

## Step 4 -- `process_candidate` skips the game on every future tick

At `driver.rs:265-268`:

```rust
if self.pending_proofs.contains_key(&game_address) {
    debug!(game = %game_address, "skipping game with pending proof session");
    return Ok(());
}
```

The game is in `pending_proofs` (from Step 3), so it is skipped. Invalid proposal detection never runs for this game. The game is **permanently non-challengeable** until the challenger process restarts.
{% endstep %}

{% step %}

## Step 5 -- Repeat (Steps 1-4 loop on every scan tick)

The ZK-prover continues returning the same non-terminal status. The dispatcher continues returning `Pending`. The driver continues short-circuiting. The game remains stuck. `retry_count` stays at 0. No external mechanism breaks the cycle.
{% endstep %}
{% endstepper %}

### Runnable PoC

A standalone single-file Rust harness (`stalled_proof.rs`) is provided in this gist. It verbatim-ports the dispatch logic and driver model, then runs four scenarios:

| # | Wire status          | Ticks  | Expected outcome                                                |
| - | -------------------- | ------ | --------------------------------------------------------------- |
| A | `5` (FAILED)         | 10     | Entry dropped after `MAX_PROOF_RETRIES + 1 = 4` ticks (bounded) |
| B | `0` (UNSPECIFIED)    | 10,000 | Entry remains in `pending_proofs`; `retry_count = 0` forever    |
| C | `6` (future/unknown) | 10,000 | Identical to B via `unwrap_or(Unspecified)` fallback            |
| D | --                   | --     | Models `process_candidate` skip-if-pending                      |

#### Build and run

```sh
rustc --edition 2021 -O stalled_proof.rs -o stalled_proof && ./stalled_proof
```

All four assertion-based scenarios pass in <100 ms.

#### Recorded output

See `stalled_proof_transcript.txt` in this gist.

### Versions

* **Rust:** any recent stable (`rustc 1.x`, edition 2021)
* **OS:** Linux x86\_64
* **Dependencies:** none (stdlib only)

### What this does NOT prove

* Does not demonstrate exploitation on a live Base network.
* Does not require a running ZK-prover service, a Base node, or network access. The PoC is a standalone code-equivalent harness.
* The fleet-wide forward-compatibility stall (Section 5.3 of the main report) is argued analytically, not demonstrated empirically, because it requires a proto version bump that has not yet occurred.
