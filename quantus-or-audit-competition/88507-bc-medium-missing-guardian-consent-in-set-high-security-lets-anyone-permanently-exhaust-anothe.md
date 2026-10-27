# 88507 bc medium missing guardian consent in set high security lets anyone permanently exhaust another account s guardian slots

**Submitted on Aug 13th 2026 at 14:00:56 UTC by @gabeonchain for** [**Audit Comp | Quantus**](https://immunefi.com/audit-competition/audit-comp-quantus)

* **Report ID:** #88507
* **Report Type:** Blockchain/DLT
* **Report severity:** Medium
* **Target:** <https://github.com/immunefi-team/audit-comp-quantus-chain>
* **Impacts:**
  * Griefing (e.g. no profit motive for an attacker, but damage to the users or the protocol)

## Description

### Brief/Intro

I was reading pallet-reversible-transfers to work out who actually holds authority over a pending transfer. The cancel path freezes the guardian at schedule time, and there's a comment explaining exactly why, so somebody clearly thought hard about this. That's what sent me looking for where the guardian relationship gets created in the first place.

set\_high\_security creates it. The guardian never gets asked.

That stopped me, because the same call writes into the guardian's own storage, and that storage has a cap.

### Vulnerability Details

set\_high\_security at pallets/reversible-transfers/src/lib.rs:376 checks that the guardian is not the caller, and that the caller is not already high security. It never checks whether the guardian agrees.

At line 394 it pushes the caller into GuardianIndex\[guardian]. That vector is capped at MaxGuardianAccounts, which the runtime sets to 32 at runtime/src/configs/mod.rs:528. The push uses try\_push behind a ?, so a full index makes the entire extrinsic fail with TooManyGuardianAccounts.

Then I checked whether those entries can ever leave. They can't. GuardianIndex gets written in exactly two places, set\_high\_security and the genesis builder, and nothing anywhere removes from it. There's no remove\_guardian, no renounce, no revoke. recover\_funds says in its own docs that it preserves the relationship deliberately.

Enrollment is permanent too. The pallet's documentation is explicit: there's no mechanism to disable high security mode.

Before concluding there's no remedy, I went looking for a privileged one. pallet\_sudo is gone from the runtime, and runtime/src/lib.rs:232 says so in a comment. The pallet has no force\_\* extrinsic, no AdminOrigin, no ensure\_root path. No migration touches the map either.

So 32 accounts under one attacker's control permanently strip a chosen account of its ability to be a guardian.

One more thing convinced me this is an oversight rather than a deliberate quota. Your genesis builder hits the same overflow on the same map and handles it the opposite way:

```rust
// Update GuardianIndex so guardian can look up their protected accounts
GuardianIndex::<T>::mutate(guardian, |accounts| {
    if !accounts.contains(who) {
        // In genesis, we use saturating push - if limit exceeded, log warning
        if accounts.try_push(who.clone()).is_err() {
            log::warn!("Guardian {:?} has too many accounts, cannot add {:?} to index", guardian, who);
        }
    }
});
```

Genesis logs a warning and carries on. The extrinsic uses ? and fails the whole call. The storage comment says the map exists so the UI can look up a guardian's accounts, which is the same best-effort role genesis treats it as. Bounding the storage never required refusing the enrollment, and the genesis path is the proof: it bounds the vector and still admits the account.

### Impact Details

Any account can be targeted, and the target does nothing to invite it. Afterwards no new user can enroll in high security with that guardian. Existing relationships keep working, since those records already sit in HighSecurityAccounts.

The account worth attacking is one that lots of users want as their guardian. A custody provider or a recovery service is the obvious case. For the price of 32 existential deposits and 32 transaction fees, you permanently remove their ability to take on anyone new.

Nothing gets stolen and nothing gets frozen. What's lost is the security feature itself, for the victim and for everyone who wanted to pick them.

## References

* pallets/reversible-transfers/src/lib.rs:376,394,248
* runtime/src/configs/mod.rs:528
* runtime/src/lib.rs:232

## Proof of Concept

The test below runs inside the pallet's own test module. It fills the index from attacker accounts, then shows that an honest, unrelated user cannot enroll, and that the state never recovers.

```
cargo test -p pallet-reversible-transfers --lib attacker_permanently_denies -- --nocapture
running 1 test
test tests::test_guardian_index_griefing::attacker_permanently_denies_the_guardian_role_of_an_uninvolved_account ... ok
test result: ok. 1 passed; 0 failed; 0 ignored; 0 measured; 51 filtered out; finished in 0.01s
```

Full source: <https://gist.github.com/gabchess/573942d2122359f37b793dfa9e0d4e88>

## Recommendation

Two changes fix this. The first one on its own is enough.

Make the guardian accept the role. Add a pending-guardian map plus a second extrinsic that the named guardian signs. set\_high\_security writes the request, and the guardian's acceptance completes it and writes GuardianIndex. Substrate already uses this shape for accept\_ownership, and it removes the write-into-someone-else's-storage primitive completely.

If you'd rather avoid a two-step flow, make the index recoverable instead: add an extrinsic letting a guardian drop an account they never agreed to protect. That's the weaker option, since the victim has to notice the attack and then pay to clean up after it, and the attacker can refill any slot that frees up.

I'd take the first. The second one treats the symptom and leaves the real rule untouched, which is that anybody can write into anybody else's bounded storage.
