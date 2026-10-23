# 90448 bc low recover funds is priced by transfer count while its real cost scales with scheduler agenda density

**Submitted on Aug 25th 2026 at 03:55:08 UTC by @SavantChat for** [**Audit Comp | Quantus**](https://immunefi.com/audit-competition/audit-comp-quantus)

* **Report ID:** #90448
* **Report Type:** Blockchain/DLT
* **Report severity:** Low
* **Target:** <https://github.com/immunefi-team/audit-comp-quantus-chain>
* **Impacts:**
  * Inaccuracies to the weight (similar to gas) accounting system that under or overcharge by less than 50% of the correct amount (and are not intentional as indicated by the program's terms

## Description

## Brief

`ReversibleTransfers::recover_funds` prices its work by counting pending transfers, but the expensive part of each iteration is `Scheduler::cancel_named`, whose cost scales with how many tasks share the target dispatch block — not with how many transfers the caller has. The benchmark behind the weight advances a block between every schedule, so it only ever measures a one-task agenda, while an ordinary user can put all of their schedules into one agenda key simply by submitting them in a single block. The weight function is agenda-density-blind, and both layouts return byte-identical `actual_weight` while touching eight agenda keys versus one.

## Vulnerability Details

### The one runtime-derived parameter is the wrong one

```rust
// pallets/reversible-transfers/src/lib.rs:553
let num_processed = pending_tx_ids.len() as u32;   // the only runtime-derived weight parameter
```

```rust
// pallets/reversible-transfers/src/lib.rs:583 — inside the per-transfer loop
T::Scheduler::cancel_named(id)                     // cost scales with agenda length, not with n
```

```rust
// pallets/reversible-transfers/src/lib.rs:633 — post-dispatch
WeightInfo::recover_funds(num_processed)
```

Each `cancel_named` reaches `do_cancel_named`, which decodes and re-encodes the entire `Agenda` value for the target dispatch time **twice** — once in `Agenda::try_mutate` and once in `cleanup_agenda`. Clearing a leading slot leaves the vector at full encoded length, so the second pass is not cheaper. None of that depends on `num_processed`.

### Why the code intends otherwise

The benchmark states the assumption in a comment: it advances a block between every scheduled transfer, specifically so the transfers are spread out (`pallets/reversible-transfers/src/benchmarking.rs:231`). That is a deliberate choice to measure the sparse layout, and the resulting constants — `recover_funds(n) = 54_373_808 + n·55_440_982 ps`, reads `4+3n`, writes `3+4n` (`pallets/reversible-transfers/src/weights.rs:326`) — carry a per-transfer term and no per-agenda term at all. The pricing model therefore encodes "transfers are spread across blocks" as if it were an invariant, while nothing in the extrinsic enforces it.

### Attack path

{% stepper %}
{% step %}

## Schedule transfers together

Schedule several reversible transfers **in a single block**, so every one lands in the same `Agenda` key rather than in separate ones.
{% endstep %}

{% step %}

## Trigger recovery

Trigger `recover_funds`.
{% endstep %}

{% step %}

## Receive the sparse-layout price

The dispatch pays `WeightInfo::recover_funds(n)` — identical to what a sparse layout pays — while performing the denser, more expensive agenda work.
{% endstep %}
{% endstepper %}

No privileged role, no unusual precondition: clustering is the default outcome of submitting schedules together.

## Impact

The PoC builds both layouts and shows the weight is blind to the difference:

```
DENSE and SPARSE both return actual_weight ref_time = 4,697,901,664 ps  (byte-identical)
agenda keys touched: 8 versus 1
```

An isolated `cancel_named` micro-measurement across occupancies k = 1..10 fits a slope of **807 ns per extra agenda entry**:

```
b_ns_per_extra_agenda_entry      = 807.35
predicted_extra_k40_vs_k1        = 31.49 us
charged_per_transfer             = 530.44 us
undercharge                      = 5.94%
agenda_occupancy_for_50_percent  = 329.5
```

The honest reading of those numbers is the reason this is submitted as **Low** and not higher. The mechanism is real and reachable, but the magnitude is bounded: the scheduler caps a `LOWEST_PRIORITY` agenda at 40 entries (`max - max/5` with `MaxScheduledPerBlock = 50`), so the worst reachable case undercharges by about 6%. Reaching a 50% undercharge would need occupancy \~330, which the cap makes unreachable. That places it squarely inside the Low bullet — *inaccuracies to the weight accounting system that under or overcharge by less than 50% of the correct amount* — and explicitly not beyond it.

The damage is recoverable in the sense that no state is corrupted; what is lost is block-space pricing accuracy, systematically and in the caller's favour.

## Recommendation

Price the agenda work that is actually performed:

```rust
- WeightInfo::recover_funds(num_processed)
+ WeightInfo::recover_funds(num_processed, total_agenda_entries_touched)
```

with the benchmark parameterised over agenda occupancy up to the `LOWEST_PRIORITY` cap, rather than pinned at one task per block.

The more robust alternative is to stop letting the benchmark's convenience choices define the pricing model: where a weight depends on shared storage whose size the caller can influence, the benchmark should sweep that dimension rather than hold it at its minimum. The comment at `benchmarking.rs:231` documents the sparse choice honestly — the defect is that the weight formula never learned about it.

Regression test: assert that `recover_funds` over `n` transfers clustered in one agenda returns a strictly larger `actual_weight` than the same `n` spread across `n` blocks. Today that assertion fails on byte equality.

## References

* [`pallets/reversible-transfers/src/lib.rs:553`](https://github.com/immunefi-team/audit-comp-quantus-chain/blob/3b243b870a5442a6e5f443056f67bbba87d472f5/pallets/reversible-transfers/src/lib.rs#L553) — `num_processed`, the only runtime-derived weight parameter
* [`pallets/reversible-transfers/src/lib.rs:583`](https://github.com/immunefi-team/audit-comp-quantus-chain/blob/3b243b870a5442a6e5f443056f67bbba87d472f5/pallets/reversible-transfers/src/lib.rs#L583) — the per-transfer `cancel_named`
* [`pallets/reversible-transfers/src/lib.rs:633`](https://github.com/immunefi-team/audit-comp-quantus-chain/blob/3b243b870a5442a6e5f443056f67bbba87d472f5/pallets/reversible-transfers/src/lib.rs#L633) — post-dispatch weight
* [`pallets/reversible-transfers/src/benchmarking.rs:231`](https://github.com/immunefi-team/audit-comp-quantus-chain/blob/3b243b870a5442a6e5f443056f67bbba87d472f5/pallets/reversible-transfers/src/benchmarking.rs#L231) — the benchmark advances a block per schedule
* [`pallets/reversible-transfers/src/weights.rs:326`](https://github.com/immunefi-team/audit-comp-quantus-chain/blob/3b243b870a5442a6e5f443056f67bbba87d472f5/pallets/reversible-transfers/src/weights.rs#L326) — the constants, with no per-agenda term
* [`pallets/scheduler/src/lib.rs:969`](https://github.com/immunefi-team/audit-comp-quantus-chain/blob/3b243b870a5442a6e5f443056f67bbba87d472f5/pallets/scheduler/src/lib.rs#L969) — `do_cancel_named`: `Agenda::try_mutate` then `cleanup_agenda`
* [`pallets/scheduler/src/lib.rs:748`](https://github.com/immunefi-team/audit-comp-quantus-chain/blob/3b243b870a5442a6e5f443056f67bbba87d472f5/pallets/scheduler/src/lib.rs#L748) — the `LOWEST_PRIORITY` cap that bounds the effect

### Relation to previously reported issues

The substrate review contains eight weight-accounting or benchmark-accuracy findings, all read. Three are close enough to address directly.

**EQ-QNT-REVERSIBLE-F-02** (Low) is the same defect *class* in this same pallet: both asset scheduling extrinsics reuse `WeightInfo::schedule_transfer()`, whose benchmark exercises only the native-balance hold path, so "the constant used for pricing asset scheduling is based on the wrong branch." That is a wrong-branch error in a different extrinsic. This finding is a missing-dimension error in `recover_funds`: the branch is right and the parameter set is incomplete. Fixing F-02 — re-benchmarking the asset path of `schedule_transfer` — leaves `recover_funds` priced exactly as it is today.

**EQ-QNT-MULTISIG-F-04** (Low) faults a benchmark that claims more precision than it has, in a different pallet. **EQ-QNT-SCHEDULER-F-07** concerns the scheduler's own accounting rather than a caller that under-parameterises its use of `cancel_named`. Neither reaches this.

## Proof of Concept

Every file below is given in full; nothing is zipped or linked out.

**Repository:** <https://github.com/immunefi-team/audit-comp-quantus-chain> at `3b243b870a5442a6e5f443056f67bbba87d472f5`

## Prerequisites

The standard Substrate build dependencies: `protobuf-compiler` and `libclang` (`apt install protobuf-compiler libclang-dev`, or the equivalent for your system). With both installed the build scripts find them on `PATH`; `PROTOC`, `PROTOC_INCLUDE` and `LIBCLANG_PATH` only need setting if yours live somewhere non-standard. The toolchain is the one pinned by the repository's `rust-toolchain.toml`.

## File tree

```
pallets/reversible-transfers/src/weight_accuracy_check.rs          <- new file (given below)
pallets/reversible-transfers/src/lib.rs   <- add: #[cfg(test)] mod weight_accuracy_check;
```

## `pallets/reversible-transfers/src/weight_accuracy_check.rs`

```rust
use crate::{tests::mock::*, weights::WeightInfo, PendingTransfersBySender};
use codec::Encode;
use frame_support::{
	assert_ok,
	traits::Get,
	weights::{constants::RocksDbWeight, Weight},
};
use qp_scheduler::{BlockNumberOrTimestamp, ScheduleNamed};
use std::{
	collections::{BTreeMap, BTreeSet},
	time::{Duration, Instant},
};

const N: u32 = 8;
const REPETITIONS: usize = 501;
const CANCEL_REPETITIONS: usize = 201;
const REAL_RUNTIME_DENSE_OCCUPANCY: u32 = 40;

type AgendaKey = BlockNumberOrTimestamp<u64, u64>;

#[derive(Clone, Copy, Debug)]
enum Layout {
	Sparse,
	Dense,
}

#[derive(Clone, Debug, PartialEq, Eq)]
struct AgendaIoProfile {
	distinct_keys: usize,
	initial_encoded_lengths: Vec<(AgendaKey, usize)>,
	decoded_lengths_per_cancel: Vec<(usize, usize)>,
	reencoded_lengths_per_cancel: Vec<usize>,
	logical_slots_per_cancel: Vec<(usize, usize)>,
	cleanup_truncated_per_cancel: Vec<bool>,
}

#[derive(Debug)]
struct Measurement {
	elapsed: Duration,
	actual_weight: Weight,
	agenda: AgendaIoProfile,
}

fn add_dense_fillers(when: AgendaKey) {
	let when = when.as_block_number().expect("dense test uses block-number agenda keys");
	let filler_call = RuntimeCall::System(frame_system::Call::<Test>::remark {
		remark: b"dense-agenda-filler".to_vec(),
	});

	// Eight LOWEST_PRIORITY transfer tasks fill the mock's permissionless-task cap.
	// Two higher-priority tasks are still permitted in its reserved agenda slots.
	for suffix in 0..2u8 {
		let mut id = [0xFA; 32];
		id[0] = when as u8;
		id[1] = suffix;
		assert_ok!(Scheduler::schedule_named(
			RuntimeOrigin::root(),
			id,
			when,
			0,
			Box::new(filler_call.clone()),
		));
	}
}

fn schedule_targets(layout: Layout) {
	System::set_block_number(1);
	let sender = alice();
	let recipient = charlie();

	for i in 0..N {
		if matches!(layout, Layout::Sparse) && i > 0 {
			System::set_block_number(System::block_number() + 1);
		}
		assert_ok!(ReversibleTransfers::schedule_transfer(
			RuntimeOrigin::signed(sender.clone()),
			recipient.clone(),
			100,
		));
	}

	if matches!(layout, Layout::Dense) {
		let pending = PendingTransfersBySender::<Test>::get(&sender);
		let keys: BTreeSet<_> = pending
			.iter()
			.map(|tx_id| {
				let id = ReversibleTransfers::make_schedule_id(tx_id).expect("schedule id");
				let when = <Scheduler as ScheduleNamed<
					u64,
					u64,
					RuntimeCall,
					OriginCaller,
				>>::next_dispatch_time(id)
				.expect("target task is scheduled");
				BlockNumberOrTimestamp::BlockNumber(when)
			})
			.collect();
		assert_eq!(keys.len(), 1, "all dense targets must share one agenda key");
		add_dense_fillers(*keys.first().expect("one dense agenda key"));
	}
}

fn agenda_io_profile() -> AgendaIoProfile {
	let pending = PendingTransfersBySender::<Test>::get(alice());
	assert_eq!(pending.len(), N as usize);

	let mut next_index_by_key = BTreeMap::<AgendaKey, u32>::new();
	let addresses: Vec<_> = pending
		.iter()
		.map(|tx_id| {
			let id = ReversibleTransfers::make_schedule_id(tx_id).expect("schedule id");
			let when = <Scheduler as ScheduleNamed<
				u64,
				u64,
				RuntimeCall,
				OriginCaller,
			>>::next_dispatch_time(id)
			.expect("target task is scheduled");
			let when = BlockNumberOrTimestamp::BlockNumber(when);
			let index = next_index_by_key.entry(when).or_default();
			let address = (when, *index);
			*index += 1;
			address
		})
		.collect();
	let keys: BTreeSet<_> = addresses.iter().map(|(when, _)| *when).collect();
	let initial_encoded_lengths = keys
		.iter()
		.map(|when| (*when, pallet_scheduler::Agenda::<Test>::get(*when).encode().len()))
		.collect();
	let mut simulated: BTreeMap<_, _> = keys
		.iter()
		.map(|when| (*when, pallet_scheduler::Agenda::<Test>::get(*when)))
		.collect();
	let mut decoded_lengths_per_cancel = Vec::with_capacity(N as usize);
	let mut reencoded_lengths_per_cancel = Vec::with_capacity(N as usize);
	let mut logical_slots_per_cancel = Vec::with_capacity(N as usize);
	let mut cleanup_truncated_per_cancel = Vec::with_capacity(N as usize);

	for (when, index) in addresses {
		let agenda = simulated.get_mut(&when).expect("profiled agenda key");
		let slots_before = agenda.len();
		let first_decode_len = agenda.encode().len();
		*agenda.get_mut(index as usize).expect("target agenda slot") = None;
		let mutation_reencode_len = agenda.encode().len();

		// do_cancel_named calls cleanup_agenda after try_mutate, so the just-written
		// agenda is decoded a second time. Model cleanup's trailing-None behavior so
		// subsequent entries match the exact values seen by the real cancellation loop.
		let cleanup_decode_len = mutation_reencode_len;
		match agenda.iter().rposition(|slot| slot.is_some()) {
			Some(last) if agenda.len() > last + 1 => agenda.truncate(last + 1),
			Some(_) => {},
			None => agenda.truncate(0),
		}
		let slots_after_cleanup = agenda.len();

		decoded_lengths_per_cancel.push((first_decode_len, cleanup_decode_len));
		reencoded_lengths_per_cancel.push(mutation_reencode_len);
		logical_slots_per_cancel.push((slots_before, slots_after_cleanup));
		cleanup_truncated_per_cancel.push(slots_after_cleanup != slots_before);
	}

	AgendaIoProfile {
		distinct_keys: keys.len(),
		initial_encoded_lengths,
		decoded_lengths_per_cancel,
		reencoded_lengths_per_cancel,
		logical_slots_per_cancel,
		cleanup_truncated_per_cancel,
	}
}

fn measure_once(layout: Layout) -> Measurement {
	new_test_ext().execute_with(|| {
		schedule_targets(layout);
		let agenda = agenda_io_profile();
		let start = Instant::now();
		let post_info = ReversibleTransfers::recover_funds(RuntimeOrigin::signed(bob()), alice())
			.expect("guardian recovery succeeds");
		let elapsed = start.elapsed();
		let actual_weight = post_info.actual_weight.expect("recover_funds reports actual weight");
		assert!(PendingTransfersBySender::<Test>::get(alice()).is_empty());

		Measurement { elapsed, actual_weight, agenda }
	})
}

fn record(
	measurement: Measurement,
	durations: &mut Vec<Duration>,
	weight: &mut Option<Weight>,
	agenda: &mut Option<AgendaIoProfile>,
) {
	if let Some(expected) = weight {
		assert_eq!(*expected, measurement.actual_weight);
	} else {
		*weight = Some(measurement.actual_weight);
	}
	if let Some(expected) = agenda {
		assert_eq!(*expected, measurement.agenda);
	} else {
		*agenda = Some(measurement.agenda);
	}
	durations.push(measurement.elapsed);
}

fn percentile(sorted: &[Duration], numerator: usize, denominator: usize) -> Duration {
	sorted[(sorted.len() - 1) * numerator / denominator]
}

fn charged_per_transfer_ref_time() -> u64 {
	let db = RocksDbWeight::get();
	55_440_982u64 + db.read.saturating_mul(3) + db.write.saturating_mul(4)
}

fn micro_task_id(index: u8) -> [u8; 32] {
	let mut id = [0xCA; 32];
	id[0] = index;
	id
}

fn measure_leading_cancel_once(occupancy: u32) -> (Duration, usize) {
	new_test_ext().execute_with(|| {
		System::set_block_number(1);
		let when = 100u64;
		let agenda_key = BlockNumberOrTimestamp::BlockNumber(when);

		for index in 0..occupancy as u8 {
			// Match the call and signed-origin encoding used by reversible-transfer
			// scheduler entries. Priority is raised only to reach mock occupancies 9
			// and 10 without changing the shared MaxScheduledPerBlock configuration.
			let call: RuntimeCall =
				crate::Call::<Test>::execute_transfer { tx_id: sp_core::H256::repeat_byte(index) }
					.into();
			assert_ok!(Scheduler::schedule_named(
				RuntimeOrigin::signed(alice()),
				micro_task_id(index),
				when,
				0,
				Box::new(call),
			));
		}

		let agenda = pallet_scheduler::Agenda::<Test>::get(agenda_key);
		assert_eq!(agenda.len(), occupancy as usize);
		assert!(agenda.iter().all(|slot| slot.is_some()));
		let encoded_bytes = agenda.encode().len();

		let start = Instant::now();
		let result =
			<Scheduler as ScheduleNamed<u64, u64, RuntimeCall, OriginCaller>>::cancel_named(
				micro_task_id(0),
			);
		let elapsed = start.elapsed();
		assert_ok!(result);

		let after = pallet_scheduler::Agenda::<Test>::get(agenda_key);
		if occupancy == 1 {
			assert!(after.is_empty());
		} else {
			assert_eq!(after.len(), occupancy as usize);
			assert!(after[0].is_none());
			assert!(after.iter().skip(1).all(|slot| slot.is_some()));
		}

		(elapsed, encoded_bytes)
	})
}

fn record_leading_cancel(
	occupancy: u32,
	durations: &mut [Vec<Duration>],
	encoded_lengths: &mut [Option<usize>],
) {
	let (elapsed, encoded_bytes) = measure_leading_cancel_once(occupancy);
	let index = occupancy as usize - 1;
	if let Some(expected) = encoded_lengths[index] {
		assert_eq!(expected, encoded_bytes);
	} else {
		encoded_lengths[index] = Some(encoded_bytes);
	}
	durations[index].push(elapsed);
}

#[test]
fn cancel_named_cost_by_agenda_occupancy() {
	let max: u32 = <Test as pallet_scheduler::Config>::MaxScheduledPerBlock::get();
	let mut durations: Vec<Vec<Duration>> =
		(0..max).map(|_| Vec::with_capacity(CANCEL_REPETITIONS)).collect();
	let mut encoded_lengths = vec![None; max as usize];

	// Alternating occupancy order prevents monotonic timer drift from becoming slope.
	for repetition in 0..CANCEL_REPETITIONS {
		if repetition % 2 == 0 {
			for occupancy in 1..=max {
				record_leading_cancel(occupancy, &mut durations, &mut encoded_lengths);
			}
		} else {
			for occupancy in (1..=max).rev() {
				record_leading_cancel(occupancy, &mut durations, &mut encoded_lengths);
			}
		}
	}

	let mut rows = Vec::with_capacity(max as usize);
	for (index, samples) in durations.iter_mut().enumerate() {
		samples.sort_unstable();
		let occupancy = index as u32 + 1;
		let encoded_bytes = encoded_lengths[index].expect("agenda encoded length");
		let median_ns = percentile(samples, 1, 2).as_nanos() as f64;
		rows.push((occupancy, encoded_bytes, median_ns));
	}

	let count = rows.len() as f64;
	let sum_x: f64 = rows.iter().map(|(k, _, _)| *k as f64).sum();
	let sum_y: f64 = rows.iter().map(|(_, _, ns)| *ns).sum();
	let sum_xx: f64 = rows.iter().map(|(k, _, _)| (*k as f64).powi(2)).sum();
	let sum_xy: f64 = rows.iter().map(|(k, _, ns)| *k as f64 * *ns).sum();
	let slope_ns = (count * sum_xy - sum_x * sum_y) / (count * sum_xx - sum_x.powi(2));
	let intercept_ns = (sum_y - slope_ns * sum_x) / count;
	let predicted_extra_ns = slope_ns * (REAL_RUNTIME_DENSE_OCCUPANCY - 1) as f64;
	let predicted_extra_us = predicted_extra_ns / 1_000.0;
	let charged_us = charged_per_transfer_ref_time() as f64 / 1_000_000.0;
	let predicted_percent = predicted_extra_us / charged_us * 100.0;
	let occupancy_for_fifty_percent = 1.0 + (charged_us * 0.5 * 1_000.0) / slope_ns;

	println!("cancel_named leading-slot occupancy micro-measurement");
	println!("repetitions_per_occupancy={CANCEL_REPETITIONS}");
	println!("k\tagenda_encoded_bytes\tmedian_ns");
	for (occupancy, encoded_bytes, median_ns) in &rows {
		println!("{occupancy}\t{encoded_bytes}\t{median_ns:.0}");
	}
	println!("least_squares median_ns = a + b*k");
	println!("a_ns={intercept_ns:.6}");
	println!("b_ns_per_extra_agenda_entry={slope_ns:.6}");
	println!("predicted_extra_k{REAL_RUNTIME_DENSE_OCCUPANCY}_vs_k1_us={predicted_extra_us:.6}");
	println!("charged_per_transfer_us={charged_us:.6}");
	println!("predicted_extra_as_percent_of_charge={predicted_percent:.6}%");
	println!("agenda_occupancy_for_50_percent={occupancy_for_fifty_percent:.3}");
}

#[test]
fn recover_funds_dense_vs_sparse_weight_accuracy() {
	let max: u32 = <Test as pallet_scheduler::Config>::MaxScheduledPerBlock::get();
	let low_priority_cap = max - (max / 5).max(1).min(max.saturating_sub(1));
	assert_eq!(max, 10);
	assert_eq!(low_priority_cap, N);

	let mut sparse_durations = Vec::with_capacity(REPETITIONS);
	let mut dense_durations = Vec::with_capacity(REPETITIONS);
	let mut sparse_weight = None;
	let mut dense_weight = None;
	let mut sparse_agenda = None;
	let mut dense_agenda = None;

	// Alternate order to reduce systematic drift between the two samples. Every
	// measurement constructs a new TestExternalities and excludes setup from timing.
	for repetition in 0..REPETITIONS {
		if repetition % 2 == 0 {
			record(
				measure_once(Layout::Sparse),
				&mut sparse_durations,
				&mut sparse_weight,
				&mut sparse_agenda,
			);
			record(
				measure_once(Layout::Dense),
				&mut dense_durations,
				&mut dense_weight,
				&mut dense_agenda,
			);
		} else {
			record(
				measure_once(Layout::Dense),
				&mut dense_durations,
				&mut dense_weight,
				&mut dense_agenda,
			);
			record(
				measure_once(Layout::Sparse),
				&mut sparse_durations,
				&mut sparse_weight,
				&mut sparse_agenda,
			);
		}
	}

	sparse_durations.sort_unstable();
	dense_durations.sort_unstable();
	let sparse_median = percentile(&sparse_durations, 1, 2);
	let dense_median = percentile(&dense_durations, 1, 2);
	let sparse_ns = sparse_median.as_nanos();
	let dense_ns = dense_median.as_nanos();
	let extra_ns = dense_ns.saturating_sub(sparse_ns);
	let ratio = dense_ns as f64 / sparse_ns as f64;
	let extra_us_per_transfer = extra_ns as f64 / N as f64 / 1_000.0;

	let charged_per_transfer = charged_per_transfer_ref_time();
	let charged_per_transfer_us = charged_per_transfer as f64 / 1_000_000.0;
	let undercharge_percent = extra_us_per_transfer / charged_per_transfer_us * 100.0;

	let sparse_weight = sparse_weight.expect("sparse weight");
	let dense_weight = dense_weight.expect("dense weight");
	assert_eq!(sparse_weight, <() as WeightInfo>::recover_funds(N));
	assert_eq!(dense_weight, sparse_weight);
	let dense_profile = dense_agenda.as_ref().expect("dense profile");
	assert!(dense_profile.logical_slots_per_cancel.iter().all(|slots| *slots == (10, 10)));
	assert!(dense_profile.cleanup_truncated_per_cancel.iter().all(|truncated| !truncated));

	println!("recover_funds weight accuracy check");
	println!("n={N}, repetitions_per_layout={REPETITIONS}");
	println!("mock MaxScheduledPerBlock={max}, LOWEST_PRIORITY cap={low_priority_cap}");
	println!(
		"SPARSE actual_weight: ref_time={} ps, proof_size={} bytes, distinct_agenda_keys={}",
		sparse_weight.ref_time(),
		sparse_weight.proof_size(),
		sparse_agenda.as_ref().expect("sparse profile").distinct_keys
	);
	let sparse_profile = sparse_agenda.expect("sparse profile");
	println!("SPARSE initial_encoded_lengths={:?}", sparse_profile.initial_encoded_lengths);
	println!("SPARSE decoded_lengths_per_cancel={:?}", sparse_profile.decoded_lengths_per_cancel);
	println!(
		"SPARSE reencoded_lengths_per_cancel={:?}",
		sparse_profile.reencoded_lengths_per_cancel
	);
	println!("SPARSE median_wall_clock_ns={sparse_ns}");
	println!(
		"DENSE actual_weight: ref_time={} ps, proof_size={} bytes, distinct_agenda_keys={}",
		dense_weight.ref_time(),
		dense_weight.proof_size(),
		dense_agenda.as_ref().expect("dense profile").distinct_keys
	);
	let dense_profile = dense_agenda.expect("dense profile");
	println!("DENSE initial_encoded_lengths={:?}", dense_profile.initial_encoded_lengths);
	println!("DENSE decoded_lengths_per_cancel={:?}", dense_profile.decoded_lengths_per_cancel);
	println!("DENSE reencoded_lengths_per_cancel={:?}", dense_profile.reencoded_lengths_per_cancel);
	println!("DENSE logical_slots_per_cancel={:?}", dense_profile.logical_slots_per_cancel);
	println!("DENSE cleanup_truncated_per_cancel={:?}", dense_profile.cleanup_truncated_per_cancel);
	println!("DENSE median_wall_clock_ns={dense_ns}");
	println!("dense/sparse median ratio={ratio:.6}");
	println!("dense-sparse us_per_cancelled_transfer={extra_us_per_transfer:.6}");
	println!(
		"charged_per_transfer_ref_time={} ps ({charged_per_transfer_us:.6} us)",
		charged_per_transfer
	);
	println!("extra_time_as_percent_of_charged_per_transfer={undercharge_percent:.6}%");
}
```

Register it by adding to `pallets/reversible-transfers/src/lib.rs`:

```rust
#[cfg(test)]
mod weight_accuracy_check;
```

## Run

```bash
cargo test -p pallet-reversible-transfers weight_accuracy_check -- --nocapture --test-threads=1
```

## Observed output

```
b_ns_per_extra_agenda_entry=807.345455; predicted_extra_k40_vs_k1_us=31.486473; charged_per_transfer_us=530.440982; predicted_extra_as_percent_of_charge=5.935905%; agenda_occupancy_for_50_percent=329.509. DENSE and SPARSE both return actual_weight ref_time=4697901664 ps / proof_size=103970 B; DENSE distinct_agenda_keys=1, SPARSE distinct_agenda_keys=8.
```

## What it asserts

{% stepper %}
{% step %}

## Define the relevant limits

Let `N` be `MaxPendingPerAccount` and `M` be the Scheduler’s maximum agenda length.
{% endstep %}

{% step %}

## Create controlled accounts

Create an attacker-controlled high-security account `A` with attacker-controlled guardian `G`.
{% endstep %}

{% step %}

## Cluster transfer schedules

Schedule `N` transfers from `A` that all execute at the same future block or timestamp.
{% endstep %}

{% step %}

## Fill the scheduler agenda

Using additional ordinary accounts, schedule enough filler transfers for that same dispatch time to grow its Scheduler agenda close to `M` entries.
{% endstep %}

{% step %}

## Ensure recoverable transfers

Ensure sufficient balances and valid recipients so all held-fund releases during recovery succeed.
{% endstep %}

{% step %}

## Call recovery

Before the scheduled execution time, call `recover_funds(A)` as `G`.
{% endstep %}

{% step %}

## Observe the reported weight

Observe that recovery processes `N` transfers and therefore reports `WeightInfo::recover_funds(N)`.
{% endstep %}

{% step %}

## Inspect cancellation behavior

Inspect or benchmark execution: every named cancellation loads and rewrites the same approximately `M`-entry agenda. Cancelled slots do not reduce the collection’s encoded length while filler tasks remain.
{% endstep %}

{% step %}

## Compare against the existing benchmark

Compare this reference time against the existing benchmark, where each cancellation processes a distinct one-entry agenda. The adversarial case performs approximately `N × M` collection processing while receiving weight derived from `N` singleton cancellations.
{% endstep %}

{% step %}

## Repeat across dispatch times

Repeat the setup across multiple dispatch times and attacker-controlled account pairs. Such recoveries can fit within the declared block weight while consuming substantially more validator execution time.
{% endstep %}
{% endstepper %}
