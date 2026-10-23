# 90180 bc high wormhole miner volume fee credits are uncollectible because no proof leaf is recorded

**Submitted on Aug 23rd 2026 at 13:54:22 UTC by @y4rvin for** [**Audit Comp | Quantus**](https://immunefi.com/audit-competition/audit-comp-quantus)

* **Report ID:** #90180
* **Report Type:** Blockchain/DLT
* **Report severity:** High
* **Target:** <https://github.com/immunefi-team/audit-comp-quantus-chain>
* **Impacts:**
  * Direct loss of funds

## Description

## Summary

`process_exit_bundle` credits the QPoW block author's Wormhole account with its share of every successful exit volume fee, but it does not record a Wormhole transfer or ZK-tree leaf for that credit. The account is keyless and cannot submit an ordinary signed transfer. Quantus's official `wormhole collect-rewards` flow discovers withdrawals only from transfer rows and requires the row's leaf index, transfer count, and Merkle proof. Consequently the credited miner fee has no collection path.

## Frozen target

* Competition repository: `https://github.com/immunefi-team/audit-comp-quantus-chain.git`
* Revision: `3b243b870a5442a6e5f443056f67bbba87d472f5`
* Official CLI evidence: `https://github.com/Quantus-Network/quantus-cli.git` at `d9e985d217f00fe8aaeb668e89136da58e75a305`

## Root cause

The successful exit-principal branch credits each recipient and immediately calls `record_transfer`, which advances the recipient transfer count and inserts a leaf (`pallets/wormhole/src/lib.rs:818-859`).

The miner-fee branch is instead at `pallets/wormhole/src/lib.rs:944-983`. It calls `Unbalanced::increase_balance` at `:955-959`, emits `MinerVolumeFeePaid` at `:961-964`, and never calls `record_transfer`. `increase_balance` calls only event-free `write_balance` (`frame/support/src/traits/tokens/fungible/regular.rs:205-233`), while `Balances::Minted` is emitted only by the separate `Mutate::done_mint_into` hook (`pallets/balances/src/impl_fungible.rs:157-194`). The runtime scanner therefore has no event from which it could create a leaf. This leaves the miner's free balance higher while its Wormhole transfer count and proof tree remain unchanged.

The beneficiary is not an ordinary signable account:

* the miner account is `Poseidon(inner_digest)` from the QPoW pre-runtime digest (`primitives/wormhole/src/lib.rs:97-148`);
* runtime signatures support only Dilithium87 and Dilithium65 (`primitives/dilithium-crypto/src/types.rs:71-98`);
* signature verification requires `hash_bytes(embedded_public_key) == signed_origin` (`primitives/dilithium-crypto/src/traits.rs:152-199`); and
* the runtime describes the corresponding address as keyless and says it can never sign (`runtime/src/genesis_config_presets.rs:301-303`).

## Why the official collection path cannot recover the credit

The official CLI documents `wormhole collect-rewards` as the miner withdrawal path. Its current implementation:

1. queries indexed Wormhole transfer rows and returns zero withdrawals if the query is empty (`quantus-cli/src/collect_rewards_lib.rs:241-265`);
2. requires each transfer's `leaf_index` and fetches a ZK Merkle proof (`:382-406`);
3. builds proof inputs from the leaf amount and transfer count, not the account's free balance (`:404-442`); and
4. obtains the leaf index and transfer count from the indexer's `transfer` table (`quantus-cli/src/subsquid/client.rs:102-134`).

`MinerVolumeFeePaid` supplies none of these proof inputs. Indexing its amount into a free-balance view cannot manufacture a transfer row, leaf, or Merkle path.

## Reproduction

```bash
git clone https://github.com/immunefi-team/audit-comp-quantus-chain.git quantus-chain
git -C quantus-chain checkout --detach 3b243b870a5442a6e5f443056f67bbba87d472f5
git -C quantus-chain apply CAND-001_miner_volume_fee_no_leaf_final.patch
git -C quantus-chain diff --check -- pallets/wormhole/src/tests.rs
cargo test \
  --manifest-path quantus-chain/Cargo.toml \
  --release \
  -p pallet-wormhole \
  test_verify_private_batch_emits_miner_volume_fee_paid \
  -- \
  --nocapture
```

Output:

```
event_semantics_control: balances_minted_events 0
affected: miner_balance 0 -> 5000000000, miner_transfer_count 0 -> 0, tree_leaf_count 0 -> 1
later_reward_control: total_balance_delta 15000000000, recorded_leaf_amount 10000000000, stranded_fee 5000000000
negative_control: accumulated_fee 10000000000, miner_transfer_count 0 -> 2, tree_leaf_count 0 -> 3
test result: ok. 1 passed; 0 failed
```

Interpretation:

* `affected` executes the real unsigned private-batch entrypoint. The miner receives `5,000,000,000` planck in free balance, but its transfer count stays zero. The only new leaf is the exit-principal leaf.
* `later_reward_control` mints and records a normal `10,000,000,000`-planck mining reward. Its leaf commits exactly that reward; it does not sweep the earlier missing fee despite the total balance increasing by `15,000,000,000`.
* `negative_control` shows the necessary repair boundary: two minimum `5,000,000,000`-planck fee shares must accumulate to one `10,000,000,000`-planck Wormhole quantum before a useful leaf can be recorded.

The unified diff included in the form's PoC field changes only the test file. It uses a real fixture proof and the production `verify_private_batch -> process_exit_bundle` path. The block-author digest and soundness-pool state are normal test-fixture setup.

## Impact

Runtime parameters set a four-basis-point volume fee with a one-quantum (`0.01 QUAN`) minimum and split it 50/50 between burning and the miner (`runtime/src/configs/mod.rs:725-752`). Therefore the minimum affected miner share is `0.005 QUAN` (`5,000,000,000` planck). Each successful author fee credit accumulates in a keyless free balance but is absent from the leaf-driven withdrawal ledger. The loss repeats on affected Wormhole exits.

## Public prior art and why this is a fix bypass

This report deliberately discloses the closest public material:

* Eiger F-03 concerned deriving the wrong miner author and burning/misdirecting its fee.
* Eiger F-05 concerned signed transfer-call variants not being recorded by the old transaction extension.
* PR #619 / commit `60ec991` publicly observed that the miner fee used `increase_balance` without `record_transfer`, but its stated impact was indexer-versus-RPC balance drift. It added only `MinerVolumeFeePaid` plus an indexer free-balance credit and tested the event and balance.

PR #619's explanation also assumed indexers saw a bare `Balances::Minted`. Frozen FRAME semantics and the release regression show that assumption is false: this `increase_balance` emits no `Balances::Minted` at all. Thus the public fix repaired only the explicitly added off-chain event consumer while leaving the on-chain proof recorder with no recognizable event and no explicit fee-leaf call.

The frozen code includes those remediations. The surviving defect is that PR #619's representation still cannot be consumed by the official collection path: it creates no transfer row, leaf index, transfer count, or Merkle proof. Neither the PR nor the cited audit finding states that the correctly-derived fee remains uncollectible through `collect-rewards`, adds the required leaf, or tests withdrawal. This is the competition's stated “bypass of a public fix” case, while the public overlap remains fully disclosed for adjudication.

## Recommended fix

Do not expose an unbacked free-balance representation to a keyless miner. Accumulate sub-quantum miner fees per beneficiary. When the accumulated amount reaches one or more full `SCALE_DOWN_FACTOR` quanta, mint only the full-quantum portion and atomically call the Wormhole proof recorder for that exact amount; retain the remainder in accumulator storage. Emit separate accrual and payout events so indexers cannot confuse pending fees with collectible balance.

Add regressions that assert:

1. every successful miner-fee balance increment has a matching transfer row and non-zero leaf;
2. minimum fees accumulate rather than quantizing into zero-value leaves;
3. `collect-rewards` discovers and proves the payout;
4. a later ordinary mining reward cannot hide an unmatched fee balance; and
5. public- and private-batch exits use the same invariant.

## Proof of Concept

This is a local regression against the exact frozen competition revision. It changes only the existing Wormhole test module and uses the real fixture proof plus the production `verify_private_batch -> process_exit_bundle` path. No production code, verifier bypass, live network, or post-call storage mutation is used.

{% stepper %}
{% step %}

## Check out the frozen target

```bash
git clone https://github.com/immunefi-team/audit-comp-quantus-chain.git quantus-chain
git -C quantus-chain checkout --detach 3b243b870a5442a6e5f443056f67bbba87d472f5
```

{% endstep %}

{% step %}

## Save and apply the test-only patch

Save the following unified diff as `CAND-001_miner_volume_fee_no_leaf_final.patch`:

```diff
diff --git a/pallets/wormhole/src/tests.rs b/pallets/wormhole/src/tests.rs
index 4473b70..8039342 100644
--- a/pallets/wormhole/src/tests.rs
+++ b/pallets/wormhole/src/tests.rs
@@ -570,7 +570,10 @@ mod private_batch_proof_tests {
 		mock::*,
 		pallet::{Error, UsedNullifiers},
 	};
-	use frame_support::{assert_noop, assert_ok};
+	use frame_support::{
+		assert_noop, assert_ok,
+		traits::fungible::Mutate,
+	};
 	use frame_system::RawOrigin;
 	use qp_wormhole_verifier::{parse_private_batch_public_inputs, ProofWithPublicInputs, C, F};
 	use sp_core::H256;
@@ -1072,6 +1075,8 @@ mod private_batch_proof_tests {
 			assert!(expected_miner_fee > 0, "fixture should produce a non-zero miner fee");
 
 			let author_balance_before = Balances::free_balance(expected_author.clone());
+			let author_transfer_count_before = Wormhole::transfer_count(&expected_author);
+			let leaf_count_before = ZkTree::leaf_count();
 
 			assert_ok!(Wormhole::verify_private_batch(
 				RawOrigin::None.into(),
@@ -1086,9 +1091,119 @@ mod private_batch_proof_tests {
 				.into(),
 			);
 			assert_eq!(
-				Balances::free_balance(expected_author),
+				Balances::free_balance(expected_author.clone()),
 				author_balance_before + expected_miner_fee
 			);
+			let minted_event_count = System::events()
+				.iter()
+				.filter(|record| {
+					matches!(
+						record.event,
+						RuntimeEvent::Balances(pallet_balances::Event::<Test>::Minted { .. })
+					)
+				})
+				.count();
+			assert_eq!(
+				minted_event_count, 0,
+				"Unbalanced::increase_balance must remain event-free for the event scanner"
+			);
+			println!("event_semantics_control: balances_minted_events {minted_event_count}");
+
+			// A miner address is a Wormhole address derived from the pre-runtime
+			// preimage. A balance credit is not spendable through Wormhole unless the
+			// corresponding leaf is recorded. The current fee path uses the event-free
+			// `Unbalanced::increase_balance` and does not call `record_transfer`, so the
+			// credit changes the balance but leaves no spendable proof material.
+			assert_eq!(
+				Wormhole::transfer_count(&expected_author),
+				author_transfer_count_before,
+				"miner volume fee must create a Wormhole leaf"
+			);
+			assert_eq!(
+				ZkTree::leaf_count(),
+				leaf_count_before +
+					inputs.account_data.iter().filter(|a| a.summed_output_amount > 0).count() as u64,
+				"the exit creates only principal leaves, not a miner-fee leaf"
+			);
+			assert!(
+				expected_miner_fee < crate::SCALE_DOWN_FACTOR,
+				"the first miner payout is also below the Wormhole leaf quantum"
+			);
+			println!(
+				"affected: miner_balance {} -> {}, miner_transfer_count {} -> {}, tree_leaf_count {} -> {}",
+				author_balance_before,
+				Balances::free_balance(expected_author.clone()),
+				author_transfer_count_before,
+				Wormhole::transfer_count(&expected_author),
+				leaf_count_before,
+				ZkTree::leaf_count(),
+			);
+
+			// Counterfactual: an ordinary later mining-reward credit records only its
+			// own amount. It cannot retroactively sweep the earlier leaf-less fee into
+			// proof material, because the leaf payload is supplied explicitly rather
+			// than derived from the beneficiary's total free balance.
+			let subsequent_reward = crate::SCALE_DOWN_FACTOR;
+			assert_ok!(Balances::mint_into(&expected_author, subsequent_reward));
+			Wormhole::record_transfer(
+				0u32,
+				&MINTING_ACCOUNT,
+				&expected_author,
+				subsequent_reward,
+			);
+			let reward_leaf = ZkTree::leaf(ZkTree::leaf_count() - 1)
+				.expect("subsequent mining reward must insert its own leaf");
+			assert_eq!(reward_leaf.to, expected_author);
+			assert_eq!(reward_leaf.amount, subsequent_reward);
+			assert_eq!(
+				Balances::free_balance(expected_author.clone()),
+				author_balance_before + expected_miner_fee + subsequent_reward
+			);
+			assert_eq!(
+				Wormhole::transfer_count(&expected_author),
+				author_transfer_count_before + 1
+			);
+			println!(
+				"later_reward_control: total_balance_delta {}, recorded_leaf_amount {}, stranded_fee {}",
+				expected_miner_fee + subsequent_reward,
+				reward_leaf.amount,
+				expected_miner_fee,
+			);
+
+			// Negative control: two minimum miner payouts reach one full Wormhole
+			// quantum. A correct fee accumulator routed through the recorder used for
+			// exit principal and ordinary mining rewards creates spendable proof material.
+			let minting_account = MINTING_ACCOUNT;
+			let accumulated_miner_fee = expected_miner_fee.saturating_mul(2);
+			assert_eq!(accumulated_miner_fee, crate::SCALE_DOWN_FACTOR);
+			Wormhole::record_transfer(
+				0u32,
+				&minting_account,
+				&expected_author,
+				accumulated_miner_fee,
+			);
+			assert_eq!(
+				Wormhole::transfer_count(&expected_author),
+				author_transfer_count_before + 2
+			);
+			assert_eq!(
+				ZkTree::leaf_count(),
+				leaf_count_before +
+					inputs.account_data.iter().filter(|a| a.summed_output_amount > 0).count() as u64 +
+					2
+			);
+			let control_leaf = ZkTree::leaf(ZkTree::leaf_count() - 1)
+				.expect("negative control must insert a miner-fee leaf");
+			assert_eq!(control_leaf.to, expected_author);
+			assert_eq!(control_leaf.amount, accumulated_miner_fee);
+			println!(
+				"negative_control: accumulated_fee {}, miner_transfer_count {} -> {}, tree_leaf_count {} -> {}",
+				accumulated_miner_fee,
+				author_transfer_count_before,
+				Wormhole::transfer_count(&expected_author),
+				leaf_count_before,
+				ZkTree::leaf_count(),
+			);
 		});
 	}
 
```

Apply and verify it:

```bash
git -C quantus-chain apply CAND-001_miner_volume_fee_no_leaf_final.patch
git -C quantus-chain diff --check -- pallets/wormhole/src/tests.rs
```

{% endstep %}

{% step %}

## Run the matched test

```bash
cargo test \
  --manifest-path quantus-chain/Cargo.toml \
  --release \
  -p pallet-wormhole \
  test_verify_private_batch_emits_miner_volume_fee_paid \
  -- \
  --nocapture
```

Expected decisive output:

```
event_semantics_control: balances_minted_events 0
affected: miner_balance 0 -> 5000000000, miner_transfer_count 0 -> 0, tree_leaf_count 0 -> 1
later_reward_control: total_balance_delta 15000000000, recorded_leaf_amount 10000000000, stranded_fee 5000000000
negative_control: accumulated_fee 10000000000, miner_transfer_count 0 -> 2, tree_leaf_count 0 -> 3
test result: ok. 1 passed; 0 failed
```

The affected path increases the miner's free balance while leaving its transfer count unchanged; the one new leaf belongs only to the exit principal. The later-reward control proves a normal leaf records only its explicit reward amount and cannot sweep the earlier fee. The negative control shows that two minimum fee shares must accumulate to one full Wormhole quantum and pass through `record_transfer` before the miner has useful proof material.
{% endstep %}
{% endstepper %}
