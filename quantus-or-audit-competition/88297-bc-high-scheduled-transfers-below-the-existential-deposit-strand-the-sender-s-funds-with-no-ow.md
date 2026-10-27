# 88297 bc high scheduled transfers below the existential deposit strand the sender s funds with no owner accessible recovery

**Submitted on Aug 12th 2026 at 17:02:04 UTC by @gabeonchain for** [**Audit Comp | Quantus**](https://immunefi.com/audit-competition/audit-comp-quantus)

* **Report ID:** #88297
* **Report Type:** Blockchain/DLT
* **Report severity:** High
* **Target:** <https://github.com/immunefi-team/audit-comp-quantus-chain>
* **Impacts:**
  * Temporary freezing of funds for at least 24 hour

## Description

### Brief/Intro

I was checking what pallet-reversible-transfers validates before it accepts a transfer. The answer is one thing: the amount can't be zero.

That seemed thin, so I followed the amount through to where it eventually gets spent. execute\_transfer builds a transfer\_keep\_alive, and transfer\_keep\_alive won't create an account below the existential deposit. Nothing between the two checks for that.

So I scheduled one and watched it fail.

### Vulnerability Details

do\_schedule\_transfer\_inner at pallets/reversible-transfers/src/lib.rs:762 checks !amount.is\_zero() and nothing else about the amount.

At execution, do\_execute\_transfer at line 693 releases the hold, removes the pending metadata, and then dispatches the transfer. The dispatch is the last step, and its result becomes the extrinsic's result. When it fails, Substrate rolls back the release and the metadata removal, which sounds safe and isn't, because the scheduler already consumed the task when it dispatched.

There's no retry to fall back on. do\_schedule\_transfer\_inner never calls set\_retry. The only Retries mentions in the pallet are benchmark annotations covering the write that cancel\_named does.

That leaves the transfer stranded: the hold is back, the pending entry is back, and nothing will ever execute it.

The owner can't clear it either. cancel\_transfer at line 874 reads the guardian frozen into the pending record. For a high-security schedule the guardian isn't the sender, so the owner gets InvalidReverser. Only the guardian can cancel, and when they do, the recipient of the released funds is the guardian, minus a 1 percent volume fee.

There's no path that gives the owner their money back.

### Impact Details

Production EXISTENTIAL\_DEPOSIT is one MILLI\_UNIT (runtime/src/lib.rs:102). Any scheduled transfer under that amount, to an address that doesn't exist yet, hits this.

That's a normal thing for people to do. Sending a small amount to a new wallet before moving a real balance is routine, and it's precisely the case that breaks.

Each stranded transfer locks its own amount, which is bounded by the existential deposit and therefore small. The slot is the real cost. MaxPendingPerAccount is 16, each stranded transfer takes one permanently, and the owner can't free it. After 16, the account can't schedule anything, and schedule\_transfer is the only outbound call a high-security account is allowed to make.

So the end state is an account that can't move its own money, holding funds it can't reach, waiting on a guardian who has to act 16 times and receives the funds each time.

## Link to Proof of Concept

<https://gist.github.com/gabchess/573942d2122359f37b793dfa9e0d4e88>

## Proof of Concept

The test runs in the runtime integration harness, so the existential deposit is the production value rather than a mock one.

```
cargo test -p quantus-runtime --test mod reversible_integration -- --nocapture --test-threads=1
test transactions::reversible_integration::sub_existential_deposit_schedule_is_permanently_unexecutable ...
recipient balance after execution window: 0
pending entries still held: 1
ok
test result: ok. 5 passed; 0 failed; 0 ignored; 0 measured; 39 filtered out; finished in 0.08s
```

It schedules EXISTENTIAL\_DEPOSIT / 2 to an unfunded address, runs past the execution window, and then checks that the recipient got nothing and the pending entry survived. It runs 500 more blocks to show nothing retries. Then it confirms the owner's cancel fails and the guardian's succeeds.

Full source in the attached gist.

## Recommendation

Reject the transfer at schedule time. Add the check next to the existing zero check at line 762:

```rust
ensure!(
    amount >= <T as pallet_balances::Config>::ExistentialDeposit::get(),
    Error::<T>::AmountBelowMinimum
);
```

This catches the case at the point where the user can still do something about it, and it costs one comparison.

Two other changes are worth making, though neither replaces the check above.

* Configure a retry on the scheduled task, so a transfer that fails for a transient reason gets another attempt instead of dying on the first failure. The scheduler already supports this through set\_retry.
* Let the owner cancel their own stranded transfer. Right now the owner is locked out of a transfer that can never execute, which serves no security purpose: the delay exists so the guardian can intervene in a transfer that would otherwise complete, and this one never will. At minimum, allow the owner to cancel a pending transfer whose scheduled block has already passed.

I'd ship the ED check first. It's the smallest change and it prevents the state from arising at all.

## References

pallets/reversible-transfers/src/lib.rs:762,693,706,722,874\
runtime/src/lib.rs:102
