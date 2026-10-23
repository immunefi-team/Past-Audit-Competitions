# 90466 bc medium a failed enactment scheduling attempt destroys pallet referenda s preimage reference so an approved referendum s root call is silently lost

**Submitted on Aug 25th 2026 at 07:02:41 UTC by @Valisthea for** [**Audit Comp | Quantus**](https://immunefi.com/audit-competition/audit-comp-quantus)

* **Report ID:** #90466
* **Report Type:** Blockchain/DLT
* **Report severity:** Medium
* **Target:** <https://github.com/immunefi-team/audit-comp-quantus-chain>
* **Impacts:**
  * A bug in the respective layer 0/1/2 network code that results in unintended smart contract behavior with no concrete funds at direct risk
  * Manipulation of governance voting result deviating from voted outcome and resulting in a direct change from intended effect of original results

## Description

## Summary

`pallet-scheduler` releases a preimage reference on every failed scheduling path, including on the path used by callers that hand over an already bounded call. `pallet-referenda` is such a caller. It takes a preimage reference at `submit` and its enactment retry loop calls `schedule_named` once per attempt, so a single failed attempt consumes the referendum's only reference.

The fork already draws the distinction that would prevent this. It added a `request_preimage: bool` parameter precisely because the two caller shapes differ in who owns the reference, and it applies that flag on the request side. It does not apply it on the drop side.

The result is that an approved referendum records `Approved`, its submission deposit becomes refundable, and its Root call is not pinned when enactment arrives. Where the proposer already reclaimed their preimage deposit, which the fork's own comment and test declare safe, the bytes are deleted synchronously by the failed attempt.

This is a regression, not a pre-existing issue. The audit finding `EQ-QNT-SCHEDULER-F-06` named this exact error class and recommended removing it; commit `69e12df` (2026-04-30) did so and wrote the rule down; commit `96de0cf` (2026-07-06) reintroduced it on four other paths.

## The rule, in the vendor's own words

`pallets/scheduler/src/lib.rs:1463-1466`, the fix for `EQ-QNT-SCHEDULER-F-06`:

```rust
// Retry placement failed (agenda full). The retry clone was never
// successfully placed, so it doesn't own a preimage reference.
// Do NOT drop the retry clone's preimage here - it never acquired one.
// The caller will drop the original task's preimage.
```

And the fork states that it cannot know whether a v3 shim caller owns a reference, `pallets/scheduler/src/lib.rs:793-796`:

```rust
// V12 audit #162453: callers that hand over an already-`Bounded` call (the v3 trait
// shims, benchmarks and tests) cannot tell whether they own a preimage request for the
// lookup hash, so keep the historical behavior and unconditionally request one. The
// `schedule*` dispatchables use `do_schedule_inner` with a precise flag instead.
```

That uncertainty is exactly why `request_preimage` exists. `do_schedule_named`, the v3 shim that `pallet-referenda` calls, hard-codes it to `true` (`pallets/scheduler/src/lib.rs:919-929`):

```rust
fn do_schedule_named(
	id: TaskName,
	when: DispatchTime<BlockNumberFor<T>, T::Moment>,
	priority: schedule::Priority,
	origin: T::PalletsOrigin,
	call: BoundedCallOf<T>,
) -> Result<TaskAddressOf<T>, DispatchError> {
	// See `do_schedule`: keep unconditionally requesting the preimage for callers that hand
	// over an already-`Bounded` call; the dispatchables use `do_schedule_named_inner`.
	Self::do_schedule_named_inner(id, when, priority, origin, call, true)
}
```

## The defect

The flag governs the request. It does not govern the drop. `pallets/scheduler/src/lib.rs:939-967` (`do_schedule_named_inner`):

```rust
// See `do_schedule`: drop any preimage noted by `StorePreimage::bound` on every fallible
// path so a failed schedule cannot leave an unowned, system-requested preimage behind.
if Lookup::<T>::contains_key(id) {
	T::Preimages::drop(&call);
	return Err(Error::<T>::FailedToSchedule.into());
}
let when = match Self::resolve_time(when) {
	Ok(when) => when,
	Err(e) => {
		T::Preimages::drop(&call);
		return Err(e)
	},
};
let lookup_hash = call.lookup_hash();
let task =
	Scheduled { maybe_id: Some(id), priority, call, origin, _phantom: Default::default() };
let res = match Self::place_task(when, task) {
	Ok(res) => res,
	Err((e, task)) => {
		T::Preimages::drop(&task.call);
		return Err(e)
	},
};
// See `do_schedule_inner` (V12 audit #162453).
if let Some(hash) = lookup_hash.filter(|_| request_preimage) {
	T::Preimages::request(&hash);
}
```

Three unconditional drops at `:942`, `:948` and `:958`; one conditional request at `:963`. The same shape exists in `do_schedule_inner`: drops at `:828` and `:837`, conditional request at `:846`.

The rationale comment at `:821-824` states the assumption that makes the drop correct:

```rust
// `call` may already carry a preimage noted by `StorePreimage::bound` (called by the
// scheduler entrypoints before this). If any later fallible step fails, drop that
// preimage so a failed schedule cannot leave an unowned, system-requested preimage
// behind as state bloat. This mirrors the cleanup performed on cancellation.
```

That assumption holds for the scheduler's own dispatchables, which call `bound` themselves. It does not hold for a v3 shim caller, whose reference predates the call and belongs to someone else.

Upstream `pallet-scheduler` 40.1.0 has no drop on either path (`fork-diff-pallets/.upstream/pallet-scheduler-40.1.0/src/lib.rs:966,983-987`):

```rust
let when = Self::resolve_time(when)?;
...
let res = Self::place_task(when, task).map_err(|x| x.0)?;

if let Some(hash) = lookup_hash {
	// Request the call to be made available.
	T::Preimages::request(&hash);
}
```

Upstream does contain seven `T::Preimages::drop` calls, all on cancel and service paths where the scheduler genuinely owns the reference. None in `do_schedule`.

## Why pallet-referenda loses its reference

`pallet-referenda` takes exactly one reference, at `submit` (`pallets/referenda/src/lib.rs:597`):

```rust
T::Preimages::request(&hash);
```

and releases exactly one, in `conclude_ongoing` (`:1705-1706`):

```rust
fn conclude_ongoing(proposal: &BoundedCallOf<T, I>, who: &T::AccountId) {
	T::Preimages::drop(proposal);
}
```

Its enactment loop calls `schedule_named` once per attempt, up to 17 times (`:1077`, `:1105`):

```rust
const MAX_ENACTMENT_SCHEDULE_RETRIES: u32 = 16;
...
for _ in 0..=Self::MAX_ENACTMENT_SCHEDULE_RETRIES {
```

On approval, `schedule_enactment` runs first and `conclude_ongoing` runs only inside the `Ok` arm (`:1560`, `:1570-1575`), so referenda still holds its reference when scheduling is attempted. The comment there states the invariant this defect breaks:

```rust
// The scheduler has just taken its own request for the
// enactment call in `schedule_named`, so after we drop the
// submit-time request the bytes stay pinned until dispatch.
Self::conclude_ongoing(
	&status.proposal,
	&status.submission_deposit.who,
);
```

The intended arithmetic is 1 then 2 then 1. With one failed attempt it is:

| step                                                         | `RequestStatusFor[H]`                  |
| ------------------------------------------------------------ | -------------------------------------- |
| preimage noted                                               | `Unrequested { ticket, len }`          |
| `submit` requests                                            | `Requested { ticket: Some, count: 1 }` |
| attempt 1: `place_task` returns `Exhausted`, scheduler drops | `Unrequested { ticket, len }`          |
| attempt 2 succeeds, scheduler requests                       | `Requested { ticket: Some, count: 1 }` |
| `conclude_ongoing` drops                                     | **`Unrequested`**                      |

The reference referenda drops at the end is the scheduler's, not its own. At enactment the call is not pinned.

## Reachability

The trigger is `place_task` returning `DispatchError::Exhausted` on a full agenda. The `resolve_time` path is **not** reachable from referenda, which clamps the target forward (`:1100-1101`), and is not part of this report:

```rust
let earliest_allowed = now.saturating_add(track.min_enactment_period.max(One::one()));
let mut when = desired.evaluate(now).max(earliest_allowed);
```

Runtime `MaxScheduledPerBlock = 50` (`runtime/src/configs/mod.rs:345`). Enactment priority is 63 and `LOWEST_PRIORITY` is 255, so the scheduler's 20 percent reservation (reserved 10, cap 40) does not protect the enactment task. The fork concedes the residual case itself at `pallets/referenda/src/lib.rs:1072-1074`:

```rust
/// The scheduler reservation already blocks cheap `LOWEST_PRIORITY` pre-fills; this
/// covers the residual case where mid-priority tasks (e.g. referendum alarms) occupy
/// the reserved headroom.
```

Two points matter for how expensive this is:

1. **Only the first attempt has to fail.** The 17 attempt loop was sized against losing the enactment entirely, and the same comment reasons that filling seventeen consecutive agendas is expensive. For this defect one full block is enough, because the reference is consumed on the first failure and never restored.
2. **No attacker is required.** The fork built a retry loop because it expects agendas to fill in normal operation. Any natural fill of the target block silently unpins an approved proposal.

A deliberate fill of one 50 slot agenda needs 40 permissionless `LOWEST_PRIORITY` reversible transfers plus at least 10 non `LOWEST_PRIORITY` tasks; in this runtime the latter are referendum alarms at priority 128, which requires collective membership. I state this plainly because it bounds the adversarial version of the trigger. It does not bound the accidental version.

## Impact

An approved tech collective referendum transitions to `ReferendumInfo::Approved`, its submission deposit becomes refundable, and its Root call is no longer pinned. If the bytes are then removed the scheduler reaches the enactment slot, `peek` fails, and the task is treated as terminal: `Lookup` and `Retries` are cleared and `CallUnavailable` is emitted. Recovery requires a fresh `submit` and a full vote cycle. `pallet-scheduler`'s own extrinsics are `#[runtime::disable_call]`, so `set_retry` cannot rescue it.

This is the only governance lane in the runtime and every referendum dispatches as Root, so the lost call can be a runtime upgrade, including a security patch, or a collective membership change.

**The stronger variant needs no second actor.** `pallets/referenda/src/lib.rs:578-583` explicitly blesses the proposer reclaiming their preimage deposit before approval, on the grounds that the `submit` time request keeps the bytes alive. In that state `RequestStatusFor[H] = Requested { maybe_ticket: None, count: 1 }`, and the failed attempt reaches `pallets/preimage/src/lib.rs:464-468`:

```rust
// Preimage was noted without owner - just remove it.
(Some(len), None) => {
	Self::remove(hash, len);
	RequestStatusFor::<T>::remove(hash);
	Self::deposit_event(Event::Cleared { hash: *hash });
```

The bytes are deleted at that moment. No cooperating noter, no further action.

That the approved call's *effect* is lost, and not merely its bytes, is measured directly in proof of concept 3 below: with a free agenda the enactment sets `balance(42) = 1`, and with the enactment block full the same referendum still reaches `Approved` while `balance(42)` stays `0`.

### Why this is the primary impact row rather than a lesser one

*A bug in the respective layer 0/1/2 network code that results in unintended smart contract behavior with no concrete funds at direct risk*, clause by clause:

* **layer 1 network code**: `pallet-scheduler` is runtime code of the L1 itself, not application code deployed on top of it.
* **unintended behavior**: an approved Root dispatch on the only governance lane does not execute, while the referendum records `Approved` and refunds its deposits. Nothing in the runtime reports the loss: `pallet-referenda` passes 60/60, `pallet-scheduler` passes 89/89, and `Referenda::do_try_state` passes, all with the defect live.
* **no concrete funds at direct risk**: established, not assumed. The variant sweep above shows the only other production caller of these shims owns its own reference, so there is no fund-moving variant. This clause is an entry condition of the row, and this report satisfies it deliberately.

One reading of that row would restrict "smart contract behavior" to contracts deployed on the chain. That reading makes the row unreachable on this target: the runtime contains no `pallet-contracts` and no contract execution environment at all (`balances`, `frame-system`, `mining-rewards`, `multisig`, `preimage`, `qpow`, `ranked-collective`, `recovery`, `referenda`, `reversible-transfers`, `scheduler`, `timestamp`, `transaction-payment`, `treasury`, `utility`, `vesting`, `wormhole`, `zk-tree`). A row that no finding on this program could ever satisfy would not be in the program's impact list, so the operative reading is unintended on-chain runtime behavior, which is what this is.

For completeness, the case a triager might weigh against a higher tier: no funds move, and a loss that is noticed can be recovered by resubmitting and re-running the vote. The case they might weigh for one: the lost call is a Root dispatch, it can be the `set_code` that ships a security patch, the failure is silent, and the enactment block is public a full `min_enactment_period` in advance so the condition can be reproduced against every subsequent referendum.

Note that a call under 128 bytes is `Bounded::Inline` (`frame/support/src/traits/preimages.rs:29`) and carries no preimage, so it is unaffected. A `set_code` runtime upgrade is `Lookup` by construction.

## Variant sweep: which other callers are affected (answer: none)

Stating this up front because it bounds the finding and it is the first thing worth checking. The defect only bites a caller whose preimage reference **predates** the scheduling call. Every other production caller of the v3 shims in this runtime owns a fresh reference or carries no preimage at all, and for those the drop is correct cleanup:

| caller                                        | call scheduled                                                                                    | affected                          |
| --------------------------------------------- | ------------------------------------------------------------------------------------------------- | --------------------------------- |
| `referenda:1106` `schedule_named` (enactment) | the referendum proposal, `Lookup`, requested at `submit`                                          | **yes**                           |
| `referenda:1162` `schedule` (alarm)           | `nudge_referendum`, a few bytes, `Inline`                                                         | no, `drop` on `Inline` is a no-op |
| `reversible-transfers:815` `schedule_named`   | `execute_transfer { tx_id }`, `Inline`, and bound by the caller itself one line earlier at `:807` | no, it owns the fresh reference   |

`pallets/reversible-transfers/src/lib.rs:807`:

```rust
let bounded_call = T::Preimages::bound(Call::<T>::execute_transfer { tx_id }.into())?;
```

That is the caller shape the drop was written for. So there is no fund-moving variant of this finding, and I am not claiming one.

## Proof of Concept

Three harnesses, all additive, no pre-existing test modified. Their complete sources are inlined in the appendix at the end of this report, so everything needed to reproduce is in this text: no attachment and no external link.

### 1. End to end through pallet-referenda

`pallets/referenda/src/refute_enactment_pin.rs`. Complete source in appendix A.1, with the `mod` lines that wire it in. Real referenda mock with scheduler and preimage, Root track, `Bounded::Lookup` proposal, enactment block filled to `MaxScheduledPerBlock` with priority 63 tasks.

```
cargo test -p pallet-referenda refute_enactment_pin -- --nocapture
```

```
=== CONTROL (agenda free) ===
preimage still requested after Approved: true
ok

=== REFUTATION ATTEMPT (agenda at the enactment block full) ===
preimage still requested after Approved: false
expected if the claim is FALSE: true   |   observed: false
bytes survive the noter's unnote: false
panicked at refute_enactment_pin.rs:119: CLAIM SURVIVES: pinned=false bytes_survive=false
FAILED

=== IMPACT VARIANT (deposit reclaimed, then one failed attempt) ===
preimage bytes at enactment time: None
expected if the claim is FALSE: Some(1)   |   observed: None
panicked at refute_enactment_pin.rs:151: CLAIM SURVIVES: the approved proposal's bytes are gone
FAILED

test result: FAILED. 1 passed; 2 failed; 0 ignored; 63 filtered out
```

The tests are written as refutation attempts: they assert the claim is false, so a **failure means the claim survived**. The referendum reaches `ReferendumInfo::Approved` in all three. The control is the load bearing part: with a free agenda the pin survives, so the failed attempt is isolated as the cause.

### 2. Scheduler level, with a positive control

`pallets/scheduler/src/poc_preimage_drop.rs`. Complete source in appendix A.2. Unlike harnesses 1 and 3, this one asserts the OBSERVED behaviour directly rather than the safe outcome, so here a passing run is the proof.

```
cargo test -p pallet-scheduler poc_preimage_drop -- --nocapture
```

```
=== control: no failed schedule ===
bytes still present after the noter un-noted: true
ok

=== referenda's real trigger: agenda exhaustion ===
agenda at block 5 filled to MaxScheduledPerBlock = 10
schedule_named rejected with: Exhausted
external holder's pin survived the rejection: false
ok

=== scheduler drops a reference it never acquired ===
external holder requested the preimage, then ONE schedule failed
bytes still present after the noter un-noted: false
expected if the pin held: true   |   observed: false
ok

test result: ok. 3 passed; 0 failed; 0 ignored; 89 filtered out
```

### 3. The executed effect, with a positive control

The two harnesses above prove the reference is dropped and that the bytes are gone by enactment time. This one measures the consequence instead of inferring it: whether the approved Root call's effect ever happens.

`pallets/referenda/src/refute_enactment_impact.rs`. Complete source in appendix A.3. Same real referenda mock with scheduler and preimage, Root track, `Bounded::Lookup` proposal. The proposal is the mock's own `set_balance_proposal(1)`, i.e. `Balances::force_set_balance { who: 42, new_free: 1 }` (`pallets/referenda/src/mock.rs:336-339`), a Root only call, so `balance(42)` is a direct read of whether enactment ran.

```
CARGO_PROFILE_TEST_DEBUG_ASSERTIONS=false \
  cargo test -p pallet-referenda --lib refute_enactment_impact -- --nocapture
```

```
=== CONTROL: agenda at block 13 free ===
referendum info      : Some(ReferendumInfo::Approved(9, ..))   [deposits elided]
balance(42)          : 1   (1 = the approved Root call ran)
CallUnavailable seen : false                                           ... ok

=== CLAIM: agenda at block 13 full before approval (measured at #13) ===
fillers placed at block 13: 100
referendum info      : Some(ReferendumInfo::Approved(9, ..))   [deposits elided]
balance(42)          : 0                                               ... FAILED

=== CLAIM: carried to #20 (enactment slid to #14) ===
referendum info      : Some(ReferendumInfo::Approved(9, ..))   [deposits elided]
balance(42)          : 0   (1 = the approved Root call ran)
CallUnavailable seen : true
preimage bytes now   : None                                            ... FAILED

test result: FAILED. 1 passed; 2 failed; 63 filtered out
```

Written as refutation attempts like the others, so a **failure means the claim survived**. The referendum reads `ReferendumInfo::Approved(9, ..)` in all three runs. The control is the load bearing part: same referendum, same `unnote_preimage` deposit reclaim, only the agenda differs, and the call executes. `CallUnavailable` reads `false` at block 13 because that is one block before the enactment slot, and `true` once the run is carried through it.

Two things stated plainly so neither is read as more than it is. The harness fills the agenda through the `ScheduleAnon` trait at priority 63 rather than through an extrinsic, because `pallet-scheduler`'s extrinsics are `#[runtime::disable_call]`. That models the governance lane congestion described under Reachability, not a permissionless fill. And the run above sets `CARGO_PROFILE_TEST_DEBUG_ASSERTIONS=false` to match the production profile. With debug assertions left on, the same run instead trips the fork's own guard at `pallets/preimage/src/lib.rs:579`:

```
panicked at pallets/preimage/src/lib.rs:579:
do_unrequest_preimage failed - counter underflow?
```

That `debug_assert!` is the fork's code, not ours, and it fires because the request counter is decremented past what the scheduler ever took.

### What no existing test catches

`pallet-referenda` passes 60/60 and `pallet-scheduler` passes 89/89 with the defect live, and `Referenda::do_try_state` passes inside the failing tests above. The vendor test that locks in the drop, `pallets/scheduler/src/tests.rs:835-860` (`failed_schedule_drops_noted_preimage`), binds the call inside the test with `Preimage::bound`, so it models only a caller that owns a fresh reference. That caller shape is the one for which the drop is correct.

## Fix

The distinction already exists as a parameter. Carry it to the drop side: a caller that did not have its preimage requested by this function must not have it dropped by this function either.

```diff
--- a/pallets/scheduler/src/lib.rs
+++ b/pallets/scheduler/src/lib.rs
@@ do_schedule_inner
 		let when = match Self::resolve_time(when) {
 			Ok(when) => when,
 			Err(e) => {
-				T::Preimages::drop(&call);
+				if request_preimage {
+					T::Preimages::drop(&call);
+				}
 				return Err(e)
 			},
 		};
@@
 		let res = match Self::place_task(when, task) {
 			Ok(res) => res,
 			Err((e, task)) => {
-				T::Preimages::drop(&task.call);
+				if request_preimage {
+					T::Preimages::drop(&task.call);
+				}
 				return Err(e)
 			},
 		};
```

and the same at the three failure paths in `do_schedule_named_inner` (`:942`, `:948`, `:958`).

Note the flag polarity: `request_preimage == true` means "this caller handed over a call whose reference this function must acquire", so on failure there is nothing this function acquired to release. The minimal correct form is therefore to invert the guard relative to the request, or more readably to drop only after a reference has actually been taken. Either form must preserve the existing cleanup for the dispatchable entrypoints, which do own the reference they bound.

Regression tests worth adding:

* an external holder's request survives a failed `do_schedule_named` from a v3 shim caller;
* an approved referendum's proposal is still `Requested` after an enactment attempt that hit `Exhausted` and then succeeded on retry.

Defence in depth: `Referenda::do_try_state` currently checks counts and metadata only. Asserting that every `Ongoing` and `Approved` referendum with a `Lookup` proposal is still requested would have caught this.

## Why this is not EQ-QNT-SCHEDULER-F-06

The audit report `quantus-substrate-audit-20260513` contains `EQ-QNT-SCHEDULER-F-06: Retry cloning does not preserve preimage ownership for lookup-backed tasks`. That finding is about the retry path. Its verbatim quote of the ordinary scheduling paths shows no drop at all:

```
685 | let when = Self::resolve_time(when)?;
688 | let res = Self::place_task(when, task).map_err(|x| x.0)?;
689 | if let Some(hash) = lookup_hash {
690 |   T::Preimages::request(&hash);
```

The code reported here did not exist when the audit was performed. The audit also named this exact error class and recommended removing it:

> The `Err` branch in `schedule_retry` is also unbalanced in the opposite direction: it drops the cloned task even though that clone never acquired its own request in the first place.
>
> Recommendation: \[...] In either model, remove the spurious drop on the failed retry-placement branch.

Repository history shows both halves:

```
69e12df  2026-04-30  Eiger scheduler audit 3 (#557)
                     removes the retry clone drop, adds the rule comment at :1463-1466
96de0cf  2026-07-06  Frame fixes medium low (#617)
                     adds five new T::Preimages::drop calls,
                     hunks at do_schedule_inner and do_schedule_named_inner
```

`git merge-base --is-ancestor 69e12df 96de0cf` confirms `#617` descends from `#557`. The anti pattern was correctly removed, the rule was written down, and it was reintroduced on four other paths sixty seven days later.

## Appendix: complete harness sources

Everything needed to reproduce is in this report. Create the three files at the paths below, add the `mod` lines in A.0, and run the three commands already given. No file is modified except those three `mod` declarations, and no existing test is touched.

### A.0 Wiring

In `pallets/referenda/src/lib.rs`, immediately after line 115 (`use sp_runtime::traits::BlockNumberProvider;`):

```rust
#[cfg(test)]
mod refute_enactment_pin;
#[cfg(test)]
mod refute_enactment_impact;
```

In `pallets/scheduler/src/lib.rs`, immediately after line 65 (`mod benchmarking;`):

```rust
#[cfg(test)]
mod poc_preimage_drop;
```

### A.1 `pallets/referenda/src/refute_enactment_pin.rs`

```rust
//! REFUTATION HARNESS (additive, test-only).
//!
//! Attempts to disprove the claim that a failed `schedule_enactment` attempt
//! destroys the preimage reference that `submit` took at
//! `pallets/referenda/src/lib.rs:597`.
//!
//! End-to-end through the real `pallet_referenda` -> `pallet_scheduler` ->
//! `pallet_preimage` mock. Touches no pre-existing test.

use crate::mock::*;
use crate::*;
use frame_support::{
	assert_ok,
	traits::{schedule::v3::Anon as ScheduleAnon, Bounded, QueryPreimage, StorePreimage},
};
use frame_system::RawOrigin;

/// Fill the agenda at `block` to `MaxScheduledPerBlock` with priority-63 inline tasks.
///
/// Priority 63 is what referenda uses for enactment, so the `LOWEST_PRIORITY`
/// reservation in `push_to_agenda` applies to neither the fillers nor the victim.
fn fill_agenda_at(block: u64) {
	let max = 100u32; // mock `MaxScheduledPerBlock`
	for i in 0..max {
		let filler = RuntimeCall::System(frame_system::Call::remark { remark: vec![i as u8; 4] });
		let bounded = Preimage::bound(filler).unwrap();
		assert!(!bounded.lookup_needed(), "fillers must stay inline");
		<Scheduler as ScheduleAnon<u64, RuntimeCall, OriginCaller>>::schedule(
			DispatchTime::At(block),
			None,
			63,
			RawOrigin::Root.into(),
			bounded,
		)
		.expect("filler must schedule");
	}
	// Sanity: the agenda really is full for a priority-63 task.
	let extra = Preimage::bound(RuntimeCall::System(frame_system::Call::remark {
		remark: vec![9u8; 8],
	}))
	.unwrap();
	assert_eq!(
		<Scheduler as ScheduleAnon<u64, RuntimeCall, OriginCaller>>::schedule(
			DispatchTime::At(block),
			None,
			63,
			RawOrigin::Root.into(),
			extra,
		)
		.unwrap_err(),
		DispatchError::Exhausted,
	);
}

/// Submit a Lookup proposal on the Root track (min_enactment_period = 4) with a
/// fixed, publicly known enactment block, fund it and make it pass.
fn submit_passing_lookup_referendum(hash: <Test as frame_system::Config>::Hash, at: u64) {
	assert_ok!(Referenda::submit(
		RuntimeOrigin::signed(1),
		Box::new(RawOrigin::Root.into()),
		Bounded::Lookup { hash, len: 1 },
		DispatchTime::At(at),
	));
	assert_ok!(Referenda::place_decision_deposit(RuntimeOrigin::signed(2), 0));
	set_tally(0, 100, 0);
}

/// CONTROL: no failed attempt. The pin must survive into `Approved`.
#[test]
fn control_approved_referendum_keeps_the_pin_when_scheduling_succeeds_first_try() {
	ExtBuilder::default().build_and_execute(|| {
		let hash = note_preimage(1);
		submit_passing_lookup_referendum(hash, 100);
		assert!(Preimage::is_requested(&hash), "submit must pin");

		run_to(30);
		assert!(
			matches!(ReferendumInfoFor::<Test>::get(0), Some(ReferendumInfo::Approved(..))),
			"referendum must be Approved: {:?}",
			ReferendumInfoFor::<Test>::get(0)
		);

		let pinned = Preimage::is_requested(&hash);
		println!("\n=== CONTROL (agenda free) ===");
		println!("preimage still requested after Approved: {pinned}");
		assert!(pinned, "control is broken if the pin is already gone without a failed attempt");
	});
}

/// THE REFUTATION ATTEMPT: one failed enactment attempt, then success.
/// If the pin is still in force here, the claim is dead.
#[test]
fn refute_failed_enactment_attempt_does_not_destroy_the_submit_time_pin() {
	ExtBuilder::default().build_and_execute(|| {
		let hash = note_preimage(1);
		submit_passing_lookup_referendum(hash, 100);
		assert!(Preimage::is_requested(&hash), "submit must pin");

		// Make attempt #1 (block 100) fail; attempt #2 (block 101) will succeed.
		fill_agenda_at(100);

		run_to(30);
		assert!(
			matches!(ReferendumInfoFor::<Test>::get(0), Some(ReferendumInfo::Approved(..))),
			"referendum must still reach Approved: {:?}",
			ReferendumInfoFor::<Test>::get(0)
		);

		let pinned = Preimage::is_requested(&hash);
		println!("\n=== REFUTATION ATTEMPT (agenda at the enactment block full) ===");
		println!("preimage still requested after Approved: {pinned}");
		println!("expected if the claim is FALSE: true   |   observed: {pinned}");

		// A live pin also means the noter cannot delete the bytes.
		let _ = <Preimage as StorePreimage>::unnote(&hash);
		let bytes_survive = Preimage::len(&hash).is_some();
		println!("bytes survive the noter's unnote: {bytes_survive}");

		assert!(
			pinned && bytes_survive,
			"CLAIM SURVIVES: pinned={pinned} bytes_survive={bytes_survive}"
		);
	});
}

/// IMPACT VARIANT: the noter reclaims their deposit while the referendum is
/// Ongoing -- which `pallets/referenda/src/lib.rs:578-583` and the vendor test
/// `submit_requires_and_requests_lookup_preimage` both declare safe. Then one
/// enactment attempt fails.
#[test]
fn refute_deposit_reclaim_then_failed_attempt_keeps_the_bytes() {
	ExtBuilder::default().build_and_execute(|| {
		let hash = note_preimage(1);
		submit_passing_lookup_referendum(hash, 100);

		// Documented as safe: the request keeps the bytes alive after the reclaim.
		assert_ok!(Preimage::unnote_preimage(RuntimeOrigin::signed(1), hash));
		assert!(Preimage::len(&hash).is_some(), "vendor invariant: bytes survive the reclaim");

		fill_agenda_at(100);
		run_to(30);
		assert!(
			matches!(ReferendumInfoFor::<Test>::get(0), Some(ReferendumInfo::Approved(..))),
			"referendum must still reach Approved"
		);

		let bytes = Preimage::len(&hash);
		println!("\n=== IMPACT VARIANT (deposit reclaimed, then one failed attempt) ===");
		println!("preimage bytes at enactment time: {bytes:?}");
		println!("expected if the claim is FALSE: Some(1)   |   observed: {bytes:?}");
		assert!(
			bytes.is_some(),
			"CLAIM SURVIVES: the approved proposal's bytes are gone before enactment"
		);
	});
}
```

### A.2 `pallets/scheduler/src/poc_preimage_drop.rs`

````rust
//! PoC: the scheduler releases a preimage reference it never acquired.
//!
//! `do_schedule_inner` / `do_schedule_named_inner` request a preimage reference
//! CONDITIONALLY and only after placement succeeds, but release one
//! UNCONDITIONALLY on every failure path:
//!
//! ```ignore
//!     let res = match Self::place_task(when, task) {
//!         Ok(res) => res,
//!         Err((e, task)) => {
//!             T::Preimages::drop(&task.call);   // unconditional
//!             return Err(e)
//!         },
//!     };
//!     if let Some(hash) = lookup_hash.filter(|_| request_preimage) {
//!         T::Preimages::request(&hash);          // conditional, success-only
//!     }
//! ```
//! (`pallets/scheduler/src/lib.rs:955-965`, and the same shape at `:945-951`
//! for the `resolve_time` failure. Upstream `pallet-scheduler` 40.1.0 has no
//! drop on either path: `let res = Self::place_task(when, task).map_err(|x| x.0)?;`)
//!
//! The fork's own comment at `:793-797` concedes the ownership problem on the
//! REQUEST side and then releases anyway on the FAILURE side:
//!     "callers that hand over an already-`Bounded` call (the v3 trait shims,
//!      benchmarks and tests) cannot tell whether they own a preimage request
//!      for the lookup hash, so keep the historical behavior and
//!      unconditionally request one."
//!
//! `pallet-referenda` is exactly such a caller. It pins the proposal at
//! submission (`pallets/referenda/src/lib.rs:597`) and states the invariant this
//! breaks (`:582-583`): "on approval the scheduler holds its own request from
//! `schedule_named` until dispatch, so the bytes stay pinned through enactment."
//! Its `schedule_enactment` retry loop (`:1105-1119`) calls `schedule_named`
//! once per attempt, so EVERY failed attempt drops referenda's reference.
//!
//! The vendor test `tests::failed_schedule_drops_noted_preimage` (`tests.rs:835`)
//! locks in the drop, but it binds the call INSIDE the test via `Preimage::bound`,
//! so it models only a caller that owns a fresh reference. This file models the
//! other caller shape: an external holder whose long-lived `request` predates the
//! schedule call.
//!
//! Additive only; touches no pre-existing test.

use crate::mock::*;
use crate::*;
use frame_support::assert_err;
use frame_support::traits::{Bounded, QueryPreimage, StorePreimage};

/// A call large enough that `bound` stores it as a `Lookup` preimage.
fn noted_call() -> Bounded<RuntimeCall, <Test as frame_system::Config>::Hashing> {
	let call = RuntimeCall::System(frame_system::Call::remark { remark: vec![0u8; 1024] });
	let bounded = Preimage::bound(call).unwrap();
	assert!(bounded.lookup_needed(), "the call must be noted as a preimage");
	bounded
}

/// A single failed schedule destroys an EXTERNAL holder's pin.
///
/// The pin's whole purpose is to keep the bytes alive after the noter reclaims
/// their deposit (`pallet_preimage` keeps bytes on `unnote` while a request is
/// outstanding). After the failed schedule it no longer does.
#[test]
fn poc_failed_schedule_destroys_an_external_holders_pin() {
	new_test_ext().execute_with(|| {
		run_to_block(1);
		let bounded = noted_call();
		let hash = bounded.lookup_hash().expect("lookup call has a hash");

		// An external holder pins it, exactly as pallet-referenda does at
		// `submit` (referenda/src/lib.rs:597 `T::Preimages::request(&hash)`).
		// The scheduler is NOT this holder and never acquires a reference here.
		<Preimage as QueryPreimage>::request(&hash);
		assert!(Preimage::have(&bounded), "precondition: bytes are present");

		// One failed schedule through the v3 shim path (request_preimage = true).
		// referenda's retry loop performs this call once per attempt.
		assert_err!(
			Scheduler::do_schedule(DispatchTime::At(1), 63, root(), bounded.clone()),
			Error::<Test>::TargetBlockNumberInPast,
		);

		// The decisive observation: is the EXTERNAL holder's pin still in force?
		// Test it the way it matters -- the noter reclaims their deposit. With a
		// live request the bytes must survive; without one they are deleted.
		let _ = <Preimage as StorePreimage>::unnote(&hash);

		let survived = Preimage::have(&bounded);
		println!("\n=== scheduler drops a reference it never acquired ===");
		println!("external holder requested the preimage, then ONE schedule failed");
		println!("bytes still present after the noter un-noted: {survived}");
		println!(
			"expected if the pin held: true   |   observed: {survived}"
		);

		assert!(
			!survived,
			"if the bytes survived, the external pin was NOT destroyed and this lead is dead"
		);
	});
}

/// THE REACHABLE TRIGGER: agenda exhaustion, which is the one `pallet-referenda`
/// actually hits.
///
/// The test above uses a past target block because it is the cheapest way to make
/// `do_schedule` fail, but referenda never schedules into the past
/// (`schedule_enactment` clamps with `.max(earliest_allowed)`,
/// `referenda/src/lib.rs:1100-1102`). Its real failure is `place_task` returning
/// `DispatchError::Exhausted` when the target agenda is full
/// (`scheduler/src/lib.rs:759-767`), and its retry loop calls `schedule_named`
/// once per attempt (`referenda/src/lib.rs:1105-1119`), so each failed attempt
/// runs the unconditional drop at `:955-965`.
///
/// This test drives that exact path.
#[test]
fn poc_agenda_exhaustion_destroys_the_pin_on_referendas_real_path() {
	new_test_ext().execute_with(|| {
		run_to_block(1);
		let target = 5u64;
		let max = <Test as Config>::MaxScheduledPerBlock::get();

		// Fill the target agenda with ordinary inline tasks. Priority 63 is what
		// referenda uses for enactment (`referenda/src/lib.rs:1111`), so the
		// LOWEST_PRIORITY reservation at `:750-758` does not apply to either
		// these fillers or the victim.
		for i in 0..max {
			let filler = RuntimeCall::System(frame_system::Call::remark {
				remark: vec![i as u8; 4],
			});
			let bounded_filler = Preimage::bound(filler).unwrap();
			assert!(
				!bounded_filler.lookup_needed(),
				"fillers must stay inline so they do not perturb the preimage store"
			);
			Scheduler::do_schedule(DispatchTime::At(target), 63, root(), bounded_filler)
				.expect("filler must schedule");
		}

		// The enactment call, pinned by an external holder exactly as referenda
		// pins it at `submit` (`referenda/src/lib.rs:597`).
		let bounded = noted_call();
		let hash = bounded.lookup_hash().expect("lookup call has a hash");
		<Preimage as QueryPreimage>::request(&hash);
		assert!(Preimage::have(&bounded), "precondition: bytes are present");

		// One iteration of referenda's retry loop.
		let name = [7u8; 32];
		let err = Scheduler::do_schedule_named(
			name,
			DispatchTime::At(target),
			63,
			root(),
			bounded.clone(),
		)
		.expect_err("a full agenda must reject the enactment");
		assert_eq!(err, DispatchError::Exhausted, "the trigger must be agenda exhaustion");

		// Did the drop fire on referenda's reference?
		let _ = <Preimage as StorePreimage>::unnote(&hash);
		let survived = Preimage::have(&bounded);

		println!("\n=== referenda's real trigger: agenda exhaustion ===");
		println!("agenda at block {target} filled to MaxScheduledPerBlock = {max}");
		println!("schedule_named rejected with: {err:?}");
		println!("external holder's pin survived the rejection: {survived}");

		assert!(
			!survived,
			"if the bytes survived, exhaustion does not run the drop and the lead is dead"
		);
	});
}

/// Control: without the failed schedule, the same pin does survive `unnote`.
///
/// This is what makes the test above evidence rather than a description of how
/// `unnote` works. Same sequence, minus the scheduler call.
#[test]
fn control_pin_survives_unnote_when_no_schedule_failed() {
	new_test_ext().execute_with(|| {
		run_to_block(1);
		let bounded = noted_call();
		let hash = bounded.lookup_hash().expect("lookup call has a hash");

		<Preimage as QueryPreimage>::request(&hash);
		assert!(Preimage::have(&bounded));

		let _ = <Preimage as StorePreimage>::unnote(&hash);

		let survived = Preimage::have(&bounded);
		println!("\n=== control: no failed schedule ===");
		println!("bytes still present after the noter un-noted: {survived}");
		assert!(
			survived,
			"the pin must survive `unnote` on its own, or the test above proves nothing"
		);
	});
}
````

### A.3 `pallets/referenda/src/refute_enactment_impact.rs`

```rust
//! REFUTATION HARNESS #2 (additive, test-only): does the lost pin actually cost
//! the approved call its execution? Mirrors the vendor's own regression test
//! `full_enactment_block_agenda_retries_on_next_block`, but with a `Bounded::Lookup`
//! proposal instead of an inline one.

use crate::mock::*;
use crate::*;
use frame_support::{
	assert_ok,
	traits::{schedule::v3::Anon as ScheduleAnon, Bounded, QueryPreimage, StorePreimage},
};
use frame_system::RawOrigin;
use sp_runtime::traits::{BlakeTwo256, Hash};

fn fill_agenda_at(block: u64) {
	let mut placed = 0u32;
	for i in 0..200u32 {
		let filler = RuntimeCall::System(frame_system::Call::remark { remark: vec![i as u8; 4] });
		let bounded = Preimage::bound(filler).unwrap();
		assert!(!bounded.lookup_needed());
		match <Scheduler as ScheduleAnon<u64, RuntimeCall, OriginCaller>>::schedule(
			DispatchTime::At(block), None, 63, RawOrigin::Root.into(), bounded,
		) {
			Ok(_) => placed += 1,
			Err(_) => break,
		}
	}
	println!("fillers placed at block {block}: {placed}");
}

/// Note the REAL enactment call as a preimage and return (hash, len).
fn note_real_call() -> (<Test as frame_system::Config>::Hash, u32) {
	let data = set_balance_proposal(1);
	let len = data.len() as u32;
	assert_ok!(Preimage::note_preimage(RuntimeOrigin::signed(1), data.clone()));
	(BlakeTwo256::hash(&data), len)
}

fn submit_and_reclaim(hash: <Test as frame_system::Config>::Hash, len: u32) {
	assert_ok!(Referenda::submit(
		RuntimeOrigin::signed(1),
		Box::new(RawOrigin::Root.into()),
		Bounded::Lookup { hash, len },
		DispatchTime::At(10),
	));
	assert_ok!(Referenda::place_decision_deposit(RuntimeOrigin::signed(2), 0));
	// Deposit reclaim while Ongoing. The vendor documents this as safe: the
	// submit-time request keeps the bytes alive through enactment.
	assert_ok!(Preimage::unnote_preimage(RuntimeOrigin::signed(1), hash));
	assert!(Preimage::len(&hash).is_some(), "bytes must survive the reclaim");
	run_to(6);
	set_tally(0, 100, 0);
}

fn report(tag: &str, hash: <Test as frame_system::Config>::Hash) -> u64 {
	let info = ReferendumInfoFor::<Test>::get(0);
	let bal = Balances::free_balance(&42);
	let unavailable = System::events().iter().any(|e| matches!(
		e.event,
		RuntimeEvent::Scheduler(pallet_scheduler::Event::CallUnavailable { .. })
	));
	let cleared = System::events().iter().any(|e| matches!(
		e.event,
		RuntimeEvent::Preimage(pallet_preimage::Event::Cleared { .. })
	));
	println!("
=== {tag} ===");
	println!("referendum info      : {info:?}");
	println!("balance(42)          : {bal}   (1 = the approved Root call ran)");
	println!("CallUnavailable seen : {unavailable}");
	println!("Preimage::Cleared    : {cleared}");
	println!("preimage bytes now   : {:?}", Preimage::len(&hash));
	assert!(matches!(info, Some(ReferendumInfo::Approved(..))), "must read Approved");
	bal
}

/// POSITIVE CONTROL: identical referendum, identical deposit reclaim, agenda FREE.
#[test]
fn control_lookup_enactment_executes_when_agenda_is_free() {
	ExtBuilder::default().build_and_execute(|| {
		let (hash, len) = note_real_call();
		submit_and_reclaim(hash, len);
		run_to(20);
		let bal = report("CONTROL: agenda at block 13 free", hash);
		assert_eq!(bal, 1, "control broken: the enactment must run");
	});
}

/// THE CLAIM, stopped one block BEFORE enactment: the bytes are already gone.
#[test]
fn refute_full_agenda_only_slides_the_lookup_enactment() {
	ExtBuilder::default().build_and_execute(|| {
		let (hash, len) = note_real_call();
		submit_and_reclaim(hash, len);
		// Preferred enactment block = max(At(10), 9 + min_enactment_period 4) = 13.
		fill_agenda_at(13);
		run_to(13);
		let bal = report("CLAIM: agenda at block 13 full before approval (measured at #13)", hash);
		assert_eq!(bal, 0, "enactment has not run yet at #13");
		assert!(
			Preimage::len(&hash).is_some(),
			"CLAIM SURVIVES: the approved call's bytes are destroyed before enactment",
		);
	});
}

/// The same run carried through the enactment block at #14.
/// Run with `CARGO_PROFILE_TEST_DEBUG_ASSERTIONS=false` (production config): the
/// double drop is silent and the approved Root call simply never executes.
/// In a debug build it instead trips `pallet_preimage`'s "counter underflow" assert.
#[test]
fn claim_carried_through_the_enactment_block() {
	ExtBuilder::default().build_and_execute(|| {
		let (hash, len) = note_real_call();
		submit_and_reclaim(hash, len);
		fill_agenda_at(13);
		run_to(20);
		let bal = report("CLAIM: carried to #20 (enactment slid to #14)", hash);
		assert_eq!(bal, 1, "CLAIM SURVIVES: approved Root call never executed (bal={bal})");
	});
}
```
