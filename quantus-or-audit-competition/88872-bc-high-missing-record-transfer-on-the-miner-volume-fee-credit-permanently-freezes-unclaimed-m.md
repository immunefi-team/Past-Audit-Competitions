# 88872 bc high missing record transfer on the miner volume fee credit permanently freezes unclaimed miner yield on a non ml dsa signable qpow wormhole author

**Submitted on Aug 15th 2026 at 10:56:36 UTC by @n3310y for** [**Audit Comp | Quantus**](https://immunefi.com/audit-competition/audit-comp-quantus)

* **Report ID:** #88872
* **Report Type:** Blockchain/DLT
* **Report severity:** High
* **Target:** <https://github.com/immunefi-team/audit-comp-quantus-chain>
* **Impacts:**
  * Permanent freezing of unclaimed yield

## Description

## Brief/Intro

Omitting `record_transfer()` after crediting the miner volume fee with event-free `Unbalanced::increase_balance()` will cause permanent freezing of unclaimed yield for QPoW miners as every wormhole exit credits that fee to the Poseidon-derived, non-ML-DSA-signable block author, which has no matching ZK-tree leaf.

This is a vulnerability path in honest settlement, not a theft attack. No privileged origin is required. It fires on every successful private-batch (and public-batch) exit that pays a non-zero miner share.

## Vulnerability Details

Quantus spends native value through a ZK Merkle tree, not through Dilithium signatures on the recipient. A credit is spendable via wormhole if and only if `pallet-zk-tree` stores a leaf `(to, transfer_count, asset_id, amount)` and `TransferCount` is advanced. `record_transfer()` is the function that inserts that leaf and emits `NativeTransferred`.

`process_exit_bundle()` is the single settlement function for both `verify_private_batch()` and `verify_public_batch()`. For each successful user exit it does:

```rust
match <T::Currency as Unbalanced<_>>::increase_balance(
    exit_account,
    *exit_balance,
    Precision::Exact,
) { /* ... */ }

minted_exit_amount = minted_exit_amount.saturating_add(*exit_balance);

// @audit user exits: event-free mint, then an explicit ZK leaf
Self::record_transfer(
    T::AssetId::default(),
    &from_account,
    &to_account,
    *exit_balance,
);
```

The comment immediately above that `increase_balance()` explains why the mint must be event-free: the runtime `WormholeProofRecorderExtension` records a leaf for every `Balances::Minted` / `Transfer` event it scans. Using `mint_into()` here would double-record the exit. So the pallet takes on the duty of calling `record_transfer()` itself. User exits honour that duty.

After the exits, the pallet computes `volume_fee_for_exit()` at the configured 4 bps (quantized ceiling, matching the circuit), burns `VolumeFeesBurnRate` (50% on the live runtime), and credits the remainder to the block author:

```rust
// Mint miner's portion of volume fee to block author
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
            Precision::Exact,
        ) {
            Ok(_) => {
                // @audit fee is on the balances pallet, but no record_transfer()
                // @audit so no ZK leaf, no TransferCount bump, no NativeTransferred
                Self::deposit_event(Event::MinerVolumeFeePaid {
                    miner: author,
                    amount: miner_fee,
                });
            },
            Err(e) => { /* burn instead of reverting the bundle */ }
        }
    }
}
```

`increase_balance()` with `Precision::Exact` writes `author.free_balance += miner_fee`. It does **not** emit `Minted`/`Transfer`, so the scanner cannot repair the miss. It does **not** bump `total_issuance`, so this is not an inflation bug. The only event is `MinerVolumeFeePaid`, which is not a leaf.

The author is not an address the miner controls with a Dilithium key. QPoW puts a 32-byte preimage in the pre-runtime digest; author extraction is:

```rust
pub fn extract_author_from_digest<'a, AccountId, I>(digest: I) -> Option<AccountId> {
    // ...
    let preimage: [u8; 32] = data.as_slice().try_into().ok()?;
    let address_bytes = derive_wormhole_address(preimage).ok()?; // @audit Poseidon rehash
    return AccountId::decode(&mut &address_bytes[..]).ok();
}
```

`derive_wormhole_address()` is `rehash_to_bytes(&inner_digest)` — the same unspendable-account derivation used for wormhole deposits. Production `Balances::transfer` requires a Dilithium signature whose public-key hash equals the account id. No such key exists for a Poseidon(preimage) account. The pallet mock does **not** check Dilithium signatures, so a `RawOrigin::Signed(miner)` transfer in a unit test would falsely succeed and must not be used as a freeze proof.

The remaining spend path is a wormhole exit. The circuit commits to the leaf amount, not to `AccountData.free_balance`. After a normal block, mining-rewards has already inserted a leaf for the **block reward only**:

```rust
match T::Currency::mint_into(miner, reward) {
    Ok(_) => {
        T::ProofRecorder::record_transfer_proof(
            None, // Native token
            mint_account,
            miner.clone(),
            reward, // @audit leaf amount = reward, not reward + later volume fees
        );
```

A later volume-fee credit raises `free_balance` but does not insert a second leaf and does not change the reward leaf's `amount`. A prover can exit the reward. They cannot exit `reward + fee` or `fee` alone, because those amounts are not in the tree. Transferring *to* the wormhole author creates a **new** leaf for the deposit, not a leaf that covers the already-credited fee.

The existing pallet test `test_verify_private_batch_emits_miner_volume_fee_paid` only asserts `MinerVolumeFeePaid` and the balance delta. It never checks `TransferCount` or the ZK tree, which is why this gap survived.

Live runtime parameters (mirrored in the pallet mock and in the committed `private_batch.hex` fixture):

* `VolumeFeeRateBps = 4`
* `VolumeFeesBurnRate = 50%` → miner share is half the volume fee (about 2 bps of exited value, with the circuit's one-quantum floor)
* `EXISTENTIAL_DEPOSIT = MILLI_UNIT` (0.001 QUAN). A failed below-ED mint burns the fee instead of freezing it. After any mining-reward credit (the normal case), and for any fee at or above the 0.005 QUAN miner floor, `increase_balance` succeeds and the freeze applies.

### Root Cause

In [`pallets/wormhole/src/lib.rs`](https://github.com/immunefi-team/audit-comp-quantus-chain/blob/3b243b870a5442a6e5f443056f67bbba87d472f5/pallets/wormhole/src/lib.rs) `process_exit_bundle()`, user exits are credited with `increase_balance()` **and then** `record_transfer()`. The miner volume-fee branch uses the same `increase_balance()` (so `WormholeProofRecorderExtension` will not see a `Balances::Minted` event) but **never** calls `record_transfer()` / `record_transfer_proof()`.

That is a mistake because the QPoW author is not a Dilithium account. [`extract_author_from_digest()`](https://github.com/immunefi-team/audit-comp-quantus-chain/blob/3b243b870a5442a6e5f443056f67bbba87d472f5/primitives/wormhole/src/lib.rs) always sets the author to `derive_wormhole_address(preimage)` — a Poseidon wormhole address with no ML-DSA key. The only spend path for that account is a wormhole exit of a ZK leaf `(recipient, transfer_count, asset, amount)`. Without a leaf, the credited fee cannot be exited, transferred, or otherwise claimed.

The mining-rewards pallet already does this correctly: `mint_into(miner, reward)` plus `record_transfer_proof(None, mint_account, miner, reward)`. The volume-fee path is the inconsistent sibling.

This finding is **only** the miner volume-fee path. The public-batch aggregator rebate also uses `increase_balance()` without a leaf, but `aggregator_address` is chosen by the aggregator and can be a spendable Dilithium account. Do not fold that into this report.

### Relationship to Previous Audits

The relevant prior report is the [Eiger audit](https://github.com/immunefi-team/audit-comp-quantus-chain/blob/3b243b870a5442a6e5f443056f67bbba87d472f5/audits/EigerWormholeAudit.pdf).

This finding is distinct from its `EQ-QNT-WORMHOLE-F-03` and `EQ-QNT-WORMHOLE-F-05` findings.

`F-03` covered incorrect author extraction: the pallet decoded the QPoW preimage directly, so the miner fee went to the wrong address or was effectively burned. Its fix derives the correct Poseidon wormhole author.

This report starts after that fix succeeds. The new failure occurs after the correct author receives an event-free balance credit: the branch omits the explicit `record_transfer()` needed to create a spendable leaf.

`F-05` covered ordinary transfer call variants missed by the old call-pattern recorder. Its fix scans emitted `Balances::Transfer` and `Balances::Minted` events and records leaves from those events.

The miner-fee credit is not one of those transfer variants. `Unbalanced::increase_balance()` emits neither scanned event, so it bypasses the event-scanner fix and requires an explicit recorder call.

The affected code path and root cause are therefore different. Under the competition's Code Update Assurance, this is a new bypass/omission, not the unpatched vulnerability disclosed in `F-03` or `F-05`.

### Vulnerability Path

{% stepper %}
{% step %}

## A QPoW miner wins a block

The header digest contains `PreRuntime(POW_ENGINE_ID, preimage)`. The runtime author is `derive_wormhole_address(preimage)`.
{% endstep %}

{% step %}

## Mining rewards create a spendable reward leaf

`pallet-mining-rewards` on finalize mints the block reward with `mint_into` and `record_transfer_proof`. The miner now has a spendable ZK leaf for the reward amount and `TransferCount = 1`.
{% endstep %}

{% step %}

## A valid batch exit is submitted

A user (or aggregator) submits `verify_private_batch` / `verify_public_batch` with a valid proof. Pool admission (`validate_unsigned`) runs cheap checks; `pre_dispatch` runs full ZK verification; dispatch runs `process_exit_bundle()`.
{% endstep %}

{% step %}

## User exits are recorded

User exits are minted and each is recorded as a ZK leaf (`NativeTransferred`).
{% endstep %}

{% step %}

## The miner fee is credited without a leaf

`process_exit_bundle()` computes the volume fee, burns 50%, and `increase_balance`s the other 50% onto the wormhole author. It emits `MinerVolumeFeePaid` and returns. No leaf is inserted. Miner `TransferCount` is unchanged.
{% endstep %}

{% step %}

## The balance and leaf diverge

The miner’s `free_balance` is now `reward + fee`. The only miner-owned leaf still commits `amount = reward`.
{% endstep %}

{% step %}

## The fee cannot be claimed

The fee cannot be moved with a Dilithium `balances.transfer` (no key). It cannot be wormhole-exited (no leaf for that amount). Subsequent mining-reward leaves commit later rewards only. The fee remains on the account forever, until a runtime upgrade inserts the missing leaves or moves the balances.
{% endstep %}
{% endstepper %}

## Impact Details

The affected party is every QPoW miner (the protocol’s block authors). They cannot claim the wormhole volume-fee share the runtime already credited to them.

Selected in-scope impact: **High — Permanent freezing of unclaimed yield.**

Why this impact, and not a Critical one:

* The frozen funds are miner protocol fees (unclaimed yield), not user funds at rest or in motion. Critical “direct theft of any user funds, other than unclaimed yield” does not apply, and nothing is stolen.
* A runtime upgrade can insert the missing leaves or move the stranded balances. Critical “permanent freezing of funds (fix requires hardfork)” does not apply.
* `increase_balance()` does not increase `total_issuance`. This is not an inflation / unauthorized-minting bug.

Loss size:

* Per successful exit bundle: `miner_fee = volume_fee - floor(volume_fee * burn_rate)`, with `volume_fee = ceil(exit_quanta * 4 / 9996) * 10^10` planck.
* On the committed fixture (19.98 QUAN exited) the PoC freezes **5\_000\_000\_000 planck = 0.005 QUAN** (half of the one-quantum minimum fee). Larger exits pay \~2 bps of exited value to the miner, all of which freeze the same way.
* The loss is continuous for the life of the chain: every private-batch and public-batch exit that pays a miner share adds another frozen credit. There is no user action that recovers it.
* USD value is not computed here: the competition snapshot does not pin a QUAN market price, and inventing one would misrepresent funds at risk. The frozen amount in native units is exact in the PoC log.

This is not the aggregator rebate, and it is not the mining-reward path.

## References

* Miner fee mint without a leaf: [`pallets/wormhole/src/lib.rs` (process\_exit\_bundle miner branch)](https://github.com/immunefi-team/audit-comp-quantus-chain/blob/3b243b870a5442a6e5f443056f67bbba87d472f5/pallets/wormhole/src/lib.rs#L942-L974)
* User exit mint **with** a leaf (the pattern the miner branch omitted): [same file, `record_transfer` after `increase_balance`](https://github.com/immunefi-team/audit-comp-quantus-chain/blob/3b243b870a5442a6e5f443056f67bbba87d472f5/pallets/wormhole/src/lib.rs#L822-L859)
* QPoW author = Poseidon wormhole address: [`primitives/wormhole/src/lib.rs` `extract_author_from_digest`](https://github.com/immunefi-team/audit-comp-quantus-chain/blob/3b243b870a5442a6e5f443056f67bbba87d472f5/primitives/wormhole/src/lib.rs#L109-L151)
* Mining rewards **do** record a leaf: [`pallets/mining-rewards/src/lib.rs` `mint_reward`](https://github.com/immunefi-team/audit-comp-quantus-chain/blob/3b243b870a5442a6e5f443056f67bbba87d472f5/pallets/mining-rewards/src/lib.rs#L220-L237)
* Scanner never sees `increase_balance`: [`runtime/src/transaction_extensions.rs` `WormholeProofRecorderExtension`](https://github.com/immunefi-team/audit-comp-quantus-chain/blob/3b243b870a5442a6e5f443056f67bbba87d472f5/runtime/src/transaction_extensions.rs#L84-L114)
* Live fee parameters: [`runtime/src/configs/mod.rs`](https://github.com/immunefi-team/audit-comp-quantus-chain/blob/3b243b870a5442a6e5f443056f67bbba87d472f5/runtime/src/configs/mod.rs#L728-L738) (`VolumeFeeRateBps = 4`, `VolumeFeesBurnRate = 50%`)
* Incomplete existing coverage: `pallets/wormhole/src/tests.rs` `test_verify_private_batch_emits_miner_volume_fee_paid` (balance + event only)

## Proof of Concept

This is Substrate / FRAME, not an EVM contract. There is no mainnet fork in Hardhat/Foundry. The program’s own build instructions are `cargo test --release` in the chain repo, and the pallet already ships a committed real private-batch proof (`pallets/wormhole/test-data/private_batch.hex`, generated with `quantus wormhole multi round`). The PoC is a full settlement-path test named `poc_miner_volume_fee_credited_without_zk_leaf` that:

* uses that real ZK proof (not a mocked verifier);
* seeds the QPoW pre-runtime digest the same way live authorship does;
* runs `validate_unsigned` → `pre_dispatch` (full ZK verify) → `verify_private_batch`;
* credits a mining-rewards-style leaf first as a positive control;
* then shows the volume fee raising `free_balance` with **no** new miner leaf;
* prints the frozen amount;
* as a negative control, calls the missing `record_transfer_proof()` and shows the fee leaf appear.

It does **not** call `Balances::transfer` from `RawOrigin::Signed(miner)`. That origin is not Dilithium-checked in the pallet mock and would be a false spend.

Nothing in this PoC talks to mainnet or a public testnet.

### Reproduce

From the `audit-comp-quantus-chain` repo root (this workspace: `codebases/chain`), on commit `3b243b870a5442a6e5f443056f67bbba87d472f5` plus the PoC test added below:

```bash
cargo test --release -p pallet-wormhole poc_miner_volume_fee -- --nocapture
```

Requirements: Rust toolchain able to build this workspace (see the chain README), no extra env vars, no network. The first `--release` build of `pallet-wormhole` regenerates ZK verifier artifacts via `build.rs` (\~20s) and then compiles. Subsequent runs reuse the artifact.

### What was added

File: `pallets/wormhole/src/tests.rs`, module `private_batch_proof_tests`.

Import changes at the top of that module (helpers `get_test_proof_bytes()`, `deserialize_test_proof()`, `setup_valid_block_state_for_test_proof()`, `ceil_volume_fee()`, `set_miner_preimage_digest()` already exist):

```rust
use frame_support::{
    assert_noop, assert_ok,
    traits::fungible::{Inspect, Mutate},
};
use qp_wormhole::TransferProofRecorder;
```

Then add the test function below (immediately after `test_verify_private_batch_cannot_replay`, before `regenerate_test_fixture`). No other production code is modified.

### Full PoC source

```rust
	/// End-to-end PoC: miner volume fees are credited to a non-ML-DSA-signable wormhole author
	/// with no ZK-tree leaf, so the fee can never be spent or exited.
	///
	/// Production path exercised:
	///   1. QPoW author = `derive_wormhole_address(preimage)` (no Dilithium key)
	///   2. Mining-rewards credits a *spendable* block reward (`mint_into` + leaf)
	///   3. A real private-batch proof is admitted, ZK-verified, and settled
	///   4. `process_exit_bundle` mints the miner's 50% volume-fee share via
	///      `increase_balance` and **never** calls `record_transfer`
	///
	/// Freeze proof (not a mock `Signed` transfer — pallet tests do not check
	/// Dilithium signatures):
	///   - author bytes == Poseidon wormhole address of the QPoW preimage
	///   - free balance increases by the miner fee
	///   - miner `TransferCount` and miner-owned leaves stay at the reward leaf only
	///   - user exits *do* get `NativeTransferred` + ZK leaves
	///   - calling the missing `record_transfer_proof` *would* insert the fee leaf
	#[test]
	fn poc_miner_volume_fee_credited_without_zk_leaf() {
		use sp_runtime::{traits::ValidateUnsigned, transaction_validity::TransactionSource};

		new_test_ext().execute_with(|| {
			println!();
			println!("==============================================================");
			println!(" PoC: miner volume fee credited with no ZK leaf (frozen yield)");
			println!("==============================================================");

			// ------------------------------------------------------------------
			// Step 0 — Load the committed private-batch fixture (real ZK proof).
			// ------------------------------------------------------------------
			println!("\n[Step 0] Load committed private-batch proof fixture");
			let proof_bytes = get_test_proof_bytes();
			let proof = deserialize_test_proof();
			let inputs = parse_private_batch_public_inputs(&proof).expect("parse public inputs");
			println!("  proof_bytes            = {} bytes", proof_bytes.len());
			println!("  asset_id               = {}", inputs.asset_id);
			println!("  volume_fee_bps         = {}", inputs.volume_fee_bps);
			println!("  proof_block_number     = {}", inputs.block_data.block_number);
			println!("  nullifiers             = {}", inputs.nullifiers.len());
			println!("  account_slots          = {}", inputs.account_data.len());
			assert_eq!(inputs.asset_id, 0, "fixture must be native");
			assert_eq!(
				inputs.volume_fee_bps,
				VolumeFeeRateBps::get(),
				"fixture fee bps must match runtime/mock (4)"
			);

			// ------------------------------------------------------------------
			// Step 1 — Wire chain state so the proof's cheap bundle checks pass.
			// ------------------------------------------------------------------
			println!("\n[Step 1] Insert matching BlockHash and advance the chain");
			setup_valid_block_state_for_test_proof();
			println!("  current block          = {}", System::block_number());

			// ------------------------------------------------------------------
			// Step 2 — Seed the QPoW pre-runtime digest (same path as live QPoW).
			// ------------------------------------------------------------------
			println!("\n[Step 2] Seed QPoW pre-runtime digest and derive miner author");
			let miner_preimage = [7u8; 32];
			set_miner_preimage_digest(miner_preimage);
			let miner = sp_core::crypto::AccountId32::from(
				qp_wormhole::derive_wormhole_address(miner_preimage)
					.expect("test preimage limbs are canonical"),
			);
			let extracted = qp_wormhole::extract_author_from_digest::<AccountId, _>(
				frame_system::Pallet::<Test>::digest().logs.iter(),
			)
			.expect("digest must yield a block author");
			println!("  miner_preimage         = 0x{}", hex::encode(miner_preimage));
			println!(
				"  derived_wormhole_addr  = 0x{}",
				hex::encode(AsRef::<[u8]>::as_ref(&miner))
			);
			println!(
				"  extract_author_digest  = 0x{}",
				hex::encode(AsRef::<[u8]>::as_ref(&extracted))
			);
			assert_eq!(
				extracted, miner,
				"live QPoW author MUST be the Poseidon wormhole address of the preimage"
			);
			println!("  [ok] author is a non-ML-DSA-signable wormhole address");

			// ------------------------------------------------------------------
			// Step 3 — Independent fee math from public inputs (not pallet internals).
			// Runtime: VolumeFeeRateBps = 4, VolumeFeesBurnRate = 50%.
			// miner_share = total_fee - total_fee/2  ≈ 2 bps of exited value.
			// ------------------------------------------------------------------
			println!("\n[Step 3] Recompute volume fee from public inputs (independent of pallet)");
			let exit_quanta: u128 = inputs
				.account_data
				.iter()
				.filter(|a| a.summed_output_amount > 0)
				.map(|a| a.summed_output_amount as u128)
				.sum();
			let exit_planck = exit_quanta * crate::SCALE_DOWN_FACTOR;
			let fee_bps = VolumeFeeRateBps::get() as u128;
			let total_fee = super::ceil_volume_fee(exit_quanta, fee_bps);
			let burn_amount = total_fee / 2;
			let expected_miner_fee = total_fee - burn_amount;
			println!("  exit_quanta            = {}", exit_quanta);
			println!(
				"  exit_amount            = {} planck ({} QUAN)",
				exit_planck,
				exit_planck as f64 / UNIT as f64
			);
			println!("  fee_bps                = {}", fee_bps);
			println!("  total_volume_fee       = {} planck", total_fee);
			println!("  burn_share (50%)       = {} planck", burn_amount);
			println!(
				"  miner_share (50%)      = {} planck ({} QUAN)",
				expected_miner_fee,
				expected_miner_fee as f64 / UNIT as f64
			);
			assert!(expected_miner_fee > 0, "fixture must produce a non-zero miner fee");

			// ------------------------------------------------------------------
			// Step 4 — Simulate the prior mining-rewards credit (the CORRECT path).
			// `pallet-mining-rewards` does mint_into + record_transfer_proof so the
			// block reward is a real ZK leaf the miner can later exit.
			// ------------------------------------------------------------------
			println!("\n[Step 4] Credit a mining-rewards block reward WITH a ZK leaf (control)");
			let block_reward: Balance = UNIT; // 1 QUAN
			assert_ok!(Balances::mint_into(&miner, block_reward));
			let recorded = <Wormhole as TransferProofRecorder<AccountId, u32, Balance>>::record_transfer_proof(
				None, // native
				MINTING_ACCOUNT,
				miner.clone(),
				block_reward,
			);
			assert!(recorded, "mining-rewards path must insert a leaf");
			let reward_count = Wormhole::transfer_count(&miner);
			let leaves_after_reward = ZkTree::leaf_count();
			let miner_balance_after_reward = Balances::balance(&miner);
			println!("  block_reward           = {} planck (1 QUAN)", block_reward);
			println!("  miner.free_balance     = {}", miner_balance_after_reward);
			println!("  miner.TransferCount    = {}", reward_count);
			println!("  ZkTree.leaf_count      = {}", leaves_after_reward);
			assert_eq!(reward_count, 1, "reward must advance miner TransferCount to 1");
			assert_eq!(leaves_after_reward, 1, "reward must insert exactly one leaf");
			let reward_leaf = ZkTree::leaf(0).expect("reward leaf");
			assert_eq!(reward_leaf.to, miner);
			assert_eq!(reward_leaf.amount, block_reward);
			assert_eq!(reward_leaf.transfer_count, 0);
			println!(
				"  leaf[0]                = to=miner amount={} transfer_count={}",
				reward_leaf.amount, reward_leaf.transfer_count
			);
			println!("  [ok] mining-rewards path: balance + spendable ZK leaf");

			// ------------------------------------------------------------------
			// Step 5 — Full unsigned inclusion path: pool admit → pre_dispatch ZK
			// verify → dispatch settlement. This is how a live block includes an exit.
			// ------------------------------------------------------------------
			println!("\n[Step 5] Admit, ZK-verify, and settle the private-batch exit");
			let call = crate::Call::<Test>::verify_private_batch { proof_bytes: proof_bytes.clone() };
			let validity = <Wormhole as ValidateUnsigned>::validate_unsigned(
				TransactionSource::External,
				&call,
			)
			.expect("fixture must be admitted to the pool");
			println!("  validate_unsigned      = Ok (priority={})", validity.priority);
			assert_ok!(<Wormhole as ValidateUnsigned>::pre_dispatch(&call));
			println!("  pre_dispatch           = Ok (full ZK verification passed)");
			assert_ok!(Wormhole::verify_private_batch(RawOrigin::None.into(), proof_bytes));
			println!("  verify_private_batch   = Ok (settlement dispatched)");

			// ------------------------------------------------------------------
			// Step 6 — Miner free balance increased by the volume fee.
			// ------------------------------------------------------------------
			println!("\n[Step 6] Miner free-balance credit (the yield that should be claimable)");
			let miner_balance_after_exit = Balances::balance(&miner);
			let frozen_fee = miner_balance_after_exit.saturating_sub(miner_balance_after_reward);
			println!("  balance before exit    = {} planck", miner_balance_after_reward);
			println!("  balance after exit     = {} planck", miner_balance_after_exit);
			println!("  credited miner fee     = {} planck", frozen_fee);
			assert_eq!(frozen_fee, expected_miner_fee, "credited amount must equal miner share");
			System::assert_has_event(
				crate::Event::<Test>::MinerVolumeFeePaid {
					miner: miner.clone(),
					amount: expected_miner_fee,
				}
				.into(),
			);
			println!("  event                  = MinerVolumeFeePaid(miner, {})", expected_miner_fee);

			// ------------------------------------------------------------------
			// Step 7 — User exits DID get NativeTransferred + ZK leaves.
			// The miner fee DID NOT.
			// ------------------------------------------------------------------
			println!("\n[Step 7] Compare user-exit leaves vs miner-fee (missing) leaf");
			let mut user_exit_events = 0u32;
			let mut miner_fee_native_events = 0u32;
			let mut user_exit_total: u128 = 0;
			for record in System::events() {
				if let RuntimeEvent::Wormhole(crate::Event::NativeTransferred {
					to,
					amount,
					leaf_index,
					transfer_count,
					..
				}) = record.event
				{
					if to == miner && amount == expected_miner_fee {
						miner_fee_native_events += 1;
						println!(
							"  UNEXPECTED miner-fee NativeTransferred leaf_index={} amount={}",
							leaf_index, amount
						);
					} else if to != miner {
						user_exit_events += 1;
						user_exit_total += amount;
						println!(
							"  user NativeTransferred  to=0x{} amount={} count={} leaf={}",
							hex::encode(AsRef::<[u8]>::as_ref(&to)),
							amount,
							transfer_count,
							leaf_index
						);
					}
				}
			}
			assert!(user_exit_events > 0, "fixture must mint at least one user exit");
			assert_eq!(
				miner_fee_native_events, 0,
				"miner fee must NOT emit NativeTransferred (that event is the leaf recorder)"
			);
			println!("  user_exit_events       = {}", user_exit_events);
			println!("  user_exit_total        = {} planck", user_exit_total);
			println!("  miner_fee NativeTransferred events = {}", miner_fee_native_events);

			let miner_count_after = Wormhole::transfer_count(&miner);
			let leaves_after_exit = ZkTree::leaf_count();
			println!("  miner.TransferCount    = {} (was {})", miner_count_after, reward_count);
			println!(
				"  ZkTree.leaf_count      = {} (was {}; delta={})",
				leaves_after_exit,
				leaves_after_reward,
				leaves_after_exit - leaves_after_reward
			);
			assert_eq!(
				miner_count_after, reward_count,
				"miner TransferCount must stay at the reward-only count — fee inserted no leaf"
			);
			assert_eq!(
				leaves_after_exit - leaves_after_reward,
				user_exit_events as u64,
				"new leaves must equal user-exit NativeTransferred events only"
			);

			let mut miner_leaves: Vec<(u64, u64, u128)> = Vec::new();
			let mut miner_leaf_covering_fee = false;
			for i in 0..leaves_after_exit {
				let leaf = ZkTree::leaf(i).expect("leaf exists");
				if leaf.to == miner {
					miner_leaves.push((i, leaf.transfer_count, leaf.amount));
					if leaf.amount == expected_miner_fee {
						miner_leaf_covering_fee = true;
					}
				}
			}
			println!("  miner-owned leaves     = {:?}", miner_leaves);
			assert_eq!(
				miner_leaves.len(),
				1,
				"miner must still own only the mining-reward leaf"
			);
			assert_eq!(miner_leaves[0].2, block_reward);
			assert!(
				!miner_leaf_covering_fee,
				"no ZK leaf may commit the miner volume fee"
			);
			println!("  [ok] no leaf (miner, amount=fee) exists in the ZK tree");

			// ------------------------------------------------------------------
			// Step 8 — Why the credit is permanently frozen (without using mock
			// Signed transfers, which skip Dilithium verification in this harness).
			//
			// Spend path A — Balances::transfer: production origin is a Dilithium
			// signature over hash(pk) == account. This account is Poseidon(preimage),
			// so no ML-DSA key exists. The pallet mock does NOT check that, so a
			// mock Signed(miner) transfer would falsely succeed and must not be used.
			//
			// Spend path B — wormhole exit: the circuit binds (recipient,
			// transfer_count, asset, amount) to an existing leaf. The only miner
			// leaf commits `block_reward`, not `block_reward + fee`. A later
			// mining-reward leaf would commit that later reward only. Transferring
			// *to* the miner creates a *new* leaf for the deposit, not one that
			// covers the already-credited fee.
			// ------------------------------------------------------------------
			println!("\n[Step 8] Freeze conditions");
			println!("  spendable_via_dilithium = false (author = Poseidon(preimage), no pk)");
			println!(
				"  spendable_via_wormhole  = false (no leaf amount={} for miner)",
				expected_miner_fee
			);
			println!(
				"  FUNDS FROZEN            = {} planck ({} QUAN)",
				frozen_fee,
				frozen_fee as f64 / UNIT as f64
			);
			println!(
				"  miner can exit at most  = {} planck (the mining-reward leaf only)",
				block_reward
			);
			println!(
				"  stuck remainder         = {} planck  <-- PERMANENTLY FROZEN YIELD",
				frozen_fee
			);

			// ------------------------------------------------------------------
			// Step 9 — Negative control: the one missing call would have recorded
			// the fee as a spendable leaf (exactly what mining-rewards does).
			// ------------------------------------------------------------------
			println!("\n[Step 9] Negative control — insert the missing record_transfer_proof");
			let would_record =
				<Wormhole as TransferProofRecorder<AccountId, u32, Balance>>::record_transfer_proof(
					None,
					MINTING_ACCOUNT,
					miner.clone(),
					expected_miner_fee,
				);
			assert!(would_record, "the missing call would have recorded the fee leaf");
			assert_eq!(Wormhole::transfer_count(&miner), reward_count + 1);
			assert_eq!(ZkTree::leaf_count(), leaves_after_exit + 1);
			let fee_leaf = ZkTree::leaf(leaves_after_exit).expect("control leaf");
			assert_eq!(fee_leaf.to, miner);
			assert_eq!(fee_leaf.amount, expected_miner_fee);
			println!(
				"  after missing call      TransferCount={} leaf_count={}",
				Wormhole::transfer_count(&miner),
				ZkTree::leaf_count()
			);
			println!(
				"  new leaf[{}]            = to=miner amount={} transfer_count={}",
				leaves_after_exit, fee_leaf.amount, fee_leaf.transfer_count
			);
			println!("  [ok] one record_transfer_proof call is the entire fix");

			println!("\n==============================================================");
			println!(
				" RESULT: {} planck miner volume fee is frozen on a non-ML-DSA-signable",
				frozen_fee
			);
			println!("         wormhole author with no ZK leaf covering that amount.");
			println!("         User exits were recorded. Mining rewards were recorded.");
			println!("         Only the miner volume-fee path omits the leaf.");
			println!("==============================================================\n");
		});
	}
```

### Run command

```bash
cd codebases/chain   # audit-comp-quantus-chain repo root
cargo test --release -p pallet-wormhole poc_miner_volume_fee -- --nocapture
```

### Full log

```
   Compiling pallet-wormhole v0.1.0 (/Users/popeye/Desktop/immunefi/AuditComp_Quants/codebases/chain/pallets/wormhole)
    Finished `release` profile [optimized] target(s) in 7.50s
     Running unittests src/lib.rs (target/release/deps/pallet_wormhole-14686b5a76865a50)

running 1 test

==============================================================
 PoC: miner volume fee credited with no ZK leaf (frozen yield)
==============================================================

[Step 0] Load committed private-batch proof fixture
  proof_bytes            = 157500 bytes
  asset_id               = 0
  volume_fee_bps         = 4
  proof_block_number     = 1
  nullifiers             = 7
  account_slots          = 14

[Step 1] Insert matching BlockHash and advance the chain
  current block          = 11

[Step 2] Seed QPoW pre-runtime digest and derive miner author
  miner_preimage         = 0x0707070707070707070707070707070707070707070707070707070707070707
  derived_wormhole_addr  = 0xd2027e5ab7dda597109a170d67b9c932746ff615b99a22ef30c13f1f2c58651b
  extract_author_digest  = 0xd2027e5ab7dda597109a170d67b9c932746ff615b99a22ef30c13f1f2c58651b
  [ok] author is a non-ML-DSA-signable wormhole address

[Step 3] Recompute volume fee from public inputs (independent of pallet)
  exit_quanta            = 1998
  exit_amount            = 19980000000000 planck (19.98 QUAN)
  fee_bps                = 4
  total_volume_fee       = 10000000000 planck
  burn_share (50%)       = 5000000000 planck
  miner_share (50%)      = 5000000000 planck (0.005 QUAN)

[Step 4] Credit a mining-rewards block reward WITH a ZK leaf (control)
  block_reward           = 1000000000000 planck (1 QUAN)
  miner.free_balance     = 1000000000000
  miner.TransferCount    = 1
  ZkTree.leaf_count      = 1
  leaf[0]                = to=miner amount=1000000000000 transfer_count=0
  [ok] mining-rewards path: balance + spendable ZK leaf

[Step 5] Admit, ZK-verify, and settle the private-batch exit
  validate_unsigned      = Ok (priority=1)
  pre_dispatch           = Ok (full ZK verification passed)
  verify_private_batch   = Ok (settlement dispatched)

[Step 6] Miner free-balance credit (the yield that should be claimable)
  balance before exit    = 1000000000000 planck
  balance after exit     = 1005000000000 planck
  credited miner fee     = 5000000000 planck
  event                  = MinerVolumeFeePaid(miner, 5000000000)

[Step 7] Compare user-exit leaves vs miner-fee (missing) leaf
  user NativeTransferred  to=0x0404040404040404040404040404040404040404040404040404040404040404 amount=19980000000000 count=0 leaf=1
  user_exit_events       = 1
  user_exit_total        = 19980000000000 planck
  miner_fee NativeTransferred events = 0
  miner.TransferCount    = 1 (was 1)
  ZkTree.leaf_count      = 2 (was 1; delta=1)
  miner-owned leaves     = [(0, 0, 1000000000000)]
  [ok] no leaf (miner, amount=fee) exists in the ZK tree

[Step 8] Freeze conditions
  spendable_via_dilithium = false (author = Poseidon(preimage), no pk)
  spendable_via_wormhole  = false (no leaf amount=5000000000 for miner)
  FUNDS FROZEN            = 5000000000 planck (0.005 QUAN)
  miner can exit at most  = 1000000000000 planck (the mining-reward leaf only)
  stuck remainder         = 5000000000 planck  <-- PERMANENTLY FROZEN YIELD

[Step 9] Negative control — insert the missing record_transfer_proof
  after missing call      TransferCount=2 leaf_count=3
  new leaf[2]            = to=miner amount=5000000000 transfer_count=1
  [ok] one record_transfer_proof call is the entire fix

==============================================================
 RESULT: 5000000000 planck miner volume fee is frozen on a non-ML-DSA-signable
         wormhole author with no ZK leaf covering that amount.
         User exits were recorded. Mining rewards were recorded.
         Only the miner volume-fee path omits the leaf.
==============================================================

test tests::private_batch_proof_tests::poc_miner_volume_fee_credited_without_zk_leaf ... ok

test result: ok. 1 passed; 0 failed; 0 ignored; 0 measured; 71 filtered out; finished in 0.02s
```

What this log proves, in order:

{% stepper %}
{% step %}

## The author is a wormhole address

The author bytes equal `derive_wormhole_address([7u8; 32])` — a wormhole address, not `hash(dilithium_pk)`.
{% endstep %}

{% step %}

## The mining reward has a matching leaf

After a mining-reward credit, the miner has 1 QUAN **and** leaf `[0] = (miner, 1 QUAN)`.
{% endstep %}

{% step %}

## The batch is verified and user exits are recorded

The real private-batch proof is ZK-verified and settled. The user exit of 19.98 QUAN gets `NativeTransferred` and leaf 1.
{% endstep %}

{% step %}

## The miner fee is credited

Miner `free_balance` rises by 5\_000\_000\_000 planck and `MinerVolumeFeePaid` fires.
{% endstep %}

{% step %}

## No leaf exists for the miner fee

Miner `TransferCount` stays 1. The only miner-owned leaf is still the reward. There is no `NativeTransferred` for the fee.
{% endstep %}

{% step %}

## The missing call resolves the issue

Calling the missing `record_transfer_proof(None, minting, miner, 5_000_000_000)` inserts leaf 2 for that exact amount. That is the whole fix.
{% endstep %}
{% endstepper %}
