# 89307 bc medium sub existential deposit block fees are minted before the account endowing reward

**Submitted on Aug 18th 2026 at 14:22:54 UTC by @Pyro for** [**Audit Comp | Quantus**](https://immunefi.com/audit-competition/audit-comp-quantus)

* **Report ID:** #89307
* **Report Type:** Blockchain/DLT
* **Report severity:** Medium
* **Target:** <https://github.com/immunefi-team/audit-comp-quantus-chain>
* **Impacts:**
  * Griefing (e.g. no profit motive for an attacker, but damage to the users or the protocol)

## Description

`on_finalize` mints collected fees to the miner before the base reward:

```rust
// Send fees to miner if any
Self::mint_reward(miner.as_ref(), tx_fees);

// Send block rewards to miner
Self::mint_reward(miner.as_ref(), miner_reward);
```

`mint_into` uses `Precision::Exact`, so a credit leaving a nonexistent account below the 1,000,000,000 planck existential deposit fails with `BelowMinimum`. The error arm redirects the fee to the treasury and records a wormhole transfer proof crediting the treasury.

The base reward on the next line then creates the same account, but the fee is already gone. This contradicts the runtime's stated rule that 100 percent of transaction fees go to the block miner.

The diversion is permanent. `CollectedFees` is cleared by the `take()` at the top of the hook, the redirect succeeds, and no retry path ever returns the amount to the miner it was owed to.

An attacker can trigger it on demand. A post-dispatch `Pays::No` extrinsic collapses the charged fee to the tip alone, so `pallet_ranked_collective::cleanup_poll` with a sub-existential-deposit tip sets the block's fee total below the threshold at will. Doing that on the block a new miner first wins denies that miner its fee revenue, at a cost to the attacker of the tip. Every miner joining the network is a fresh target, since each new `rewards_inner_hash` derives an author address with no prior balance.

## Recommendation

Aggregate both miner-directed amounts into one credit, checked once against the existential deposit. The single-credit form recommended in H-02 already does this and fixes both defects together.

```diff
-		// Send fees to miner if any
-		Self::mint_reward(miner.as_ref(), tx_fees);
-
-		// Send block rewards to miner
-		Self::mint_reward(miner.as_ref(), miner_reward);
+		// Single miner-directed credit, so a sub-ED fee is never rejected on its own.
+		Self::mint_reward(miner.as_ref(), tx_fees.saturating_add(miner_reward));
```

## Proof of Concept

Add to `pallets/mining-rewards/src/tests.rs`. `TestMiner(7)` is not endowed by the mock genesis, so it is a fresh author address.

```bash
cargo test -p pallet-mining-rewards poc_sub_ed_fee_redirected_to_treasury -- --nocapture
```

```rust
#[test]
fn poc_sub_ed_fee_redirected_to_treasury() {
	ExistentialDeposit::set(1_000_000_000);
	new_test_ext().execute_with(|| {
		let miner = TestMiner(7);
		assert_eq!(Balances::free_balance(miner.account_id()), 0, "fresh author address");

		let fees = 500_000_000u128; // 0.0005 QUAN, below the 1e9 ED
		MiningRewards::collect_transaction_fees(fees);
		set_miner_preimage_digest(miner.preimage());

		let current_supply = Balances::total_issuance() + fees;
		let total_reward = (MaxSupply::get() - current_supply) / EmissionDivisor::get();
		let treasury_reward = Treasury::portion().mul_floor(total_reward);
		let miner_reward = total_reward - treasury_reward;
		let treasury_before = Balances::free_balance(Treasury::account_id());

		MiningRewards::on_finalize(1);

		let miner_got = Balances::free_balance(miner.account_id());
		let treasury_got = Balances::free_balance(Treasury::account_id()) - treasury_before;

		println!("fees collected   : {}", fees);
		println!("miner balance    : {} (base reward {})", miner_got, miner_reward);
		println!("treasury gained  : {} (its share {})", treasury_got, treasury_reward);
		println!("retry bucket     : {}", MiningRewards::collected_fees());

		System::assert_has_event(
			Event::MinerRewardRedirected { miner: miner.account_id(), reward: fees }.into(),
		);
		assert_eq!(miner_got, miner_reward, "the miner got the base reward but not the fee");
		assert_eq!(treasury_got, treasury_reward + fees, "the fee went to the treasury");
		assert_eq!(MiningRewards::collected_fees(), 0, "and nothing retries it");
	});
}
```

Output:

```
fees collected   : 500000000
miner balance    : 692449530206 (base reward 692449530206)
treasury gained  : 692949530205 (its share 692449530205)
retry bucket     : 0
```
