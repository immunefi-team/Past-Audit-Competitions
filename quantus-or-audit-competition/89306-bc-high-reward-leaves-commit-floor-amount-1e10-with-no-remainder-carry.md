# 89306 bc high reward leaves commit floor amount 1e10 with no remainder carry

**Submitted on Aug 18th 2026 at 14:20:53 UTC by @Pyro for** [**Audit Comp | Quantus**](https://immunefi.com/audit-competition/audit-comp-quantus)

* **Report ID:** #89306
* **Report Type:** Blockchain/DLT
* **Report severity:** High
* **Target:** <https://github.com/immunefi-team/audit-comp-quantus-chain>
* **Impacts:**
  * Permanent freezing of unclaimed yield

## Description

`mint_reward` mints the raw reward and hands the same raw amount to the proof recorder:

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

`hash_leaf` commits only whole quanta:

```rust
let amount_quantized =
	u32::try_from(amount_u128 / AMOUNT_SCALE_DOWN_FACTOR).unwrap_or(u32::MAX);
```

`AMOUNT_SCALE_DOWN_FACTOR` is 10,000,000,000 planck. The sub-quantum remainder never enters the Poseidon preimage, so no circuit can witness it, and `process_exit_bundle` reconstructs `summed_output_amount * SCALE_DOWN_FACTOR`. Entitlement stops at whole quanta while the balance is the raw amount.

There is no accumulator. `on_finalize` makes three independent `mint_reward` calls and each opens its own leaf at the next `transfer_count`, with its own floor. The nullifier binds to `(secret, transfer_count)`, so a leaf spends once and a later leaf cannot absorb an earlier remainder.

The runtime already names this hazard and enforces the invariant elsewhere. `VestingPayoutQuantum` is defined as `pallet_wormhole::SCALE_DOWN_FACTOR`, with the comment that “a sub-quantum transfer would be committed as a zero-value leaf, stranding funds paid to keyless beneficiaries”. Pallet-vesting honours it:

```rust
/// Round down to a multiple of the payout quantum.
fn quantize_down(amount: BalanceOf<T>) -> BalanceOf<T> {
	let remainder = amount % T::PayoutQuantum::get();
	amount.checked_sub(&remainder).expect("remainder never exceeds the dividend")
}
```

Vesting pays whole quanta and keeps the remainder in the pot for the next claim. Mining-rewards mints the remainder into an address that can never claim it.

The same chokepoint accepts unaligned credits from the event scanner, so a third-party transfer sized for a wormhole recipient loses its remainder the same way. `record_transfer_proof` rejects only a zero amount. There is no alignment check and no usable minimum.

The shortfall is permanent and recurs on every block. At the 12,000 ms target that is 7,200 block rewards a day, each stranding its own remainder at an address that can never claim it, while total issuance rises by the full raw amount and the unclaimable part consumes the 21,000,000 QUAN supply budget. The shortfall grows as a share of the reward as emission decays.

Example:

{% stepper %}
{% step %}

## Start from the Planck preset

The chain starts from the Planck preset, so genesis issuance is 100,009,001,000,000,000 planck.
{% endstep %}

{% step %}

## Produce the first block

The first block carries a PoW pre-runtime digest and collects no transaction fees.
{% endstep %}

{% step %}

## Calculate the total reward

`on_finalize` computes `total_reward = (2.1e19 - 100_009_001_000_000_000) / 15_163_560 = 1_378_303_709_617` planck.
{% endstep %}

{% step %}

## Split the treasury portion

The 50 percent treasury portion floors to 689,151,854,808 planck, so `miner_reward` is 689,151,854,809 planck.
{% endstep %}

{% step %}

## Mint the raw reward

`mint_into` credits the full 689,151,854,809 planck and total issuance rises by the same amount.
{% endstep %}

{% step %}

## Commit whole quanta

`hash_leaf` commits `689_151_854_809 / 10_000_000_000 = 68` quanta, worth 680,000,000,000 planck.
{% endstep %}

{% step %}

## Observe the stranded entitlement

9,151,854,809 planck of that block's reward is entitlement that was never created, 1.33 percent of the reward.
{% endstep %}

{% step %}

## Produce the next block

The next block opens a new leaf at `transfer_count + 1` with no remainder carried in, and the shortfall repeats for every block author.
{% endstep %}
{% endstepper %}

## Recommendation

Apply the rule pallet-vesting already applies. Pay the miner one credit, quantized down to whole leaf quanta, and roll the remainder into the next block's bucket.

```diff
-		// Send fees to miner if any
-		Self::mint_reward(miner.as_ref(), tx_fees);
-
-		// Send block rewards to miner
-		Self::mint_reward(miner.as_ref(), miner_reward);
+		// One credit, one leaf. Pay whole leaf quanta only and carry the sub-quantum
+		// remainder into the next block, so nothing is minted that no leaf can commit.
+		let miner_total = tx_fees.saturating_add(miner_reward);
+		let quantum = T::LeafQuantum::get();
+		let remainder = miner_total % quantum;
+		Self::retain_unminted(remainder);
+		Self::mint_reward(miner.as_ref(), miner_total.saturating_sub(remainder));
```

`LeafQuantum` is a new pallet constant, wired in the runtime to `pallet_wormhole::SCALE_DOWN_FACTOR`, exactly as `VestingPayoutQuantum` already is.

Add the same guard at the shared chokepoint so no caller can bypass it. In `record_transfer_proof`, reject a native credit to a wormhole-form recipient that is not a multiple of `AMOUNT_SCALE_DOWN_FACTOR`, or that is below the two-quantum minimum the circuit fee relation needs for a nonzero output.

## Proof of Concept

Two parts. The first proves the remainder is absent from the commitment, the second proves it is minted anyway and never carried.

{% stepper %}
{% step %}

## The leaf cannot represent the remainder

Add to `pallets/zk-tree/src/tests.rs`.

```bash
cargo test -p pallet-zk-tree poc_remainder_is_not_committed -- --nocapture
```

```rust
#[test]
fn poc_remainder_is_not_committed() {
	let minted = 692_449_530_321u128;
	let floored = (minted / tree::AMOUNT_SCALE_DOWN_FACTOR) * tree::AMOUNT_SCALE_DOWN_FACTOR;

	println!("minted   : {} planck", minted);
	println!("committed: {} planck", floored);
	println!("stranded : {} planck", minted - floored);

	assert_eq!(
		tree::hash_leaf::<Test>(&amount_leaf(minted)),
		tree::hash_leaf::<Test>(&amount_leaf(floored)),
		"the remainder is not part of the leaf commitment"
	);
}
```

Output:

```
minted   : 692449530321 planck
committed: 690000000000 planck
stranded : 2449530321 planck
```

{% endstep %}

{% step %}

## The remainder is minted and never carried forward

Add to `pallets/mining-rewards/src/tests.rs`.

```bash
cargo test -p pallet-mining-rewards poc_reward_remainder_is_never_carried -- --nocapture
```

```rust
#[test]
fn poc_reward_remainder_is_never_carried() {
	new_test_ext().execute_with(|| {
		const QUANTUM: u128 = 10_000_000_000;
		MockProofRecorder::clear();
		set_miner_preimage_digest(MINER_1.preimage());

		// Two consecutive blocks, same author.
		MiningRewards::on_finalize(1);
		MiningRewards::on_finalize(2);

		let leaves: Vec<u128> = MockProofRecorder::get_recorded_proofs()
			.into_iter()
			.filter(|p| p.to == MINER_1.account_id())
			.map(|p| p.amount)
			.collect();

		let mut stranded = 0u128;
		for (i, minted) in leaves.iter().enumerate() {
			let committed = (minted / QUANTUM) * QUANTUM;
			stranded += minted - committed;
			println!("leaf {}: minted {} committed {} stranded {}", i, minted, committed, minted - committed);
		}
		println!("total stranded over 2 blocks: {} planck", stranded);
		println!("retry bucket: {}", MiningRewards::collected_fees());

		assert_eq!(leaves.len(), 2, "each block opens its own leaf");
		assert!(stranded > 0, "both leaves strand their remainder");
		assert_eq!(
			MiningRewards::collected_fees(),
			0,
			"no remainder is carried into the next block"
		);
	});
}
```

Output:

```
leaf 0: minted 692449530321 committed 690000000000 stranded 2449530321
leaf 1: minted 692449484656 committed 690000000000 stranded 2449484656
total stranded over 2 blocks: 4899014977 planck
retry bucket: 0
```

{% endstep %}
{% endstepper %}
