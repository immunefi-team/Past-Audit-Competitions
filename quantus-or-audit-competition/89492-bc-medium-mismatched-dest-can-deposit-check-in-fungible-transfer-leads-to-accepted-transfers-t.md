# 89492 bc medium mismatched dest can deposit check in fungible transfer leads to accepted transfers that credit nothing

**Submitted on Aug 19th 2026 at 16:18:54 UTC by @v4rvl for** [**Audit Comp | Quantus**](https://immunefi.com/audit-competition/audit-comp-quantus)

* **Report ID:** #89492
* **Report Type:** Blockchain/DLT
* **Report severity:** Medium
* **Target:** <https://github.com/immunefi-team/audit-comp-quantus-chain>
* **Impacts:**
  * A bug in the respective layer 0/1/2 network code that results in unintended smart contract behavior with no concrete funds at direct risk

## Description

## Summary

Commit `db94e2a196` ("Harden FRAME primitives against audit findings (#616)") changed the destination precondition in the `fungible::Mutate::transfer` default body from the upstream `can_deposit(dest, amount, Extant)` to `can_deposit(dest, amount + dust, Extant)`, while the credit actually applied stayed `debited`.

For `pallet-balances`, the only implementor on this chain, `debited` equals `amount` and never `amount + dust`: `Unbalanced::decrease_balance` returns `old_balance.saturating_sub(new_balance)` at `frame/support/src/traits/tokens/fungible/regular.rs:196`, and the reaping dust is routed separately to `handle_dust` at `:193`. The commit's premise, stated verbatim in its own comment at `:337-339` as "`debited` can exceed `amount`", is false on this cut.

The consequence is a precondition that stopped covering the value it is meant to protect. When an `Expendable` transfer reaps the source, the check passes against `amount + dust` while the credit applied is the strictly smaller `amount`. If that credit lands below the existential deposit, `Unbalanced::increase_balance` under `BestEffort` takes the silent no-op branch at `:218-219` and returns `Ok(Default::default())` without writing anything. The `let _ =` at `:340` discards it, `done_transfer` emits a `Transfer` event for the full amount, and the function returns `Ok(debited)`.

Because nothing errors, the `with_storage_layer` wrapper that `frame/support-procedural/src/pallet/expand/call.rs:241-247` places around every dispatchable body has nothing to roll back. The debit commits, the credit does not, and `total_issuance` is never adjusted. The runtime therefore reaches a state the state machine is not supposed to be able to reach: a dispatchable returns `Ok`, emits `Transfer { from, to, amount }`, and the destination balance is unchanged, while `total_issuance` diverges from `sum(balances)` plus what dust removal accounted for.

Upstream rejects the same call with `TokenError::BelowMinimum`, because upstream checks `can_deposit` against the value it credits.

**What is claimed.** This is a state-machine correctness defect in the chain's own FRAME code: an accepted extrinsic reports a transfer that did not happen and leaves the issuance identity broken. No concrete funds are at direct risk. The value involved is structurally capped strictly below one existential deposit (`EXISTENTIAL_DEPOSIT = MILLI_UNIT = 1_000_000_000` planck = `0.001 UNIT`) and belongs to the signed caller, who is also the only actor who can trigger the path. That cap is the evidence for the selected row's "no concrete funds at direct risk" qualifier, not an impact claim. A generic conservation property (`auditooor_prop_fungible_conservation` at `PROPTEST_CASES=20000`) later reached the same state without a hand-written sequence, which is independent reproduction of the same behavior.

## Impact

The impact claimed is an unintended state transition in the chain's own runtime code, with no concrete funds at direct risk.

A signed `transfer_allow_death` is accepted, returns `Ok`, and emits `Transfer { from: SOURCE, to: DEST, amount }`, yet the destination account is never written. Every observer of the runtime, on-chain and off-chain, is told a transfer completed that did not. The runtime simultaneously leaves `total_issuance` out of step with `sum(balances)`, breaking an identity that FRAME maintains everywhere else and that commit `db94e2a196` explicitly set out to preserve.

In the PoC state transition, `SOURCE=150`, `DEST=0`, `total_issuance=150` before the call, and `SOURCE=0`, `DEST=0`, `total_issuance=90` after it. Dust 60 is burned as designed. The remaining 90 is the accounting divergence, and it is what the intended semantics forbid.

No funds impact is claimed and none is provable: the divergence is capped strictly below `0.001 UNIT` per call by `regular.rs:216`, and only the signed caller's own extrinsic can produce it. Those numbers are reachability and bounding detail supporting the selected row's "no concrete funds at direct risk" qualifier.

## Root Cause

**1. The precondition and the credit are computed from different values.**

`frame/support/src/traits/tokens/fungible/regular.rs:327-342`:

```rust
let dust = Self::can_withdraw(source, amount).into_result(preservation != Expendable)?;
let actual = amount.saturating_add(dust);
Self::can_deposit(dest, actual, Extant).into_result()?;
if source == dest {
    // Nothing is debited and no reap occurs on the self path, so the reported amount must
    // not include the hypothetical dust.
    return Ok(amount)
}

let debited = Self::decrease_balance(source, amount, BestEffort, preservation, Polite)?;
// Credit the amount actually removed from the source. When an expendable transfer reaps
// the source, `debited` can exceed `amount`; ignoring it desynchronizes issuance from
// account balances.
let _ = Self::increase_balance(dest, debited, BestEffort);
Self::done_transfer(source, dest, debited);
Ok(debited)
```

The check at `:329` uses `actual`. The credit at `:340` uses `debited`.

**2. `debited` never equals `actual` when dust is nonzero.**

`Unbalanced::decrease_balance`, `frame/support/src/traits/tokens/fungible/regular.rs:184-197`:

```rust
let new_balance = old_balance.checked_sub(&amount).ok_or(TokenError::FundsUnavailable)?;
if let Some(dust) = Self::write_balance(who, new_balance)? {
    Self::handle_dust(Dust(dust));
}
Ok(old_balance.saturating_sub(new_balance))
```

`new_balance` is `old_balance - amount`, so the return at `:196` is exactly `amount`. The dust is a separate value returned by `write_balance` at `:193` and handed to `handle_dust`; it is never folded into the return. `pallet-balances` does not override `decrease_balance` (`pallets/balances/src/impl_fungible.rs:153-189` overrides only `handle_dust`, `write_balance`, `set_total_issuance`, `deactivate`, `reactivate`), so the default body above is what runs.

Therefore, whenever the source is reaped, `debited = amount` and `actual = amount + dust > debited`.

**3. The under-checked credit hits the silent no-op branch.**

`Unbalanced::increase_balance`, `frame/support/src/traits/tokens/fungible/regular.rs:209-232`:

```rust
let old_balance = Self::balance(who);
let new_balance = if let BestEffort = precision {
    old_balance.saturating_add(amount)
} else {
    old_balance.checked_add(&amount).ok_or(ArithmeticError::Overflow)?
};
if new_balance < Self::minimum_balance() {
    // Attempt to increase from 0 to below minimum -> stays at zero.
    if let BestEffort = precision {
        Ok(Default::default())
    } else {
        Err(TokenError::BelowMinimum.into())
    }
} else {
```

The transfer body passes `BestEffort`, so the branch taken at `:218-219` returns `Ok(0)` and writes nothing.

**4. Nothing errors, so the transactional layer does not fire.**

`frame/support-procedural/src/pallet/expand/call.rs:241-247` wraps every dispatchable body in `with_storage_layer`, whose implementation is `frame/support/src/storage/transactional.rs:188`. It rolls back on `Err`. Here `increase_balance` returns `Ok(0)`, `transfer` returns `Ok(debited)`, and `transfer_allow_death` returns `Ok(())` at `pallets/balances/src/lib.rs:678`. The debit is committed.

Note the precise shape: the discarded `Result` at `:340` is not itself the defect. On this cut `increase_balance` has no reachable `Err` branch under `BestEffort` (the only `?` is `write_balance` at `:227`, and every error path inside `pallets/balances/src/lib.rs:1092-1186` requires a provider or consumer transition that an increase above ED cannot produce). Binding and checking the `Result` would not catch this, because the failed credit is reported as `Ok(0)`.

**5. Upstream did not have this hole.**

`git show refs/audit-bases/polkadot-stable2512-2:substrate/frame/support/src/traits/tokens/fungible/regular.rs`:

```rust
let _extra = Self::can_withdraw(source, amount).into_result(preservation != Expendable)?;
Self::can_deposit(dest, amount, Extant).into_result()?;
...
Self::decrease_balance(source, amount, BestEffort, preservation, Polite)?;
// This should never fail as we checked `can_deposit` earlier. But we do a best-effort
// anyway.
let _ = Self::increase_balance(dest, amount, BestEffort);
```

Upstream checks and credits the same value, which is why its own comment can say the credit "should never fail". Quantus kept the discard but broke the equality the discard relied on.

## Attack Path

Concretely, with `ED = 100` in the PoC mock, which is the same shape as `ED = 1_000_000_000` on the runtime. The live window is source in `[ED, 2*ED)`, dest empty, `amount < ED`. A 1 UNIT transfer does not hit it.

{% stepper %}
{% step %}

## Account `SOURCE` holds `150`

Account `DEST` does not exist.
{% endstep %}

{% step %}

## `SOURCE` calls `Balances::transfer_allow_death(DEST, 90)`

This is `call_index(0)`, `ensure_signed`, permissionless (`pallets/balances/src/lib.rs:670-679`), and passes `Preservation::Expendable` at `:678`.
{% endstep %}

{% step %}

## `can_withdraw(SOURCE, 90)` returns dust

`can_withdraw(SOURCE, 90)` computes `new_free = 60 < ED` and returns `ReducedToZero(60)` (`pallets/balances/src/impl_fungible.rs:132-137`). `into_result(false)` yields `dust = 60` (`frame/support/src/traits/tokens/misc.rs:124`).
{% endstep %}

{% step %}

## The widened deposit check passes

`actual = 90 + 60 = 150`. `can_deposit(DEST, 150, Extant)` returns `Success` because `0 + 150 >= ED` (`pallets/balances/src/impl_fungible.rs:88-92`).
{% endstep %}

{% step %}

## The source is reaped

`decrease_balance(SOURCE, 90, BestEffort, Expendable, Polite)` writes `SOURCE.free = 60`, which is under ED, so `write_balance` returns `Some(60)`, `handle_dust` burns it through `DustRemoval`, and `SOURCE` is reaped. The return is `90`.
{% endstep %}

{% step %}

## The destination credit is silently dropped

`increase_balance(DEST, 90, BestEffort)` computes `0 + 90 = 90 < ED` and returns `Ok(0)` without writing.
{% endstep %}

{% step %}

## The transfer is reported as successful

`done_transfer(SOURCE, DEST, 90)` emits `Transfer { from: SOURCE, to: DEST, amount: 90 }`. `transfer` returns `Ok(90)`. `transfer_allow_death` returns `Ok(())`.
{% endstep %}

{% step %}

## The issuance counter and account map diverge

Final state: `SOURCE = 0`, `DEST = 0`, `total_issuance = 90`, `sum(balances) = 0`. The issuance counter and the account map disagree by 90, and nothing on the path reconciles them.

The same call under upstream fails at step 4 with `TokenError::BelowMinimum`, because upstream checks `can_deposit(DEST, 90, Extant)`.
{% endstep %}
{% endstepper %}

## Proof of Concept

File: `src/quantus-chain/pallets/balances/tests/auditooor_l6_transfer_credit_drop.rs`

Run:

```bash
cd src/quantus-chain
cargo test -p pallet-balances --test auditooor_l6_transfer_credit_drop
```

```rust
parameter_types! {
	pub static ExistentialDeposit: Balance = 100;
}

const SOURCE: u64 = 1;
const DEST: u64 = 9;

fn new_test_ext() -> sp_io::TestExternalities {
	let mut storage = frame_system::GenesisConfig::<Test>::default().build_storage().unwrap();
	pallet_balances::GenesisConfig::<Test> {
		balances: vec![(SOURCE, 150)],
		dev_accounts: None,
	}
	.assimilate_storage(&mut storage)
	.unwrap();
	let mut ext = sp_io::TestExternalities::new(storage);
	ext.execute_with(|| System::set_block_number(1));
	ext
}

#[test]
fn transfer_reaping_source_destroys_the_credit_and_still_returns_ok() {
	new_test_ext().execute_with(|| {
		let ed = ExistentialDeposit::get();
		assert_eq!(ed, 100);

		let issuance_before = <Balances as Inspect<u64>>::total_issuance();
		assert_eq!(issuance_before, 150);
		assert_eq!(<Balances as Inspect<u64>>::balance(&DEST), 0);

		// The precondition Quantus checks: amount + dust.
		assert_eq!(
			<Balances as Inspect<u64>>::can_deposit(&DEST, 150, Provenance::Extant),
			DepositConsequence::Success,
			"the widened precondition passes"
		);
		// The precondition upstream checked, and the value actually credited.
		assert_eq!(
			<Balances as Inspect<u64>>::can_deposit(&DEST, 90, Provenance::Extant),
			DepositConsequence::BelowMinimum,
			"the credit that is actually applied would NOT have passed the upstream check"
		);

		let reported =
			<Balances as Mutate<u64>>::transfer(&SOURCE, &DEST, 90, Preservation::Expendable)
				.expect("transfer returns Ok - nothing errors, so nothing rolls back");

		// The caller is told 90 moved.
		assert_eq!(reported, 90);

		// Nothing arrived.
		assert_eq!(
			<Balances as Inspect<u64>>::balance(&DEST),
			0,
			"destination was never credited"
		);
		// The source was reaped.
		assert_eq!(<Balances as Inspect<u64>>::balance(&SOURCE), 0);

		// 60 of dust was burned by `DustRemoval = ()`. The remaining 90 is gone from every
		// account while total issuance still counts it: unbacked issuance, permanently.
		let issuance_after = <Balances as Inspect<u64>>::total_issuance();
		assert_eq!(issuance_after, 90, "issuance dropped only by the 60 of dust");

		let summed: Balance = [SOURCE, DEST]
			.iter()
			.map(|who| <Balances as Inspect<u64>>::total_balance(who))
			.sum();
		assert_eq!(summed, 0, "no account holds anything");
		assert_eq!(
			issuance_after - summed,
			90,
			"sum(balances) < total_issuance by exactly the destroyed credit"
		);
	});
}

#[test]
fn production_dispatchable_returns_ok_and_the_state_commits() {
	new_test_ext().execute_with(|| {
		assert_eq!(
			Balances::transfer_allow_death(RuntimeOrigin::signed(SOURCE), DEST, 90),
			Ok(())
		);
		assert_eq!(<Balances as Inspect<u64>>::balance(&DEST), 0);
		assert_eq!(<Balances as Inspect<u64>>::balance(&SOURCE), 0);
		assert_eq!(<Balances as Inspect<u64>>::total_issuance(), 90);
	});
}

#[test]
fn credit_at_or_above_ed_lands_normally_so_the_loss_is_capped_by_ed() {
	new_test_ext().execute_with(|| {
		// 150 - 100 = 50 left behind, below ED, so the source is still reaped and dust is 50.
		let reported =
			<Balances as Mutate<u64>>::transfer(&SOURCE, &DEST, 100, Preservation::Expendable)
				.expect("transfer succeeds");
		assert_eq!(reported, 100);
		assert_eq!(
			<Balances as Inspect<u64>>::balance(&DEST),
			100,
			"a credit at ED is written, so the destruction window is strictly below ED"
		);
	});
}
```

Transcript:

```
    Finished `test` profile [unoptimized + debuginfo] target(s) in 0.75s
     Running tests/auditooor_l6_transfer_credit_drop.rs

running 5 tests
test test_genesis_config_builds ... ok
test __construct_runtime_integrity_test::runtime_integrity_tests ... ok
test production_dispatchable_returns_ok_and_the_state_commits ... ok
test credit_at_or_above_ed_lands_normally_so_the_loss_is_capped_by_ed ... ok
test transfer_reaping_source_destroys_the_credit_and_still_returns_ok ... ok

test result: ok. 5 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.01s
```

Suite result: ok

## What the tests prove

* `transfer_reaping_source_destroys_the_credit_and_still_returns_ok` proves the whole chain in one run: the widened precondition passes for `amount + dust`, the upstream precondition would have refused the credit actually applied, the transfer returns `Ok` reporting the full amount, the destination balance stays zero, the source is reaped, and `total_issuance - sum(balances)` equals exactly the credit that was never written.
* `production_dispatchable_returns_ok_and_the_state_commits` proves the caller returns `Ok`, which is the load-bearing claim for the transactional layer. `with_storage_layer` reverts on `Err` only, so an `Ok` return commits the debit.
* `credit_at_or_above_ed_lands_normally_so_the_loss_is_capped_by_ed` fixes the upper bound: at a credit of exactly ED the write happens, so the divergence window is strictly below ED. This is the executed proof of impossibility behind refusing every funds row.

The same defect is not only the three named cases. After the fixture ED was set to 100 (production-shaped, not the vacuous ED=1 default) and the amount strategy put a third of its mass on `0..2*ED`, `auditooor_prop_fungible_conservation` in `pallets/balances/tests/auditooor_prop_fungible_traits.rs` failed at `PROPTEST_CASES=20000` (1\_000\_000 calls). Flat `0..5000` amounts at the same ED ran 20\_000 cases clean: the window existed and the sampler almost never entered it.

Failure, verbatim:

```
issuance 5107 -> 5008, balances 5107 -> 5000, entitlement 0
Transfer { from: 1, to: 4, amount: 8, keep_alive: false }
```

Shrunk load-bearing pair:

```
BurnFrom { who: 1, amount: 893, exact: false, keep_alive: false },
Transfer { from: 1, to: 4, amount: 8, keep_alive: false },
```

Account 1 is driven to 107, then sends 8 to empty account 4. `can_deposit(4, 8+99)` passes; `increase_balance` drops the 8; dust 99 burns; issuance falls by 99 and summed balances by 107. Divergence equals the transfer amount and stays `< ED`. Same three conditions as the named test (`150/90` there, `107/8` here). This is generic reproduction, not a second mechanism.

A separate `set_balance` return-value miss (`reported 1`, account holds `0`) also appears once ED is 100. That path is byte-identical to `refs/audit-bases/polkadot-stable2512-2` and is not claimed here.

## Recommended Fix

Check the value that will actually be credited, or credit the value that was actually checked. The minimal fix is to restore the equality the discard relies on:

```rust
let debited = Self::decrease_balance(source, amount, BestEffort, preservation, Polite)?;
Self::increase_balance(dest, debited, BestEffort)?;
```

together with checking `can_deposit(dest, amount, Extant)` for the credited value rather than `amount + dust`. If the goal of `#616` genuinely is to move the reaping dust to the destination as well, then `decrease_balance` must return `amount + dust` for that path, and the destination credit must use that same value. Either way, the precondition and the credit must be the same number, and `increase_balance` must not be allowed to no-op silently on a value the caller already committed to debiting.
