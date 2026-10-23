# 90133 bc high mining rewards are minted at raw precision but their zk leaves commit only floor amount 0 01 quan permanently freezing every remainder and making ordinary fee yield wholly unc&#x20;

**Submitted on Aug 23rd 2026 at 07:21:23 UTC by @tinnohofficial for** [**Audit Comp | Quantus**](https://immunefi.com/audit-competition/audit-comp-quantus)

* **Report ID:** #90133
* **Report Type:** Blockchain/DLT
* **Report severity:** High
* **Target:** <https://github.com/immunefi-team/audit-comp-quantus-chain>
* **Impacts:**
  * Permanent freezing of unclaimed yield

## Description

## Brief/Intro

QPoW miners are paid to Poseidon-derived wormhole addresses. That derivation produces no ML-DSA keypair, so a miner has no signing key and the **only** way to spend a credit is to prove the ZK-tree leaf that commits it.

`MiningRewards::on_finalize` mints transaction fees and the block reward as two separate, full-precision balance credits and records a leaf for each. The leaf hash commits only:

```
floor(raw_amount / 10_000_000_000)      // 10^10 planck = 0.01 QUAN
```

The remainder exists in the miner's keyless account and in no proof the miner can ever construct. Two consequences follow, both automatic and both recurring:

1. **An ordinary block's entire fee yield can be unclaimable.** A real ML-DSA-65 `transfer_keep_alive` pays `10,994,787,000` planck in fee. That floors to **one quantum**, and at the configured 4 bps wormhole rate a one-quantum leaf permits **zero** output. The miner earned real, minted, on-chain value with no positive claim against it.
2. **Every non-aligned block reward strands its remainder forever.** The integer emission formula is almost never quantum-aligned, so `miner_reward mod 10^10` is lost on essentially every block.

## Vulnerability Details

### The miner cannot sign; the leaf is the only spend path

`extract_author_from_digest` hashes the miner-supplied preimage with Poseidon and decodes the image as the reward account:

```rust
let preimage: [u8; 32] = data.as_slice().try_into().ok()?;
let address_bytes = derive_wormhole_address(preimage).ok()?;
return AccountId::decode(&mut &address_bytes[..]).ok();
```

`primitives/wormhole/src/lib.rs:113-150`. A Poseidon image is not an ML-DSA public key, so no ordinary signed transfer is possible from it.

### Fees and block rewards are minted raw, and recorded raw

Fees are routed into the pallet by the production payment adapter (`runtime/src/configs/mod.rs:477-480`) and accumulated in `CollectedFees` (`pallets/mining-rewards/src/lib.rs:309-318`). At finalization they are paid out as two separate credits:

```rust
Self::mint_reward(miner.as_ref(), tx_fees);
Self::mint_reward(miner.as_ref(), miner_reward);
```

`pallets/mining-rewards/src/lib.rs:192-196`. Each call mints the full raw amount and records that same raw value as an independent proof:

```rust
match T::Currency::mint_into(miner, reward) {
    Ok(_) => {
        T::ProofRecorder::record_transfer_proof(None, mint_account, miner.clone(), reward);
```

`pallets/mining-rewards/src/lib.rs:220-241`. There is no quantum alignment and no minimum-payout check on either call.

### The authoritative leaf hash floors the amount

The tree stores the raw `u128`, but the root hashes a quantized `u32`:

```rust
pub const AMOUNT_SCALE_DOWN_FACTOR: u128 = 10_000_000_000;

let amount_u128: u128 = leaf.amount.into();
let amount_quantized =
    u32::try_from(amount_u128 / AMOUNT_SCALE_DOWN_FACTOR).unwrap_or(u32::MAX);
felts.extend(bytes_to_felts_compact_lossy(&(amount_quantized as u64).to_le_bytes()));
```

`pallets/zk-tree/src/tree.rs:30-34,134-149`. Leaves whose raw amounts differ only below `10^10` planck hash identically.

### A one-quantum leaf has zero output capacity

Every leaf proof independently enforces:

```
(output_1 + output_2) * 10000 <= input * (10000 - fee_bps)
```

`qp-zk-circuits/wormhole/circuit/src/zk_merkle_proof.rs:491-499`, where `input` is that leaf's own quantized amount. With `VolumeFeeRateBps = 4` and `input = 1`, a single output quantum would require `1 * 10000 <= 1 * 9996`, which is false. **Maximum output is zero.**

On chain, an exit scales only the proven integer output back to planck:

```rust
let exit_balance_u128 = (account_data.summed_output_amount as u128)
    .saturating_mul(crate::SCALE_DOWN_FACTOR);
```

`pallets/wormhole/src/lib.rs:776-806`. Nothing reads the account's free balance to derive exit capacity.

### No later proof can recover a remainder

Aggregation sums valid outputs; it cannot move an omitted remainder into another leaf or let a leaf exceed its own bound. The omitted `raw mod 10^10` is absent from the first proof and from every possible subsequent proof, so choosing a smaller output and "redeeming the rest later" cannot reach it.

## Impact Details

The frozen value is protocol-generated miner yield credited to an account that cannot sign, not a user's voluntary dust transfer.

**Wholly unclaimable fee yield.** Any block whose total fees floor to exactly one quantum produces a leaf with zero output capacity — a 100% loss of that credit. The PoC demonstrates it with a single ordinary signed transfer through the real `Executive::apply_extrinsic` pipeline.

**Permanently stranded block-reward remainders.** Running the frozen runtime's own integer emission equations forward for `10,519,200` blocks (four years at the 12 s target):

```
total_reward    = floor((MAX_SUPPLY - supply) / 15_163_560)
treasury_reward = floor(total_reward / 2)
miner_reward    = total_reward - treasury_reward
uncommitted    += miner_reward mod 10_000_000_000
```

yields `3,501,001.704268688139` QUAN of miner yield against `52,302.704268688139` QUAN of uncommitted remainder — approximately **1.49%** of miner block-reward yield omitted from the only commitments a miner can spend, before counting fee credits. The percentage is a property of the emission formula; the absolute figure depends on the starting supply used to seed the projection.

Recovery of already-stranded balances requires a runtime migration or another privileged mechanism. This matches **High - Permanent freezing of unclaimed yield**.

## Why This Is Not a Known Issue or Duplicate

### The runtime guards vesting against this exact hazard, and not mining

The team identified this failure mode and defended the vesting pallet against it:

```rust
/// Vesting payouts are rounded down to multiples of the wormhole leaf quantum
/// (`SCALE_DOWN_FACTOR`): a sub-quantum transfer would be committed as a
/// zero-value leaf, stranding funds paid to keyless beneficiaries.
pub const VestingPayoutQuantum: Balance = pallet_wormhole::SCALE_DOWN_FACTOR;
/// One QUAN keeps every payout above the existential deposit and the Wormhole
/// circuit's fee-consuming minimum.
pub const VestingMinimumPayout: Balance = UNIT;
```

`runtime/src/configs/mod.rs:562-580`. The pallet states the reason on the config item itself — "a payout below one quantum would create a zero-value leaf: funds moved to a keyless beneficiary would be irrecoverable" (`pallets/vesting/src/lib.rs:147-157`) — and enforces it as a hard construction-time assertion:

```rust
assert!((minimum % quantum).is_zero(), "MinimumPayout must be quantum-aligned");
let two_quanta = quantum.checked_add(&quantum).expect(...);
assert!(minimum >= two_quanta, "MinimumPayout must contain at least two quanta");
```

`pallets/vesting/src/lib.rs:295-310`. The **two-quanta** floor is exactly the zero-output condition demonstrated above: one quantum cannot produce a positive output at a nonzero fee rate, so the minimum must contain two.

Mining rewards pay the same kind of keyless beneficiary through the chain's primary issuance path and apply **neither** control — no quantum alignment, no minimum, no assertion.

This establishes the intended producer-side invariant. It does not disclose or accept the `MiningRewards` violation of it; it is evidence that the violation is unintended.

### `EQ-QNT-DOC-F-01` is a different mechanism

The closest published finding concerns QIP-5's false claim that a user can partially withdraw one existing leaf and redeem the remainder later. Its example withdraws 60 from a 100-unit leaf and explains that the remaining 40 is unreachable because the first proof consumed the leaf's nullifier. Its recommendation is to correct the documentation.

That is a **user-chosen** partial spend of a correctly committed leaf, resolved at spend time by nullifier consumption. This report is a **protocol-created** credit that is under-committed at leaf creation: the value never enters any proof, so no choice of output can recover it, and the one-quantum case has no valid positive output at all. Different trigger, different actor, different fix.

`EQ-QNT-WORMHOLE-F-03` is also distinct — it covers decoding the miner preimage as an account instead of deriving its Poseidon address, and that derivation is correct in the scoped commit.

### Distinct from the miner-volume-fee report

A separate report covers two `process_exit_bundle` credit paths in `pallets/wormhole` that create **no leaf at all**. Here `pallets/mining-rewards` does create leaves and does advance `TransferCount`; the defect is that each leaf commits **less than was minted**, and small fee leaves have zero output capacity. Adding the missing `record_transfer` calls to `process_exit_bundle` does not touch this path, and quantum-aligning mining payouts does not fix that one.

### Not a post-audit regression

The payout path and the two-decimal amount representation both predate the March 2026 review, so this is not a claim of new code slipping in after an audit — it is a defect that was reviewable at the time and is absent from the published findings.

## Recommended Fix

Do not mint to a keyless miner an amount that cannot be committed and exited.

{% stepper %}
{% step %}

## Combine fees and rewards

Combine `tx_fees` and `miner_reward` into a single payout before crediting.
{% endstep %}

{% step %}

## Carry pending remainder

Carry the sub-quantum difference in a per-miner pending remainder instead of minting it.
{% endstep %}

{% step %}

## Emit only claimable credits

Emit only a quantum-aligned credit whose leaf has positive output capacity; withhold a payout that would produce a one-quantum leaf at a nonzero fee rate.
{% endstep %}

{% step %}

## Migrate stranded balances

Migrate already-stranded balances into claimable, quantum-aligned leaves.
{% endstep %}
{% endstepper %}

Minimum invariants to enforce before `mint_into`:

```
amount % AMOUNT_SCALE_DOWN_FACTOR == 0
floor(amount / AMOUNT_SCALE_DOWN_FACTOR) * (10000 - fee_bps) / 10000 > 0
```

## References

* `primitives/wormhole/src/lib.rs:97-150` - keyless miner reward-address derivation.
* `runtime/src/configs/mod.rs:138-147` - production MiningRewards wiring.
* `runtime/src/configs/mod.rs:477-480` - transaction fees routed to MiningRewards.
* `pallets/mining-rewards/src/lib.rs:129-199` - reward calculation and separate fee/reward payouts.
* `pallets/mining-rewards/src/lib.rs:220-241` - raw mint followed by proof recording.
* `pallets/mining-rewards/src/lib.rs:309-318` - fee collector.
* `pallets/wormhole/src/lib.rs:1280-1310` - one independent leaf per recorded credit.
* `pallets/zk-tree/src/tree.rs:30-34,134-149` - flooring in the authoritative leaf hash.
* `qp-zk-circuits/wormhole/circuit/src/zk_merkle_proof.rs:486-504` - range and per-leaf fee constraints.
* `pallets/wormhole/src/lib.rs:776-806` - output scaled from quantized proof data only.
* `runtime/src/configs/mod.rs:562-580`, `pallets/vesting/src/lib.rs:147-157,295-310` - vesting's explicit protection against the same hazard.

## Proof of Concept

The PoC is a runtime integration test. It drives the production `Runtime`, `TransactionFeesCollector`, `MiningRewards`, `Wormhole`, `ZkTree`, the real transaction extensions and the ML-DSA-65 signature pipeline — no mock proof recorder and no simulated fee. It shows, in order:

* `MiningRewards::on_finalize` mints a fee credit and a block-reward credit at raw precision and records a separate leaf for each, and each leaf hashes identically to the same leaf with its remainder removed — the raw remainder is not committed;
* a one-quantum fee leaf has `max_output_quanta == 0`, so it can never fund a positive exit;
* an **actual** ML-DSA-65 `Balances::transfer_keep_alive` pushed through `Executive::apply_extrinsic` pays a `10,994,787,000` planck fee that `MiningRewards` records as exactly that raw value, producing a leaf with zero output capacity;
* projecting the frozen runtime's own integer emission equations over four years strands `52,302.704268688139` QUAN of miner block-reward remainders.

### Run

The PoC is a standalone integration test and needs no edits to production files. From `audit-comp-quantus-chain` at frozen commit `3b243b8`:

{% stepper %}
{% step %}

## Copy the PoC

Copy the PoC to `runtime/tests/mining_rewards_quantization.rs`.
{% endstep %}

{% step %}

## Run the test

```bash
cargo test -p quantus-runtime --test mining_rewards_quantization -- --nocapture
```

{% endstep %}
{% endstepper %}

### Observed Output

Tested against frozen chain commit `3b243b8`:

```
running 3 tests
test mining_rewards_mint_raw_dust_that_the_real_zk_leaves_do_not_commit ... ok
actual ML-DSA-65 transfer fee: 10994787000 planck
test an_actual_single_transfer_fee_becomes_a_zero_output_miner_leaf ... ok
four-year uncommitted miner yield: 52302704268688139 planck (52302 whole QUAN)
test four_year_projection_from_live_planck_strands_over_52_thousand_quan ... ok

test result: ok. 3 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out
```

### PoC Code

```rust
use codec::Encode;
use frame_support::traits::{Currency, Hooks};
use qp_dilithium_crypto::Dilithium65Pair;
use quantus_runtime::{
	transaction_extensions::{ReversibleTransactionExtension, WormholeProofRecorderExtension},
	Balances, BalancesCall, Executive, MiningRewards, Runtime, RuntimeCall, RuntimeEvent,
	Signature, SignedPayload, System, TxExtension, UncheckedExtrinsic, Wormhole, ZkTree,
	MAX_SUPPLY, UNIT, VERSION,
};
use sp_consensus_qpow::POW_ENGINE_ID;
use sp_core::{crypto::AccountId32, Pair};
use sp_runtime::{
	generic::Era, traits::IdentifyAccount, BuildStorage, DigestItem, MultiAddress, Permill,
};

const QUANTUM: u128 = pallet_zk_tree::tree::AMOUNT_SCALE_DOWN_FACTOR;

fn test_ext() -> sp_io::TestExternalities {
	let mut storage = frame_system::GenesisConfig::<Runtime>::default()
		.build_storage()
		.expect("system genesis builds");
	pallet_treasury::GenesisConfig::<Runtime> {
		treasury_account: Some(AccountId32::new([9u8; 32])),
		treasury_portion: Some(Permill::from_percent(50)),
	}
	.assimilate_storage(&mut storage)
	.expect("treasury genesis builds");

	let mut ext = sp_io::TestExternalities::new(storage);
	ext.execute_with(|| System::set_block_number(1));
	ext
}

fn signed_transfer(
	pair: &Dilithium65Pair,
	sender: AccountId32,
	dest: AccountId32,
	value: u128,
) -> UncheckedExtrinsic {
	let genesis_hash = System::block_hash(0);
	let call: RuntimeCall =
		BalancesCall::transfer_keep_alive { dest: MultiAddress::Id(dest), value }.into();
	let tx_ext: TxExtension = (
		frame_system::CheckNonZeroSender::<Runtime>::new(),
		frame_system::CheckSpecVersion::<Runtime>::new(),
		frame_system::CheckTxVersion::<Runtime>::new(),
		frame_system::CheckGenesis::<Runtime>::new(),
		frame_system::CheckEra::<Runtime>::from(Era::immortal()),
		frame_system::CheckNonce::<Runtime>::from(0),
		frame_system::CheckWeight::<Runtime>::new(),
		ReversibleTransactionExtension::<Runtime>::new(),
		WormholeProofRecorderExtension::<Runtime>::new(),
		pallet_transaction_payment::ChargeTransactionPayment::<Runtime>::from(0),
		frame_metadata_hash_extension::CheckMetadataHash::<Runtime>::new(false),
		frame_system::WeightReclaim::<Runtime>::new(),
	);
	let payload = SignedPayload::from_raw(
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
	let signature = payload.using_encoded(|bytes| pair.sign(bytes));
	UncheckedExtrinsic::new_signed(
		call,
		MultiAddress::Id(sender),
		Signature::Dilithium65(signature),
		tx_ext,
	)
}

#[test]
fn mining_rewards_mint_raw_dust_that_the_real_zk_leaves_do_not_commit() {
	test_ext().execute_with(|| {
		let preimage = [42u8; 32];
		let miner = AccountId32::new(
			qp_wormhole::derive_wormhole_address(preimage).expect("canonical miner preimage"),
		);
		System::deposit_log(DigestItem::PreRuntime(POW_ENGINE_ID, preimage.to_vec()));

		// A one-transaction block can produce a fee just over one 0.01 QUAN quantum.
		// MiningRewards mints this independently from the block reward.
		let raw_fee = QUANTUM + 1;
		MiningRewards::collect_transaction_fees(raw_fee);
		MiningRewards::on_finalize(1);

		assert_eq!(ZkTree::leaf_count(), 3, "fee, miner reward, and treasury reward");

		let fee_leaf = ZkTree::leaf(0).expect("fee leaf");
		let reward_leaf = ZkTree::leaf(1).expect("miner block-reward leaf");
		assert_eq!(fee_leaf.to, miner);
		assert_eq!(reward_leaf.to, miner);
		assert_eq!(fee_leaf.transfer_count, 0);
		assert_eq!(reward_leaf.transfer_count, 1);
		assert_eq!(fee_leaf.amount, raw_fee);

		// The raw extra planck exists in the miner's keyless account, but does not
		// affect the Merkle commitment: QUANTUM and QUANTUM + 1 hash identically.
		let mut rounded_fee_leaf = fee_leaf.clone();
		rounded_fee_leaf.amount = QUANTUM;
		assert_eq!(
			pallet_zk_tree::tree::hash_leaf::<Runtime>(&fee_leaf),
			pallet_zk_tree::tree::hash_leaf::<Runtime>(&rounded_fee_leaf),
		);

		// At the runtime's 4 bps rate, one input quantum cannot fund one output
		// quantum: 1 * 10_000 > 1 * (10_000 - 4). The whole fee leaf is frozen.
		let input_quanta = fee_leaf.amount / QUANTUM;
		let max_output_quanta = input_quanta * (10_000 - 4) / 10_000;
		assert_eq!(input_quanta, 1);
		assert_eq!(max_output_quanta, 0);

		// The protocol-generated block reward is also not quantum-aligned. Because
		// it is a separate leaf, neither credit can absorb the other's remainder.
		assert_ne!(reward_leaf.amount % QUANTUM, 0);
		let committed_reward = reward_leaf.amount / QUANTUM * QUANTUM;
		let mut rounded_reward_leaf = reward_leaf.clone();
		rounded_reward_leaf.amount = committed_reward;
		assert_eq!(
			pallet_zk_tree::tree::hash_leaf::<Runtime>(&reward_leaf),
			pallet_zk_tree::tree::hash_leaf::<Runtime>(&rounded_reward_leaf),
		);

		assert_eq!(
			Balances::free_balance(&miner),
			fee_leaf.amount + reward_leaf.amount,
			"the raw amounts were minted even though their ZK commitments were rounded down",
		);
		assert!(Balances::total_issuance() <= MAX_SUPPLY);

		// Keep this tied to the production recorder rather than a mock.
		assert_eq!(Wormhole::transfer_count(&miner), 2);
	});
}

#[test]
fn an_actual_single_transfer_fee_becomes_a_zero_output_miner_leaf() {
	test_ext().execute_with(|| {
		let pair = Dilithium65Pair::from_seed_slice(&[7u8; 32]).expect("valid seed");
		let sender = pair.public().into_account();
		let destination = AccountId32::new([8u8; 32]);
		Balances::make_free_balance_be(&sender, 1_000 * UNIT);

		let preimage = [42u8; 32];
		let miner = AccountId32::new(
			qp_wormhole::derive_wormhole_address(preimage).expect("canonical miner preimage"),
		);
		System::deposit_log(DigestItem::PreRuntime(POW_ENGINE_ID, preimage.to_vec()));

		let result = Executive::apply_extrinsic(signed_transfer(&pair, sender, destination, UNIT))
			.expect("the signed extrinsic is valid");
		assert!(result.is_ok(), "the transfer dispatch succeeds");

		let actual_fee = System::events()
			.into_iter()
			.find_map(|record| match record.event {
				RuntimeEvent::TransactionPayment(
					pallet_transaction_payment::Event::TransactionFeePaid { actual_fee, .. },
				) => Some(actual_fee),
				_ => None,
			})
			.expect("transaction fee event");
		assert_eq!(MiningRewards::collected_fees(), actual_fee);
		assert!(actual_fee >= QUANTUM, "fee is at least one leaf quantum");
		assert!(actual_fee < 2 * QUANTUM, "fee is below two leaf quanta");

		MiningRewards::on_finalize(1);
		let fee_leaf = (0..ZkTree::leaf_count())
			.filter_map(ZkTree::leaf)
			.find(|leaf| leaf.to == miner && leaf.amount == actual_fee)
			.expect("mining-rewards recorded the exact runtime fee as a leaf");

		let input_quanta = fee_leaf.amount / QUANTUM;
		let max_output_quanta = input_quanta * (10_000 - 4) / 10_000;
		assert_eq!(input_quanta, 1);
		assert_eq!(max_output_quanta, 0, "the fee leaf cannot produce a positive exit");

		println!("actual ML-DSA-65 transfer fee: {actual_fee} planck");
	});
}

#[test]
fn four_year_projection_from_live_planck_strands_over_52_thousand_quan() {
	const FOUR_YEARS_IN_BLOCKS: u64 = 10_519_200;
	const EMISSION_DIVISOR: u128 = 15_163_560;
	// TotalIssuance at Planck block 946,683 (spec version 144).
	const LIVE_PLANCK_SUPPLY: u128 = 7_003_948_600_961_369_811;

	let mut supply = LIVE_PLANCK_SUPPLY;
	let mut miner_yield = 0u128;
	let mut uncommitted_remainders = 0u128;

	for _ in 0..FOUR_YEARS_IN_BLOCKS {
		let total_reward = (MAX_SUPPLY - supply) / EMISSION_DIVISOR;
		let treasury_reward = total_reward / 2;
		let miner_reward = total_reward - treasury_reward;
		supply += total_reward;
		miner_yield += miner_reward;
		uncommitted_remainders += miner_reward % QUANTUM;
	}

	assert_eq!(miner_yield, 3_501_001_704_268_688_139);
	assert_eq!(uncommitted_remainders, 52_302_704_268_688_139);
	assert!(uncommitted_remainders > 52_000 * UNIT);
	println!(
		"four-year uncommitted miner yield: {} planck ({} whole QUAN)",
		uncommitted_remainders,
		uncommitted_remainders / UNIT,
	);
}
```
