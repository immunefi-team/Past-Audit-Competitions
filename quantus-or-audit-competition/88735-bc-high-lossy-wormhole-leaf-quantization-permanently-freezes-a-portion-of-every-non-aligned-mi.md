# 88735 bc high lossy wormhole leaf quantization permanently freezes a portion of every non aligned mining reward and transaction fee payout

**Submitted on Aug 14th 2026 at 18:43:26 UTC by @Pig46940 for** [**Audit Comp | Quantus**](https://immunefi.com/audit-competition/audit-comp-quantus)

* **Report ID:** #88735
* **Report Type:** Blockchain/DLT
* **Report severity:** High
* **Target:** <https://github.com/immunefi-team/audit-comp-quantus-chain>
* **Impacts:**
  * Permanent freezing of funds

## Description

## Brief/Intro

Quantus mints mining rewards and collected transaction fees to keyless Wormhole addresses using their full planck-denominated values, but the ZK-tree leaf commits only `floor(amount / 10^10)`. Wormhole withdrawals can mint only whole multiples of `10^10` planck. Consequently, whenever a miner payout is not divisible by `10^10`, its remainder is credited to the miner's on-chain Wormhole balance but is absent from the ZK commitment and can never be withdrawn. The configured emission formula produces such non-aligned rewards during normal block finalization, so miner funds are permanently frozen without requiring malicious or privileged behavior.

## Vulnerability Details

### Security invariant

For every credit to a keyless Wormhole account, the amount represented by the ZK leaf must equal the amount credited on chain. If the circuit intentionally supports only quantized amounts, the runtime must pay only whole quanta or retain the remainder in recoverable state for a later payout.

In mathematical terms, for a keyless credit `A` and the Wormhole quantum `Q`:

```
credited_amount == withdrawable_amount
```

or, if quantization is intentional:

```
credited_amount % Q == 0
```

The current mining-reward path satisfies neither condition.

### Root cause

The root cause is a missing reconciliation boundary between two Quantus-specific accounting systems:

* The native currency layer accounts for exact planck-denominated balances.
* The Wormhole ZK layer can commit and later reconstruct only whole `10^10`-planck quanta.

`pallet-mining-rewards::on_finalize` in `pallets/mining-rewards/src/lib.rs` chooses the reward solely from the monetary-policy formula `remaining_supply / emission_divisor`. Nothing between that calculation and `mint_into` either aligns the payout to `SCALE_DOWN_FACTOR` or defers the unrepresentable remainder. The raw amount consequently reaches `record_transfer_proof` and then `hash_leaf`, where integer division silently removes part of the credited value from the commitment.

The defect is therefore not narrowly the absence of one particular dust-accumulator implementation. The missing requirement is that every credit intended to be exited from a keyless Wormhole account must be reconciled with the ZK amount quantum before or during recording.

{% stepper %}
{% step %}

### The mining pallet calculates an arbitrary planck-denominated reward

The block reward is calculated from the remaining supply and divided by `EmissionDivisor`. It is then split using `Permill::mul_floor`:

* `pallets/mining-rewards/src/lib.rs`: `fn on_finalize()`

```rust
let total_reward = remaining_supply
		.checked_div(&emission_divisor)
		.unwrap_or_else(BalanceOf::<T>::zero);

// Split the reward between treasury and miner
let treasury_portion = T::Treasury::portion();
let treasury_reward = treasury_portion.mul_floor(total_reward);
let miner_reward = total_reward.saturating_sub(treasury_reward);
```

There is no rounding to the Wormhole quantum and no remainder accumulator.

Transaction fees and the miner block reward are then paid separately:

* `pallets/mining-rewards/src/lib.rs`: `fn on_finalize()`

```rust
// Send fees to miner if any
Self::mint_reward(miner.as_ref(), tx_fees);

// Send block rewards to miner
Self::mint_reward(miner.as_ref(), miner_reward);
```

Processing them separately makes the problem worse: even if their sum happened to be aligned, each amount is committed in its own independently rounded leaf.
{% endstep %}

{% step %}

### The full amount is minted to a keyless Wormhole address

Mining rewards are sent to an address derived from the miner's pre-runtime digest. Quantus documentation states that all mining rewards are automatically sent to these Wormhole addresses. They are not ordinary accounts controlled by a transaction-signing key; value is recovered through a Wormhole ZK exit.

`mint_reward` mints the complete, unrounded amount and passes exactly that amount to the proof recorder:

* `pallets/mining-rewards/src/lib.rs`: `fn mint_reward()`

```rust
match T::Currency::mint_into(miner, reward) {
	Ok(_) => {
		T::ProofRecorder::record_transfer_proof(
			None, // Native token
			mint_account,
			miner.clone(),
			reward,
	         );
	        Self::deposit_event(Event::MinerRewarded {
			miner: miner.clone(),
			reward,
	        });
},
```

At this point the miner's balance has increased by `reward` planck.
{% endstep %}

{% step %}

### The ZK-tree silently discards the remainder

The ZK leaf represents its amount as a quantized `u32`. `hash_leaf` performs integer division by `10_000_000_000`:

* `pallets/zk-tree/src/tree.rs`

```rust
pub const AMOUNT_SCALE_DOWN_FACTOR: u128 = 10_000_000_000;
```

* `pallets/zk-tree/src/tree.rs`: `fn hash_leaf()`

```rust
let amount_u128: u128 = leaf.amount.into();
let amount_quantized =
    u32::try_from(amount_u128 / AMOUNT_SCALE_DOWN_FACTOR).unwrap_or(u32::MAX);
```

Integer division rounds down. If:

```
reward = n * 10^10 + remainder
0 < remainder < 10^10
```

the leaf commits only `n`. A leaf for `n * 10^10 + remainder` is identical to a leaf for exactly `n * 10^10`, assuming all other fields are equal.

A positive credit smaller than `10^10` commits the same quantized amount as a zero credit.
{% endstep %}

{% step %}

### No ZK exit can recover the discarded remainder

During withdrawal, the runtime converts the circuit's quantized output back into a balance by multiplying it by the same factor:

* `pallets/wormhole/src/lib.rs`: `fn process_exit_bundle()`

```rust
let exit_balance_u128 = (account_data.summed_output_amount as u128)
    .saturating_mul(crate::SCALE_DOWN_FACTOR);
```

Therefore every amount produced by a valid exit is a multiple of `10^10`. The prover cannot claim the discarded remainder because it was never committed into the leaf, and attempting to claim a larger quantized amount would not match the authoritative Merkle root.

The permanently inaccessible amount for each leaf is consequently:

```
frozen_remainder = credited_amount % 10_000_000_000
```

{% endstep %}

{% step %}

### Quantus already enforces this invariant for vesting, but not mining

The runtime's vesting configuration explicitly recognizes this failure mode:

* `runtime/src/configs/mod.rs`

```rust
/// Vesting payouts are rounded down to multiples of the wormhole leaf quantum
/// (`SCALE_DOWN_FACTOR`): a sub-quantum transfer would be committed as a
/// zero-value leaf, stranding funds paid to keyless beneficiaries.
pub const VestingPayoutQuantum: Balance = pallet_wormhole::SCALE_DOWN_FACTOR;
```

Vesting does **not** suffer from the reported loss. It rounds the currently owed amount down before transferring anything:

* `pallets/vesting/src/lib.rs`: `fn claim_plan()`

```rust
let owed = vested.checked_sub(&schedule.claimed).ok_or(ArithmeticError::Underflow)?;
let candidate = Self::quantize_down(owed);

fn quantize_down(amount: BalanceOf<T>) -> BalanceOf<T> {
    let remainder = amount % T::PayoutQuantum::get();
    amount.checked_sub(&remainder).expect("remainder never exceeds the dividend")
}
```

Only the quantized `payable` value is transferred and recorded. `settle` advances `schedule.claimed` by that payable value, rather than by the larger unrounded `owed` value:

* `pallets/vesting/src/lib.rs`

```rust
Self::pay_out(&Self::pot_account_id(), &schedule.beneficiary, amount)?;
schedule.claimed = schedule.claimed.checked_add(&amount).ok_or(ArithmeticError::Overflow)?;
```

The sub-quantum difference therefore never enters the beneficiary's keyless account and is never falsely treated as paid. It remains part of the schedule's unpaid obligation and can be included in a later claim after enough additional value has vested. Schedule totals must themselves be quantum-aligned, so an ordinarily completed schedule can ultimately drain exactly. If an administrator ends a schedule early, the implementation returns the sub-quantum vested dust to the signature-controlled treasury instead of transferring it to the keyless beneficiary.

Vesting consequently reconciles exact planck accounting with the ZK quantum by rounding **before** transfer and preserving the unpaid obligation. It needs no separate dust ledger because `schedule.total - schedule.claimed` already performs that role.

Mining rewards and transaction fees use the same Wormhole proof recorder but have no equivalent reconciliation step or per-miner unpaid obligation. They mint the raw amount first, after which the ZK-tree's floor division makes the remainder unrecoverable. The contrast with vesting demonstrates that the loss is not an unavoidable property of Wormhole quantization; it is specific to the unprotected mining-reward path.
{% endstep %}
{% endstepper %}

### Concrete example

Using the repository's test runtime constants (`MAX_SUPPLY = 21,000,000 * 10^12`, `EmissionDivisor = 15,163,560`, 50% treasury portion, and initial issuance of three planck), the first reward is:

```
total_reward  = 1,384,899,060,642 planck
miner_reward  =   692,449,530,321 planck
quantized     = floor(692,449,530,321 / 10,000,000,000)
              = 69
withdrawable  =   690,000,000,000 planck
remainder     =     2,449,530,321 planck
```

The miner receives a balance credit of `692,449,530,321` planck, but the corresponding leaf can authorize at most `690,000,000,000` planck before any applicable exit fee. The remaining `2,449,530,321` planck is not represented in the proof system.

The exact remainder changes as total issuance changes. It does not naturally accumulate into a later leaf because every reward leaf is quantized independently.

### Distinction from the published known issue

The known-issues list states that mining rewards may eventually fall "well below the quantized minimum for the circuit" and that the circuit and chain will be upgraded when that point approaches.

* `https://immunefi.com/audit-competition/audit-comp-quantus/information`

This report concerns a different present-day condition:

* The reward does not need to be below the minimum. A reward many times larger than one quantum still loses `reward % 10^10`.
* The loss happens on every currently non-aligned mining-reward leaf, not only at a future stage when the whole reward becomes sub-quantum.
* Transaction fees are recorded separately and suffer the same loss independently of the future emission level.

An upgrade planned for the future case where the entire mining reward falls below one quantum does not recover remainders already credited but omitted from historical leaves.

## Impact Details

### Selected impact

**Permanent freezing of funds**

The affected funds are real native tokens included in total issuance and credited to the miner's Wormhole account. However, no valid proof can represent or withdraw them. Because the account is keyless, the miner cannot use an ordinary signed balance transfer as an alternative recovery path. Recovering historical remainders would require protocol intervention, such as a runtime migration or another privileged state correction.

This is triggered by normal block production; no compromised credential, privileged actor, malicious genesis block, Sybil attack, or upstream Substrate defect is required.

For every affected reward or fee leaf, the permanently frozen amount is between one planck and `9,999,999,999` planck. Although each individual remainder is small, mining rewards occur every block and fee payouts create additional independently rounded leaves. The aggregate loss therefore grows continually over the lifetime of the chain and affects miners receiving the protocol's intended compensation.

The defect does not permit an attacker to steal the remainder. The impact is permanent unavailability of protocol-issued miner funds.

## Suggested Remediation

There are multiple valid ways to restore the invariant. The essential requirement is to reconcile the exact reward with the Wormhole quantum before a credit is finalized.

{% stepper %}
{% step %}

### Option 1: Round before minting and preserve the unpaid entitlement

Mining can follow the vesting strategy: calculate an aligned payable amount, mint and record only that amount, and leave the shortfall as an unpaid entitlement for the same miner. Unlike vesting, mining currently has no per-beneficiary schedule, so preserving attribution would require adding equivalent per-miner state. Merely rounding the reward down and returning the difference to the global remaining supply prevents inaccessible minting, but it may underpay the miner who earned that block.
{% endstep %}

{% step %}

### Option 2: Carry the remainder into a later miner leaf

Maintain a per-miner remainder accumulator and use previously minted, uncommitted dust when constructing a later aligned leaf:

```
available = pending_dust[miner] + miner_reward + transaction_fees
payable = floor(available / quantum) * quantum
pending_dust[miner] = available - payable
mint the new reward and record only the newly representable payable amount
```

The exact minting and recording order must ensure that cumulative recorded value never exceeds the miner's cumulative credited balance. The accumulator must be per miner; a global accumulator could give one miner the remainder belonging to another miner.

Combining the current block's fee payout and mining reward before quantization also avoids losing two independent remainders, but it does not replace the need to carry the remaining dust forward. Existing historical remainders require a migration or another explicit recovery mechanism.
{% endstep %}

{% step %}

### Option 3: Remove the lossy amount boundary

A longer-term alternative is to modify the circuit and leaf encoding to commit the complete native-token balance. That requires coordinated circuit, verifier, runtime, and proof-format changes.
{% endstep %}
{% endstepper %}

## References

* Quantus chain repository: <https://github.com/immunefi-team/audit-comp-quantus-chain>
* Mining reward calculation and minting: <https://github.com/immunefi-team/audit-comp-quantus-chain/blob/audit-comp-ready/pallets/mining-rewards/src/lib.rs#L155-L199>
* Mining reward proof recording: <https://github.com/immunefi-team/audit-comp-quantus-chain/blob/audit-comp-ready/pallets/mining-rewards/src/lib.rs#L220-L241>
* ZK-tree amount quantization: <https://github.com/immunefi-team/audit-comp-quantus-chain/blob/audit-comp-ready/pallets/zk-tree/src/tree.rs#L134-L149>
* Wormhole exit scaling: <https://github.com/immunefi-team/audit-comp-quantus-chain/blob/audit-comp-ready/pallets/wormhole/src/lib.rs#L787-L798>
* Vesting's explicit payout-quantum protection: <https://github.com/immunefi-team/audit-comp-quantus-chain/blob/audit-comp-ready/runtime/src/configs/mod.rs#L562-L580>
* Vesting claim quantization and settlement: <https://github.com/immunefi-team/audit-comp-quantus-chain/blob/audit-comp-ready/pallets/vesting/src/lib.rs#L528-L620>
* Vesting quantization regression tests: <https://github.com/immunefi-team/audit-comp-quantus-chain/blob/audit-comp-ready/pallets/vesting/src/tests.rs#L731-L805>
* Mining documentation describing automatic Wormhole reward addresses: <https://github.com/immunefi-team/audit-comp-quantus-chain/blob/audit-comp-ready/MINING.md#L41-L48>
* Wormhole ZK documentation: <https://github.com/immunefi-team/audit-comp-quantus-chain/blob/audit-comp-ready/docs/wormhole-zk.md>
* Quantus competition information and known issues: <https://immunefi.com/audit-competition/audit-comp-quantus/information/>

## Proof of Concept

The PoC consists of two focused unit tests. The first demonstrates that the production mining formula mints and records a non-quantized miner reward. The second demonstrates that the ZK-tree commitment discards the remainder and cannot distinguish the credited amount from its rounded-down value.

{% stepper %}
{% step %}

### Add the mining-reward test

Add the following test to `pallets/mining-rewards/src/tests.rs`:

```rust
#[test]
fn poc_miner_reward_is_not_aligned_to_wormhole_quantum() {
    new_test_ext().execute_with(|| {
        const WORMHOLE_QUANTUM: Balance = 10_000_000_000;

        set_miner_preimage_digest(MINER_1.preimage());
        let current_supply = Balances::total_issuance();
        let total_reward = (MaxSupply::get() - current_supply) / EmissionDivisor::get();
        let miner_reward = total_reward - Treasury::portion().mul_floor(total_reward);

        assert_ne!(
            miner_reward % WORMHOLE_QUANTUM,
            0,
            "the configured mining reward unexpectedly happens to be quantum-aligned"
        );

        MockProofRecorder::clear();
        MiningRewards::on_finalize(1);

        let miner_proof = MockProofRecorder::get_recorded_proofs()
            .into_iter()
            .find(|proof| proof.to == MINER_1.account_id())
            .expect("a miner reward proof must be recorded");

        assert_eq!(miner_proof.amount, miner_reward);
        assert_ne!(miner_proof.amount % WORMHOLE_QUANTUM, 0);
    });
}
```

This proves that the configured emission formula produces a non-aligned miner reward and that the complete non-aligned amount is passed to the Wormhole proof-recording path.
{% endstep %}

{% step %}

### Add the ZK-tree commitment test

`pallets/zk-tree/src/tests.rs` already contains the `amount_leaf` helper. Add the following test after that helper:

```rust
#[test]
fn poc_non_aligned_credit_remainder_is_absent_from_leaf_commitment() {
    let quantum = tree::AMOUNT_SCALE_DOWN_FACTOR;
    let committed = 138 * quantum;
    let credited = committed + 4_899_060_642;

    assert_eq!(
        tree::hash_leaf::<Test>(&amount_leaf(credited)),
        tree::hash_leaf::<Test>(&amount_leaf(committed)),
        "the leaf commitment discards the credited sub-quantum remainder"
    );

    assert_eq!(
        tree::hash_leaf::<Test>(&amount_leaf(quantum - 1)),
        tree::hash_leaf::<Test>(&amount_leaf(0)),
        "a positive sub-quantum credit commits as zero"
    );
}
```

This proves that the ZK-tree commitment contains only the rounded-down amount. The credited amount and rounded-down amount generate exactly the same leaf hash. It also demonstrates that an entire positive sub-quantum credit is indistinguishable from zero in the leaf commitment.
{% endstep %}

{% step %}

### Run the tests

From the repository root, run:

```bash
cargo test -p pallet-mining-rewards \
  poc_miner_reward_is_not_aligned_to_wormhole_quantum -- --nocapture

cargo test -p pallet-zk-tree \
  poc_non_aligned_credit_remainder_is_absent_from_leaf_commitment -- --nocapture
```

{% endstep %}

{% step %}

### Expected result

Both tests pass.

The first passing test establishes that the mining pallet creates and records an amount with a non-zero remainder modulo `10^10`. The second passing test establishes that this remainder does not affect the authoritative leaf commitment.

Finally, inspection of `pallets/wormhole/src/lib.rs` lines 791-793 shows that every accepted exit amount is calculated as the circuit's `summed_output_amount * 10^10`. Therefore, the omitted remainder cannot be expressed by any valid exit and remains permanently frozen in the miner's keyless Wormhole account.
{% endstep %}
{% endstepper %}
