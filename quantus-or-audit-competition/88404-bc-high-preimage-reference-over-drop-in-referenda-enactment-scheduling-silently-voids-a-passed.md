# 88404 bc high preimage reference over drop in referenda enactment scheduling silently voids a passed governance decision

**Submitted on Aug 13th 2026 at 01:38:38 UTC by @Ispolin11 for** [**Audit Comp | Quantus**](https://immunefi.com/audit-competition/audit-comp-quantus)

* **Report ID:** #88404
* **Report Type:** Blockchain/DLT
* **Report severity:** High
* **Target:** <https://github.com/immunefi-team/audit-comp-quantus-chain>
* **Impacts:**
  * Manipulation of governance voting result deviating from voted outcome and resulting in a direct change from intended effect of original results
  * A bug in the respective layer 0/1/2 network code that results in unintended smart contract behavior with no concrete funds at direct risk

## Description

## Brief/Intro

Quantus' forked `pallet-scheduler` releases a preimage reference on **every** failed scheduling attempt, including the case where the reference belongs to the *caller* (`pallet-referenda`) rather than to the scheduler. Because referenda schedules a passed referendum's enactment through a Quantus-added 17-attempt retry loop over `schedule_named`, a single attempt that fails against a full block agenda strips the pin that keeps the approved proposal's call bytes available.

The approved call is then left with **zero** outstanding preimage requests, so the noter's ordinary deposit reclaim (`unnote_preimage`) deletes its bytes. On mainnet this **silently voids a tech-collective referendum that already passed at the required threshold** — the chain's sole lane for Root/governance dispatch: at dispatch the scheduler can only emit `CallUnavailable` (indistinguishable from routine scheduler cleanup), the referendum still reads `Approved`, and neither call nor origin is retained, so it cannot be retried.

Recovery requires re-running the full multi-day prepare/decide/confirm cycle, which the same mechanism can defeat again. The corruption needs no attacker (an honest deposit reclaim while the agenda happens to be full is enough) and can also be deliberately triggered and repeated by a single collective member.

## Vulnerability Details

**Referenda owns exactly one preimage reference for a `Lookup` proposal, and releases it exactly once.**

`submit` takes a preimage `request` so a `Lookup` proposal's bytes stay pinned until enactment, even if the noter reclaims their deposit (`pallets/referenda/src/lib.rs:597`, guarded by an existence/length check at `:589-595`; a V12-audit change). That single reference is released by `conclude_ongoing`'s `T::Preimages::drop(proposal)` (`pallets/referenda/src/lib.rs:1706`) on every terminal transition.

**Enactment is scheduled through the v3 `ScheduleNamed` trait, which requests the preimage on success but drops it unconditionally on failure.**

The scheduler's trait entry routes to `do_schedule_named_inner` with `request_preimage = true` (`pallets/scheduler/src/lib.rs:928`):

```rust
fn do_schedule_named(/* ... */) -> Result<TaskAddressOf<T>, DispatchError> {
    // "keep unconditionally requesting the preimage for callers that hand
    //  over an already-`Bounded` call; the dispatchables use do_schedule_named_inner"
    Self::do_schedule_named_inner(id, when, priority, origin, call, true)
}
```

Inside `do_schedule_named_inner`, the **success** path adds the scheduler's *own* reference (gated on `request_preimage`), but **every fallible path** drops unconditionally, ignoring `request_preimage` (`pallets/scheduler/src/lib.rs:942`, `:948`, `:958`):

```rust
// fails here when the target block's agenda is full:
let res = match Self::place_task(when, task) {
    Ok(res) => res,
    Err((e, task)) => {
        T::Preimages::drop(&task.call);   // drops a reference the CALLER (referenda) owns
        return Err(e)
    },
};
// success path — adds the scheduler's own reference, leaving the caller's intact:
if let Some(hash) = lookup_hash.filter(|_| request_preimage) {
    T::Preimages::request(&hash);
}
```

The success path treats referenda as the owner: it re-`request`s its own and leaves referenda's reference untouched, so `conclude_ongoing` still has something to drop. The failure path treats the scheduler as the owner: it releases the caller's reference. The two paths are inconsistent, and the caller is never told.

The scheduler's own comment scopes the drop to *"callers that hand over an already-`Bounded` call"* — which is exactly the class that owns its own reference — yet the drop runs for them anyway.

The trait contract settles which model is correct:

> "NOTE: This will request `call` to be made available"

`frame/support/src/traits/schedule.rs:471`

`schedule_named` *adds* a reference for the scheduler's own use; it does not assume ownership of the caller's. So on failure the scheduler holds no reference of its own to release, and dropping here can only consume the caller's.

**Referenda calls this path up to 17 times in a loop** (`pallets/referenda/src/lib.rs:1077` `MAX_ENACTMENT_SCHEDULE_RETRIES = 16`, `:1105`):

```rust
for _ in 0..=Self::MAX_ENACTMENT_SCHEDULE_RETRIES {          // 0..=16 => 17 attempts
    match T::Scheduler::schedule_named(name, DispatchTime::At(when), None, 63, origin.clone(), call.clone()) {
        Ok(_)  => return Ok(()),
        Err(e) => { last_err = Some(e); when = when.saturating_add(One::one()); }, // slide + retry
    }
}
```

**Result — a single failed first attempt leaves the approved call completely unpinned.**

Accounting on `RequestStatusFor[hash]` for one approval with one failed attempt:

| Step                                      | `RequestStatusFor[hash]`                               |
| ----------------------------------------- | ------------------------------------------------------ |
| noter `note_preimage`                     | `Unrequested { ticket }`                               |
| `submit` → `request`                      | `Requested { count: 1 }`                               |
| attempt #1 fails → scheduler `drop`       | `Unrequested` ← **referenda's pin gone**               |
| attempt #2 succeeds → scheduler `request` | `Requested { count: 1 }` (this one is the scheduler's) |
| `conclude_ongoing` → `drop`               | `Unrequested` ← **scheduler's pin gone**               |

`conclude_ongoing` believes it is releasing the submit-time reference, but that one was already consumed by the failed attempt, so it releases the scheduler's instead. The remaining 15 attempts are no-ops only because `Unrequested` makes `do_unrequest_preimage` return `NotRequested`.

The referendum is now `Approved`, with an enactment task pending, and **nothing** holding its bytes.

**Two ways the bytes are then deleted (either suffices):**

1. **Honest noter, no attacker.** The submit-time pin was added (`#654`) precisely so a noter can reclaim their storage deposit via `unnote_preimage` while the bytes stay pinned through enactment. With the pin gone, that same ordinary, permissionless action silently destroys the approved proposal. If the enactment block's agenda is full for any reason — including normal governance activity — the decision is lost with no attacker and no intent.
2. **Deliberate, single member.** `note_preimage` is permissionless and `submit` only checks that the hash *exists*, never who noted it. A member who noted the blob (routine — the proposal must be on-chain first) then votes and calls `unnote_preimage`, so the entire exploit sequence is attacker-controlled (no honest third party required).

**Reachability of a full agenda.**

`MaxScheduledPerBlock = 50` (`runtime/src/configs/mod.rs:345`); `push_to_agenda` reserves `10` slots against `LOWEST_PRIORITY`, so the permissionless surface (reversible transfers, `LOWEST`) can fill at most 40 — the last 10 need non-`LOWEST` tasks.

Referendum alarms are priority 128, and **every vote** re-arms the voted referendum's alarm at `now + 1`. One member voting on ten ongoing referenda in block `X-1` drops ten priority-128 tasks into block `X`; combined with 40 pre-scheduled reversible transfers, block `X` is full.

`X` is deterministic and public (`status.enactment` in `ReferendumInfoFor`, `min_enactment_period` 1 day), and only the **first** attempt must fail.

**Which proposals are affected.**

`StorePreimage::bound` picks `Inline` for encoded calls ≤128 bytes, where `Preimages::drop` is a no-op, and `Lookup` above that. Calls >64 KiB are rejected at `submit` (`PreimageTooBig`).

The affected window is `Lookup` calls of 128 B – 64 KiB:

* `Utility::batch` / `batch_all`
* Multi-member `TechCollective` changes
* Treasury-spend batches
* `System::set_storage`

Standard `set_code(wasm)` runtime upgrades are **not** affected — the wasm exceeds `MaxProposalSize`, so upgrades go via `authorize_upgrade(code_hash)` (\~40 bytes, `Inline`, immune).

**This is a fresh regression in `HEAD`, not aged code.**

Three ingredients are required, and the last one created the defect:

* Scheduler drop-on-every-fallible-path (`#617`, 2026-07-06)
* Referenda's 17-attempt retry loop (`#630`, 2026-07-25)
* Referenda's submit-time `request` + `conclude_ongoing` drop (`#654`, 2026-08-10)

Commit `3b243b8` ("V12 Vesting (#654)") is `HEAD` itself; among the many changes it bundles is *"fix: pin referendum proposal preimage for its whole lifecycle"*. That change is what first gives referenda a reference the scheduler can steal.

Before it, referenda held no reference and a failed attempt's `drop` was a no-op, so the end state was correct. No prior audit could have covered it.

The project's own full-agenda tests miss it because they all use inline bounded calls (`set_balance_proposal_bounded`), where `Preimages::drop` is a no-op.

## Impact Details

A passed tech-collective referendum — the chain's **sole** lane for Root dispatch (`TechCollectiveTracksInfo`: one track, Root proposals only) — is silently discarded.

This maps to the two selected in-scope impacts:

* **`Manipulation of governance voting result deviating from voted outcome and resulting in a direct change from intended effect of original results`.** The collective votes to enact call *X*; the bug causes *X* to be silently discarded while the recorded result stays `Approved`. The **intended effect of the original result** — enactment of *X* — is directly changed from "*X* executes" to "*X* never executes." In the deliberate variant this is a single actor purposely overriding a decision passed at the required ≥61% threshold — a silent, repeatable veto the governance thresholds are specifically designed to deny any single member.
* **`A bug in the respective layer 0/1/2 network code that results in unintended smart contract behavior with no concrete funds at direct risk`.** This is the exact characterization: L1 pallet code (`referenda` × `scheduler`) produces unintended behavior (an approved proposal is discarded), with no funds at direct risk.

**The team documented this exact failure mode — and this bug re-opens it.**

The `submit` comment added by `#654` (`pallets/referenda/src/lib.rs:574-583`) describes precisely the loss demonstrated here, as the thing the fix was meant to prevent:

> // V12 audit: a lookup proposal whose preimage is missing at enactment is\
> // dropped by the scheduler as terminal (`CallUnavailable`) after the referendum\
> // record has already been replaced with `Approved`, which retains neither the call\
> // nor a retry path — the voted proposal silently disappears.

The fix's stated guarantee is that *"on approval the scheduler holds its own request from `schedule_named` until dispatch, so the bytes stay pinned through enactment"* (`:582-583`).

That is exactly what the enactment-retry over-drop breaks: one failed attempt consumes referenda's submit-time request, `conclude_ongoing` then drops the scheduler's own request, and the enactment call ends with zero references — reproducing verbatim the "`CallUnavailable` … voted proposal silently disappears" outcome the comment set out to eliminate.

The invariant the team wrote down is the one that fails.

**Concrete consequences on mainnet:**

* The governance action never takes effect, yet no event distinguishes it from a normal enactment except `CallUnavailable`. The referendum reads `Approved` forever and retains neither call nor origin, so it cannot be retried on-chain.
* Recovery requires re-noting, re-submitting and re-voting through a full prepare/decide/confirm cycle (≥3 days per `min_enactment_period` and track timings), which the same single-member mechanism can defeat again — an indefinite, low-cost denial of the collective's decisions.
* **Secondary accounting corruption:** pallet-preimage's own `debug_assert!(res.is_ok(), "do_unrequest_preimage failed - counter underflow?")` (`pallets/preimage/src/lib.rs:579`) fires from the scheduler's `service_task` cleanup. A `try-runtime` or debug build **panics inside `on_initialize`**; production release builds compile the check out and fail silently.

**Boundaries, stated honestly (no over-claim):**

* The chain keeps confirming transactions: the `CallUnavailable` arm of `service_task` is graceful, and the only assertion is release-compiled-out.
* No funds are stolen, minted, or frozen. An unexecuted treasury batch simply leaves the funds in the treasury.
* The loss is recoverable by re-proposal.

The damage is to **governance integrity**: a decision passed at the required threshold does not take effect, silently and repeatably.

**On the "privileged actor within attributed privileges" boundary.**

The deliberate variant involves a tech-collective member, but the effect is outside any member's attributed privileges, and that exclusion does not apply — for two independent reasons.

First, a member is granted exactly two powers: to *submit* and to *vote* (`RootOrMemberForTechReferendaOrigin`, `runtime/src/governance/definitions.rs`). The 61%-approval / 60%-support curves (`runtime/src/governance/definitions.rs:119-127`) exist precisely to deny any single member control over the outcome.

What this bug yields — a silent, repeatable veto over a decision the collective already passed at those thresholds — is a capability the governance design deliberately withholds, obtained through a defect rather than a granted privilege. A bug that hands an actor a power the system is built to deny them is the opposite of "within attributed privileges."

Second, the loss needs no privileged actor at all: route 1, an honest noter's permissionless `unnote_preimage` while the enactment block's agenda is full for any reason, including ordinary governance activity, destroys the approved proposal with no attacker and no intent.

This is a correctness / data-loss defect first, and a single-member weapon second.

## References

Asset #1 — `audit-comp-quantus-chain`, branch `audit-comp-ready`, commit `3b243b8` (the audited commit in `SCOPE.md`; current tip of the branch):

* Scheduler unconditional drop on failure — `pallets/scheduler/src/lib.rs:958` (also `:942`, `:948`), and success-only request at `:963-965`
* Scheduler trait entry passing `request_preimage = true` — `pallets/scheduler/src/lib.rs:928`
* Referenda 17-attempt enactment retry loop — `pallets/referenda/src/lib.rs:1105` (`MAX_ENACTMENT_SCHEDULE_RETRIES` at `:1077`)
* Referenda submit-time preimage `request` — `pallets/referenda/src/lib.rs:597` (existence/length guard at `:589-595`)
* Referenda single release point — `conclude_ongoing` → `T::Preimages::drop(proposal)`, `pallets/referenda/src/lib.rs:1706`
* Preimage counter-underflow assertion reached via `service_task` — `pallets/preimage/src/lib.rs:579`
* Agenda cap and `LOWEST_PRIORITY` reservation — `runtime/src/configs/mod.rs:345`; referendum-alarm priority `128` and `now + 1` re-arm on every vote — `pallets/referenda/src/lib.rs:1165` (`set_alarm`), `:936`/`:956` (`access_poll`/`try_access_poll`)
* Regression commit — `3b243b8` "V12 Vesting (#654)" (2026-08-10), which bundles the change *"fix: pin referendum proposal preimage for its whole lifecycle"*; builds on `96de0cf` "Frame fixes medium low (#617)" (2026-07-06) and `5507d6c` "V12 review node (#630)" (2026-07-25)
* v3 scheduler trait contract (an *added*, not transferred, reference): *"NOTE: This will request `call` to be made available"* — `frame/support/src/traits/schedule.rs:471`

## Link to Proof of Concept

<https://gist.github.com/Ispolin11/cc0477c88fb65bc286fc41608a11798d>

## Proof of Concept

Six Rust tests reproduce the finding against the audited tree. They are added to `pallets/referenda/src/tests.rs` (the pallet's own unit-test module, so they reuse its mock runtime, `ExtBuilder`, `run_to`, `set_tally`, etc.).

The complete patch is also in the linked secret gist; the full source is reproduced below so this PoC is self-contained.

### Environment

* **Asset #1** — `audit-comp-quantus-chain`, branch `audit-comp-ready`, commit `3b243b8` (the commit `SCOPE.md` names; current tip).
* **Toolchain** — Rust `1.93.0`, pinned in `rust-toolchain.toml` and selected automatically by `rustup`.
* If the build cannot locate libclang, set `LIBCLANG_PATH` to your `libclang.so` directory; no other setup is required.

### Steps to reproduce

{% stepper %}
{% step %}

## Apply the PoC tests

From the root of an `audit-comp-quantus-chain` checkout at commit `3b243b8`, with the PoC tests applied:

```bash
git apply poc-referenda-preimage-unpin.patch
```

The patch is in the gist. Alternatively, add the source block at the end of this report to `pallets/referenda/src/tests.rs`.
{% endstep %}

{% step %}

## Run the lifecycle tests

```bash
cargo test --release -p pallet-referenda poc_enactment_preimage_unpin
```

{% endstep %}

{% step %}

## Run the isolated-defect tests

```bash
cargo test --release -p pallet-referenda verify_scheduler_preimage_ownership
```

{% endstep %}

{% step %}

## Run the attacker-model bound

```bash
cargo test --release -p pallet-referenda verify_lowest_priority_reservation
```

{% endstep %}

{% step %}

## Run the full pallet suite

```bash
cargo test --release -p pallet-referenda
```

{% endstep %}
{% endstepper %}

{% hint style="warning" %}
**`--release` is required.** pallet-preimage's `debug_assert!(res.is_ok(), "…counter underflow?")` trips on the corrupted accounting, so a *debug* build panics inside `on_initialize` before the impact assertion is reached.

Release compiles that check out and reproduces production behaviour — which is itself part of the finding. The `poc_unpinned_enactment_is_silently_discarded` test is therefore gated `#[cfg(not(debug_assertions))]`.
{% endhint %}

These are pallet unit tests, so they run against pallet-referenda's **mock runtime**. The block numbers below (13 = enactment, 14 = retry) are the mock's timing (`min_enactment_period = 4`); the over-drop mechanism is config-independent — production timing differs but the accounting is identical.

### The attack, step by step

Mapped to `poc_enactment_retry_unpins_approved_proposal_preimage` (the bug) and `poc_unpinned_enactment_is_silently_discarded` (the impact):

{% stepper %}
{% step %}

## Note and submit the proposal

`NOTER` `note_preimage`s a >128-byte call, so its `Bounded` form is a `Lookup` (not `Inline`); a referendum is submitted for it.

`submit` requests the preimage:

```
RequestStatusFor[hash] = Requested { count: 1 }
```

This is referenda's single reference.
{% endstep %}

{% step %}

## Fill the enactment block and approve the referendum

The enactment block (13) is pre-filled to capacity. The referendum passes at block 9, and referenda calls `schedule_named` to place the enactment task.
{% endstep %}

{% step %}

## Fail the first scheduling attempt

Attempt #1 targets block 13:

```
place_task fails (agenda full)
→ forked scheduler error path runs Preimages::drop
→ Unrequested
```

**Referenda's pin is gone.**
{% endstep %}

{% step %}

## Schedule the retry

Attempt #2 slides to block 14 and succeeds:

```
scheduler requests
→ Requested { count: 1 }
```

This reference is the **scheduler's own**.
{% endstep %}

{% step %}

## Conclude the referendum

The referendum record becomes `Approved`.

`conclude_ongoing` runs its single `Preimages::drop`:

```
Unrequested
```

It believes it is releasing the submit-time reference; that one is already gone, so it releases the scheduler's. **The enactment call now has zero references.**
{% endstep %}

{% step %}

## Delete the preimage

`NOTER` reclaims their storage deposit with `unnote_preimage`.

The bytes are **deleted**:

```
QueryPreimage::len(&hash) = None
```

{% endstep %}

{% step %}

## Attempt enactment

At block 14 the scheduler dispatches the enactment task:

```
preimage missing
→ CallUnavailable
```

`ReferendumInfo::Approved` retains neither call nor origin, so there is no retry path. The passed decision is silently lost.
{% endstep %}
{% endstepper %}

The **control** test (`enactment_preimage_stays_pinned_when_first_attempt_succeeds`) runs the identical flow with block 13 *free*: attempt #1 succeeds, status stays `Requested { count: 1 }`, `unnote_preimage` leaves the bytes intact, and enactment dispatches normally — proving the loss is caused solely by the retry.

### Isolation and bounding

* **`verify_scheduler_preimage_ownership`** exercises only `<T::Scheduler as ScheduleNamed>::schedule_named` — the exact call `schedule_enactment` makes — from a state where the caller owns exactly one request, varying nothing but whether the agenda is full:

  | Same start (`Requested { count: 1 }`), same call | Result                                                                |
  | ------------------------------------------------ | --------------------------------------------------------------------- |
  | `schedule_named` **succeeds**                    | `Requested { count: 2 }` — scheduler adds its own, caller's untouched |
  | `schedule_named` **fails**                       | `Unrequested` — scheduler released **the caller's**                   |

  The success case forecloses the "referenda intends to hand ownership over" objection: if it did, the scheduler would not `request` again on success, and `conclude_ongoing` would have nothing to drop.
* **`verify_lowest_priority_reservation_bounds_permissionless_agenda_fill`** confirms the honest limit on the attacker model: `LOWEST_PRIORITY` scheduling saturates at `max - max/5` (40 of 50), so the permissionless surface alone can never fill an agenda — the reserved fifth needs higher-priority tasks (referendum alarms, i.e. votes).

### Expected output

Verified on `3b243b8`, rustc 1.93.0, release:

```
running 3 tests
test tests::poc_enactment_preimage_unpin::enactment_preimage_stays_pinned_when_first_attempt_succeeds ... ok
test tests::poc_enactment_preimage_unpin::poc_enactment_retry_unpins_approved_proposal_preimage ... ok
test tests::poc_enactment_preimage_unpin::poc_unpinned_enactment_is_silently_discarded ... ok
test result: ok. 3 passed; 0 failed; 0 ignored; 0 measured; 63 filtered out

running 2 tests
test tests::verify_scheduler_preimage_ownership::failed_schedule_named_releases_the_callers_request ... ok
test tests::verify_scheduler_preimage_ownership::successful_schedule_named_adds_a_request_and_keeps_the_callers ... ok
test result: ok. 2 passed; 0 failed; 0 ignored; 0 measured; 64 filtered out

running 1 test
test tests::verify_lowest_priority_reservation_bounds_permissionless_agenda_fill ... ok
test result: ok. 1 passed; 0 failed; 0 ignored; 0 measured; 65 filtered out
```

Full suite unaffected:

```
cargo test --release -p pallet-referenda → 66 passed; 0 failed
```

### Honest scope of what the tests prove

The three lifecycle tests fill the enactment block's agenda **directly** (root-scheduled priority-100 tasks) to isolate the defect. They prove that *given a full agenda at the enactment block, one failed retry unpins the approved proposal and its bytes become deletable, and the enactment is then discarded as `CallUnavailable`.*

They do **not** claim the agenda-fill itself is permissionless.

Reachability of a full agenda on mainnet — a member's votes re-arming priority-128 alarms into a target block; permissionless reversible transfers capped at 40 by the reservation — is argued in the Description and bounded by `verify_lowest_priority_reservation` above. It is not re-derived inside these tests.

### Test source

Add the following to the end of `pallets/referenda/src/tests.rs`.

```rust
mod poc_enactment_preimage_unpin {
	use super::*;
	use codec::Encode;
	use frame_support::traits::Bounded;

	const NOTER: u64 = 5;
	/// Approval lands at block 9; mock track 0's `min_enactment_period` is 4 => `max(At(10), 9+4)`.
	const PREFERRED_ENACTMENT: u64 = 13;

	/// Note a >128-byte call so `Bounded` must be a `Lookup`, owned by `NOTER` — the production
	/// shape: somebody `note_preimage`s the proposal, then a referendum is submitted for it.
	fn note_lookup_proposal() -> (BoundedCallOf<Test, ()>, <Test as frame_system::Config>::Hash) {
		let call = RuntimeCall::System(frame_system::Call::remark { remark: vec![7u8; 256] });
		let encoded = call.encode();
		let len = encoded.len() as u32;
		assert!(len > 128, "must exceed BoundedInline's bound so this is a Lookup, not Inline");
		let hash = BlakeTwo256::hash(&encoded);
		assert_ok!(Preimage::note_preimage(RuntimeOrigin::signed(NOTER), encoded));
		(Bounded::Lookup { hash, len }, hash)
	}

	/// Fill block `n`'s agenda with mid-priority tasks (100 != LOWEST_PRIORITY, so the
	/// reservation does not hold them back), exactly as the existing tests in this file do.
	fn fill_agenda(n: u64) {
		let max = <<Test as pallet_scheduler::Config>::MaxScheduledPerBlock as frame_support::traits::Get<
			u32,
		>>::get();
		let filler = RuntimeCall::System(frame_system::Call::remark { remark: vec![] });
		for _ in 0..max {
			assert_ok!(Scheduler::schedule(RuntimeOrigin::root(), n, 100, Box::new(filler.clone())));
		}
	}

	/// Submit referendum 0 for `proposal` and drive it to approval at block 9.
	fn submit_and_approve(proposal: BoundedCallOf<Test, ()>) {
		assert_ok!(Referenda::submit(
			RuntimeOrigin::signed(1),
			Box::new(RawOrigin::Root.into()),
			proposal,
			DispatchTime::At(10),
		));
		assert_ok!(Referenda::place_decision_deposit(RuntimeOrigin::signed(2), 0));
		run_to(6);
		set_tally(0, 100, 0);
		run_to(9);
		assert_eq!(approved_since(0), 9);
	}

	#[allow(dead_code)]
	fn scheduler_saw_call_unavailable() -> bool {
		System::events().iter().any(|r| {
			matches!(
				r.event,
				RuntimeEvent::Scheduler(pallet_scheduler::Event::CallUnavailable { .. })
			)
		})
	}

	/// CONTROL — the intended behaviour, with the preferred enactment block free.
	/// The approved proposal's bytes stay pinned even after the noter reclaims their deposit.
	#[test]
	fn enactment_preimage_stays_pinned_when_first_attempt_succeeds() {
		ExtBuilder::default().build_and_execute(|| {
			let (proposal, hash) = note_lookup_proposal();
			submit_and_approve(proposal);

			// The scheduler holds the only outstanding request; referenda dropped its own.
			assert_matches!(
				pallet_preimage::RequestStatusFor::<Test>::get(hash),
				Some(pallet_preimage::RequestStatus::Requested { count: 1, .. })
			);

			// The noter reclaims their deposit — the bytes MUST survive, because they are
			// still requested by the scheduler.
			assert_ok!(Preimage::unnote_preimage(RuntimeOrigin::signed(NOTER), hash));
			assert!(
				<Preimage as QueryPreimage>::len(&hash).is_some(),
				"pinned preimage must survive unnote_preimage"
			);

			// Enactment dispatches normally at block 13.
			run_to(PREFERRED_ENACTMENT);
			assert!(!scheduler_saw_call_unavailable());
		});
	}

	/// POC — a single failed enactment-scheduling attempt strips the pin.
	///
	/// Block 13's agenda is full, so attempt #1 fails and the forked scheduler's error path
	/// runs `Preimages::drop` on referenda's request. Attempt #2 at block 14 succeeds and takes
	/// the scheduler's own request, which `conclude_ongoing` then immediately drops — believing
	/// it is releasing the submit-time one. Net: the enactment call is left completely unpinned,
	/// and the noter can delete the approved proposal's bytes before it ever runs.
	#[test]
	fn poc_enactment_retry_unpins_approved_proposal_preimage() {
		ExtBuilder::default().build_and_execute(|| {
			let (proposal, hash) = note_lookup_proposal();
			fill_agenda(PREFERRED_ENACTMENT);
			submit_and_approve(proposal);

			// BUG: no outstanding request at all — the referendum was approved, its enactment
			// is scheduled for block 14, yet nothing pins the call any more.
			assert_matches!(
				pallet_preimage::RequestStatusFor::<Test>::get(hash),
				Some(pallet_preimage::RequestStatus::Unrequested { .. }),
				"enactment preimage lost its request reference"
			);

			// The noter now deletes the approved proposal's bytes outright.
			assert_ok!(Preimage::unnote_preimage(RuntimeOrigin::signed(NOTER), hash));
			assert!(
				<Preimage as QueryPreimage>::len(&hash).is_none(),
				"approved proposal's bytes were deleted while still pending enactment"
			);
		});
	}

	/// POC, impact half — the approved call is silently discarded at enactment.
	///
	/// Block 13 was full, so the retry landed at 14; by then the noter has deleted the bytes and
	/// the scheduler can only report `CallUnavailable`. The passed governance decision is lost
	/// with no retry path, because `ReferendumInfo::Approved` retains neither call nor origin.
	///
	/// Only meaningful without debug assertions: `QueryPreimage::unrequest` carries a
	/// `debug_assert!(res.is_ok(), "do_unrequest_preimage failed - counter underflow?")`, and the
	/// scheduler's own `service_task` cleanup trips it on the corrupted accounting. A debug build
	/// therefore panics inside `on_initialize` instead of reaching the assertion below — which is
	/// itself the point: pallet-preimage's invariant check catches the over-drop, and production
	/// (release) builds compile that check out and fail silently.
	#[test]
	#[cfg(not(debug_assertions))]
	fn poc_unpinned_enactment_is_silently_discarded() {
		ExtBuilder::default().build_and_execute(|| {
			let (proposal, hash) = note_lookup_proposal();
			fill_agenda(PREFERRED_ENACTMENT);
			submit_and_approve(proposal);
			assert_ok!(Preimage::unnote_preimage(RuntimeOrigin::signed(NOTER), hash));

			run_to(PREFERRED_ENACTMENT + 1);
			assert!(
				scheduler_saw_call_unavailable(),
				"enactment must have been dropped as CallUnavailable"
			);
		});
	}
}

mod verify_scheduler_preimage_ownership {
	use super::*;
	use codec::Encode;
	use frame_support::traits::Bounded;

	const TARGET: u64 = 50;

	/// Note a >128-byte blob and take ONE request on it, as the caller (referenda) owns.
	fn noted_and_requested() -> (BoundedCallOf<Test, ()>, <Test as frame_system::Config>::Hash, u32) {
		let call = RuntimeCall::System(frame_system::Call::remark { remark: vec![3u8; 256] });
		let encoded = call.encode();
		let len = encoded.len() as u32;
		let hash = BlakeTwo256::hash(&encoded);
		assert_ok!(Preimage::note_preimage(RuntimeOrigin::signed(5), encoded));
		<Preimage as QueryPreimage>::request(&hash);
		assert_matches!(
			pallet_preimage::RequestStatusFor::<Test>::get(hash),
			Some(pallet_preimage::RequestStatus::Requested { count: 1, .. }),
			"precondition: caller owns exactly one request"
		);
		(Bounded::Lookup { hash, len }, hash, len)
	}

	/// The exact trait call `schedule_enactment` makes: `T::Scheduler`'s `ScheduleNamed` impl.
	fn try_schedule(call: BoundedCallOf<Test, ()>) -> Result<(), sp_runtime::DispatchError> {
		<<Test as Config>::Scheduler as ScheduleNamed<u64, RuntimeCall, OriginCaller>>::schedule_named(
			[9u8; 32],
			DispatchTime::At(TARGET),
			None,
			63,
			RawOrigin::Root.into(),
			call,
		)
		.map(|_| ())
	}

	/// BASELINE — on success the scheduler adds its OWN request and leaves the caller's alone.
	/// This pins down the ownership model: referenda still holds a reference afterwards, which
	/// is exactly why `conclude_ongoing` is entitled to drop one later.
	#[test]
	fn successful_schedule_named_adds_a_request_and_keeps_the_callers() {
		ExtBuilder::default().build_and_execute(|| {
			let (proposal, hash, _) = noted_and_requested();
			assert_ok!(try_schedule(proposal));
			assert_matches!(
				pallet_preimage::RequestStatusFor::<Test>::get(hash),
				Some(pallet_preimage::RequestStatus::Requested { count: 2, .. }),
				"scheduler must ADD its own reference (1 caller + 1 scheduler)"
			);
		});
	}

	/// THE DEFECT — on failure the scheduler releases the CALLER's request.
	/// Same starting state, same call, only the agenda differs.
	#[test]
	fn failed_schedule_named_releases_the_callers_request() {
		ExtBuilder::default().build_and_execute(|| {
			let (proposal, hash, _) = noted_and_requested();

			// Fill TARGET's agenda so `place_task` must fail.
			let max = <<Test as pallet_scheduler::Config>::MaxScheduledPerBlock as frame_support::traits::Get<
				u32,
			>>::get();
			let filler = RuntimeCall::System(frame_system::Call::remark { remark: vec![] });
			for _ in 0..max {
				assert_ok!(Scheduler::schedule(
					RuntimeOrigin::root(),
					TARGET,
					100,
					Box::new(filler.clone()),
				));
			}

			assert!(try_schedule(proposal).is_err(), "agenda is full, scheduling must fail");

			// A failed call that scheduled nothing has nonetheless consumed the caller's
			// reference: 1 -> 0. The caller is never told, and still owes one `drop`.
			assert_matches!(
				pallet_preimage::RequestStatusFor::<Test>::get(hash),
				Some(pallet_preimage::RequestStatus::Unrequested { .. }),
				"BUG: failed schedule_named released a reference it never took"
			);
		});
	}
}

#[test]
fn verify_lowest_priority_reservation_bounds_permissionless_agenda_fill() {
	use frame_support::traits::schedule::LOWEST_PRIORITY;

	ExtBuilder::default().build_and_execute(|| {
		const TARGET: u64 = 60;
		let max = <<Test as pallet_scheduler::Config>::MaxScheduledPerBlock as frame_support::traits::Get<
			u32,
		>>::get();
		let reserved = (max / 5).max(1).min(max.saturating_sub(1));
		let cap = max - reserved;
		let filler = RuntimeCall::System(frame_system::Call::remark { remark: vec![] });

		// The permissionless surface saturates at `cap`, not at `max`.
		let mut placed_lowest = 0u32;
		for _ in 0..(max * 2) {
			if Scheduler::schedule(
				RuntimeOrigin::root(),
				TARGET,
				LOWEST_PRIORITY,
				Box::new(filler.clone()),
			)
			.is_err()
			{
				break
			}
			placed_lowest += 1;
		}
		assert_eq!(placed_lowest, cap, "LOWEST_PRIORITY must stop at the reservation boundary");

		// The reserved fifth is still available - but only to higher-priority tasks, which on
		// this chain means referendum alarms (priority 128) or enactment (63).
		for _ in 0..reserved {
			assert_ok!(Scheduler::schedule(
				RuntimeOrigin::root(),
				TARGET,
				100,
				Box::new(filler.clone()),
			));
		}
		// Only now is the agenda genuinely full, and enactment scheduling would fail.
		assert!(Scheduler::schedule(RuntimeOrigin::root(), TARGET, 63, Box::new(filler)).is_err());
	});
}
```
