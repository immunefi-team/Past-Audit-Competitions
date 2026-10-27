# 90369 bc low a failed enactment placement destroys an approved referendum s proposal so the voted root call never executes

**Submitted on Aug 24th 2026 at 17:37:30 UTC by @gzeon for** [**Audit Comp | Quantus**](https://immunefi.com/audit-competition/audit-comp-quantus)

* **Report ID:** #90369
* **Report Type:** Blockchain/DLT
* **Report severity:** Low
* **Target:** <https://github.com/immunefi-team/audit-comp-quantus-chain>
* **Impacts:**
  * Contract fails to deliver promised returns, but doesn't lose value

## Description

## Brief/Intro

When a referendum passes, `schedule_enactment` asks the scheduler to place the enactment task, retrying on later blocks if the target agenda is full. On each failed attempt the Quantus scheduler calls `T::Preimages::drop` on the proposal, but it takes its own preimage reference only after a *successful* placement. On the failure path it therefore drops a reference it never held: the one `Referenda::submit` took to pin the proposal for its whole lifecycle. If the preimage noter has already called `unnote_preimage`, that was the last reference, and the proposal bytes are deleted. A later retry then places the enactment successfully, so the referendum is recorded `Approved` while its call no longer exists. At the enactment block the scheduler finds the call unavailable and terminally removes the task. The vote is recorded as passed and the Root action it authorised never happens. The drop is a Quantus regression: `96de0cf9` (2026-07-06) added it to the two schedule paths two months after `69e12df9` (2026-04-30) had removed the identical drop from `schedule_retry` on an auditor's recommendation. The trigger is a full agenda at the enactment block. Permissionless traffic alone cannot produce one, because the scheduler reserves the top fifth of every agenda against `LOWEST_PRIORITY` tasks and the enactment itself takes one of the reserved slots, so this fires when ten higher-priority tasks already sit on that block rather than on demand.

## Vulnerability Details

### The scheduler drops a reference it never took

```rust
// pallets/scheduler/src/lib.rs:955
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
Ok(res)
```

The `request` happens after `place_task` succeeds. On the failure path the scheduler holds no reference of its own, so the `drop` decrements somebody else's. Upstream `pallet-scheduler` 40.1.0 has no drop on that path:

```rust
// pallet-scheduler-40.1.0/src/lib.rs:1074
let res = Self::place_task(when, task).map_err(|x| x.0)?;

if let Some(hash) = lookup_hash {
    // Request the call to be made available.
    T::Preimages::request(&hash);
}

Ok(res)
```

The intent of the Quantus addition is stated at `pallets/scheduler/src/lib.rs:939`: "drop any preimage noted by `StorePreimage::bound` on every fallible path so a failed schedule cannot leave an unowned, system-requested preimage behind." That is right for a caller that handed over a call the scheduler itself bound and noted. It is wrong for a caller that passes an existing `Bounded::Lookup` it already owns a request on, and that is exactly what referenda does.

The same file states the correct rule 500 lines further down, in `schedule_retry`:

```rust
// pallets/scheduler/src/lib.rs:1462
Err((_, retry_task)) => {
    // Retry placement failed (agenda full). The retry clone was never
    // successfully placed, so it doesn't own a preimage reference.
    // Do NOT drop the retry clone's preimage here - it never acquired one.
    // The caller will drop the original task's preimage.
```

That is the rule `do_schedule_named_inner` and `do_schedule_inner` break. `request_preimage` is `true` on the referenda path because the v3 `Named` trait routes through `do_schedule_named` (`pallets/scheduler/src/lib.rs:1633`), which calls `do_schedule_named_inner(..., true)` (`:928`). The dispatchables instead pass `lookup_already_requested(&call)` (`:439`, `:476`, `:509`), which is what makes the trait path the one that hands the scheduler a reference it does not yet own.

### This is a regression introduced while fixing the audited sibling

Equilibrium's May 2026 review of this pallet reported `EQ-QNT-SCHEDULER-F-06`, "Retry cloning does not preserve preimage ownership for lookup-backed tasks" (Low, `chain#557`), against `schedule_retry`. Its recommendation names the rule this finding breaks:

> The `Err` branch in `schedule_retry` is also unbalanced in the opposite direction: it drops the cloned task even though that clone never acquired its own request in the first place.
>
> Recommendation: \[...] In either model, remove the spurious drop on the failed retry-placement branch.

The same report quotes the schedule paths as they stood at review time, and they have no drop on the failure path:

```rust
// Equilibrium report, pallets/scheduler/src/lib.rs:679-762 as reviewed 2026-05-13
let res = Self::place_task(when, task).map_err(|x| x.0)?;
if let Some(hash) = lookup_hash {
    T::Preimages::request(&hash);
}
```

So `schedule_retry` was fixed, and it now carries the rule in a comment (`pallets/scheduler/src/lib.rs:1462`). The drop it was told to remove was then installed in `do_schedule_inner` and `do_schedule_named_inner`, under a different V12 item (`#162453`, cited in the code at `:963`). `F-06` covered `schedule_retry`. This is the same defect in the two functions the audit had quoted as correct.

Git dates both steps:

```
$ git log --format='%h %ad %s' --date=short -S "Do NOT drop the retry clone" -- pallets/scheduler/src/lib.rs
69e12df9 2026-04-30 Eiger scheduler audit 3 (#557)

$ git log --format='%h %ad %s' --date=short -S "T::Preimages::drop(&task.call);
				return Err(e)" -- pallets/scheduler/src/lib.rs
96de0cf9 2026-07-06 Frame fixes medium low (#617)
```

`69e12df9` carries the audit fix. Two months later `96de0cf9` adds four failure-path drops across `do_schedule_inner` and `do_schedule_named_inner`, including the two at `:942` and `:948` that share the same ownership problem.

### Referenda owns a request and depends on it surviving to enactment

`submit` verifies the lookup preimage exists, bounds its size, and requests it (`pallets/referenda/src/lib.rs:597`). The stated reason is in the source at `:583`: the request holds the bytes through enactment so `unnote` cannot delete them. That also lets the proposer reclaim the preimage deposit while the bytes stay pinned, which is the documented workflow.

After `unnote_preimage`, the referendum's request is the only thing keeping the bytes alive: `RequestStatusFor` is `Requested { maybe_ticket: None, count: 1, maybe_len: Some(len) }`. One `unrequest` takes `count` to zero, and `pallets/preimage/src/lib.rs:459` then removes the bytes outright.

### An `Exhausted` placement failure is handled inside the same call

```rust
// pallets/referenda/src/lib.rs:1105
for _ in 0..=Self::MAX_ENACTMENT_SCHEDULE_RETRIES {
    match T::Scheduler::schedule_named(name, DispatchTime::At(when), None, 63, origin.clone(), call.clone()) {
        Ok(_) => return Ok(()),
        Err(e) => { last_err = Some(e); when = when.saturating_add(One::one()); },
    }
}
```

Sliding forward one block on a full agenda was added so a full block cannot lose an enactment (`#160560`, `#162455`). The retry loop calls the buggy path once per failed attempt, so a single congested block is enough: attempt one at `T` fails and destroys the proposal, attempt two at `T+1` succeeds, and the referendum is committed as `Approved` with a proposal that no longer exists.

### What it takes to reach it, and what does not

`MaxScheduledPerBlock` is 50 (`runtime/src/configs/mod.rs:345`) and `push_to_agenda` reserves the top fifth against `LOWEST_PRIORITY` tasks, so permissionless traffic tops out at 40 slots. The reservation exists for exactly this reason:

```rust
// pallets/scheduler/src/lib.rs:742
// `LOWEST_PRIORITY` tasks (a best-effort, potentially permissionless surface such
// as reversible transfers) may not consume the last ~20% of an agenda, so
// higher-priority tasks such as governance enactment always retain headroom and
// cannot be censored by pre-filling their target block.
if what.priority == schedule::LOWEST_PRIORITY {
    let occupied = agenda.iter().filter(|slot| slot.is_some()).count() as u32;
    let max = T::MaxScheduledPerBlock::get();
    let reserved = (max / 5).max(1).min(max.saturating_sub(1));
    let cap = max.saturating_sub(reserved);
    if occupied >= cap {
        return Err((DispatchError::Exhausted, what));
    }
}
```

With `MaxScheduledPerBlock` at 50 that is `reserved = 10`, `cap = 40`. That guard holds, and it is why this finding is rated where it is. The enactment runs at priority 63, which is exempt from that reservation, so with only the 40 permissionless tasks present it lands in slot 41 and placement succeeds. Reaching the bug needs all 50 slots occupied at the moment `schedule_enactment` runs, which means ten tasks above `LOWEST_PRIORITY` already sitting on the enactment block.

Ordering matters, because the guard tests total occupancy rather than the count of `LOWEST_PRIORITY` tasks. The 40 permissionless tasks have to arrive while occupancy is still under 40; if ten higher-priority tasks land first, permissionless insertion stops at 30 and the block holds 40, leaving headroom.

The higher-priority sources are referendum alarms (priority 128), other referenda's enactments (priority 63), and, since `ScheduleOrigin` is `EnsureRoot` (`runtime/src/configs/mod.rs:353`), any task a passed Root proposal schedules directly. None of the first two can be aimed at a chosen block on short notice: a vote re-arms an alarm to the next block only (`pallets/referenda/src/lib.rs:956`), and the enactment block is at least one `min_enactment_period` after approval, so a vote-driven alarm always arrives after the placement it would have to precede. Ten enactments can share a distant target, but `max_deciding` is 1, so producing them takes about ten days. The undeciding-timeout alarm can be aimed exactly and fires 45 days after submission.

Every deliberate construction of those ten therefore runs through the Technical Collective. This report does not claim one. The trigger it relies on is ordinary scheduling load arriving in the order above, which the retry loop exists to handle gracefully, and the defect is the unbalanced accounting that fires when it does.

### Why the in-tree test misses it

`full_enactment_block_agenda_retries_on_next_block` (`pallets/referenda/src/tests.rs:284`) congests the enactment block and asserts the retry executes on the next one. It passes, because `set_balance_proposal_bounded` produces a `Bounded::Inline` call for a proposal that small. Inline proposals carry their bytes in the task itself and have no preimage reference to lose. The PoC below is that same test with the proposal changed to `Bounded::Lookup`, which is the shape `submit`'s request path exists to serve. `StorePreimage::bound` inlines anything up to the 128-byte `BoundedInline` cap, so small Root calls are `Inline` and unaffected; runtime upgrades and anything larger are `Lookup`.

## Impact Details

**(Blockchain / DLT) Contract fails to deliver promised returns, but doesn't lose value.** The voted Root call is silently discarded. No value moves and nobody gains: the referendum's outcome is destroyed rather than altered.

The chain records `ReferendumInfo::Approved` and emits the approval, and the Root call the collective voted for never runs. The scheduler reaches the enactment block, `Preimages::peek` fails, and the task is terminally removed as `CallUnavailable` (`peek` fails at `pallets/scheduler/src/lib.rs:1294`, event at `:1306`). Nothing re-queues it and the referendum cannot be re-run, because it is no longer `Ongoing`.

The single tech-collective track is the chain's only governance lane and dispatches Root, so the suppressed action can be a runtime upgrade, a treasury spend, a collective membership change, or any other Root call.

The code states the invariant this breaks. At the approval branch, `pallets/referenda/src/lib.rs:1570` reads: "The scheduler has just taken its own request for the enactment call in `schedule_named`, so after we drop the submit-time request the bytes stay pinned until dispatch." That holds when placement succeeds first time. After a failed attempt the scheduler's request is byte-less, because the failed attempt deleted the bytes, and `conclude_ongoing` then drops the submit-time request as designed, leaving neither bytes nor request.

Recovery is possible and is not free. Anyone may `note_preimage` the public bytes before the enactment block, which restores what `Preimages::peek` needs, but the new note is `Unrequested` and carries its noter's deposit, and its noter can `unnote` it again before dispatch. It also depends on somebody noticing a `Cleared` event from `pallet-preimage` and connecting it to a pending enactment.

## References

* `pallets/scheduler/src/lib.rs:958` the drop on the placement-failure path, inside the `match` opening at `:955`
* `pallets/scheduler/src/lib.rs:964` the scheduler's own request, taken only after success, guarded at `:963`
* `pallets/scheduler/src/lib.rs:939` the stated reason for the drop
* `pallets/scheduler/src/lib.rs:748` the `LOWEST_PRIORITY` agenda reservation
* `pallets/scheduler/src/lib.rs:1294`, `:1306` terminal removal on `CallUnavailable`
* `pallet-scheduler-40.1.0/src/lib.rs:1074` upstream, which does not drop on failure
* `pallets/referenda/src/lib.rs:574`-`:583` why the request is taken, `:587`-`:596` the existence and size checks, `:597` the request itself
* `pallets/scheduler/src/lib.rs:1463`-`:1466` the same file stating the rule this breaks
* Equilibrium, *Quantus Network: Substrate and Node Review*, 2026-05-13, `EQ-QNT-SCHEDULER-F-06` (pages 56-57): the audited sibling in `schedule_retry`, its recommendation to remove the drop, and the schedule paths quoted without one
* `pallets/scheduler/src/lib.rs:1633`, `:928` where `request_preimage` becomes `true`, and `:439`, `:476`, `:509` where the dispatchables pass `lookup_already_requested` instead
* `pallets/referenda/src/lib.rs:1105` the enactment retry loop
* `pallets/referenda/src/lib.rs:956` a vote re-arms the referendum alarm for the next block
* `pallets/referenda/src/lib.rs:1165` alarms run at priority 128
* `pallets/referenda/src/tests.rs:284` the in-tree test that covers this path with an inline call
* `pallets/preimage/src/lib.rs:459` the count-one branch, `:465`-`:468` where the bytes and status are removed
* `pallets/preimage/src/lib.rs:359`-`:361`, `:378`-`:380` repopulating a byte-less request
* `pallets/reversible-transfers/src/lib.rs:467` the signed entrypoint, `:776`-`:785` the caller-chosen target, `:818` the `LOWEST_PRIORITY` it schedules at
* `runtime/src/configs/mod.rs:242`, `:345` `MaxActiveReferendaPerAccount`, `MaxScheduledPerBlock`
* `runtime/src/governance/definitions.rs:117` `confirm_period`, `:118` `min_enactment_period`, `:119` `min_approval`

## Proof of Concept

End to end in the referenda pallet's own test harness, which wires the real `pallet-preimage` and the real `pallet-scheduler`. Local only.

What is the mock rather than the runtime, none of which affects the over-drop: `MaxScheduledPerBlock` is 100 here against the runtime's 50; `SubmitOrigin` is `EnsureSigned` rather than members-only and there is no ranked collective; `prepare_period`, `decision_period`, `confirm_period` and `min_enactment_period` are 2 to 4 blocks rather than hours or days; `UndecidingTimeout` is 20; deposits, `MaxActive`, `MaxActivePerAccount`, `MaxQueued` and `MaxProposalSize` differ; the runtime's 16-pending-transfer-per-account cap has no counterpart; and the congestion is placed by Root. Those change how a full agenda is reached and what it costs, not what happens once it is full.

The proposal is also an 11-byte call wrapped as `Bounded::Lookup` by hand. That is legal and it is what makes the path visible in one short test, but a call that small would be `Inline` in production. The shape this reaches in production is any proposal over the 128-byte `BoundedInline` cap, runtime upgrades included.

{% stepper %}
{% step %}

## Setup

```bash
git clone https://github.com/immunefi-team/audit-comp-quantus-chain
cd audit-comp-quantus-chain && git checkout 3b243b870a5442a6e5f443056f67bbba87d472f5
export LIBCLANG_PATH=$(dirname $(find /usr/lib -name 'libclang.so*' | head -1))
```

{% endstep %}

{% step %}

## Add the test

Append to `pallets/referenda/src/tests.rs`:

```rust
// ---------------------------------------------------------------------------
// PoC: a full agenda at the enactment block destroys the approved proposal
// ---------------------------------------------------------------------------
//
// This is `full_enactment_block_agenda_retries_on_next_block` with one change:
// the proposal is `Bounded::Lookup` instead of the inline call that test uses.
// The retry still lands on the next block and the referendum is still recorded
// Approved, but the failed first attempt has already destroyed the preimage the
// referendum owns, so the voted call never executes.

/// How many tasks currently sit on `when`, so the congestion is visible rather
/// than asserted.
fn agenda_len(when: u64) -> usize {
	pallet_scheduler::Agenda::<Test>::get(qp_scheduler::BlockNumberOrTimestamp::BlockNumber(when))
		.iter()
		.filter(|s| s.is_some())
		.count()
}

/// The scheduler's terminal verdict on a task whose call it cannot fetch.
fn call_unavailable_tasks() -> Vec<(u64, u32)> {
	System::events()
		.into_iter()
		.filter_map(|r| match r.event {
			RuntimeEvent::Scheduler(pallet_scheduler::Event::CallUnavailable { task, .. }) =>
				match task.0 {
					qp_scheduler::BlockNumberOrTimestamp::BlockNumber(n) => Some((n, task.1)),
					qp_scheduler::BlockNumberOrTimestamp::Timestamp(_) => None,
				},
			_ => None,
		})
		.collect()
}

#[test]
fn poc_full_enactment_agenda_destroys_a_lookup_proposal() {
	ExtBuilder::default().build_and_execute(|| {
		// A Lookup-backed proposal: note the bytes, then reference them by hash.
		// `StorePreimage::bound` inlines anything up to the 128-byte cap, so this is the
		// shape every proposal too large to inline takes, runtime upgrades included.
		let call =
			RuntimeCall::Balances(pallet_balances::Call::force_set_balance { who: 42, new_free: 1 });
		let encoded = call.encode();
		let hash = BlakeTwo256::hash(&encoded);
		let len = encoded.len() as u32;
		assert_ok!(Preimage::note_preimage(RuntimeOrigin::signed(1), encoded.clone()));
		let proposal: BoundedCallOf<Test, ()> =
			frame_support::traits::Bounded::Lookup { hash, len };
		println!("step 1  noted proposal preimage: {len} bytes, hash {hash:?}");

		assert_ok!(Referenda::submit(
			RuntimeOrigin::signed(1),
			Box::new(RawOrigin::Root.into()),
			proposal,
			DispatchTime::At(10),
		));
		assert_ok!(Referenda::place_decision_deposit(RuntimeOrigin::signed(2), 0));
		println!("step 2  submitted referendum 0, Root track, proposal = force_set_balance(42, 1)");

		// `submit` requested the preimage, so the noter may reclaim the deposit while the
		// bytes stay pinned by the referendum. This is the documented workflow.
		assert_ok!(Preimage::unnote_preimage(RuntimeOrigin::signed(1), hash));
		assert_eq!(
			<Preimage as QueryPreimage>::len(&hash),
			Some(len),
			"before the attack the referendum's own request keeps the proposal bytes alive"
		);
		println!(
			"step 3  noter reclaimed their deposit with unnote_preimage; bytes still pinned by \
			 the referendum's own request, len = {:?}",
			<Preimage as QueryPreimage>::len(&hash)
		);

		// Congest the preferred enactment block, exactly as the in-tree test does.
		let preferred_enactment = 13u64;
		let max = <<Test as pallet_scheduler::Config>::MaxScheduledPerBlock as frame_support::traits::Get<
			u32,
		>>::get();
		let filler = RuntimeCall::System(frame_system::Call::remark { remark: vec![] });
		for _ in 0..max {
			assert_ok!(Scheduler::schedule(
				RuntimeOrigin::root(),
				preferred_enactment,
				100,
				Box::new(filler.clone()),
			));
		}
		println!(
			"step 4  block {preferred_enactment} agenda filled: {} / {max} slots taken",
			agenda_len(preferred_enactment)
		);

		run_to(6);
		set_tally(0, 100, 0);
		run_to(9);

		// The vote succeeded and the chain records it as such.
		assert_eq!(approved_since(0), 9);
		println!("step 5  referendum 0 APPROVED at block {}", approved_since(0));

		// But the failed first placement dropped a request the scheduler never owned,
		// and with the noter already refunded that was the last one holding the bytes.
		assert_eq!(
			<Preimage as QueryPreimage>::len(&hash),
			None,
			"the approved referendum's proposal bytes must have been destroyed"
		);
		println!(
			"step 6  proposal preimage len is now {:?}  <-- the approved referendum's call is gone",
			<Preimage as QueryPreimage>::len(&hash)
		);

		// The retry slid the enactment onto the next block, which is the behaviour the
		// in-tree test asserts. The task is there; its call is not.
		assert_eq!(agenda_len(preferred_enactment + 1), 1);
		println!(
			"step 7  enactment task placed on block {} ({} task), exactly as the in-tree test expects",
			preferred_enactment + 1,
			agenda_len(preferred_enactment + 1)
		);

		// The in-tree inline-call test sees the call execute here.
		run_to(preferred_enactment + 1);
		assert_eq!(Balances::free_balance(&42), 0, "the voted call must not have executed");
		let unavailable = call_unavailable_tasks();
		assert_eq!(
			unavailable,
			vec![(preferred_enactment + 1, 0)],
			"the scheduler must have terminally dropped the task as CallUnavailable"
		);
		println!(
			"step 8  block {}: Scheduler::CallUnavailable for task {:?}; free_balance(42) = {}",
			System::block_number(),
			unavailable[0],
			Balances::free_balance(&42)
		);

		run_to(preferred_enactment + 10);
		assert_eq!(Balances::free_balance(&42), 0, "and it never will");
		assert!(
			matches!(ReferendumInfoFor::<Test>::get(0), Some(ReferendumInfo::Approved(..))),
			"the chain still records the referendum as Approved"
		);
		println!(
			"step 9  block {}: referendum 0 is still recorded Approved, free_balance(42) = {} \
			 -- the voted Root call never ran and never will",
			System::block_number(),
			Balances::free_balance(&42)
		);
	});
}
```

{% endstep %}

{% step %}

## Run and output

Run the PoC alongside the in-tree test it is derived from, so the difference is visible in one command:

```bash
cargo test --release -p pallet-referenda --lib -- --nocapture --test-threads=1 \
  full_enactment_block_agenda_retries_on_next_block \
  poc_full_enactment_agenda_destroys_a_lookup_proposal
```

```
running 2 tests
test tests::full_enactment_block_agenda_retries_on_next_block ... ok
test tests::poc_full_enactment_agenda_destroys_a_lookup_proposal ... step 1  noted proposal preimage: 11 bytes, hash 0x87c261c0fdd64e1df7485405bf5ab632aef316decd3f24cb9959808f23eec7fe
step 2  submitted referendum 0, Root track, proposal = force_set_balance(42, 1)
step 3  noter reclaimed their deposit with unnote_preimage; bytes still pinned by the referendum's own request, len = Some(11)
step 4  block 13 agenda filled: 100 / 100 slots taken
step 5  referendum 0 APPROVED at block 9
step 6  proposal preimage len is now None  <-- the approved referendum's call is gone
step 7  enactment task placed on block 14 (1 task), exactly as the in-tree test expects
step 8  block 14: Scheduler::CallUnavailable for task (14, 0); free_balance(42) = 0
step 9  block 23: referendum 0 is still recorded Approved, free_balance(42) = 0 -- the voted Root call never ran and never will
ok

test result: ok. 2 passed; 0 failed; 0 ignored; 0 measured; 59 filtered out; finished in 0.00s
```

Same congestion, same approval at block 9, same retry onto block 14. The in-tree test's inline proposal executes and sets `free_balance(42)` to 1. The `Bounded::Lookup` proposal is destroyed by the failed first attempt, and `force_set_balance(42, 1)` never runs at block 14 or at any block after it.
{% endstep %}
{% endstepper %}

## Suggested fix

On the placement-failure path, drop only a reference the scheduler owns.

```rust
let res = match Self::place_task(when, task) {
    Ok(res) => res,
    Err((e, task)) => {
        // Only drop what this call noted itself. A caller-supplied `Bounded::Lookup`
        // carries the caller's request, which the scheduler has not taken yet.
        if !request_preimage {
            T::Preimages::drop(&task.call);
        }
        return Err(e)
    },
};
```

The same pattern is in `do_schedule_inner` (`pallets/scheduler/src/lib.rs:828`, `:837`), and the two unconditional drops earlier in `do_schedule_named_inner` itself (`:942` on a `Lookup` name collision, `:948` on a `resolve_time` error) have the same ownership problem. Upstream 40.1.0 does not drop on any of these paths, and `schedule_retry` at `:1462` already documents why.

A regression test belongs next to `full_enactment_block_agenda_retries_on_next_block`, using a `Bounded::Lookup` proposal and an `unnote`d noter, since the existing test's inline call cannot reach this path.
