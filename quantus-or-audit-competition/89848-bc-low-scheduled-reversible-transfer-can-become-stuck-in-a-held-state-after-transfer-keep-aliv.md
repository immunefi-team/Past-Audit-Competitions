# 89848 bc low scheduled reversible transfer can become stuck in a held state after transfer keep alive failure pallet reversible transfers&#x20;

**Submitted on Aug 21st 2026 at 13:48:11 UTC by @Pelican29685 for** [**Audit Comp | Quantus**](https://immunefi.com/audit-competition/audit-comp-quantus)

* **Report ID:** #89848
* **Report Type:** Blockchain/DLT
* **Report severity:** Low
* **Target:** <https://github.com/immunefi-team/audit-comp-quantus-chain>
* **Impacts:**
  * Temporary freezing of funds for at least 24 hour

## Description

## Brief/Intro

`do_execute_transfer` builds and dispatches an inner `transfer_keep_alive` call, then returns that inner dispatch's result directly as its own return value. Because every `#[pallet::call]` dispatchable is wrapped in `with_storage_layer` (frame/support-procedural/src/pallet/expand/call.rs:244), an `Err` from the inner call rolls back everything `do_execute_transfer` did: the released hold, the `PendingTransfers::remove`, and the `TransactionExecuted` event.

Meanwhile `pallet_scheduler::service_task` (pallets/scheduler/src/lib.rs, \~line 1342) treats ANY dispatch outcome (Ok or Err) as terminal when no retry config is set, and permanently removes the scheduled task. **`reversible-transfers` never calls `set_retry`/`set_retry_named` for its `execute_transfer` tasks** (confirmed via grep — no matches in `do_schedule_transfer_inner`). This is the crux of the impact: it is not enough that the inner dispatch can fail — the finding only matters because there is no retry path once it does.

Net effect: if the inner `transfer_keep_alive` fails, the reversible transfer's hold rolls back to "still pending", but the Scheduler has already permanently dropped the task, producing a three-way state inconsistency:

| Component                  | State after the failed execution                 |
| -------------------------- | ------------------------------------------------ |
| `PendingTransfers` storage | still present (as if never executed)             |
| Held balance               | still held                                       |
| Scheduler task             | **removed** (Scheduler believes the job is done) |

No normal actor can resolve this mismatch directly: `execute_transfer` can only be called by the pallet's own sovereign account (`ensure!(who == Self::account_id(), ...)`, line \~443), so the sender cannot re-trigger it by hand, and there is no reschedule/retry extrinsic in this pallet at all.

## Why this is inconsistent with the pallet's intended failure handling

The same file's `cancel_transfer` (\~line 865-897) explicitly guards against this exact failure shape: it separates the fund-release step so a later failure in canceling the scheduled task cannot roll back the release, with an inline comment describing almost verbatim the bug reported here ("...propagating rolls back the whole transactional extrinsic...permanently freezing the held funds..."). `do_execute_transfer` (the auto-execution path) has no equivalent protection.

This is not a speculative interpretation — it is evidence of the project's own design intent: the developers were already aware that "a transactional rollback here can freeze funds" and built a guard for it in `cancel_transfer`, but the same guard was never applied to `do_execute_transfer`.

An existing test, `cancel_releases_funds_when_scheduled_task_already_gone`, only *simulates* the scheduler losing the task (calls `cancel_named` manually). No existing test drives a real `execute_transfer` failure through the actual Scheduler pallet — that is the gap this PoC fills.

## Impact

The scheduled transfer fails to complete, while the funds remain held and the Scheduler task is permanently removed without a retry path.

* The recipient receives nothing; the sender's funds remain held with no automatic recovery.
* No clear on-chain signal identifies which reversible transfer is stuck — the pallet's own `TransactionExecuted` event is rolled back with everything else. Only the Scheduler's generic `Dispatched{Err}` event remains, which does not say which pallet or which transfer.
* `recover_funds` (guardian-only) does not help: it seizes everything to the guardian with a fee, it does not retry the original transfer to the intended recipient.

**This is not a fund-loss bug — it is an availability/liveness failure.** The reversible-transfer guarantee ("your transfer executes automatically after the delay, or you can cancel it") is silently broken for the affected transfer.

**Manual cancellation provides a recovery path, but it does not prevent the liveness failure**: the sender must first detect the failure (via a generic, unattributed Scheduler event), identify which of their pending transfers is affected, and submit a separate `cancel()` transaction. The bug is that the pallet's core promise — automatic execution or explicit cancellation, with no silent third state — is broken; recoverability changes the severity, not the validity, of that break.

## Additional low-complexity trigger identified by code analysis (2026-08-20)

The original GHSA explicitly left this unverified: "I did not verify the low-amount/new-account trigger against the full runtime." Identified via code inspection (not yet re-run as a live test against the real runtime):

* Real runtime `EXISTENTIAL_DEPOSIT = MILLI_UNIT = 1_000_000_000` (runtime/src/lib.rs) — not the mock's `ExistentialDeposit = 1`. This value is confirmed in the current public runtime source.
* Standard Substrate/FRAME semantics: an account cannot be created with a balance below the existential deposit. Sending `value < ED` to a brand-new (never-before-used) recipient address fails deterministically under standard FRAME balance semantics, regardless of `transfer_keep_alive` vs `transfer_allow_death` — separate from (and in addition to) `transfer_keep_alive`'s own sender-side ED guarantee.
* Consequence: a user scheduling a reversible transfer of any amount below 1 MILLI\_UNIT to a fresh recipient address would hit the same `do_execute_transfer` failure/rollback/stuck-hold bug described above, without depending on timing or other balance activity during the delay.
* **Who can trigger this**: this is not a remote attacker locking another user's funds. The condition is created by the sender's own choices (either their balance changes during the delay, or they schedule a small amount to a new address) — closer to "any ordinary user could hit this by accident" than "an attacker can remotely lock a victim's funds." This materially affects how Privileges Required / Attack Complexity should be read; flagged explicitly here rather than argued away.
* Not independently re-verified with a live `cargo test` run against the real `EXISTENTIAL_DEPOSIT` value (the existing PoC used the mock's ED=1 and a balance-drain workaround instead). This is a code-reasoning argument backed by well-documented Substrate semantics, not a second passing test.

## What I did not verify

* Not tested against a running multi-node testnet, only the pallet's own unit-test runtime.
* The mock's `ExistentialDeposit = 1` meant I could not trigger the failure through the most natural real-world path (small amount to a brand-new never-used recipient account) inside the test; I modeled the equivalent "sender's other balance spent down during the delay" scenario directly via storage mutation instead. Did not independently re-verify the low-amount/new-account trigger against the full runtime.
* Did not check whether the same pattern exists in `recover_funds`'s own transfer-out step (and in fact confirmed separately that `recover_funds` already handles this failure class correctly — see below).

## Suggested Fix

Make `do_execute_transfer` handle a failed inner dispatch the same way `cancel_transfer` already handles a failed `cancel_named`: release the hold and remove the pending-transfer bookkeeping in their own transactional scope so a later failure of the inner transfer does not roll them back; dispatch the inner transfer as best-effort; on failure, emit a clear event ("this reversible transfer's auto-execution failed, please check on it") instead of silently losing the event to the storage-layer rollback.

## Disclosure note

I privately reported this same bug to the Quantus-Network/chain maintainers via GitHub Security Advisory GHSA-f38q-8qpf-7m3c on 2026-08-15 (3 days after this competition's Aug 12 launch), through the project's official SECURITY.md channel. As of today it is still in Triage status — not published, not fixed, visible only to me (as reporter) and the Quantus-Network owners. Per this program's "Private Known Issues Reward Policy" ("Private known issues — known issues that were not publicly disclosed — are valid for a reward at their full severity"), I'm submitting it here as well since it remains unpublished/unfixed and is not listed among the program's stated "Known Issues".

## AI tooling disclosure

Claude Code (Anthropic) was used to help read the codebase, form and check the hypothesis, and write/run the proof-of-concept test. The analysis, PoC construction choices, and this report are mine.

## Proof of Concept

Repo: audit-comp-quantus-chain (matches Quantus-Network/chain @commit c3f7f3b8bd364ba8487f27c06680150f7fdf0d59 at time of writing)

{% stepper %}
{% step %}

## Add the test

Add the test below to `pallets/reversible-transfers/src/tests/test_reversible_transfers.rs` (also requires changing the import line: add `Get` to the `frame_support::traits` import).
{% endstep %}

{% step %}

## Run the targeted test

`cargo test -p pallet-reversible-transfers auto_execution_permanently_freezes_funds_on_keep_alive_failure --lib -- --nocapture`
{% endstep %}

{% step %}

## Run the full pallet suite

`cargo test -p pallet-reversible-transfers --lib` (52 passed; 0 failed — 51 existing + 1 new, in original testing)
{% endstep %}
{% endstepper %}

### Test logic

{% stepper %}
{% step %}

## Fund the test account

Give test account `victim` just enough balance to schedule a transfer and keep the existential deposit free.
{% endstep %}

{% step %}

## Schedule the transfer

`schedule_transfer_with_delay` for the full amount (held).
{% endstep %}

{% step %}

## Simulate reduced free balance

Simulate the victim's remaining free balance being spent to 0 by ordinary activity during the delay (models fees/other payments; written directly via `frame_system::Account::<Test>::mutate` only because the mock's `ExistentialDeposit = 1` makes the more realistic "small amount to a brand-new recipient account" trigger unreachable in this test environment; the real runtime's much larger `EXISTENTIAL_DEPOSIT = MILLI_UNIT` makes that natural trigger far easier to hit for real).
{% endstep %}

{% step %}

## Advance the Scheduler

Let the real Scheduler pallet run forward past the delay (`run_to_block(7)`) — **the actual dispatch happens, nothing is manually simulated**: schedule → blocks advance → Scheduler executes → inner dispatch fails → Scheduler removes task.
{% endstep %}

{% step %}

## Assert the stuck state

Assert: `PendingTransfers` still has the entry, the hold is still on the victim, the recipient got 0, and the Scheduler's Agenda is completely empty (task permanently gone, no retry pending).
{% endstep %}

{% step %}

## Assert recovery

Assert: only a manual `cancel()` afterward recovers the funds.
{% endstep %}
{% endstepper %}

### State transition observed by the PoC's assertions

| State              | Before execution         | After failed execution |
| ------------------ | ------------------------ | ---------------------- |
| `PendingTransfers` | present                  | present                |
| Held balance       | held                     | still held             |
| Recipient balance  | 0                        | 0                      |
| Scheduler task     | present                  | removed                |
| Recovery path      | scheduled auto-execution | manual `cancel()` only |

(Full Rust diff/patch available — same as attached to GHSA-f38q-8qpf-7m3c, Quantus-Network/chain, currently in private Triage status, not yet published, reporter: gapbbong.)
