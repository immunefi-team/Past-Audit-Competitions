# 89889 bc medium permissionless vesting claim strands a schedule in an un terminable state blocking the treasury s clawback of the final tranche

**Submitted on Aug 21st 2026 at 17:22:45 UTC by @zealynx for** [**Audit Comp | Quantus**](https://immunefi.com/audit-competition/audit-comp-quantus)

* **Report ID:** #89889
* **Report Type:** Blockchain/DLT
* **Report severity:** Medium
* **Target:** <https://github.com/immunefi-team/audit-comp-quantus-chain>
* **Impacts:**
  * Griefing (e.g. no profit motive for an attacker, but damage to the users or the protocol)

## Description

### Brief/Intro

`Vesting::claim` is permissionless: any funded account can trigger a payout on any schedule. Its non-final cap guarantees the schedule retains a minimum-sized *total* remainder, but not that the *already-vested and unpaid* portion is either zero or at least `MinimumPayout` — and `end_schedule` rejects exactly that middle case. An unrelated account can therefore call `claim` at the right moment on a schedule the administrator is about to terminate, permanently denying the treasury its clawback of the final unvested tranche and blocking termination until natural maturity. The denied amount vests to the original beneficiary instead of returning to the treasury.

### Vulnerability Details

`Vesting::claim` calls `ensure_signed(origin)` and accepts any signer for any schedule id.

For a non-final claim, `claim_plan` caps the payout at `remaining - MinimumPayout`, which ensures the schedule retains one minimum-sized *total* tranche. It does not ensure that the amount which is **already vested but unpaid** is either zero or at least `MinimumPayout`.

When the uncapped candidate exceeds that cap by less than one minimum payout, the claim succeeds and leaves the difference vested and unpaid. `end_schedule` independently rejects every non-zero unpaid vested amount below `MinimumPayout`:

```rust
ensure!(vested_paid.is_zero() || vested_paid >= T::MinimumPayout::get(), Error::<T>::PayoutBelowMinimum);
```

A schedule that was terminable immediately before the claim therefore becomes impossible to end.

The cap only binds when the candidate exceeds `remaining - MinimumPayout`, which requires `total - vested < MinimumPayout`. The unvested portion at that instant — the amount the treasury would have reclaimed — is consequently always **less than one `MinimumPayout`**, at any schedule size.

The condition is one-shot per schedule: after the capped claim, `remaining == MinimumPayout`, so any subsequent claim hits `max_non_final = 0 < minimum` and returns `WouldLeaveDust`. `MinClaimInterval` does not gate the attack either, because `last_claim_at` is `None` on a schedule that has never been claimed.

#### Vulnerable scenario

{% stepper %}
{% step %}

### Create the schedule

A legitimate one-year schedule holds 100 QUAN for a beneficiary.
{% endstep %}

{% step %}

### Trigger a permissionless claim

When 99.01 QUAN is vested, an unrelated account calls the permissionless `claim` before the administrator's termination call.
{% endstep %}

{% step %}

### Strand the schedule

`claim_plan` pays the capped 99 QUAN, leaving a 1-QUAN total remainder but only 0.01 QUAN currently vested and unpaid.
{% endstep %}

{% step %}

### Block termination

`end_schedule` rejects the 0.01-QUAN payout with `PayoutBelowMinimum` until natural maturity.
{% endstep %}
{% endstepper %}

The attacker needs only an arbitrary funded account able to submit a signed extrinsic — no privileged origin, no relationship to the schedule, no knowledge of the beneficiary. (In the PoC, the only `Root` calls create the legitimate pre-existing schedule and model the administrator's *failed* termination — Root appears on the victim side only; the harmful `claim` is submitted by an unrelated account.)

**Recommended fix.** When the non-final cap would leave a non-zero, sub-minimum *vested* remainder, reduce the claim by enough to preserve at least `MinimumPayout` of the amount already vested; if that would make the claim itself smaller than `MinimumPayout`, reject it with `ClaimWouldLeaveDust`:

```diff
// pallets/vesting/src/lib.rs  (claim_plan, non-final branch)
-				candidate.min(max_non_final)
+				if candidate <= max_non_final {
+					candidate
+				} else {
+					let payable = candidate.checked_sub(&minimum).ok_or(ArithmeticError::Underflow)?;
+					if payable < minimum {
+						return Ok(ClaimPlan::WouldLeaveDust);
+					}
+					payable
+				}
```

This can defer a near-maturity claim on a small schedule until the final claim, but preserves immediate administrative termination and avoids the inconsistent vested remainder.

### Impact Details

Any signed account can deny the administrator's immediate clawback of the final unvested tranche and block schedule termination for a bounded window — a permissionless griefing of the treasury's clawback path, with no profit to the attacker.

* **Amount denied to the treasury:** bounded by `MinimumPayout - PayoutQuantum`. With production values (`MinimumPayout = 1 QUAN`, `PayoutQuantum = 0.01 QUAN`) that is **0.99 QUAN**, and it is bounded below one `MinimumPayout` by construction regardless of schedule size. Without mitigation it vests to the original beneficiary at maturity — it is never returned to the treasury.
* **Duration termination is blocked:** `duration × (MinimumPayout − PayoutQuantum) / total`. The demonstrated 100-QUAN one-year schedule blocks termination for **\~3.6 days**; the window scales inversely with grant size, so larger grants are blocked for proportionally less time (minutes for large grants).
* **Partial mitigation available to the admin:** retargeting the schedule to the treasury prevents the beneficiary capturing the remainder, but still cannot recover the tranche or end the schedule before maturity.
* **Attacker prerequisites:** one signed extrinsic from any funded account.

Selected in-scope impact: **Griefing** — an unprivileged actor damages the protocol/treasury (a permanently-denied sub-`MinimumPayout` clawback plus a bounded termination block) with no profit motive.

## References

Frozen revision: `quantus-chain @ 3b243b870a5442a6e5f443056f67bbba87d472f5`

* <https://github.com/immunefi-team/audit-comp-quantus-chain/blob/3b243b870a5442a6e5f443056f67bbba87d472f5/pallets/vesting/src/lib.rs#L319-L350> — `claim`, `ensure_signed` result discarded
* <https://github.com/immunefi-team/audit-comp-quantus-chain/blob/3b243b870a5442a6e5f443056f67bbba87d472f5/pallets/vesting/src/lib.rs#L401-L442> — `end_schedule`, the `PayoutBelowMinimum` guard
* <https://github.com/immunefi-team/audit-comp-quantus-chain/blob/3b243b870a5442a6e5f443056f67bbba87d472f5/pallets/vesting/src/lib.rs#L528-L560> — `claim_plan`, the non-final cap

## Proof of Concept

Runs entirely locally. **No test was performed against mainnet or any public testnet.** The test is added to the existing `runtime/tests/governance/vesting.rs` (which supplies the `account()`, `new_test_ext()`, `set_time()` helpers) and uses production runtime constants.

{% stepper %}
{% step %}

### Control

Create a 100-QUAN one-year schedule, advance to 99.01% vested, and `end_schedule` — the treasury correctly gets 0.99 QUAN back.
{% endstep %}

{% step %}

### Attack

Use the identical schedule, but have an unrelated `account(8)` call `claim` first. It pays the capped 99 QUAN, leaving 0.01 QUAN vested-unpaid. `end_schedule` now fails with `PayoutBelowMinimum` — at 99.01%, one millisecond before maturity, and only succeeds at maturity, by which point the remainder has vested to the beneficiary. The treasury never recovers it.
{% endstep %}

{% step %}

### Retarget mitigation

Even after retargeting the schedule to the treasury, `end_schedule` still fails until maturity.
{% endstep %}
{% endstepper %}

Apply this patch and run:

```diff
diff --git a/runtime/tests/governance/vesting.rs b/runtime/tests/governance/vesting.rs
--- a/runtime/tests/governance/vesting.rs
+++ b/runtime/tests/governance/vesting.rs
@@ -91,6 +91,106 @@ mod tests {
 		assert_eq!(VestingMinClaimInterval::get(), 24 * 60 * 60 * 1000);
 	}
 
+	#[test]
+	fn audit_permissionless_claim_prevents_clawback_until_natural_end() {
+		const DAY: u64 = 24 * 60 * 60 * 1_000;
+		const ONE_YEAR: u64 = 365 * DAY;
+		const ATTACK_TIME: u64 = ONE_YEAR * 9_901 / 10_000; // 99.01% vested
+		const TOTAL: Balance = 100 * UNIT;
+		let beneficiary = account(9);
+		let treasury = account(4);
+		let quantum = VestingPayoutQuantum::get();
+		let denied_clawback = 99 * quantum;
+		assert_eq!(VestingMinimumPayout::get() - quantum, denied_clawback);
+
+		// Control: ending normally at 99.01% returns 0.99 QUAN to the treasury.
+		new_test_ext(Some(treasury.clone())).execute_with(|| {
+			Balances::make_free_balance_be(&treasury, 1_000 * UNIT);
+			assert_ok!(Vesting::create_schedule(RuntimeOrigin::root(), beneficiary.clone(), 0, 0, ONE_YEAR, TOTAL));
+			set_time(ATTACK_TIME);
+			assert_ok!(Vesting::end_schedule(RuntimeOrigin::root(), 0));
+			assert_eq!(Balances::total_balance(&beneficiary), 99 * UNIT + quantum);
+			assert_eq!(Balances::total_balance(&treasury), 900 * UNIT + denied_clawback);
+		});
+
+		// Attack: an unrelated claim strands the schedule; treasury never recovers the tranche.
+		new_test_ext(Some(treasury.clone())).execute_with(|| {
+			Balances::make_free_balance_be(&treasury, 1_000 * UNIT);
+			assert_ok!(Vesting::create_schedule(RuntimeOrigin::root(), beneficiary.clone(), 0, 0, ONE_YEAR, TOTAL));
+			set_time(ATTACK_TIME);
+			assert_ok!(Vesting::claim(RuntimeOrigin::signed(account(8)), 0));
+			assert_eq!(Balances::total_balance(&treasury), 900 * UNIT);
+			assert_noop!(Vesting::end_schedule(RuntimeOrigin::root(), 0), pallet_vesting::Error::<Runtime>::PayoutBelowMinimum);
+			set_time(ONE_YEAR - 1);
+			assert_noop!(Vesting::end_schedule(RuntimeOrigin::root(), 0), pallet_vesting::Error::<Runtime>::PayoutBelowMinimum);
+			set_time(ONE_YEAR);
+			assert_ok!(Vesting::end_schedule(RuntimeOrigin::root(), 0));
+			assert_eq!(Balances::total_balance(&treasury), 900 * UNIT);
+			assert_eq!(Balances::total_balance(&beneficiary), TOTAL);
+		});
+
+		// Retargeting prevents beneficiary capture but still cannot end the schedule early.
+		new_test_ext(Some(treasury.clone())).execute_with(|| {
+			Balances::make_free_balance_be(&treasury, 1_000 * UNIT);
+			assert_ok!(Vesting::create_schedule(RuntimeOrigin::root(), beneficiary.clone(), 0, 0, ONE_YEAR, TOTAL));
+			set_time(ATTACK_TIME);
+			assert_ok!(Vesting::claim(RuntimeOrigin::signed(account(8)), 0));
+			assert_ok!(Vesting::retarget_schedule(RuntimeOrigin::root(), 0, treasury.clone()));
+			assert_noop!(Vesting::end_schedule(RuntimeOrigin::root(), 0), pallet_vesting::Error::<Runtime>::PayoutBelowMinimum);
+			set_time(ONE_YEAR);
+			assert_ok!(Vesting::end_schedule(RuntimeOrigin::root(), 0));
+			assert_eq!(Balances::total_balance(&treasury), 901 * UNIT);
+			assert_eq!(Balances::total_balance(&beneficiary), 99 * UNIT);
+		});
+	}
+
 	#[test]
 	fn treasury_multisig_creates_and_ends_schedules() {
 		new_test_ext(Some(treasury_multisig())).execute_with(|| {
```

```bash
SKIP_WASM_BUILD=1 cargo test -p quantus-runtime --test mod \
  governance::vesting::tests::audit_permissionless_claim_prevents_clawback_until_natural_end -- --nocapture
```

### Observed output

```
test governance::vesting::tests::audit_permissionless_claim_prevents_clawback_until_natural_end ... ok
test result: ok. 1 passed; 0 failed
```

With the recommended fix applied, the unchanged test fails at the first expected `PayoutBelowMinimum` (because `end_schedule` now succeeds), pinning the harm to the `claim_plan` cap. A sweep over grants of 2–8 minimum payouts and every quantized vesting point confirms every successful claim then leaves `end_schedule` immediately usable, and the existing Vesting suite stays green.
