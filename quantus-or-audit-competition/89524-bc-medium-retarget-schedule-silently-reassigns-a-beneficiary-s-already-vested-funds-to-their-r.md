# 89524 bc medium retarget schedule silently reassigns a beneficiary s already vested funds to their replacement

**Submitted on Aug 19th 2026 at 18:12:10 UTC by @ilov3c4ndy for** [**Audit Comp | Quantus**](https://immunefi.com/audit-competition/audit-comp-quantus)

* **Report ID:** #89524
* **Report Type:** Blockchain/DLT
* **Report severity:** Medium
* **Target:** <https://github.com/immunefi-team/audit-comp-quantus-chain>
* **Impacts:**
  * Theft of unclaimed yield
  * Griefing (e.g. no profit motive for an attacker, but damage to the users or the protocol)

## Description

**Target:** Substrate L1 node, `pallet-vesting` (runtime index 22)

**Severity:** Medium. Loss of vested user funds, in violation of the invariant the function documents for itself. The vulnerable state is reachable and holdable permissionlessly by an unrelated account; the final step is a routine administrative action. Per schedule the loss is bounded by one `MinClaimInterval` of vesting, and it multiplies across a grantee's schedules.

## Summary

`retarget_schedule` is documented to pay out whatever the outgoing beneficiary could currently claim before swapping in the new one, precisely so the result does not depend on transaction ordering. It does not. When the outgoing beneficiary has claimed within `MinClaimInterval`, the settlement is skipped and their accrued entitlement transfers to the incoming beneficiary.

On the live runtime `MinClaimInterval` is `MILLIS_PER_DAY`, so the window is a full day.

The beneficiary does not have to be an active claimer for this to bite. `claim` is permissionless by design, so any account can stamp `last_claim_at` on any schedule. An unrelated third party who gains nothing from the outcome can therefore put a schedule into the vulnerable window, and keep it there indefinitely for one transaction per schedule per interval, so that whenever the treasury next performs a routine rotation the outgoing beneficiary is silently shorted.

## Root cause

`retarget_schedule` settles through `claim_plan`, then treats three distinct outcomes as "nothing to settle":

```rust
let vested_paid = match Self::claim_plan(schedule, now)? {
    ClaimPlan::Pay(amount) => { Self::settle(schedule, amount, now)?; amount },
    ClaimPlan::NothingToClaim | ClaimPlan::TooSoon | ClaimPlan::WouldLeaveDust => Zero::zero(),
};
let old_beneficiary = core::mem::replace(&mut schedule.beneficiary, new_beneficiary.clone());
```

`NothingToClaim` genuinely means nothing is owed. `TooSoon` does not. It is a rate limit, returned after the payable amount has already been computed:

```rust
let candidate = Self::quantize_down(owed);
if candidate < minimum { return Ok(ClaimPlan::NothingToClaim); }
let payable = /* ... */;
if let Some(last) = schedule.last_claim_at {
    if now.checked_sub(last)? < T::MinClaimInterval::get() {
        return Ok(ClaimPlan::TooSoon);
    }
}
Ok(ClaimPlan::Pay(payable))
```

`schedule.claimed` is never advanced, so the unpaid amount stays in the schedule and is paid to whoever holds `beneficiary` at the next successful claim.

## The invariant this breaks

The doc comment on the extrinsic:

> Settle any payout a permissionless claim could currently force, then change the beneficiary. This makes retargeting independent of claim transaction ordering.

And PR #646, which introduced the pallet:

> `retarget_schedule` first settles exactly the payout a permissionless claim could currently force to the old beneficiary, then changes the beneficiary. This makes the result independent of claim/retarget transaction ordering.

The outcome depends directly on when the outgoing beneficiary last claimed, which is the ordering dependence both statements rule out.

`end_schedule` confirms the intent. It pays `quantize_down(vested - claimed)` to the beneficiary with no `MinClaimInterval` gate at all, so the two administrative exits disagree about whether an outgoing beneficiary is owed their accrued amount.

## Why the existing regression test does not catch it

The pallet already has a dedicated test for this invariant, and it passes:

```rust
#[test]
fn settles_a_front_runnable_claim_before_retargeting() {
    new_test_ext(vec![default_schedule(BOB)]).execute_with(|| {
        set_time(300_000);
        assert_ok!(Vesting::retarget_schedule(RuntimeOrigin::signed(TREASURY), 0, CHARLIE));
        assert_eq!(free(&BOB), TOTAL / 2);
        assert_eq!(stored(0).claimed, TOTAL / 2);
        ...
```

It retargets a schedule that has never been claimed, so `last_claim_at` is `None`. The `TooSoon` check is guarded by exactly that:

```rust
if let Some(last) = schedule.last_claim_at {
    if now.checked_sub(last)? < T::MinClaimInterval::get() {
        return Ok(ClaimPlan::TooSoon);
    }
}
```

With `last_claim_at == None` the branch is unreachable and `claim_plan` can only return `Pay`, so the test exercises the settling path every time and never the one that skips it.

The first proof of concept below uses the same fixture (`default_schedule(BOB)`) and the same starting timestamp (`300_000`) as that test. The only difference is one prior claim.

## Impact

The outgoing beneficiary permanently loses everything that vested since their last claim, capped at one full day of the schedule's linear vesting rate. The funds are not burned or frozen: they are paid to the incoming beneficiary at the next successful claim.

Magnitude scales with the allocation. The only schedules currently in the tree are the testnet presets (`GENESIS_VESTING_TOTAL = 10_000 QUAN` over 365 days, marked "Testnet example grant size"), which put one day at roughly 27 QUAN. The pallet exists to carry a real allocation table that PR #642 explicitly deferred ("A real per-allocation table (amounts, varying terms) is deliberately left for a follow-up PR"), so the per-incident loss on a production table scales linearly with grant size and inversely with vesting duration.

The precondition is attacker-controlled, not incidental. A second test drives the whole sequence from an unrelated account:

```
3 schedules armed by an unrelated account: BOB lost 1500000 total, CHARLIE received 4500000
```

MALLORY is neither the beneficiary nor the treasury and takes nothing. BOB never touches his schedules. MALLORY claims on his behalf, which pays BOB correctly and is not itself an attack, but stamps `last_claim_at`; every rotation afterwards is silently unsettled. Because one account may hold any number of schedules, a grantee holding N schedules loses N times the per-schedule amount in a single rotation.

Keeping a schedule armed is cheap and durable. `MinClaimInterval` is both the rate limit and the refresh period, so one claim per schedule per day holds `last_claim_at` permanently inside the window. Any grant vesting at least `MinimumPayout` (1 QUAN) per day qualifies, which the shipped preset does by a wide margin at roughly 27 QUAN per day.

The loss is silent. No error is returned, and the emitted `ScheduleRetargeted` event carries `vested_paid: 0`, which is indistinguishable from the legitimate `NothingToClaim` case. Neither party has any on-chain signal that value moved between them.

Triggering it requires `AdminOrigin` (Root, or the treasury multisig via `EnsureTreasury`), but no malice: rotating a grant to a new address is the documented purpose of the call. The condition is not rare, it is the default state of any beneficiary claiming at the cadence the pallet's own rate limit implies.

The final step still needs `AdminOrigin`, and that gate is structural: `TreasuryAccount` is a genesis-only storage value with no setter extrinsic, and `retarget_schedule` is the only code path that mutates `beneficiary`. What an attacker controls is the precondition, not the trigger. Front-running a specific retarget with a claim is not the attack and would protect the outgoing party; the attack is holding schedules in the vulnerable window so that the treasury's own routine maintenance destroys value whenever it happens.

## Public History check

* The Equilibrium chain audit (13 May 2026) reports no `pallet-vesting` findings at all; its components are MULTISIG, SCHEDULER, TREASURY, REVERSIBLE and RUNTIME.
* `pallet-vesting` was added on 8 Aug 2026 in PR #646, after that audit.
* The two V12 vesting passes touch `retarget_schedule` on unrelated points: #656 rejects a `treasury_and_pot()` lookup suggestion, #649 extracts the shared `settle` block. Neither mentions `TooSoon`, `MinClaimInterval` or `last_claim_at` in relation to retargeting.
* None of the three published Known Issues relate.
* Still unfixed upstream: `Quantus-Network/chain` at `94c4ca7a`, later than the pinned `3b243b87`, carries the identical `ClaimPlan::TooSoon => Zero::zero()` arm.

## Remediation

Settle the outgoing beneficiary before the swap regardless of the rate limit. `MinClaimInterval` exists to bound how often a permissionless caller can force a payout, and a retarget is neither permissionless nor repeatable, so the limit should not apply to it.

The smallest correct fix is to compute the payout without the interval gate on this path, for example by having `claim_plan` take a flag that skips the `last_claim_at` check, and treating only `NothingToClaim` and `WouldLeaveDust` as zero. `WouldLeaveDust` should additionally be reconsidered: it strands up to two minimum payouts with the outgoing beneficiary's replacement for the same reason.

## Link to Proof of Concept

<https://gist.github.com/joksdz/dd81e3671d368d0a71b4970bd35a2f42>

## Proof of Concept

Two tests against the pinned tree, repo `audit-comp-quantus-chain`, commit `3b243b870a5442a6e5f443056f67bbba87d472f5`. Both append to `pallets/vesting/src/tests.rs` and use only the pallet's existing mock. Full runnable sources: <https://gist.github.com/joksdz/dd81e3671d368d0a71b4970bd35a2f42>

Mock constants differ from the runtime's and are scaled down: `MinClaimInterval` is 100\_000 in the mock versus `MILLIS_PER_DAY` live, and the default schedule vests `TOTAL = 4_000_000` linearly between `START = 100_000` and `END = 500_000`.

### Test 1: a routine rotation silently keeps the outgoing beneficiary's accrued funds

```
cargo test -p pallet-vesting --lib poc_retarget_within_min_claim_interval -- --nocapture
```

{% stepper %}
{% step %}

## Pay BOB everything vested so far at `t = 300_000`

This stamps `last_claim_at`. Half the schedule has vested by this point, so `claimed` becomes `TOTAL / 2`.

```rust
set_time(300_000);
assert_ok!(Vesting::claim(RuntimeOrigin::signed(PINGER), 0));
let bob_after_first_claim = free(&BOB);
let claimed_then = stored(0).claimed;
assert_eq!(claimed_then, TOTAL / 2);
```

{% endstep %}

{% step %}

## Advance to `t = 350_000`

That is 50\_000 elapsed against a `MinClaimInterval` of 100\_000, so the schedule is inside the rate-limit window. Assert a genuinely claim-worthy amount has accrued to BOB in that gap, so this is not the `NothingToClaim` case.

```rust
set_time(350_000);
let s = stored(0);
let vested_now = Vesting::vested_amount(&s, 350_000);
let accrued_for_bob = vested_now - s.claimed;
assert!(accrued_for_bob >= MinimumPayout::get());
```

{% endstep %}

{% step %}

## The treasury rotates the grant to a new address

Nothing here is malicious; this is the documented purpose of the call.

```rust
assert_ok!(Vesting::retarget_schedule(RuntimeOrigin::signed(TREASURY), 0, CHARLIE));
```

{% endstep %}

{% step %}

## Verify that BOB was paid nothing

BOB is no longer the beneficiary. The extrinsic returns `Ok`, and `claimed` is untouched, so the accrued amount is still owed by the schedule but now to someone else.

```rust
assert_eq!(free(&BOB), bob_after_first_claim);   // no settlement
assert_eq!(stored(0).claimed, claimed_then);     // no payout recorded
assert_eq!(stored(0).beneficiary, CHARLIE);
```

{% endstep %}

{% step %}

## CHARLIE claims later

CHARLIE receives BOB's accrued entitlement along with his own.

```rust
set_time(450_000);
let charlie_before = free(&CHARLIE);
assert_ok!(Vesting::claim(RuntimeOrigin::signed(PINGER), 0));
let charlie_gain = free(&CHARLIE) - charlie_before;
assert!(charlie_gain >= accrued_for_bob);
```

{% endstep %}
{% endstepper %}

Output:

```
BOB lost 500000 of already-vested funds; CHARLIE received 1500000
test tests::poc_retarget_within_min_claim_interval_reassigns_accrued_entitlement ... ok
```

### Test 2: an unrelated account can arm this, and the loss multiplies

```
cargo test -p pallet-vesting --lib poc_third_party_can_arm -- --nocapture
```

This one starts from three separate grants to BOB, the shape a real allocation table produces, and never has BOB touch any of them.

{% stepper %}
{% step %}

## MALLORY claims on BOB's behalf across all three schedules

MALLORY is neither the beneficiary nor the treasury and takes nothing from the outcome. This pays BOB correctly and is not itself an attack, but it stamps `last_claim_at` on every schedule.

```rust
let schedules = vec![default_schedule(BOB), default_schedule(BOB), default_schedule(BOB)];
new_test_ext(schedules).execute_with(|| {
    let mallory = PINGER;
    set_time(300_000);
    for id in 0..3u64 {
        assert_ok!(Vesting::claim(RuntimeOrigin::signed(mallory.clone()), id));
    }
    let bob_after_arming = free(&BOB);
```

{% endstep %}

{% step %}

## The treasury rotates all three grants inside the same window

Each one asserts a claim-worthy amount had accrued, then asserts nothing was settled.

```rust
    set_time(350_000);
    let mut total_lost = 0u128;
    for id in 0..3u64 {
        let s = stored(id);
        let accrued = Vesting::vested_amount(&s, 350_000) - s.claimed;
        assert!(accrued >= MinimumPayout::get());
        total_lost += accrued;
        assert_ok!(Vesting::retarget_schedule(RuntimeOrigin::signed(TREASURY), id, CHARLIE));
        assert_eq!(stored(id).claimed, s.claimed);   // no payout recorded
    }
    assert_eq!(free(&BOB), bob_after_arming);        // nothing across all rotations
```

{% endstep %}

{% step %}

## CHARLIE collects every one of them

Output:

```
3 schedules armed by an unrelated account: BOB lost 1500000 total, CHARLIE received 4500000
test tests::poc_third_party_can_arm_the_loss_and_it_scales_with_schedule_count ... ok
```

{% endstep %}
{% endstepper %}
