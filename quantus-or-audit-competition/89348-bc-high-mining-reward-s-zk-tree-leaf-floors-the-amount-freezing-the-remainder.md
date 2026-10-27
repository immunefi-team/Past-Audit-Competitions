# 89348 bc high mining reward s zk tree leaf floors the amount freezing the remainder

## #89348 \[BC-High] Mining reward's zk-tree leaf floors the amount, freezing the remainder

**Submitted on Aug 18th 2026 at 18:19:05 UTC by @AlexSR for** [**Audit Comp | Quantus**](https://immunefi.com/audit-competition/audit-comp-quantus)

* **Report ID:** #89348
* **Report Type:** Blockchain/DLT
* **Report severity:** High
* **Target:** <https://github.com/immunefi-team/audit-comp-quantus-chain>
* **Impacts:**
  * Permanent freezing of unclaimed yield

### Description

## Mining rewards are minted at full precision but their zk-tree leaf commits only the floored amount, so the remainder is permanently unclaimable

### Summary

`pallet-mining-rewards` mints the miner's reward at full `u128` precision and records that same exact amount as a wormhole transfer proof (`pallets/mining-rewards/src/lib.rs:220-237`). The proof's authoritative commitment is the zk-tree leaf hash, and that hash commits `floor(amount / 10^10)`, not `amount` (`pallets/zk-tree/src/tree.rs:146-149`). Exits reconstruct value the other way, as `summed_output_amount * 10^10` (`pallets/wormhole/src/lib.rs:792-793`).

So for a reward `amount = q*S + r` with `S = 10^10` and `0 < r < S`, the miner is credited `q*S + r` at an address whose only claim instrument is a withdrawal proof, and the largest proof that address can ever produce for that credit is worth at most `q*S` (the circuit's volume-fee relation puts the practical ceiling below that, `pallets/wormhole/src/lib.rs:1066-1076`). The `r` planck is earned yield the miner can never claim by any permissionless action.

The recipient is keyless by construction. `on_finalize` derives the author address from the 32-byte QPoW pre-runtime preimage as `poseidon(preimage)` (`mining-rewards/src/lib.rs:205-207`, `primitives/wormhole/src/lib.rs:97-110,122,146-147`); that preimage is itself the inner digest `poseidon(salt || secret)`. A signable account id is a different derivation over a different input, `poseidon(dilithium_public_key)` (`primitives/dilithium-crypto/src/scheme_macro.rs:238-245`). A miner cannot mine to an account it can sign for, so the withdrawal proof is the only exit.

The project already defends the same hazard on its other keyless-recipient payout path, and says why in its own words at three separate sites. `pallet-vesting` rounds every payout down to the leaf quantum because "a payout below one quantum would create a zero-value leaf: funds moved to a keyless beneficiary would be irrecoverable" (`pallets/vesting/src/lib.rs:147-152`). The same reasoning routes early-termination dust to the treasury rather than the beneficiary (`:401-407`, `:422-433`) and backs a compile-time assert that no payout may exceed the leaf's `u32` ceiling (`runtime/src/configs/mod.rs:584-591`).

Mining quantizes nothing, and its recipient is keyless.

Verified on the frozen in-scope tree (commit `3b243b87`); both defect files are fork-authored Quantus code.

### Root cause

`mint_reward` credits first and records second, with no alignment step between the two. The call arguments are folded onto one line here for width; the source spreads them across `:232-237`:

```rust
match T::Currency::mint_into(miner, reward) {      // @audit credits full u128 precision
    Ok(_) => {
        T::ProofRecorder::record_transfer_proof(   // @audit records the same unaligned amount
            None, mint_account, miner.clone(), reward,   // None = native token
        );
```

(`mining-rewards/src/lib.rs:230-237`.)

Nothing downstream aligns it. `Wormhole::record_transfer_proof` rejects only an exact zero and passes every other native amount straight through (`pallets/wormhole/src/lib.rs:1354-1368`), and `record_transfer` stores the raw planck value in the leaf struct before hashing (`pallets/wormhole/src/lib.rs:1296-1310`, `pallets/zk-tree/src/lib.rs:259,283-288`).

The loss happens in the hash, which divides by `AMOUNT_SCALE_DOWN_FACTOR = 10^10` and commits the `u32` quotient (`pallets/zk-tree/src/tree.rs:146-149`; the factor is at `:34`). The project's own doc states the split: "The amount stored on-chain is the raw planck value. When hashing for the ZK circuit, amounts are quantized by dividing by 10^10 \[...]" (`docs/zk-trie-architecture.md:422`; the sentence continues "to fit in a single field element with 2 decimal places of precision").

The 2-decimal wire format is not the bug. The bug is that an origination path mints an amount the wire format cannot represent, to a recipient whose only spend instrument is that format.

The reward is not aligned by any explicit step. `total_reward = remaining_supply / EmissionDivisor` with `EmissionDivisor = 15_163_560` (`mining-rewards/src/lib.rs:155-157`, `runtime/src/configs/mod.rs:144`), then a `Permill::mul_floor` split (`mining-rewards/src/lib.rs:161-162`). Neither step produces a multiple of `10^10` except by coincidence.

### Why the remainder is permanently frozen

**The leaf cannot represent it.** `hash_leaf(amount)` and `hash_leaf(floor(amount/S)*S)` are the same 32 bytes, so the commitment is blind to `r`. A withdrawal proof is checked against that commitment and the exit converts the proven output back with `saturating_mul(SCALE_DOWN_FACTOR)` (`pallets/wormhole/src/lib.rs:792-793`), so no proof against that leaf can pay out more than `q*S`. The attached `leaf_commitment_remainder_poc.rs` shows this directly.

**The remainders never combine.** Each credit gets its own leaf, keyed `(recipient, transfer_count)` (`pallets/wormhole/src/lib.rs:1296-1310`), and each is floored independently. `on_finalize` makes up to two separate miner credits per block, one for collected fees and one for the block reward (`mining-rewards/src/lib.rs:193`, `:196`; the fee call is skipped when the block collected none, since `mint_reward` returns on a zero amount at `:221-223`), so a block can strand two remainders. Thousands of them never sum to a claimable quantum: no leaf holds the sum.

**The balance is not a fallback.** The address is keyless, so no signed `Balances` transfer exists, and an exit does not spend it either: the exit issues the claim to the exit account with `increase_balance` and never debits the address the leaf was recorded against (`pallets/wormhole/src/lib.rs:828-833`). The leaf is the spend instrument; the balance is its locked counterpart. `pallet-sudo` was removed from this runtime (`runtime/src/lib.rs:232`), so recovery needs a privileged `pallet-balances` call through Root governance, such as `force_transfer` or `force_set_balance`, or a `set_code` migration. None of those is a permissionless action available to the miner, and the runtime keeps no ledger separating residue from the counterpart of unexited leaves.

### Attack path

No attacker and no extrinsic. It is the ordinary reward path: any block whose miner credit is not an exact multiple of `10^10` strands the difference, and nothing in the emission arithmetic aims for that multiple.

{% stepper %}
{% step %}

### Compute the miner reward and derive the block author

`on_finalize` computes `miner_reward` from remaining supply and the treasury split, unaligned (`mining-rewards/src/lib.rs:155-157,161-162`), and derives the block author from the QPoW digest as a keyless wormhole address (`:165`, `:205-207`).
{% endstep %}

{% step %}

### Mint and record the full reward

`mint_reward` mints `miner_reward` exactly, then records a proof for `miner_reward` exactly (`:230-237`).
{% endstep %}

{% step %}

### Quantize the recorded amount in the leaf

`record_transfer_proof` accepts it and `insert_leaf` hashes it, committing `floor(miner_reward / 10^10)`.
{% endstep %}

{% step %}

### Strand the remainder

`miner_reward mod 10^10` is now balance at a keyless address with no claim behind it.
{% endstep %}

{% step %}

### Repeat on subsequent blocks

Repeat next block, and again for the fee credit.
{% endstep %}
{% endstepper %}

### Impact

Permanent freezing of a slice of the miner's unclaimed yield, on every credit that is not quantum-aligned.

Per credit the frozen amount is bounded by one quantum, `r < 10^10` planck `= 0.01 QUAN` (`zk-tree/src/tree.rs:34`, `runtime/src/lib.rs:97`). The rate is what matters. At `TARGET_BLOCK_TIME_MS = 12_000` (`runtime/src/lib.rs:87`) the chain produces 2,628,000 blocks a year and each block strands up to two remainders, so the block-reward arm alone loses on the order of `10^4` QUAN a year, ceiling 26,280 QUAN.

The emission budget is charged for it: `total_reward` is derived from `T::Currency::total_issuance()` (`mining-rewards/src/lib.rs:143`), which counts the stranded planck.

As a fraction of the miner's own yield it is a few tenths of a percent per credit at launch-era supply. The real `on_finalize`, run in the pallet mock at the shipped existential deposit with the block's collected fees added back, credits a 692,449,528,904 planck reward and strands 2,449,528,904 of it, 0.354%. That fraction grows over the life of the chain: the reward decays with remaining supply while `r` stays bounded by a fixed `10^10`. Individual blocks vary, since `r = miner_reward % 10^10` takes any value in that range, zero included. This report is scoped to the above-minimum stretch of the emission schedule, where the reward sits well above the quantized minimum and only its tail is lost. The chain operates in that stretch today, and will for a long time to come.

The direction is a freeze, not theft or inflation. No user principal is affected and no state is corrupted.

### Severity self-assessment

The governing rung, quoted verbatim from the Blockchain/DLT ladder:

> Permanent freezing of unclaimed yield

Any miner credit that is not an exact multiple of `10^10` strands its remainder at an address with no key and no second instrument to claim it with, and nothing on the mining path aims for that multiple. The quantity is unclaimed mining yield and the loss is permanent, so the finding lands on **High**.

#### Why known issue 3 does not reach this

The clause mentions:

> At some point the volume of transactions will cause the zk-tree depth to exceed the current maximum, both on chain and in circuit. When that moment approaches we will do a runtime upgrade to update both the circuit and the chain. A similar reasoning works for when / if mining rewards ever drop well below the quantized minimum for the circuit.

Its subject is the reward and its predicate is that the reward drops well below the quantized minimum. That is false of every credit in this report. The block reward at launch-era supply commits 69 whole quanta, and the finding is that a reward far above the minimum still loses its tail.

The two cases also need different changes, and that is what separates them. The clause promises "a runtime upgrade to update both the circuit and the chain". An increase in precision is sufficient for the disclosed case: once the quantum sits below the reward, `floor(reward / S)` stops being zero and the credit becomes representable again. It is not sufficient here. This loss is `reward % S`, and no smaller `S` guarantees that is zero, because nothing on the mining path aligns the credit to `S`. Only `S = 1` makes it identically zero, and that is not a retune of the constant: the leaf commits the quantized amount as a `u32` (`pallets/zk-tree/src/tree.rs:138-148`) under a compile-time assert that `MAX_SUPPLY < u32::MAX * AMOUNT_SCALE_DOWN_FACTOR` (`runtime/src/configs/mod.rs:588-591`), so dropping quantization altogether means widening the circuit's amount field. Aligning the credit before it is minted removes the loss at any `S` and touches no circuit at all.

### Proof of Concept

Five drop-in unit tests across the two target pallets, each extending that pallet's own test suite and, where it touches runtime state, driving that pallet's own mock. `mining_reward_remainder_poc.rs` appends to `pallets/mining-rewards/src/tests.rs`, extending `miner_reward_works` (`tests.rs:8`) and `miner_reward_with_transaction_fees_works` (`tests.rs:40`) and reusing their mock and helpers. `leaf_commitment_remainder_poc.rs` appends to `pallets/zk-tree/src/tests.rs` and drives the pallet's own `insert_leaf`, the same function the wormhole's `record_transfer` calls (`zk-tree/src/lib.rs:352-359`) and the same one that suite's `insert_first_leaf_works` (`tests.rs:182`) drives. Every symbol used already exists in those two test builds; nothing is authored that either suite already provides.

The mining-rewards tests raise that mock's existential deposit to the shipped `MILLI_UNIT = 1e9` (`runtime/src/lib.rs:98,102`) rather than its default of 1, so the issuance the reward is computed from is the shipped magnitude. The zk-tree mock has no balances pallet and derives the reward arithmetically from the same shipped constants. The two constants the finding turns on need no adjustment: the mock's `MaxSupply = 21_000_000 * UNIT` and `EmissionDivisor = 15_163_560` (`mining-rewards/src/mock.rs:38-39`) are identical to the runtime's (`runtime/src/lib.rs:105`, `configs/mod.rs:143-144`). Full parity table in `RUN.md`.

* `poc_block_reward_credit_exceeds_what_its_leaf_can_ever_claim` runs the real `on_finalize` (`lib.rs:129`) and asserts the claim **across the whole collection of the miner's recorded proofs**: every credited planck was recorded, yet `sum(floor(a/S) * S)` is strictly less than the total credited, and **the entire shortfall equals `block_reward % 10^10`**. An aligned fee credit of exactly four quanta is the control: it rides along in the same call and loses nothing.
* `poc_fee_credit_can_strand_one_planck_short_of_a_full_quantum` pins the per-credit maximum: `3*10^10 + 9_999_999_999` planck commits 3 quanta and strands 9,999,999,999.
* `poc_tree_root_cannot_distinguish_the_sub_quantum_remainder` works at the level a withdrawal is checked. Recording the full credit and recording only its floored part leave the **real** tree with a byte-identical root. The control is the next assertion: one more whole quantum *does* move it.
* `poc_stored_leaf_keeps_the_remainder_the_commitment_drops` pins the divergence inside one state: `Leaves` holds the full amount while the hash over that same leaf equals the floored amount's.
* `poc_remainders_never_combine_into_a_claimable_quantum` shows two credits stranding just under a quantum each commit 1 + 1 quanta, not the 3 the same total would commit as one credit.

A disclosed stub: the mining-rewards mock wires `type ProofRecorder = MockProofRecorder` (`mock.rs:129,172`), a recording stub rather than the live wormhole recorder. The first two tests therefore establish what the miner is credited and what amount reaches the recorder; the zk-tree tests run the real tree and establish what a recorded amount commits to. Neither file claims the other's half.

```bash
git checkout 3b243b870a5442a6e5f443056f67bbba87d472f5
cat mining_reward_remainder_poc.rs   >> pallets/mining-rewards/src/tests.rs
cat leaf_commitment_remainder_poc.rs >> pallets/zk-tree/src/tests.rs
cargo test -p pallet-mining-rewards --lib poc_mining_reward_remainder -- --nocapture
cargo test -p pallet-zk-tree        --lib poc_leaf_commitment_remainder -- --nocapture
```

### Suggested fix

Round the miner's credit down to the leaf quantum before minting and carry the remainder forward, the shape `pallet-vesting` ships for its keyless beneficiaries (`quantize_down`, `vesting/src/lib.rs:562-567`). At the top of `mint_reward`, take `remainder = reward % T::LeafQuantum::get()` only when the recipient is a miner and the quantum is non-zero, subtract it from the credit before `mint_into`, and add it to `CollectedFees`. The treasury payout is left exact, following the project's own reasoning that dust is safe on a signature-controlled account and only stranded on a keyless one (`vesting/src/lib.rs:404-406`). `CollectedFees` is the right carry bucket: the pallet already treats its contents as "already-scheduled supply" that the remaining-supply calculation adds back (`mining-rewards/src/lib.rs:134-141`), so the emission schedule is unchanged.

`quantize_mining_reward_before_mint.patch` (attached) is this change plus a runtime binding of `LeafQuantum` straight to `pallet_zk_tree::tree::AMOUNT_SCALE_DOWN_FACTOR`, following the pattern `VestingPayoutQuantum` sets, though vesting anchors on `pallet_wormhole::SCALE_DOWN_FACTOR` (`:563-567`) which the runtime asserts equal to the zk-tree factor (`:574-581`). It is forkless, and already-stranded residue is reconstructible from the `MinerRewarded { miner, reward }` history (`mining-rewards/src/lib.rs:238-241`).

A guard at the recorder instead would be worse than nothing: the mint has already happened by then, so rejecting the credit would suppress the whole leaf and strand the entire reward, not just its tail.

### mining\_reward\_remainder.rs

```rust
// Proof of concept, part 1 of 2: the miner credit path.
//
// `pallet-mining-rewards` mints the miner's reward at full u128 precision and records
// that exact amount as a wormhole transfer proof. The proof's leaf commits
// `floor(amount / AMOUNT_SCALE_DOWN_FACTOR)`, so `amount % 10^10` is not in any claim
// the (keyless) miner address can make.
//
// Drop-in unit tests for the mining-rewards pallet. Append to
// `pallets/mining-rewards/src/tests.rs`. These drop-in tests extend that suite's own
// `miner_reward_works` (tests.rs:8) and `miner_reward_with_transaction_fees_works`
// (tests.rs:40), reusing the same mock runtime and the same helpers the two parent tests
// use: `new_test_ext`, `MINER_1`, `set_miner_preimage_digest`, `MaxSupply`,
// `EmissionDivisor` and `Treasury::portion()`. These drop-in tests also use
// `MockProofRecorder`, the recording stub the pallet's own mock provides (mock.rs:129-151);
// the two parent tests never reference it. Nothing here is authored that the suite
// already provides.
//
// The entry point is the pallet's real `on_finalize` hook (lib.rs:129), the same hook
// the runtime's Executive calls on every block. There is no extrinsic above it.
//
// Both tests run at the SHIPPED existential deposit (`MILLI_UNIT = 1e9`,
// `runtime/src/lib.rs:98,102`) rather than the mock's default of 1, so the genesis
// issuance the reward is computed from is the shipped magnitude. The mock declares
// `ExistentialDeposit` as `pub static` for this (`mock.rs:41-42`). The two constants the
// finding actually turns on need no adjustment: the mock's `MaxSupply = 21_000_000 * UNIT`
// and `EmissionDivisor = 15_163_560` (mock.rs:38-39) are identical to the runtime's
// (`runtime/src/lib.rs:105`, `runtime/src/configs/mod.rs:143-144`).
//
// Disclosed stub: this pallet's mock wires `type ProofRecorder = MockProofRecorder`
// (mock.rs:129,172), a recording stub rather than the live wormhole recorder. These
// tests therefore prove what the miner is CREDITED and what amount REACHES the
// recorder. Part 2 runs the real zk tree and proves what a recorded amount commits to.
//
// Part 2 (`leaf_commitment_remainder_poc.rs`) closes the loop on the commitment side.
//
// No node, network, or live state is involved; these are in-process unit tests.

#[cfg(test)]
mod poc_mining_reward_remainder {
	use crate::mock::*;
	use frame_support::traits::{Currency, Hooks};
	use pallet_treasury::TreasuryProvider;
	use sp_core::crypto::AccountId32;

	/// The zk-tree leaf amount quantum. `hash_leaf` commits `amount / S` as a u32, and
	/// the wormhole exit reconstructs `summed_output_amount * S`, so a credit's
	/// claimable value is `(amount / S) * S`.
	const S: Balance = pallet_zk_tree::tree::AMOUNT_SCALE_DOWN_FACTOR;

	/// The shipped runtime's existential deposit, `MILLI_UNIT` (`runtime/src/lib.rs:98,102`).
	const RUNTIME_EXISTENTIAL_DEPOSIT: Balance = 1_000_000_000;

	fn use_runtime_existential_deposit() {
		ExistentialDeposit::set(RUNTIME_EXISTENTIAL_DEPOSIT);
	}

	/// Every native transfer proof recorded for `who`, in recording order.
	///
	/// The whole collection, not one lookup: the claim is that the miner's leaves
	/// cannot together claim what the miner was credited, so a single proof is not
	/// enough to assert it against.
	fn native_proof_amounts_to(who: &AccountId32) -> Vec<Balance> {
		MockProofRecorder::get_recorded_proofs()
			.into_iter()
			.filter(|p| &p.to == who && p.asset_id.is_none())
			.map(|p| p.amount)
			.collect()
	}

	/// What a recorded credit of `amount` can ever be withdrawn for: the leaf commits
	/// `floor(amount / S)` quanta and the exit reconstructs `quanta * S`.
	fn claimable(amount: Balance) -> Balance {
		(amount / S) * S
	}

	/// The involuntary arm: the block reward, with an aligned fee credit beside it as
	/// the control.
	///
	/// `on_finalize` credits the collected fees and the block reward to the same miner
	/// through the same `mint_reward`, in the same block. Choosing the fee amount to be
	/// a whole number of quanta makes it the control: same path, same recorder, nothing
	/// stranded. The block reward is `remaining_supply / EmissionDivisor` split by a
	/// Permill `mul_floor`, and neither step aligns to `S`.
	#[test]
	fn poc_block_reward_credit_exceeds_what_its_leaf_can_ever_claim() {
		use_runtime_existential_deposit();
		new_test_ext().execute_with(|| {
			MockProofRecorder::clear();

			// The control credit: a whole number of quanta, through the same path.
			let aligned_fees: Balance = 4 * S;
			MiningRewards::collect_transaction_fees(aligned_fees);

			// Seed the block author via the QPoW pre-runtime digest. The derived address
			// is poseidon(preimage) over the miner-supplied preimage (itself the inner
			// digest poseidon(salt || secret)), i.e. a keyless wormhole address: its only
			// spend instrument is a withdrawal proof against a leaf.
			set_miner_preimage_digest(MINER_1.preimage());
			let miner = MINER_1.account_id();
			let balance_before = Balances::free_balance(miner.clone());

			// Reward arithmetic, computed exactly as `on_finalize` computes it: the
			// collected fees are added back to the issuance before the division
			// (lib.rs:132-162), because they were burned when they were charged.
			let current_supply = Balances::total_issuance() + aligned_fees;
			let total_reward = (MaxSupply::get() - current_supply) / EmissionDivisor::get();
			let treasury_reward = Treasury::portion().mul_floor(total_reward);
			let block_reward = total_reward - treasury_reward;

			MiningRewards::on_finalize(1);

			// (1) Both credits land at full u128 precision.
			let credited = Balances::free_balance(miner.clone()) - balance_before;
			assert_eq!(
				credited,
				aligned_fees + block_reward,
				"the miner is credited the fees and the block reward at full precision",
			);

			// (2) @audit THE CLAIM, over the whole collection of the miner's proofs.
			// Every planck credited was recorded, and yet the total those records can
			// ever be withdrawn for is strictly less than the total credited. This is
			// the freeze: it is asserted across ALL of the miner's leaves, so it cannot
			// be explained away by a second leaf somewhere carrying the difference.
			let recorded = native_proof_amounts_to(&miner);
			assert_eq!(recorded.len(), 2, "one proof for the fee credit, one for the block reward");
			let recorded_total: Balance = recorded.iter().copied().sum();
			let claimable_total: Balance = recorded.iter().copied().map(claimable).sum();
			assert_eq!(recorded_total, credited, "every planck credited was recorded");
			assert!(
				claimable_total < credited,
				"the miner's leaves cannot together claim what the miner was credited",
			);

			// (3) The control, in the same block and the same call: the aligned credit
			// keeps all of its value, the unaligned one loses its tail, and the entire
			// shortfall is that tail. A test that only showed a shortfall could be a
			// property of the path; this shows it is a property of the ALIGNMENT.
			assert!(recorded.contains(&aligned_fees), "the fee credit was recorded as itself");
			assert_eq!(aligned_fees % S, 0, "the control credit is a whole number of quanta");
			assert_eq!(claimable(aligned_fees), aligned_fees, "so its leaf claims all of it");
			assert!(block_reward % S > 0, "the block reward is not a whole number of quanta");
			assert_eq!(
				credited - claimable_total,
				block_reward % S,
				"the entire shortfall is the block reward's sub-quantum remainder",
			);

			// (4) This is the ABOVE-minimum tier, not the sub-quantum tier described by
			// the programme's known issue: the credit commits whole quanta and still
			// loses a tail.
			assert!(
				block_reward / S > 0,
				"this report is about rewards above the quantized minimum, not below it",
			);
			assert!(block_reward % S < S, "a single credit strands strictly less than one quantum");

			println!(
				"credited = {credited} planck (fees {aligned_fees} + block reward {block_reward}); \
				 claimable = {claimable_total} planck; stranded = {} planck; block reward commits \
				 {} quanta",
				credited - claimable_total,
				block_reward / S,
			);
		});
	}

	/// The deterministic worst case, on a block's fee credit.
	///
	/// `on_finalize` credits collected transaction fees through the same `mint_reward`,
	/// so a block can strand two remainders. Choosing the fee amount pins one of them at
	/// its maximum, one planck short of a whole quantum.
	#[test]
	fn poc_fee_credit_can_strand_one_planck_short_of_a_full_quantum() {
		use_runtime_existential_deposit();
		new_test_ext().execute_with(|| {
			MockProofRecorder::clear();

			// Three whole quanta plus the largest representable remainder.
			let fees: Balance = 3 * S + (S - 1);
			MiningRewards::collect_transaction_fees(fees);

			set_miner_preimage_digest(MINER_1.preimage());
			let miner = MINER_1.account_id();

			MiningRewards::on_finalize(1);

			let recorded = native_proof_amounts_to(&miner);
			let fee_proof = recorded
				.iter()
				.copied()
				.find(|a| *a == fees)
				.expect("the fee credit must be recorded at full precision");

			assert_eq!(fee_proof / S, 3, "three claimable quanta");
			assert_eq!(
				fee_proof - claimable(fee_proof),
				S - 1,
				"9,999,999,999 planck stranded on this one credit",
			);

			println!("fee credit = {fee_proof} planck; stranded = {} planck", fee_proof % S);
		});
	}
}
```

### leaf\_commitment\_remainder\_poc.rs

```rust
// Proof of concept, part 2 of 2: the commitment side.
//
// Part 1 shows the miner is credited an unaligned, full-precision amount and that the
// same unaligned amount reaches the recorder. This part shows what happens when such an
// amount is recorded into the REAL tree: the authoritative Merkle root the withdrawal
// circuit verifies against is byte-identical to the root for the floored amount, so the
// remainder is not in the only thing a proof can be checked against.
//
// Drop-in unit tests for the zk-tree pallet. Append to `pallets/zk-tree/src/tests.rs`.
// They reuse that suite's `Test` mock, its `new_test_ext`, `make_account` and
// `amount_leaf` helpers, and the pallet's own `insert_leaf`, the same entry point the
// wormhole's `record_transfer` calls (`zk-tree/src/lib.rs:352-359`) and the same one the
// suite's own `insert_first_leaf_works` (tests.rs:182) and `insert_multiple_leaves_works`
// (tests.rs:203) drive. Nothing is stubbed: this is the real 4-ary Poseidon tree, the
// real leaf hashing and the real root update.
//
// In the first two tests the amount is DERIVED from the shipped emission constants
// rather than typed in, so those tests are about the actual block reward and not about a
// chosen literal. The third test is a pure integer identity about how two separate
// credits floor independently, and uses constructed worst-case values for that.

#[cfg(test)]
mod poc_leaf_commitment_remainder {
	use super::*;

	/// The leaf amount quantum (`tree.rs:34`).
	const S: u128 = tree::AMOUNT_SCALE_DOWN_FACTOR;

	/// The shipped runtime's emission constants: `MAX_SUPPLY = 21_000_000 * UNIT` with
	/// `UNIT = 1e12` (`runtime/src/lib.rs:97,105`) and `EmissionDivisor = 15_163_560`
	/// (`runtime/src/configs/mod.rs:143-144`). The mining-rewards mock carries the same
	/// two values (`pallets/mining-rewards/src/mock.rs:38-39`), which is why part 1 and
	/// part 2 compute the same reward formula. They print slightly different planck
	/// figures because they run at different issuances: part 1 inside that mock with the
	/// block's collected fees added back, part 2 from three existential deposits here.
	/// Neither figure is asserted by either part; both parts assert only relationships.
	const MAX_SUPPLY: u128 = 21_000_000 * 1_000_000_000_000;
	const EMISSION_DIVISOR: u128 = 15_163_560;

	/// The shipped existential deposit (`runtime/src/lib.rs:98,102`), used only to give
	/// the derivation a realistic starting issuance.
	const EXISTENTIAL_DEPOSIT: u128 = 1_000_000_000;

	/// The miner's share of a block reward at a given issuance, computed the way
	/// `on_finalize` computes it: divide the remaining supply, then keep what the
	/// treasury Permill floors away (`mining-rewards/src/lib.rs:155-162`). That share is
	/// runtime storage, not a constant (`pallets/treasury/src/lib.rs:81`), and the
	/// genesis presets set it to 50% (`runtime/src/genesis_config_presets.rs:387`), which
	/// is also the mining-rewards mock's value (`mining-rewards/src/mock.rs:109-110`).
	fn miner_block_reward(issuance: u128) -> u128 {
		let total = (MAX_SUPPLY - issuance) / EMISSION_DIVISOR;
		total - total / 2
	}

	/// Record one credit into a fresh REAL tree and return the authoritative root.
	///
	/// The recipient, transfer count and asset match `amount_leaf`, so the only thing
	/// varying between calls is the amount.
	fn root_after_recording(amount: u128) -> Hash256 {
		let mut ext = new_test_ext();
		ext.execute_with(|| {
			ZkTree::insert_leaf(make_account(0x11), 7, 5u32, amount);
			ZkTree::root()
		})
	}

	/// The root the withdrawal circuit checks against cannot tell a credit apart from
	/// that credit with its sub-quantum remainder removed.
	#[test]
	fn poc_tree_root_cannot_distinguish_the_sub_quantum_remainder() {
		// The first block's miner reward on a chain whose issuance is three existential
		// deposits, derived rather than typed. Every assertion below is a property of
		// the derivation; the exact planck figures are printed, not asserted.
		let credited = miner_block_reward(3 * EXISTENTIAL_DEPOSIT);
		let quanta = credited / S;
		let claimable = quanta * S;
		let remainder = credited - claimable;

		assert!(
			quanta > 0,
			"the credit is above the quantized minimum, not the sub-quantum tier known issue 3 describes",
		);
		assert!(remainder > 0, "the credit is not a whole number of quanta");

		// @audit THE CLAIM, at the level a withdrawal is actually checked. Recording the
		// full credit and recording only its floored part leave the tree in states with
		// the same authoritative root, so no Merkle proof can distinguish them and no
		// proof can claim the remainder.
		assert_eq!(
			root_after_recording(credited),
			root_after_recording(claimable),
			"the root commits floor(amount / S); the remainder is not in it",
		);

		// The control. One more WHOLE quantum does move the root, so the root is not
		// insensitive to the amount in general; only the sub-quantum tail vanishes.
		assert_ne!(
			root_after_recording(claimable + S),
			root_after_recording(claimable),
			"a whole extra quantum does change the root",
		);

		// The worst case a single credit can strand, one planck short of a quantum, is
		// invisible to the root as well.
		assert_eq!(
			root_after_recording(claimable + S - 1),
			root_after_recording(claimable),
			"one planck short of a quantum is uncommitted",
		);

		println!(
			"derived miner block reward = {credited} planck; commits {quanta} quanta = \
			 {claimable} planck; stranded = {remainder} planck",
		);
	}

	/// The divergence inside a single state: the chain stores the full amount while the
	/// commitment over it carries only the floored value.
	#[test]
	fn poc_stored_leaf_keeps_the_remainder_the_commitment_drops() {
		let credited = miner_block_reward(3 * EXISTENTIAL_DEPOSIT);
		let claimable = (credited / S) * S;
		assert!(credited > claimable, "the derived reward carries a remainder");

		new_test_ext().execute_with(|| {
			ZkTree::insert_leaf(make_account(0x11), 7, 5u32, credited);

			// The stored leaf keeps every planck.
			let stored = ZkTree::leaf(0).expect("the leaf was inserted");
			assert_eq!(stored.amount, credited, "storage holds the full credited amount");

			// The commitment over that same stored leaf is the commitment for the
			// floored amount. Storage says one number, the only checkable artefact says
			// another, and the difference is unspendable.
			assert_eq!(
				tree::hash_leaf::<Test>(&stored),
				tree::hash_leaf::<Test>(&amount_leaf(claimable)),
				"the leaf hash of the full credit equals the leaf hash of its floored part",
			);
			assert_ne!(
				tree::hash_leaf::<Test>(&stored),
				tree::hash_leaf::<Test>(&amount_leaf(claimable + S)),
				"but a whole extra quantum is committed",
			);
		});
	}

	/// Remainders never accumulate into a claimable quantum.
	///
	/// Each credit gets its own leaf and each leaf is floored independently, so two
	/// credits that each strand just under a quantum commit 1 + 1 quanta, not the 3 the
	/// same total would commit if it arrived as one credit.
	#[test]
	fn poc_remainders_never_combine_into_a_claimable_quantum() {
		let a = S + (S - 1);
		let b = S + (S - 1);
		assert_eq!(a / S + b / S, 2, "two credits commit one quantum each");
		assert_eq!((a + b) / S, 3, "the same total as one credit would commit three");
	}
}
```
