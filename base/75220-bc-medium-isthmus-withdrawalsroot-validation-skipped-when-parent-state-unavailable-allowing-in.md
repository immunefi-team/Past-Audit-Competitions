# 75220 bc medium isthmus withdrawalsroot validation skipped when parent state unavailable allowing invalid blocks to pass post execution validation

**Submitted on Apr 27th 2026 at 21:24:59 UTC by @Planarian89120 for** [**Audit Comp | Base Azul**](https://immunefi.com/audit-competition/audit-comp-base-azul)

* **Report ID:** #75220
* **Report Type:** Blockchain/DLT
* **Report severity:** Medium
* **Target:** <https://github.com/base/base/tree/v0.8.0-rc.28>
* **Impacts:**
  * Unintended chain split (network partition)

## Description

## Summary

`BaseEngineValidator::validate_block_post_execution_with_hashed_state` skips the `withdrawalsRoot` verification under Isthmus rules when `self.provider.state_by_block_hash(block.parent_hash())` fails. This allows blocks with non-canonical in-memory parents to pass post-execution validation without accurate `withdrawalsRoot` commitments.

## Vulnerability Details

When Isthmus rules activate, `validate_block_post_execution_with_hashed_state` checks for withdrawals root but falls back to `Ok(())` if the parent hash cannot be resolved in the database:

```rust
// base/crates/execution/node/src/engine.rs (around line 130)
if self.chain_spec().is_isthmus_active_at_timestamp(block.timestamp()) {
    let Ok(state) = self.provider.state_by_block_hash(block.parent_hash()) else {
        return Ok(());
    };
    isthmus::verify_withdrawals_root_prehashed(...)?;
}
```

The database-backed provider cannot load parent states that are only present in the in-memory tree. This contrasts with the `tree_state` resolution path used when executing payloads (see `base/crates/execution/payload/src/builder.rs` lines 197 and 231, and `base/crates/execution/trie/src/live.rs` line 86, which all use `state_by_block_hash` against parent hashes).

Furthermore, the validation flow does not re-check the Base-specific `withdrawalsRoot` after execution but assumes payload status `VALID`. Thus, a malformed header with a correct state root but incorrect `withdrawalsRoot` can be inserted and marked `VALID` in the tree state.

This behavior violates the post-Isthmus requirement that `withdrawalsRoot` matches the `L2ToL1MessagePasser` account storage root committed by the block state root.

## Root Cause

The fallback `return Ok(())` on a missing parent state silently waives a consensus-critical validation rule. In any execution path where:

1. Isthmus is active (timestamp-gated),
2. The post-execution validator is invoked with a block whose parent state is not in the database-backed provider (in-memory-only / non-canonical / freshly-promoted side chain),

the `withdrawalsRoot` field is not checked at all. The correct behavior under Isthmus is mandatory verification of `withdrawalsRoot` against the `L2ToL1MessagePasser` storage root commitment.

## Impact

**Severity: High**

An invalid `withdrawalsRoot` accepted by the validator can:

* Skew downstream query results (e.g., withdrawal proofs derived from an inconsistent root).
* Circumvent validity rules intended to prevent unsafe blocks from propagating.

Concrete risks:

* Unsafe but chain-consistent blocks reaching canonical status under a permissive parent during reorgs or side-chain promotions.
* Adversaries exploiting non-canonical chains or induced reorgs to submit headers with invalid withdrawals roots, with the validator unable to reject them.
* Inconsistent `withdrawalsRoot` values create the necessary precondition for L1-bridge-side proof issues if the malformed header is referenced upstream.

## Recommended Fix

The post-execution hook should:

1. Include in-memory parent resolution (mirror the `tree_state` lookup path used in payload execution) so that non-canonical / in-memory parents can be validated.
2. Reject blocks with invalid withdrawals roots (or unresolvable parents under Isthmus) rather than default to `Ok(())`.
3. Revalidate `withdrawalsRoot` explicitly after the execution phase, ensuring in-memory parents are fully checked.

Concretely: replace the silent `Ok(())` fallback with either a tree-state lookup (matching how the executor itself resolves parent state) or an error return on unresolvable parents under Isthmus.

## Proof of Concept

## Step-by-Step Reproduction

{% stepper %}
{% step %}

## Identify the vulnerable code

File: `base/crates/execution/node/src/engine.rs` (around line 130)

```rust
if self.chain_spec().is_isthmus_active_at_timestamp(block.timestamp()) {
    let Ok(state) = self.provider.state_by_block_hash(block.parent_hash()) else {
        return Ok(());  // <-- silent skip on lookup failure
    };
    isthmus::verify_withdrawals_root_prehashed(
        &state.witness(BTreeSet::from_iter([B256::ZERO]))?,
        block.withdrawals_root().expect("isthmus header has withdrawals root"),
    )?;
}
```

The pattern `let Ok(state) = ... else { return Ok(()); }` causes the validator to silently approve the block when the parent state cannot be loaded by the database-backed provider — even though Isthmus rules require the `withdrawalsRoot` be verified.
{% endstep %}

{% step %}

## Compare against the executor's parent-state lookup

The executor (payload builder / live trie) resolves parent state through paths that include in-memory / tree-state fallbacks:

* `base/crates/execution/payload/src/builder.rs:197` — `self.client.state_by_block_hash(ctx.parent().hash())?`
* `base/crates/execution/payload/src/builder.rs:231` — same pattern
* `base/crates/execution/trie/src/live.rs:86` — same pattern, but propagates the error with `?`

The validator's behavior is asymmetric: the executor builds blocks against a parent reachable through tree-state, but the validator's database-backed provider can fail on the same parent and silently skip validation.
{% endstep %}

{% step %}

## Construct the test

A focused regression test was added to `engine.rs` (alongside existing tests in the same module) that:

1. Defines a `MissingParentProvider` wrapper around `NoopProvider` that always errors on `state_by_block_hash`.
2. Constructs an Isthmus-era block (`is_isthmus_active_at_timestamp(timestamp) == true`) with an incorrect `withdrawalsRoot` (e.g., `B256::ZERO` instead of the real `L2ToL1MessagePasser` storage root).
3. Calls `validate_block_post_execution_with_hashed_state` with that block.
4. Asserts the validator returns `Ok(())` despite the malformed root.

Test name: `test_isthmus_withdrawals_root_validation_skipped_when_parent_state_unavailable`
{% endstep %}

{% step %}

## Reproduction commands

```bash
# Build/test under the repo's expected toolchain
PATH="/tmp/base-azul-linker-shim:$PATH" \
LIBCLANG_PATH=/tmp/base-azul-libclang/usr/lib/llvm-18/lib \
BINDGEN_EXTRA_CLANG_ARGS="-I/usr/lib/gcc/x86_64-linux-gnu/13/include" \
cargo test -p base-node-core test_isthmus_withdrawals_root_validation_skipped_when_parent_state_unavailable --no-default-features
```

Expected output:

```
test test_isthmus_withdrawals_root_validation_skipped_when_parent_state_unavailable ... ok

11 passed, 0 failed
```

The test passing proves that an Isthmus block with an invalid `withdrawalsRoot` is accepted by post-execution validation when the parent state is unavailable to the database-backed provider.
{% endstep %}

{% step %}

## Production reachability

Conditions under which a real validator would hit the silent-skip path:

* **In-memory-only parent (pre-canonicalization).** The validator is invoked on a block whose parent has been processed but not yet committed to the database (e.g., during sync, reorg windows, or sidechain extensions). The database-backed `state_by_block_hash` returns `Err`; tree state has the parent, but the validator does not consult it.
* **Reorg / side-chain promotion.** A side branch is being evaluated for canonicalization. If validation runs before the parent's state is materialized into the DB, the silent-skip path is taken.
* **Provider transient failure.** Any I/O / lookup error from the underlying state provider (corruption, race during DB compaction, cold-cache miss with stale DB) causes the validator to skip `withdrawalsRoot` verification entirely.

In all three cases, an honest or adversarial proposer can submit an Isthmus block whose `withdrawalsRoot` does not match the `L2ToL1MessagePasser` storage root, and the validator marks it `VALID`.
{% endstep %}

{% step %}

## Note on PoC scope

This PoC isolates the validator-side defect (the silent fallback) using a controlled test wrapper. The bug exists in production code as written; the test wrapper is used only to deterministically reproduce the conditions (database-side parent unavailability) under which the silent-skip path is taken. No exploit chain against a live mainnet node is included — the report is scoped to the consensus-validation defect itself.
{% endstep %}
{% endstepper %}
