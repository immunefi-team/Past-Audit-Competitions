# 90207 bc critical missing zk tree leaf on the wormhole miner fee credit leads to permanent freezing of the block author s volume fee

**Submitted on Aug 23rd 2026 at 18:04:43 UTC by @halfounce for** [**Audit Comp | Quantus**](https://immunefi.com/audit-competition/audit-comp-quantus)

* **Report ID:** #90207
* **Report Type:** Blockchain/DLT
* **Report severity:** Critical
* **Target:** <https://github.com/immunefi-team/audit-comp-quantus-chain>
* **Impacts:**
  * Permanent freezing of funds

## Description

## Brief/Intro

On every wormhole settlement, the miner's share of the volume fee is credited to the QPoW block author with a bare balance increase and **no corresponding ZK-tree leaf**. The block author address is wormhole-derived (Poseidon over Goldilocks) and therefore has no private key, and wormhole-held value is movable only by a circuit proof of membership of a ZK-tree leaf. With no leaf recorded, the fee is unspendable by any key-holder and unreachable through the wormhole exit path; it is movable only by a governance referendum. Because the credit uses `increase_balance` rather than the burn path, `total_issuance` is not reduced — so this is a **freeze, not a burn**, and reported supply permanently overstates usable supply, growing with every settlement. In production this silently strands \~2 bps of all exit volume, without bound in time.

*Disclosure, per the project's `SECURITY.md` ("If you used AI tooling to find or write up the report, please say so"): **yes** — AI tooling was used in discovery and drafting. Every factual claim below was verified directly against the project's source, every quotation checked character-for-character against the cited file, and the proof of concept was executed with an inverted control (see PoC section).*

## Vulnerability Details

### Root cause: a safe idiom separated from the condition that made it safe

The codebase has **two** ways a credit becomes leaf-recorded, and the miner-fee path falls in the blind spot of both:

1. **Explicit** — a `Self::record_transfer(..)` call.
2. **Automatic** — `transaction_extensions::WormholeProofRecorderExtension<Runtime>`, installed in the runtime's `TransactionExtension` pipeline (`runtime/src/lib.rs:167`), which records a transfer proof for every `Balances::Minted` event it scans.

The exit path documents precisely why the *event-free* credit is safe there. From `pallets/wormhole/src/lib.rs:822-827`, directly above the exit credit:

> "NOTE: this must stay `Unbalanced::increase_balance` (event-free). The runtime's `WormholeProofRecorderExtension` records a transfer proof for every `Balances::Minted` event it scans, and this exit already records its own proof via `record_transfer` below — switching to `mint_into` (which emits `Minted`) could double-record the credit."

The safety of that idiom is **conditional**: it is correct *only because* the exit records its own proof on the following lines. The miner-fee path at `:955` uses the same event-free `increase_balance` with **no `record_transfer` beneath it**. The condition is absent while the idiom was carried over, and the consequences compose exactly wrong:

* event-free credit → no `Balances::Minted` event → the **automatic** recorder never fires;
* no explicit call → the **explicit** mechanism never fires;
* → value is minted and orphaned.

This is why the defect is easy to miss: the credit *looks* correct because it uses the blessed idiom from the same file. It occurs **twice in one function** — the public-batch aggregator rebate at `:919-933` has the identical omission — so it is a pattern, not a slip.

### The decisive contrast: the same function BURNS this fee in one branch and STRANDS it in the other

Two adjacent branches, \~10 lines apart, in `pallets/wormhole/src/lib.rs`.

**No-author branch (`:975-981`) — genuinely burned:**

```rust
} else {
    // No block author found - add miner fee to burn amount
    burn_amount_u128 = burn_amount_u128.saturating_add(miner_fee_u128);
}
```

Executed at `:990-993`, which really does destroy supply:

```rust
if !burn_amount.is_zero() {
    let current = <T::Currency as FungibleInspect<_>>::total_issuance();
    <T::Currency as Unbalanced<_>>::set_total_issuance(current.saturating_sub(burn_amount));
}
```

**Author-found branch (`:944-970`) — stranded, not burned:**

```rust
match <T::Currency as Unbalanced<_>>::increase_balance(&author, miner_fee, Precision::Exact) {
    Ok(_)  => { Self::deposit_event(Event::MinerVolumeFeePaid { miner: author, amount: miner_fee }); }
    Err(e) => { /* burn instead */ }
}
```

No `record_transfer(..)` on this path.

The code's own accounting comment at `:880-887` confirms the distinction:

> "- Minting miner fee: increases balance but NOT issuance (increase\_balance)"
>
> "- Burning: decreases total issuance by burn\_amount"

A burn reduces `total_issuance`; this credit does not. **Frozen, not destroyed.** It also bears on intent: a team that wanted this value destroyed had the burn path written ten lines away and did not route the author-found branch into it.

### The recipient is a keyless address

`primitives/wormhole/src/lib.rs:122-149` — `extract_author_from_digest` reads the 32-byte QPoW pre-runtime preimage and returns `derive_wormhole_address(preimage)` = `rehash_to_bytes(&inner_digest)` (`:109-111`) — the same Poseidon-over-Goldilocks construction used for unspendable wormhole accounts. Signing for it would require a Dilithium public key `P` with `hash_bytes(P)` equal to that Poseidon output, i.e. a preimage attack. Note these are **different functions**: `hash_bytes` absorbs bytes; `rehash_to_bytes` parses its input as Goldilocks field elements. A miner cannot choose a preimage landing on an address it controls.

### A leaf is the required pattern for this exact address, and the codebase shows it

`pallets/mining-rewards/src/lib.rs:228-238` credits the **same** block-author address via `mint_into` and immediately calls `T::ProofRecorder::record_transfer_proof(None, mint_account, miner.clone(), reward)`. The wormhole exit loop (`:828-859`) likewise records transfers for the amounts it credits. Only the miner-fee path and the aggregator rebate omit it.

`record_transfer` (`:1280-1330`) does three things: increments `TransferCount`, calls `T::ZkTree::record_transfer(...)` to insert the leaf, and emits `NativeTransferred`. **None of the three happens for the miner fee.**

### This contradicts the project's own specification

`docs/wormhole-zk.md` §2 states the design invariant for minting:

> "4. Mint `sum · 10^10` ... to each surviving exit; record each transfer in `pallet-zk-tree` so the new mint becomes a fresh leaf available for future wormhole exits."

And what is supposed to happen to the fee:

> "5. ... burn portion reduces `total_issuance`, miner portion **minted to the QPoW block author** from the pre-runtime digest. **If no author is found, the miner portion is burned instead.**"

The spec explicitly contrasts *minted to the author* with *burned*, and names burning as the fallback. If stranding were intended, the spec would say burned — the word is used for the sibling branch. The implementation satisfies "minted to the author" and omits the leaf-recording that step 4 states is what makes minted value usable.

### Prior art — EQ-QNT-WORMHOLE-F-03 (disclosed, not left for triage to find)

`audits/EigerWormholeAudit.pdf` (Equilibrium Group, 2026-03-20, public, shipped inside this repository) contains **EQ-QNT-WORMHOLE-F-03, "Miner Rewards Silently Burned Due to Incorrect Preimage Handling"** (Medium), on this same code path.

**F-03 is a different defect and its fix is already present.** F-03 names exactly three problems, all *address derivation*: (1) `AccountId::decode` on the raw preimage instead of hashing it; (2) `.find_map(|item| item.as_pre_runtime())` with no `POW_ENGINE_ID` filter; (3) a silent burn on extraction failure. All three are fixed at `audit-comp-ready`. The "Correct Implementation" F-03 holds up is `mining-rewards::extract_miner_from_digest`, praised solely for **deriving the address** correctly.

**F-03 = wrong address. This report = right address, missing leaf.** The words `record_transfer_proof`, leaf, ZkTree and leaf-membership appear nowhere in F-03. Notably the same audit critiques `mining-rewards::mint_reward`'s `record_transfer_proof` call site twice elsewhere (for `let _ =` and an unnecessary `.clone()`) — the contrast establishing this defect was in view from the mining-rewards side and was not connected to the wormhole miner-fee path.

**Stated risk:** a reviewer may argue F-03 already covers "the miner does not receive the volume fee", since the economic impact reads identically. The distinction offered is mechanism, not impact: F-03's remedy is fully applied and this defect persists behind it. **If triage judges this covered by F-03, it should be withdrawn rather than argued.**

## Impact Details

An exit mints exactly `summed_output_amount`, justified by a circuit proof of membership of a leaf `(to, transfer_count, asset_id, amount)`; the chain never consults the address's actual balance. Value with no leaf can therefore never be named by any future proof.

{% stepper %}
{% step %}

## Unspendable by any key-holder

The address is keyless by construction.
{% endstep %}

{% step %}

## Unreachable via the wormhole exit path

An exit mints strictly what a leaf-membership proof justifies, and no leaf covering this value exists. Verified exhaustively in the PoC: every leaf in the real ZK tree is enumerated and **zero** name the author.
{% endstep %}

{% step %}

## Movable only by governance

`pallets/balances/src/lib.rs:685` exposes `force_transfer`, gated by `ensure_root(origin)` at `:691`. There is **no sudo pallet**: `runtime/src/lib.rs:232` records *"Index 4 was `pallet_sudo` (removed). Kept vacant so downstream pallet indices stay stable"*. Root origin therefore requires a governance referendum or a runtime upgrade.
{% endstep %}

{% step %}

## No migration or sweeper recovers it

`pallets/wormhole/src/migrations.rs` holds exactly one migration (v1→v2 `RemoveSoundnessCounters`) which `kill()`s two counters and touches no balances; the wormhole pallet exposes no recovery/sweep/reclaim/`force_*` call and contains no `ensure_root`; and the runtime's entire `Migrations` tuple (`runtime/src/lib.rs:186`) is that counter deletion plus `pallet_treasury::MigrateV0ToV1`.
{% endstep %}

{% step %}

## Supply accounting is corrupted

`increase_balance` does not reduce `total_issuance`, so reported supply permanently overstates usable supply.
{% endstep %}

{% step %}

## Accrues without bound

`VolumeFeesBurnRate` is 50% of the 4 bps volume fee, so the stranded share is **\~2 bps of all exit volume**: small per settlement, unbounded over time. Balances already stranded are not recoverable retroactively by any normal mechanism.
{% endstep %}

{% step %}

## Same omission on the aggregator rebate

The omission also occurs on the aggregator rebate (`:919-933`) whenever the aggregator uses a wormhole address — the natural choice, since `aggregator_address` is a public input.
{% endstep %}
{% endstepper %}

This maps to the in-scope impact **"Permanent freezing of funds"**. The magnitude is stated above so triage has the full picture; no grade is argued for.

Note it is **not** "(fix requires hardfork)" — this is a Substrate runtime, so the fix ships as a runtime upgrade through governance.

### Remediation (both patterns already exist in this codebase)

1. **Record the leaf**, mirroring `mining-rewards/src/lib.rs:228-238`: call `Self::record_transfer(..)` alongside the credit so a leaf `(author, transfer_count, 0, miner_fee / SCALE_DOWN_FACTOR)` is inserted. Apply to the aggregator rebate too.
2. **Or route to the burn accumulator**, as the sibling no-author branch at `:975-981` already does — making the outcome honest in `total_issuance`.
3. **Durable fix:** provide a helper performing the event-free credit *and* the `record_transfer` together, so the pairing cannot be separated again.

## References

* `pallets/wormhole/src/lib.rs` — `:822-827` (the conditional-safety comment), `:880-887` (accounting), `:919-933` (aggregator rebate), `:944-970` (miner-fee credit), `:975-981` (burn branch), `:990-993` (issuance reduction), `:1280-1330` (`record_transfer`)
* `primitives/wormhole/src/lib.rs:109-149` — `derive_wormhole_address` / `extract_author_from_digest`
* `pallets/mining-rewards/src/lib.rs:228-238` — the correct pattern for the same address
* `runtime/src/lib.rs:167` (recorder extension), `:186` (Migrations tuple), `:232` (sudo removed)
* `pallets/balances/src/lib.rs:685-691` — root-gated `force_transfer`
* `docs/wormhole-zk.md` §2 — design spec, steps 4 and 5
* `audits/EigerWormholeAudit.pdf` — EQ-QNT-WORMHOLE-F-03 (prior art, disclosed above)

## Link to Proof of Concept

<https://gist.github.com/05oz/85966b1ddc413f1a1d65dfcad32003fb>

## Proof of Concept

The complete test source is in the linked secret Gist (`C01-poc-test.rs`). It extends the project's own `test_verify_private_batch_emits_miner_volume_fee_paid` fixture — real ZK proof, mock runtime with the **real** `pallet-zk-tree` wired in (`pallets/wormhole/src/mock.rs:127`: *"Real ZK tree so tests exercise the actual leaf hashing … not the no-op recorder"*).

{% stepper %}
{% step %}

## Clone the repository

Clone `immunefi-team/audit-comp-quantus-chain` at branch `audit-comp-ready`.
{% endstep %}

{% step %}

## Add the test source

Open `pallets/wormhole/src/tests.rs` and append the body of the linked Gist file to the **end of the existing `mod private_batch_proof_tests { ... }` block**, immediately before that module's closing brace.

**No additional imports are required.** This was verified by diffing the modified tree against a pristine checkout: 133 inserted lines in that one file, zero deleted, no other file in the repository touched. Every path the test needs (`pallet_zk_tree::*`, `sp_core::crypto::AccountId32`, `qp_wormhole::derive_wormhole_address`, `super::ceil_volume_fee`, `crate::*`) already resolves in that module.
{% endstep %}

{% step %}

## Run the test

```bash
cargo test -p pallet-wormhole audit_c01 -- --nocapture
```

Only `pallet-wormhole` and its mock runtime are required — a full node build is not needed (\~2 minutes).
{% endstep %}
{% endstepper %}

**Expected result on the unpatched code — the test PASSES, and passing is the bug:**

```
running 1 test
test tests::private_batch_proof_tests::audit_c01_miner_volume_fee_is_credited_without_any_zk_leaf ... ok
test result: ok. 1 passed; 0 failed; 0 ignored; 0 measured; 71 filtered out

C-01 permanence: 1 leaves in tree, 0 name the author (balance credited: 5000000000)
```

### What the test asserts

{% stepper %}
{% step %}

## The fee is credited

`free_balance(author)` rises by `expected_miner_fee`.
{% endstep %}

{% step %}

## The transfer count does not advance

`TransferCount::<Test>::get(&author)` does **not** advance.
{% endstep %}

{% step %}

## No leaf event names the author

No `NativeTransferred` event names the author.
{% endstep %}

{% step %}

## No leaf exists for the author

**Permanence** — it exhaustively enumerates every leaf in the real ZK tree (`pallet_zk_tree::LeafCount` / `Leaves`) and asserts **none** names the author. The batch's genuine exit recipient receives the one leaf; the author receives none while being credited the fee. Since an exit mints only what a leaf-membership proof justifies, and no leaf for this address exists, no such proof can ever be constructed.
{% endstep %}

{% step %}

## Sensitivity control

The tree is asserted non-empty and the genuine exit recipients *in the same batch* **do** emit `NativeTransferred` leaf events, so assertions (2), (3) and (4) cannot pass merely because nothing was recorded at all.
{% endstep %}
{% endstepper %}

### Inverted control

**Inverted control (proving the assertions are load-bearing, not vacuous).** Flipping assertion (2) to `assert_ne!` — demanding that a leaf *does* exist — makes the test fail:

```
test ... FAILED
panicked at pallets/wormhole/src/tests.rs: assertion `left != right` failed
test result: FAILED. 0 passed; 1 failed
```

Restoring it makes the test pass again. Both directions were verified.

Assertions (2) and (3) are worded to fail with a message stating the bug is fixed, so the test doubles as a regression test after a patch.

### Why this satisfies the PoC bar

`SECURITY.md` requires a PoC that "run\[s] directly against this project, using a release build or locally built version" and excludes "simulations that only reproduce the theory without exercising the project." This test invokes the pallet's real `verify_private_batch` extrinsic, running the actual `process_exit_bundle` code path, against the project's own ZK proof fixture, in the project's own mock runtime, with the real ZK tree. Nothing is simulated or reimplemented — the observed behaviour is the project's own code executing.
