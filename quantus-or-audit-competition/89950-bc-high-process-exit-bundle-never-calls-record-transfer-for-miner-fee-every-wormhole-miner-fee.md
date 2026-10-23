# 89950 bc high process exit bundle never calls record transfer for miner fee every wormhole miner fee credit is permanently unspendable

**Submitted on Aug 22nd 2026 at 02:54:52 UTC by @volpasec for** [**Audit Comp | Quantus**](https://immunefi.com/audit-competition/audit-comp-quantus)

* **Report ID:** #89950
* **Report Type:** Blockchain/DLT
* **Report severity:** High
* **Target:** <https://github.com/immunefi-team/audit-comp-quantus-chain>
* **Impacts:**
  * Permanent freezing of funds

## Description

## Brief/Intro

`process_exit_bundle` (`pallets/wormhole/src/lib.rs`) credits the block author's share of the wormhole exit volume fee (`miner_fee`) via `Unbalanced::increase_balance`, to an account produced by `qp_wormhole::extract_author_from_digest` — a Poseidon2 hash of the PreRuntime digest, structurally identical to the codebase's own `WormholeAddress` "unspendable account" type. Unlike the `exit_account` credit 90 lines earlier in the same function, this branch never calls `Self::record_transfer`, so no leaf is ever inserted into `pallet-zk-tree`'s Merkle tree. The recipient has no ML-DSA keypair capable of signing for it and no Merkle leaf any wormhole exit proof can reference, so the credited value is permanently unreachable. This fires on every `verify_private_batch`/`verify_public_batch` call with a nonzero fee — at the real runtime's `VolumeFeeRateBps = 4`, every exit with nonzero volume — with no attacker and no privileged actor involved.

## Vulnerability Details

**The bug.** `process_exit_bundle`'s `miner_fee` branch:\
<https://github.com/immunefi-team/audit-comp-quantus-chain/blob/3b243b870a5442a6e5f443056f67bbba87d472f5/pallets/wormhole/src/lib.rs#L944-L983>

```rust
			if !miner_fee.is_zero() {
				let digest = frame_system::Pallet::<T>::digest();
				if let Some(author) = qp_wormhole::extract_author_from_digest::<
					<T as frame_system::Config>::AccountId,
					_,
				>(digest.logs.iter())
				{
					match <T::Currency as Unbalanced<_>>::increase_balance(
						&author,
						miner_fee,
						frame_support::traits::tokens::Precision::Exact,
					) {
						Ok(_) => {
							Self::deposit_event(Event::MinerVolumeFeePaid {
								miner: author,
								amount: miner_fee,
							});
						},
						Err(e) => {
							burn_amount_u128 = burn_amount_u128.saturating_add(miner_fee_u128);
						},
					}
				} else {
					burn_amount_u128 = burn_amount_u128.saturating_add(miner_fee_u128);
				}
			}
```

No call to `Self::record_transfer` exists in this branch. `author` is always the output of `derive_wormhole_address` (Poseidon2 rehash) via `extract_author_from_digest`:\
<https://github.com/immunefi-team/audit-comp-quantus-chain/blob/3b243b870a5442a6e5f443056f67bbba87d472f5/primitives/wormhole/src/lib.rs#L97-L151>

```rust
pub fn derive_wormhole_address(inner_digest: [u8; 32]) -> Result<[u8; 32], &'static str> {
	rehash_to_bytes(&inner_digest)
}

pub fn extract_author_from_digest<'a, AccountId, I>(digest: I) -> Option<AccountId>
where
	AccountId: Decode,
	I: IntoIterator<Item = &'a DigestItem>,
{
	for log in digest {
		let DigestItem::PreRuntime(engine_id, data) = log else { continue };
		if *engine_id != POW_ENGINE_ID { continue; }
		if data.len() != 32 { return None; }
		let preimage: [u8; 32] = data.as_slice().try_into().ok()?;
		let address_bytes = derive_wormhole_address(preimage).ok()?;
		return AccountId::decode(&mut &address_bytes[..]).ok();
	}
	None
}
```

**The correct pattern exists 90 lines earlier, in the same function.** The `exit_account` credit uses the identical `increase_balance` call, immediately followed by `Self::record_transfer`:\
<https://github.com/immunefi-team/audit-comp-quantus-chain/blob/3b243b870a5442a6e5f443056f67bbba87d472f5/pallets/wormhole/src/lib.rs#L818-L859>

```rust
			for (exit_account, exit_balance) in &processed_accounts {
				match <T::Currency as Unbalanced<_>>::increase_balance(
					exit_account,
					*exit_balance,
					frame_support::traits::tokens::Precision::Exact,
				) {
					Ok(_) => {},
					Err(e) => { continue; },
				}
				minted_exit_amount = minted_exit_amount.saturating_add(*exit_balance);
				let from_account: <T as Config>::WormholeAccountId = mint_account.clone().into();
				let to_account: <T as Config>::WormholeAccountId = exit_account.clone().into();
				Self::record_transfer(
					T::AssetId::default(),
					&from_account,
					&to_account,
					*exit_balance,
				);
			}
```

**`record_transfer` is the only function in this pallet that ever inserts a leaf:**\
<https://github.com/immunefi-team/audit-comp-quantus-chain/blob/3b243b870a5442a6e5f443056f67bbba87d472f5/pallets/wormhole/src/lib.rs#L1280-L1330>

```
$ grep -rn "insert_leaf(" --include="*.rs" . | grep -v "/target/" | grep -v tests.rs
./pallets/zk-tree/src/lib.rs:253:  pub fn insert_leaf(
./pallets/zk-tree/src/lib.rs:358:  let (leaf_index, _root) = Self::insert_leaf(to, transfer_count, asset_id, amount);

$ grep -n "Self::record_transfer(" pallets/wormhole/src/lib.rs
486:    # on_initialize, genesis
854:    # exit_account, the correct pattern above
1367:   # record_transfer_proof, native branch
1386:   # record_transfer_proof, asset branch
```

Four call sites, none of them `miner_fee` or `aggregator_fee`.

**Why the recipient has no signature path.** `WormholeAddress` and `DilithiumSigner` are disjoint account derivations:\
<https://github.com/immunefi-team/audit-comp-quantus-chain/blob/3b243b870a5442a6e5f443056f67bbba87d472f5/primitives/dilithium-crypto/src/traits.rs#L77-L85>

```rust
pub struct WormholeAddress(pub H256);
// AccountID32 for a wormhole address is the same as the address itself
impl IdentifyAccount for WormholeAddress {
	type AccountId = AccountId32;
	fn into_account(self) -> Self::AccountId {
		AccountId32::new(self.0.as_bytes().try_into().unwrap())
	}
}
```

<https://github.com/immunefi-team/audit-comp-quantus-chain/blob/3b243b870a5442a6e5f443056f67bbba87d472f5/primitives/dilithium-crypto/src/traits.rs#L191-L201>

```rust
impl IdentifyAccount for DilithiumSigner {
	type AccountId = AccountId32;
	fn into_account(self) -> AccountId32 {
		match self {
			Self::Dilithium87(who) => hash_bytes(who.as_ref()).into(),
			Self::Dilithium65(who) => hash_bytes(who.as_ref()).into(),
		}
	}
}
```

A signature-capable account's `AccountId` is `hash_bytes(ml_dsa_public_key)`. A wormhole address's `AccountId` is the Poseidon2-rehash output itself — no public key anywhere in the derivation. Signing for a `miner_fee` credit requires an ML-DSA keypair whose `hash_bytes(pubkey)` equals a value the runtime already fixed: a second-preimage attack on a \~128-bit hash (per the March 2026 Equilibrium audit's own Poseidon2 security analysis, listed on this program's Audits page).

**Production values, not mock artifacts.** `runtime/src/configs/mod.rs#L734-L736`: `VolumeFeeRateBps: u32 = 4`, `VolumeFeesBurnRate = Permill::from_percent(50)`. Identical in `pallets/wormhole/src/mock.rs#L98-L106`.

**The same pallet family gets this right elsewhere.** `pallet-mining-rewards::mint_reward` pays the same block author (via the same `extract_author_from_digest`) and calls `record_transfer_proof` on every credit:\
<https://github.com/immunefi-team/audit-comp-quantus-chain/blob/3b243b870a5442a6e5f443056f67bbba87d472f5/pallets/mining-rewards/src/lib.rs#L220-L241>

```rust
		fn mint_reward(maybe_miner: Option<&T::AccountId>, reward: BalanceOf<T>) {
			match maybe_miner {
				Some(miner) => {
					match T::Currency::mint_into(miner, reward) {
						Ok(_) => {
							T::ProofRecorder::record_transfer_proof(
								None, mint_account, miner.clone(), reward,
							);
							Self::deposit_event(Event::MinerRewarded { miner: miner.clone(), reward });
						},
```

The base block reward gets a leaf. The volume-fee share does not.

**Trigger path — no attacker required:**

{% stepper %}
{% step %}

### Submit a valid exit

Any account submits a valid `verify_private_batch`/`verify_public_batch` extrinsic with a nonzero exit amount (unsigned, permissionless — the only way to withdraw from the wormhole).
{% endstep %}

{% step %}

### Compute and split the fee

`process_exit_bundle` computes the fee and splits it into `burn_amount`/`miner_fee`.
{% endstep %}

{% step %}

### Derive the block author

The block author is derived via `extract_author_from_digest` — always a wormhole-style address.
{% endstep %}

{% step %}

### Credit the miner fee

`miner_fee` is credited. `record_transfer` is not called.
{% endstep %}

{% step %}

### Finalize the block

The block finalizes.
{% endstep %}

{% step %}

### Funds become unreachable

The credit is now unreachable by signature and unreachable by proof.

Every exit with `miner_fee > 0` reproduces this.
{% endstep %}
{% endstepper %}

**Why this is permanent, addressing the three most likely objections directly:**

<details>

<summary>Is there another leaf-insertion path?</summary>

No — confirmed above by exhaustive grep; four call sites, all accounted for.

</details>

<details>

<summary>Could governance recover it?</summary>

`pallet_balances::force_transfer` (`pallets/balances/src/lib.rs#L685`) requires Root origin. A runtime upgrade could backfill a leaf. Both require the Technical Collective acting within its attributed privileges — out of scope under this program's rules.

</details>

<details>

<summary>Could an unrelated later credit to the same address rescue it?</summary>

`WormholeProofRecorderExtension::record_proofs_from_events_since` (`runtime/src/transaction_extensions.rs#L309-L396`) scans `Balances::Transfer`/`Minted`/`TransferOnHold`/`ReserveRepatriated` events — a fresh signed transfer into the miner's wormhole address would be caught and would insert a genuine new leaf. That leaf does not rescue the frozen credit: `pallet-zk-tree`'s `ZkLeaf` (`pallets/zk-tree/src/lib.rs#L139-L148`) binds a fixed `amount` and a specific, monotonically assigned `transfer_count` at insertion time. A later top-up makes only that top-up's amount exitable. No operation in this codebase inserts a leaf at a past `transfer_count` position, and no leaf's amount derives from the account's current `free_balance`.

</details>

<details>

<summary>Is this EQ-QNT-WORMHOLE-F-03 (already fixed)?</summary>

No. F-03 (Equilibrium, March 20 2026) found the pallet decoding the PreRuntime preimage directly as an `AccountId` instead of Poseidon2-hashing it — misdirecting the fee to an account the miner didn't control at all. That's fixed in this commit (`extract_author_from_digest` now hashes correctly, matching `pallet-mining-rewards`). At the time of that audit, `record_transfer` wrote into a flat `TransferProof` StorageMap with no Merkle tree (per EQ-QNT-WORMHOLE-R-08's quoted code) — `pallet-zk-tree` postdates that audit. F-03 fixed which address the fee resolves to; it never evaluated whether that address's credit satisfies the Merkle-leaf requirement introduced afterward.

</details>

<details>

<summary>Is this a published known issue?</summary>

The program lists three (referendum vote removal, plaintext miner-node connection, zk-tree depth ceiling). None describes fee distribution or leaf coverage.

</details>

## Impact Details

100% of `miner_fee` credited by `process_exit_bundle` is permanently frozen — the branch is unconditional on amount, so every exit producing a nonzero fee reproduces this identically, at the production `VolumeFeeRateBps = 4`. Not delayed, not recoverable after any number of blocks, not recoverable by the recipient or by any permissionless mechanism in this codebase.

`aggregator_fee` (`pallets/wormhole/src/lib.rs#L892-935`, same function) shares the identical pattern, but `aggregator_address` is a prover-supplied public input rather than a protocol-derived wormhole address — whether a given rebate is affected depends on the address the aggregator chooses to supply. Noted as a related pattern, not the basis of this report's severity claim.

## References

* Commit audited: <https://github.com/immunefi-team/audit-comp-quantus-chain/tree/3b243b870a5442a6e5f443056f67bbba87d472f5>
* `pallets/wormhole/src/lib.rs` (miner\_fee branch): <https://github.com/immunefi-team/audit-comp-quantus-chain/blob/3b243b870a5442a6e5f443056f67bbba87d472f5/pallets/wormhole/src/lib.rs#L944-L983>
* `pallets/wormhole/src/lib.rs` (exit\_account, correct pattern): <https://github.com/immunefi-team/audit-comp-quantus-chain/blob/3b243b870a5442a6e5f443056f67bbba87d472f5/pallets/wormhole/src/lib.rs#L818-L859>
* `pallets/wormhole/src/lib.rs` (record\_transfer): <https://github.com/immunefi-team/audit-comp-quantus-chain/blob/3b243b870a5442a6e5f443056f67bbba87d472f5/pallets/wormhole/src/lib.rs#L1280-L1330>
* `primitives/wormhole/src/lib.rs` (derive\_wormhole\_address, extract\_author\_from\_digest): <https://github.com/immunefi-team/audit-comp-quantus-chain/blob/3b243b870a5442a6e5f443056f67bbba87d472f5/primitives/wormhole/src/lib.rs#L97-L151>
* `primitives/dilithium-crypto/src/traits.rs` (WormholeAddress vs DilithiumSigner): <https://github.com/immunefi-team/audit-comp-quantus-chain/blob/3b243b870a5442a6e5f443056f67bbba87d472f5/primitives/dilithium-crypto/src/traits.rs#L77-L201>
* `runtime/src/configs/mod.rs` (production fee rate): <https://github.com/immunefi-team/audit-comp-quantus-chain/blob/3b243b870a5442a6e5f443056f67bbba87d472f5/runtime/src/configs/mod.rs#L734-L736>
* `pallets/mining-rewards/src/lib.rs` (comparator, mint\_reward): <https://github.com/immunefi-team/audit-comp-quantus-chain/blob/3b243b870a5442a6e5f443056f67bbba87d472f5/pallets/mining-rewards/src/lib.rs#L220-L299>
* `runtime/src/transaction_extensions.rs` (WormholeProofRecorderExtension): <https://github.com/immunefi-team/audit-comp-quantus-chain/blob/3b243b870a5442a6e5f443056f67bbba87d472f5/runtime/src/transaction\\_extensions.rs#L309-L396>
* `pallets/balances/src/lib.rs` (force\_transfer, Root-only): <https://github.com/immunefi-team/audit-comp-quantus-chain/blob/3b243b870a5442a6e5f443056f67bbba87d472f5/pallets/balances/src/lib.rs#L685>
* EQ-QNT-WORMHOLE-F-03 / EQ-QNT-WORMHOLE-R-08 — Equilibrium, "Quantus Network: Security and Performance Review," 2026-03-20, listed on this program's Audits page.

## Proof of Concept

This PoC runs inside `pallets/wormhole`'s own `mock.rs`/`new_test_ext` harness, at `VolumeFeeRateBps = 4` and `VolumeFeesBurnRate = 50%` (identical to the production runtime), against `pallet-zk-tree`'s real Merkle-tree implementation:

<https://github.com/immunefi-team/audit-comp-quantus-chain/blob/3b243b870a5442a6e5f443056f67bbba87d472f5/pallets/wormhole/src/mock.rs#L108-L127>

{% stepper %}
{% step %}

### Add the test

In `pallets/wormhole/src/tests.rs`, inside `mod private_batch_proof_tests` (opens L568, closes L1168), paste the test below immediately after the line containing exactly `author_balance_before + expected_miner_fee` (L1090) and its closing `);`/`});`/`}` sequence, and before the `#[test]` that immediately precedes `fn test_verify_private_batch_cannot_replay() {` (L1096).
{% endstep %}

{% step %}

### Run the test

```bash
cd pallets/wormhole
cargo test miner_fee_credit_moves_balance_but_records_no_zk_leaf_for_the_author -- --nocapture
```

{% endstep %}

{% step %}

### Test

```rust
	#[test]
	fn miner_fee_credit_moves_balance_but_records_no_zk_leaf_for_the_author() {
		new_test_ext().execute_with(|| {
			let proof = deserialize_test_proof();
			let inputs = parse_private_batch_public_inputs(&proof).expect("Should parse");

			let block_number = inputs.block_data.block_number as u64;
			let block_hash_bytes: [u8; 32] =
				inputs.block_data.block_hash.as_ref().try_into().unwrap();
			let block_hash = H256::from(block_hash_bytes);
			frame_system::BlockHash::<Test>::insert(block_number, block_hash);
			System::set_block_number(block_number + 10);

			let miner_preimage = [7u8; 32];
			set_miner_preimage_digest(miner_preimage);
			let expected_author = sp_core::crypto::AccountId32::from(
				qp_wormhole::derive_wormhole_address(miner_preimage)
					.expect("test preimage limbs are canonical"),
			);

			let canonical_bytes = pallet_zk_tree::tree::canonicalize_account_bytes(
				<[u8; 32]>::try_from(expected_author.as_ref()).expect("32 bytes"),
			);
			let canonical_author = AccountId::from(canonical_bytes);

			let leaf_count_before = ZkTree::leaf_count();
			let transfer_count_before = crate::TransferCount::<Test>::get(&canonical_author);
			let author_balance_before = Balances::free_balance(expected_author.clone());
			assert_eq!(transfer_count_before, 0, "author must start with no recorded transfers");

			assert_ok!(Wormhole::verify_private_batch(
				RawOrigin::None.into(),
				get_test_proof_bytes()
			));

			let author_balance_after = Balances::free_balance(expected_author.clone());
			let leaf_count_after = ZkTree::leaf_count();
			let transfer_count_after = crate::TransferCount::<Test>::get(&canonical_author);

			assert!(
				author_balance_after > author_balance_before,
				"sanity: miner_fee must actually be credited"
			);
			assert!(
				leaf_count_after > leaf_count_before,
				"sanity: the exit itself must record at least one leaf (for the exit account)"
			);
			assert_eq!(
				transfer_count_after, transfer_count_before,
				"BUG: miner_fee increased the author's balance without ever calling \
				 record_transfer for the author (lib.rs:955 uses increase_balance only) — \
				 no leaf exists for this credit, so no withdrawal proof can ever reference it"
			);

			use qp_wormhole::TransferProofRecorder;
			assert!(
				<Wormhole as TransferProofRecorder<AccountId, u32, u128>>::record_transfer_proof(
					None,
					MintingAccount::get(),
					expected_author.clone(),
					1_000 * UNIT,
				),
				"a native credit to this same author must be recordable"
			);
			assert_eq!(
				crate::TransferCount::<Test>::get(&canonical_author),
				transfer_count_before + 1,
				"once record_transfer IS called for this account, TransferCount DOES advance \
				 — the miner_fee path's omission is the anomaly, not a structural impossibility"
			);
		});
	}
```

{% endstep %}

{% step %}

### Output

```
running 1 test
test tests::private_batch_proof_tests::miner_fee_credit_moves_balance_but_records_no_zk_leaf_for_the_author ... ok

test result: ok. 1 passed; 0 failed; 0 ignored; 0 measured; 71 filtered out; finished in 0.10s
```

{% endstep %}
{% endstepper %}
