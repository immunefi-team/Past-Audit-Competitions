# 89832 bc critical unbounded transaction tips bypass high security containment and drain compromised accounts

**Submitted on Aug 21st 2026 at 12:11:12 UTC by @Grant4969 for** [**Audit Comp | Quantus**](https://immunefi.com/audit-competition/audit-comp-quantus)

* **Report ID:** #89832
* **Report Type:** Blockchain/DLT
* **Report severity:** Critical
* **Target:** <https://github.com/immunefi-team/audit-comp-quantus-chain>
* **Impacts:**
  * Direct loss of funds

## Description

## Brief/Intro

Quantus high-security mode expressly exists to contain a compromised account signing key: after enrollment, immediate value exits must remain blocked in favor of delayed transfers that a guardian can cancel. However, `ReversibleTransactionExtension` enforces this policy only over `RuntimeCall`. A transaction tip is separate signed data inside `ChargeTransactionPayment`, so the same restricted signer can attach an almost balance-sized tip to the explicitly whitelisted `Vesting::claim`. The payment is withdrawn and finalized even when the claim fails, immediately reducing the high-security account to its existential deposit without creating any reversible hold. The full-runtime PoC also proves that transaction-pool validation accepts the extrinsic and that later deposits can be drained repeatedly.

## Vulnerability Details

### This is a high-security policy escape, not ordinary use of an unrestricted key

The competition's general exclusions mention impacts requiring leaked keys or credentials. This report does not claim that key compromise itself is a protocol vulnerability, nor that an unrestricted account owner used authority that the protocol intended to grant.

High-security is a Quantus-specific containment control whose documented purpose is to keep enforcing restrictions after the ordinary signing key is compromised. Its pallet documentation states that enrollment is permanent so that an attacker who gains access to the account cannot disable the protections and that funds subsequently sent to a compromised account remain protected and recoverable by the guardian.

Consequently, possession of the signing key intentionally grants only restricted-signer authority. The vulnerability is that this restricted signer escapes the Quantus-enforced boundary through another value-bearing field in the same signed extrinsic that the custom policy never inspects. This is analogous to a constrained session key bypassing its spending limit through an unchecked signed field. The missing enforcement and affected security control are Quantus-owned runtime integration code; standard transaction-payment behavior only exposes the gap.

### This does not rely on a trusted role or an accepted off-chain safeguard

No allocator, curator, approver, guardian, governance actor, operator, or other permissioned role participates in the exploit. The attacker has only the ordinary account key that high-security mode expressly treats as an untrusted, restricted signer after enrollment.

Nor is the report asking Quantus to introduce a new solvency invariant or replace a documented off-chain trust assumption. Quantus already enforces this exact containment invariant on-chain: `ReversibleTransactionExtension` rejects signed transactions whose `RuntimeCall` is outside the high-security whitelist. The exploit supplies an explicitly allowed call but moves value through an unchecked sibling field in the same signed payload. It therefore bypasses an existing code-enforced constraint rather than relying on the absence of a constraint the protocol deliberately left to trusted actors.

The competition's disclosed Known Issues do not describe transaction tips, `ChargeTransactionPayment`, or this split policy boundary.

### The Quantus policy checks only `RuntimeCall`

`ReversibleTransactionExtension` implements `TransactionExtension<RuntimeCall>`. Its validation logic passes only `call` to the high-security policy:

```rust
let who = ensure_signed(origin.clone())?;
if !HighSecurityConfig::is_call_allowed(&who, call) {
    return Err(InvalidTransaction::Custom(1).into());
}
```

The Quantus whitelist explicitly admits `Vesting::claim` because the call cannot redirect a legitimate vesting payout:

```rust
RuntimeCall::Vesting(pallet_vesting::Call::claim { .. })
```

However, the production signed-extension tuple also contains:

```rust
pallet_transaction_payment::ChargeTransactionPayment<Runtime>
```

The tip is an arbitrary compact-encoded balance stored in that separate extension, outside `RuntimeCall`. `ReversibleTransactionExtension` has no reference to it and therefore cannot enforce the high-security invariant over it. Reordering the two extensions would not fix this because one extension does not automatically receive another extension's value.

### Transaction payment immediately settles the unchecked value exit

During transaction preparation, the predicted fee including the caller-selected tip is withdrawn from the signer. During post-dispatch settlement, standard transaction-payment logic computes and deposits the actual fee including the same tip irrespective of whether dispatch succeeded. This is normal behavior for unrestricted accounts and is not alleged to be an upstream Substrate bug. The Quantus vulnerability is composing this value-bearing field with a custom security policy that examines only the call.

Therefore, a restricted signer can submit an allowed `Vesting::claim { schedule_id: 0 }` with:

```
tip = free_balance - existential_deposit - inclusion_fee
```

The high-security extension accepts the call. The payment extension withdraws the amount. The nonexistent schedule makes the claim fail, which demonstrates that the business call produces no payout or useful state transition, but the full tip remains charged. Failure is not required for exploitation; it is only the cleanest minimal reproduction because any successful whitelisted call can carry the same tip.

No pending reversible transfer is created, so the guardian has no delay window or cancellation target. Quantus' `TransactionFeesCollector` records the payment in `MiningRewards::CollectedFees`; `MiningRewards::on_finalize` subsequently remints the collected transaction fees to the current block author.

### Attack sequence

{% stepper %}
{% step %}

## The victim enrolls the account

The victim enrolls the account in high-security mode and appoints a different guardian.
{% endstep %}

{% step %}

## The ordinary signing key is compromised

The ordinary signing key is later compromised, which is the state the feature expressly promises to contain.
{% endstep %}

{% step %}

## The restricted signer creates a high-tip transaction

The restricted signer creates an explicitly whitelisted `Vesting::claim` and sets the transaction tip to virtually the full free balance.
{% endstep %}

{% step %}

## Transaction-pool validation accepts the extrinsic

Production transaction-pool validation accepts the extrinsic because the custom policy examines only `RuntimeCall`.
{% endstep %}

{% step %}

## Transaction payment withdraws the value

Transaction payment withdraws the fee and tip before dispatch.
{% endstep %}

{% step %}

## The claim fails but the payment is retained

The claim fails because the schedule does not exist, but payment settlement retains the full tip.
{% endstep %}

{% step %}

## The account is drained without a reversible transfer

The account is left with only its existential deposit and no pending transfer that the guardian can cancel.
{% endstep %}

{% step %}

## The drain can be repeated

If a third party later replenishes the compromised account, the signer increments the nonce and repeats the same drain.
{% endstep %}
{% endstepper %}

The attack requires no guardian key, governance privilege, valid vesting schedule, malicious token, majority hash power, reorganization, race, or arithmetic edge case.

## Impact Details

The selected in-scope impact is **Direct loss of funds**, rated Critical by the Quantus competition. The victim's high-security balance is immediately and irreversibly reduced regardless of which miner includes the transaction. The loss is limited only by the account's balance, bypasses both the guardian intervention window and the reversible hold, and can be repeated against every later deposit.

Miner receipt is secondary evidence rather than a prerequisite for the Critical impact. If the attacker produces an otherwise normal block and privately includes the transaction, Quantus remints the collected payment to that block author. This does not require majority hash power or a chain reorganization. Even when an unrelated miner includes it, the victim suffers the same deterministic direct loss.

## References

* High-security's compromised-key guarantee: <https://github.com/immunefi-team/audit-comp-quantus-chain/blob/3b243b870a5442a6e5f443056f67bbba87d472f5/pallets/reversible-transfers/src/lib.rs#L339-L366>
* Call-only high-security transaction extension: <https://github.com/immunefi-team/audit-comp-quantus-chain/blob/3b243b870a5442a6e5f443056f67bbba87d472f5/runtime/src/transaction\\_extensions.rs#L60-L80>
* High-security whitelist including `Vesting::claim`: <https://github.com/immunefi-team/audit-comp-quantus-chain/blob/3b243b870a5442a6e5f443056f67bbba87d472f5/runtime/src/configs/mod.rs#L660-L699>
* Production `TxExtension` composition: <https://github.com/immunefi-team/audit-comp-quantus-chain/blob/3b243b870a5442a6e5f443056f67bbba87d472f5/runtime/src/lib.rs#L153-L176>
* Standard tip storage and settlement mechanics: <https://github.com/immunefi-team/audit-comp-quantus-chain/blob/3b243b870a5442a6e5f443056f67bbba87d472f5/pallets/transaction-payment/src/lib.rs#L794-L808>
* Quantus fee collection and miner reminting: <https://github.com/immunefi-team/audit-comp-quantus-chain/blob/3b243b870a5442a6e5f443056f67bbba87d472f5/pallets/mining-rewards/src/lib.rs#L192-L217>

## Recommended Fix

Enforce the high-security invariant over the complete signed extrinsic, not only `RuntimeCall`. At minimum, reject every nonzero `ChargeTransactionPayment::tip()` from a high-security signer. A per-transaction cap is insufficient because a compromised key can submit repeated transactions; any permitted allowance must be cumulative over the guardian delay. Architecturally, the policy should inspect every signed extension capable of debiting the origin, or the tip restriction should be integrated directly into transaction payment. Ordinary inclusion fees may remain payable, but discretionary tips must be forbidden or routed into a guardian-cancellable hold.

## Link to Proof of Concept

<https://gist.github.com/Grant3124/fdb8e665a48f363316dabed94a9f023b>

## Proof of Concept

The PoC runs against the exact in-scope commit:

```
3b243b870a5442a6e5f443056f67bbba87d472f5
```

It uses an ML-DSA-65 account and the production runtime's complete signed extrinsic, transaction-extension tuple, transaction-pool validation, dispatch pipeline, transaction-payment adapter, and mining-rewards finalization.

### Steps to reproduce

{% stepper %}
{% step %}

## Apply the patch

Download the raw file from the linked secret Gist as `immunefi-c01-poc.patch`, place it in the repository root, and apply it:

```bash
git apply immunefi-c01-poc.patch
```

The patch adds `audit_poc_high_security_balance_can_be_drained_via_tip`, its maximum-tip helper, and the required imports to `runtime/tests/transactions/ml_dsa_65.rs`.
{% endstep %}

{% step %}

## Run the test

From the repository root, run:

```bash
SKIP_WASM_BUILD=1 cargo test -p quantus-runtime --test mod \
  audit_poc_high_security_balance_can_be_drained_via_tip \
  --release --locked -- --nocapture
```

{% endstep %}

{% step %}

## Observe the result

```
running 1 test
test transactions::ml_dsa_65::audit_poc_high_security_balance_can_be_drained_via_tip ... ok

test result: ok. 1 passed; 0 failed; 0 ignored; 0 measured; 43 filtered out
```

{% endstep %}
{% endstepper %}

### What the test proves

{% stepper %}
{% step %}

## Fund and enroll the account

It funds an ML-DSA-65 account with 1000 UNIT and enrolls it in high-security mode with a distinct guardian.
{% endstep %}

{% step %}

## Verify the zero-tip control

As a control, it applies the same nonexistent `Vesting::claim` with `tip=0` and proves that only the ordinary inclusion fee is charged.
{% endstep %}

{% step %}

## Compute the maximum tip

It computes the maximum tip that leaves exactly the existential deposit:

```rust
let tip = balance - EXISTENTIAL_DEPOSIT - inclusion_fee;
```

{% endstep %}

{% step %}

## Construct the signed exploit extrinsic

It constructs a fully signed production extrinsic containing the explicitly whitelisted claim and that tip.
{% endstep %}

{% step %}

## Validate through the public transaction pool

It proves the ordinary external transaction-pool path accepts the exploit:

```rust
Executive::validate_transaction(
    TransactionSource::External,
    xt.clone(),
    System::block_hash(0),
).expect("the public transaction pool must accept the high-tip extrinsic");
```

{% endstep %}

{% step %}

## Apply the transaction

It applies the transaction through every production extension with `Executive::apply_extrinsic`. The vesting call fails because schedule 0 does not exist, but the transaction itself remains valid and payment is finalized.
{% endstep %}

{% step %}

## Verify the charged tip

It obtains `TransactionPayment::TransactionFeePaid` and proves that the charged tip equals the attacker-selected amount.
{% endstep %}

{% step %}

## Verify the account drain

It proves the account is reduced to exactly `EXISTENTIAL_DEPOSIT` and that `PendingTransfersBySender` is empty.
{% endstep %}

{% step %}

## Replenish the compromised account

It transfers another 1000 UNIT into the compromised account from a third party.
{% endstep %}

{% step %}

## Repeat the drain

With nonce 2, it proves transaction-pool validation accepts the repeated exploit and drains the replenished account back to `EXISTENTIAL_DEPOSIT`.
{% endstep %}

{% step %}

## Verify fee collection

It proves all three payments—the zero-tip control and both drains—enter `MiningRewards::collected_fees()`.
{% endstep %}

{% step %}

## Verify miner reminting

It sets an attacker-controlled QPoW author preimage and finalizes the block. A dedicated event proves that all collected transaction fees are reminted to that author:

```rust
assert!(System::events().into_iter().any(|record| matches!(
    record.event,
    RuntimeEvent::MiningRewards(
        pallet_mining_rewards::Event::MinerRewarded { ref miner, reward }
    ) if miner == &author && reward == collected_fees
)));
```

{% endstep %}
{% endstepper %}

The tip=0 control isolates the discretionary tip as the drain vector. Transaction-pool validation rules out a block-author-only invalid extrinsic. The second drain directly tests the pallet's promise that funds later sent to a compromised high-security account remain protected. The exact miner event distinguishes fee reminting from the separate base block reward.
