# 89552 bc high qpow miners will permanently freeze the sub quantum remainder of every block reward because mint reward credits full planck while hash leaf only commits floor amount 10 1&#x20;

**Submitted on Aug 19th 2026 at 20:29:40 UTC by @n3310y for** [**Audit Comp | Quantus**](https://immunefi.com/audit-competition/audit-comp-quantus)

* **Report ID:** #89552
* **Report Type:** Blockchain/DLT
* **Report severity:** High
* **Target:** <https://github.com/immunefi-team/audit-comp-quantus-chain>
* **Impacts:**
  * Permanent freezing of unclaimed yield

## Description

## Brief/Intro

Minting the miner share in full Planck units and then hashing the leaf with integer division by `10^10` will cause permanent freezing of unclaimed yield for every QPoW miner as honest block production credits a Poseidon (non-ML-DSA) author with a remainder that is present in `Balances` and in leaf storage but absent from the Merkle commitment the wormhole circuit is bound to. On the production runtime the first block of the PoC mints `692_383_582_714` planck (`0.692383582714 QUAN`), commits only 69 quanta (`0.69 QUAN`), and leaves `2_383_582_714` planck (`0.002383582714 QUAN`) unexitable. That residue is above the existential deposit, so Substrate will not reap it. The miner cannot Dilithium-sign it out (`BadProof`) and cannot prove it in a wormhole exit. The freeze repeats on every subsequent block until the whole miner share itself falls below one quantum (the separately listed late-emission known issue). This is not the Eiger leftover-redemption finding: that issue is a user-chosen partial exit of an already-committed amount; this freeze is mint-side truncation before any exit.

## Vulnerability Details

### Root Cause

In `chain/pallets/mining-rewards/src/lib.rs:195-196` `on_finalize()` calls `mint_reward()` with the undivided miner share. In `lib.rs:230-237` that path `mint_into()`s `reward` and `record_transfer_proof(..., reward)`. In `chain/pallets/zk-tree/src/tree.rs:146-148` the leaf hash uses `amount / AMOUNT_SCALE_DOWN_FACTOR` (`10^10`). Miner identity is `derive_wormhole_address(preimage)` (`chain/primitives/wormhole/src/lib.rs:147-148`), which is Poseidon, not `hash_bytes(dilithium_pk)`, so there is no ML-DSA spend of the residue.

### Details

Quantus pays QPoW authors from remaining supply and then relies on the zk-wormhole to let those authors move value off a keyless account. Two independent type systems collide on that path.

Spendable Substrate accounts are ML-DSA-65 (or 87) public keys hashed with Poseidon:

```rust
// chain/primitives/dilithium-crypto/src/traits.rs
fn into_account(self) -> AccountId32 {
    match self {
        Self::Dilithium87(who) => hash_bytes(who.as_ref()).into(),
        Self::Dilithium65(who) => hash_bytes(who.as_ref()).into(),
        // @audit AccountId that can sign a Balances transfer is hash_bytes(dilithium_pk)
    }
}
```

QPoW authors are not that. The pre-runtime digest carries a 32-byte mining preimage. `MiningRewards::on_finalize()` extracts the author through `extract_author_from_digest()`, which re-hashes that preimage into a wormhole address:

```rust
// chain/primitives/wormhole/src/lib.rs
let preimage: [u8; 32] = data.as_slice().try_into().ok()?;
let address_bytes = derive_wormhole_address(preimage).ok()?;
// @audit Poseidon re-hash of the mining preimage, not hash_bytes(dilithium_pk)
return AccountId::decode(&mut &address_bytes[..]).ok();
```

That account can receive `mint_into()`, but it cannot produce a `Signature::Dilithium65` that `Verify` will accept for that `AccountId`. The intended spend path is therefore a wormhole exit: insert a ZK-tree leaf at mint time, later prove membership of that leaf and remint the committed amount to a Dilithium account.

The mint itself is computed in full 12-decimal Planck units from production constants (`MAX_SUPPLY = 21_000_000 * 10^12`, `EmissionDivisor = 15_163_560`, treasury `Permill` 50%):

```rust
// chain/pallets/mining-rewards/src/lib.rs
let total_reward = remaining_supply.checked_div(&emission_divisor).unwrap_or_else(BalanceOf::<T>::zero);
let treasury_reward = treasury_portion.mul_floor(total_reward);
let miner_reward = total_reward.saturating_sub(treasury_reward);
// @audit miner_reward is a raw Balance, not rounded to the circuit quantum

Self::mint_reward(miner.as_ref(), tx_fees);
Self::mint_reward(miner.as_ref(), miner_reward);
// @audit full miner share, not quantized
```

`mint_reward()` then credits that exact Planck amount and records a leaf with the same number:

```rust
match T::Currency::mint_into(miner, reward) {
    Ok(_) => {
        T::ProofRecorder::record_transfer_proof(
            None, // Native token
            mint_account,
            miner.clone(),
            reward, // @audit records 692_383_582_714; hash commits 69
        );
```

The Merkle commitment does not use `leaf.amount` as stored. `hash_leaf()` divides by `10^10` (two decimal places of QUAN) before hashing, matching the circuit's `u32` amount:

```rust
// chain/pallets/zk-tree/src/tree.rs
pub const AMOUNT_SCALE_DOWN_FACTOR: u128 = 10_000_000_000; // 0.01 QUAN

let amount_u128: u128 = leaf.amount.into();
let amount_quantized =
    u32::try_from(amount_u128 / AMOUNT_SCALE_DOWN_FACTOR).unwrap_or(u32::MAX);
// @audit 692_383_582_714 / 10^10 = 69; residue 2_383_582_714 is hash-invisible
```

The wormhole circuit's private `input_amount` is a `u32` quantized amount, not a Planck `u128`:

```rust
// qp-zk-circuits/wormhole/circuit/src/inputs.rs
pub input_amount: u32, // @audit 0.01 QUAN units; bound to hash_leaf's floored felt
```

Settlement remints only that quantized public output times `10^10`:

```rust
// chain/pallets/wormhole/src/lib.rs
let exit_balance_u128 = (account_data.summed_output_amount as u128)
    .saturating_mul(crate::SCALE_DOWN_FACTOR); // @audit output_amount * 10^10
```

A valid proof of the block-1 leaf can therefore only ever mint at most `69 * 10^10 = 690_000_000_000` planck. Claiming `70` quanta would require a different leaf hash, which the tree does not contain. The circuit's fee constraint (`qp-zk-circuits/wormhole/circuit/src/zk_merkle_proof.rs`) is `(output_1 + output_2) * 10000 <= input * (10000 - fee_bps)`, so outputs cannot exceed the already-truncated `input_amount`.

The residue `2_383_582_714` planck sits on the Poseidon miner in `Balances`. It is `2.38×` the existential deposit (`10^9`), so it is not dust and will not be reaped. It cannot be moved with `Balances::transfer_keep_alive`: `Executive::apply_extrinsic` with a real Dilithium key claiming the miner origin returns `InvalidTransaction::BadProof` and the miner balance is unchanged. Remainders from later blocks cannot be added together to make one extra quantum, because each `record_transfer_proof` is a distinct `(to, transfer_count)` leaf.

This is not known issue #4. That path credits the miner volume fee with `Unbalanced::increase_balance()` and **omits** `record_transfer()`. This path **does** call `record_transfer_proof`; the leaf exists; only the committed amount is truncated.

This is not known issue #3. That text is the late-emission case where mining rewards drop **well below** the quantized minimum (the whole reward floors to 0, planned circuit upgrade). Block 1 of the production runtime commits **69 quanta**, not 0. The freeze is live from genesis-scale rewards.

This is not Eiger `EQ-QNT-DOC-F-01` (“Leftover Amount Redemption Issue in QIP-5”, `chain/audits/EigerWormholeAudit.pdf`). That prior finding is a **user-controlled partial withdrawal**: the prover may set `output_amount_1 + output_amount_2 + fee < input_amount`, the nullifier is consumed, and any leftover of the **already-committed quantized amount** cannot be redeemed again. The user chose to leave it. H-1 is an **unavoidable mining-mint truncation**. `mint_reward()` credits full Planck and `hash_leaf()` floors the commitment **before any user action**. A maximal exit of the block-1 leaf (`input_amount = 69`) still cannot recover `2_383_582_714` planck, because those units were never a felt in the leaf. They cannot appear as change on `output_amount_2`. Honest QPoW production is enough; no partial-exit choice is involved.

The treasury half of the same `on_finalize` mint is **not** this bug. Treasury is a pallet account; governance can spend it. Only the Poseidon miner remainder is frozen.

### Attack Path

{% stepper %}
{% step %}

## Mine a valid QPoW block

A miner finds a valid QPoW nonce. The 32-byte preimage is placed in the block header as `DigestItem::PreRuntime(POW_ENGINE_ID, preimage)`.
{% endstep %}

{% step %}

## Compute the miner reward

`Executive::initialize_block` + `Executive::finalize_block` run. `MiningRewards::on_finalize()` computes `miner_reward = (MaxSupply - total_issuance) / EmissionDivisor` after the 50% treasury split.
{% endstep %}

{% step %}

## Derive a keyless miner account

`extract_author_from_digest()` sets the author to `derive_wormhole_address(preimage)` (Poseidon), not `hash_bytes(dilithium_pk)`.
{% endstep %}

{% step %}

## Mint and record the full Planck reward

`mint_reward()` `mint_into()`s the full Planck amount onto that Poseidon account and `record_transfer_proof(..., reward)` inserts a native leaf with `leaf.amount = reward`.
{% endstep %}

{% step %}

## Commit only the quantized amount

`hash_leaf()` commits `floor(reward / 10^10)` only. Storage still holds the full amount; the Merkle root does not.
{% endstep %}

{% step %}

## Prove only the quantized amount

The miner builds a wormhole proof of that leaf. The circuit `input_amount` can only be the quantized value (69 on block 1 of the PoC). The remainder never enters the proof and is not reminted on exit.
{% endstep %}

{% step %}

## Fail to transfer the residue directly

Any attempt to `Balances::transfer_keep_alive` the remainder with an ML-DSA key claiming the miner origin fails `Verify` with `InvalidTransaction::BadProof`. Miner balance is unchanged.
{% endstep %}

{% step %}

## Repeat on later blocks

The next block repeats the same mint against a new `transfer_count`. Remainders cannot be combined into one extra quantum.
{% endstep %}
{% endstepper %}

## Impact Details

**In-scope impact:** High — Permanent freezing of unclaimed yield.

The affected party is every QPoW miner. The protocol mints yield onto an account that cannot sign, then commits a truncated amount into the only exit path. No attacker needs to act. Honest block production is the attack path.

Measured on the production runtime (`quantus-runtime`, `MAX_SUPPLY = 21_000_000 QUAN`, `EmissionDivisor = 15_163_560`, 50/50 split, 12-second blocks):

| Quantity                                            |                Planck |               QUAN |
| --------------------------------------------------- | --------------------: | -----------------: |
| Block-1 miner share (`MinerRewarded`)               |     `692_383_582_714` |   `0.692383582714` |
| Committed by `hash_leaf` (69 quanta)                |     `690_000_000_000` |   `0.690000000000` |
| Frozen remainder (block 1)                          |       `2_383_582_714` |   `0.002383582714` |
| Remainder / reward                                  |                     — |          `0.3443%` |
| Existential deposit                                 |       `1_000_000_000` |            `0.001` |
| Frozen over 200 produced blocks (sum of remainders) |     `475_807_892_516` |   `0.475807892516` |
| Sum of those 200 miner block rewards                | `138_475_807_892_516` | `138.475807892516` |
| Remainders / rewards (200 blocks)                   |                     — |          `0.3436%` |

Lead with what was measured, not with an annual projection. The 200-block figure is a direct runtime measurement, not a unit-test constant. Miner free balance after 200 blocks is `138.486803679516 QUAN`; the extra `~0.011 QUAN` versus the reward sum is the block-2 transaction-fee mint (out of scope for this report). The frozen sum above is **only** `miner_reward % 10^10` from `MinerRewarded` events.

Immediate funds at risk, right now, are the measured block-1 remainder (`2_383_582_714` planck = `0.002383582714 QUAN`) and every following block of the same shape. The `6_264.055372392 QUAN` number later in the PoC log is **only an envelope**: `7_200 blocks/day × 365 ×` the block-1 remainder, assuming that remainder stayed constant. It is not a guaranteed annual loss. The remainder is a sawtooth (`miner_reward % 10^10`) that decays only as remaining supply decays.

The freeze continues until the **whole** miner share drops below one quantum. That happens when `0.5 * remaining / 15_163_560 < 10^10`, i.e. remaining supply `< 303_271.2 QUAN`. The PoC still had `20_997_723 QUAN` remaining after 200 blocks. Known issue #3 only covers the tail after that threshold. Almost the entire emission schedule is this bug, not the listed exception.

The miner does not steal from users. They lose their own unclaimed yield. Treasury's matching remainder is spendable and is not counted in the loss.

### Risk Breakdown

* **Severity:** High, matching Immunefi impact “Permanent freezing of unclaimed yield”.
* **Likelihood:** Certain. Every block whose miner share is not an exact multiple of `10^10` (the typical case) freezes the residue. No special transaction, no victim interaction, no privileged role.
* **Complexity:** None. Happens in honest operation on the production runtime.
* **Privileges required:** None beyond mining a valid QPoW block, which is the normal authoring path.
* **Impact window:** Immediate, and repeating every 12-second block from genesis-scale rewards until the whole miner share itself is `< 1` quantum.

## References

* `chain/pallets/mining-rewards/src/lib.rs` — `on_finalize()`, `extract_miner_from_digest()`, `mint_reward()`
* `chain/pallets/zk-tree/src/tree.rs` — `AMOUNT_SCALE_DOWN_FACTOR`, `hash_leaf()`
* `chain/primitives/wormhole/src/lib.rs` — `derive_wormhole_address()`, `extract_author_from_digest()`
* `chain/primitives/dilithium-crypto/src/traits.rs` — `IdentifyAccount` for Dilithium (`hash_bytes(pk)`)
* `chain/pallets/wormhole/src/lib.rs` — `SCALE_DOWN_FACTOR`, exit remint is `summed_output_amount * 10^10`
* `qp-zk-circuits/wormhole/circuit/src/inputs.rs` — `PrivateCircuitInputs.input_amount: u32`
* `qp-zk-circuits/wormhole/circuit/src/zk_merkle_proof.rs` — fee constraint `output_1 + output_2 <= input` after bps
* `chain/runtime/src/lib.rs` — `UNIT`, `EXISTENTIAL_DEPOSIT`, `MAX_SUPPLY`
* `chain/runtime/src/configs/mod.rs` — `EmissionDivisor = 15_163_560`
* Program known issue #3 (whole reward well below one quantum; planned upgrade) — **not this bug**
* Program known issue #4 (volume fee `increase_balance` with no `record_transfer`) — **not this bug**
* Eiger `EQ-QNT-DOC-F-01` leftover-amount redemption in QIP-5 (`chain/audits/EigerWormholeAudit.pdf`) — **not this bug** (user-controlled partial exit of an already-committed amount)

## Proof of Concept

This is not a `hash_leaf` unit test. It is a `quantus-runtime` integration test that drives the **production** runtime through `Executive::initialize_block` / `finalize_block` (the same hooks a real QPoW block runs) and `Executive::apply_extrinsic` (the same Dilithium signature + `TxExtension` pipeline a real transfer runs). It never talks to a public network.

The test builds a synthetic `Header` with `PreRuntime(POW_ENGINE_ID, preimage)` only so `Executive` will run those production hooks. It does not simulate a QPoW nonce search. The freeze does not depend on the nonce: any accepted block whose miner share is not a multiple of `10^10` truncates the same way.

What the PoC proves, in order:

{% stepper %}
{% step %}

## Identify the QPoW author

The QPoW author is `derive_wormhole_address(preimage)`, not `hash_bytes(dilithium_pk)`.
{% endstep %}

{% step %}

## Verify the block-1 reward

Block 1’s `MinerRewarded` amount equals the live production formula `(MAX_SUPPLY - total_issuance) / 15_163_560` after the 50% treasury split: `692_383_582_714` planck.
{% endstep %}

{% step %}

## Verify the full balance mint

`Balances::free_balance(miner)` equals that full mint.
{% endstep %}

{% step %}

## Verify the leaf truncation

The ZK leaf stores the full mint, but `hash_leaf(full) == hash_leaf(mint - remainder) != hash_leaf(next quantum)`.
{% endstep %}

{% step %}

## Verify a Dilithium control spend

A Dilithium control account **can** spend through `Executive::apply_extrinsic` (`Ok(Ok(()))`).
{% endstep %}

{% step %}

## Verify the miner account cannot spend

The same Dilithium key claiming the miner origin **cannot** (`Err(Invalid(BadProof))`); miner balance unchanged.
{% endstep %}

{% step %}

## Verify repeating remainders

200 consecutive blocks produce 200 separate miner-reward leaves; remainders sum to `475_807_892_516` planck (`0.475807892516 QUAN`) and cannot merge.
{% endstep %}

{% step %}

## Distinguish from known issues

Block 1 committed 69 quanta, so this is not known issue #3. A leaf exists, so this is not known issue #4. The remainder is missing from the leaf hash before any exit, so this is not Eiger leftover redemption (user-controlled partial withdrawal).
{% endstep %}
{% endstepper %}

## 0. Step-by-step reproduction (triager)

Do not run this against a public testnet or mainnet. The PoC is an in-process runtime test.

{% stepper %}
{% step %}

## Clone the repository

Clone the in-scope asset `https://github.com/immunefi-team/audit-comp-quantus-chain` and enter the `chain/` directory. The repo pins Rust via `chain/rust-toolchain` (`channel = "1.93.0"`, target `wasm32v1-none`). No extra environment variables are required.
{% endstep %}

{% step %}

## Add the PoC test file

Add the PoC file at `chain/runtime/tests/transactions/h1_mining_reward_remainder.rs` (full source in section 2 below). Nothing else in the runtime is patched; the test only **calls** production `Executive` hooks. The synthetic header is a vehicle for those hooks, not a consensus exploit.
{% endstep %}

{% step %}

## Register the module

Register the module in `chain/runtime/tests/transactions/mod.rs` by adding this line at the top of the file, next to the existing `pub mod` entries:

```rust
pub mod h1_mining_reward_remainder;
```

{% endstep %}

{% step %}

## Run the test

From `chain/`, run the command in section 3. First compile of `pallet-wormhole` generates ZK circuit binaries (\~40s) and then compiles `quantus-runtime` tests. The test itself finishes in \~1.4s. Use `--nocapture` so the step log prints. Expected result: `test result: ok. 1 passed`.
{% endstep %}

{% step %}

## Compare the output

Compare stdout to the full log in section 4. The exact Planck figures are a function of `total_issuance()` before block 1 (the PoC endows `1000 UNIT + 1000 UNIT + ED`). They must match the formula printed in STEP 2. Hashes in STEP 4 must show `hash_leaf(full) == hash_leaf(truncated)` and a different next-quantum hash. STEP 5 must show `BadProof`.
{% endstep %}
{% endstepper %}

No Foundry/Hardhat fork is applicable; this is a Substrate L1. `Executive::initialize_block` / `finalize_block` / `apply_extrinsic` is the equivalent of a production-runtime fork test.

## 1. What was added (nothing in production logic is changed)

| Path                                                             | Change                                          |
| ---------------------------------------------------------------- | ----------------------------------------------- |
| `chain/runtime/tests/transactions/h1_mining_reward_remainder.rs` | **New file.** Full E2E PoC.                     |
| `chain/runtime/tests/transactions/mod.rs`                        | One line: `pub mod h1_mining_reward_remainder;` |

No pallet, runtime config, genesis preset, or circuit is modified. A passing test is the exploit demonstration: the production hooks already freeze the remainder.

## 2. Full PoC source

Save as `chain/runtime/tests/transactions/h1_mining_reward_remainder.rs`:

```rust
//! End-to-end PoC for H-1: QPoW mining rewards are minted in full planck to a
//! Poseidon (non-ML-DSA) miner address, but the ZK-tree leaf only commits
//! `floor(amount / 10^10)`. The residue cannot be Dilithium-signed out and
//! cannot be proven in a wormhole exit — permanent freeze of unclaimed miner yield.
//!
//! This is a `quantus-runtime` integration test. It drives the **production**
//! runtime (`MAX_SUPPLY`, `EmissionDivisor`, `UNIT`, wormhole quantum, 50/50
//! treasury split) through `Executive::initialize_block` / `finalize_block`
//! (the same hooks a real block runs) and `Executive::apply_extrinsic` (the
//! same signature + `TxExtension` pipeline a real transfer runs). It never
//! talks to a public network.
//!
//! Run from chain/:
//! cargo test -p quantus-runtime --test mod -- --nocapture --exact
//!   transactions::h1_mining_reward_remainder::tests::prove_miner_remainder_frozen

use codec::Encode;
use frame_support::{assert_ok, traits::Currency};
use pallet_zk_tree::{
	tree::{hash_leaf, AMOUNT_SCALE_DOWN_FACTOR},
	ZkLeaf,
};
use qp_dilithium_crypto::Dilithium65Pair;
use qp_poseidon_core::hash_bytes;
use qp_wormhole::{derive_wormhole_address, TestMiner, POW_ENGINE_ID};
use quantus_runtime::{
	transaction_extensions::{ReversibleTransactionExtension, WormholeProofRecorderExtension},
	Balances, BalancesCall, Executive, Header, Runtime, RuntimeCall, RuntimeEvent, Signature,
	SignedPayload, System, TxExtension, UncheckedExtrinsic, EXISTENTIAL_DEPOSIT, MAX_SUPPLY, UNIT,
	VERSION,
};
use sp_consensus_qpow::POW_ENGINE_ID as QPOW_ID;
use sp_core::{
	crypto::{AccountId32, Ss58AddressFormat, Ss58Codec},
	Pair, H256,
};
use sp_runtime::{
	generic::{Digest, DigestItem, Era},
	transaction_validity::{InvalidTransaction, TransactionValidityError},
	traits::{Header as HeaderT, IdentifyAccount},
	MultiAddress,
};

const SS58: u16 = 189;
const QUANTUM: u128 = AMOUNT_SCALE_DOWN_FACTOR;
/// How many consecutive production-runtime blocks the PoC mines to show the
/// remainder repeats and cannot be combined across leaves.
const ACCUMULATION_BLOCKS: u32 = 200;
const TARGET_BLOCK_TIME_MS: u64 = 12_000;
const EMISSION_DIVISOR: u128 = 15_163_560;

fn ss58(id: &AccountId32) -> String {
	id.to_ss58check_with_version(Ss58AddressFormat::custom(SS58))
}

fn hex32(bytes: &[u8; 32]) -> String {
	bytes.iter().map(|b| format!("{b:02x}")).collect()
}

fn account_bytes(id: &AccountId32) -> [u8; 32] {
	AsRef::<[u8]>::as_ref(id).try_into().expect("AccountId32 is 32 bytes")
}

fn quan(planck: u128) -> String {
	format!("{}.{:012}", planck / UNIT, planck % UNIT)
}

/// Build genesis: production treasury split + a Dilithium control signer that
/// *can* spend, plus a destination. No miner endowment — they are created by
/// `mint_into` on the first block, like a real QPoW author.
fn test_ext(control: &AccountId32, dest: &AccountId32) -> sp_io::TestExternalities {
	use quantus_runtime::{configs::TreasuryPalletId, BuildStorage};
	use sp_runtime::{traits::AccountIdConversion, Permill};

	let mut t = frame_system::GenesisConfig::<Runtime>::default().build_storage().unwrap();
	let treasury: AccountId32 = TreasuryPalletId::get().into_account_truncating();
	pallet_treasury::GenesisConfig::<Runtime> {
		treasury_account: Some(treasury.clone()),
		treasury_portion: Some(Permill::from_percent(50)),
	}
	.assimilate_storage(&mut t)
	.unwrap();

	let mut ext = sp_io::TestExternalities::new(t);
	ext.execute_with(|| {
		// Enough to pay fees and a 10 UNIT control transfer. Issuance is still
		// negligible vs MAX_SUPPLY, so block 1's miner share matches a
		// near-genesis mainnet block.
		Balances::make_free_balance_be(control, 1_000 * UNIT);
		Balances::make_free_balance_be(dest, EXISTENTIAL_DEPOSIT);
		Balances::make_free_balance_be(&treasury, 1_000 * UNIT);
	});
	ext
}

fn miner_digest(preimage: [u8; 32]) -> Digest {
	Digest { logs: vec![DigestItem::PreRuntime(POW_ENGINE_ID, preimage.to_vec())] }
}

fn signed_transfer(
	pair: &Dilithium65Pair,
	sender: AccountId32,
	dest: AccountId32,
	value: u128,
	nonce: u32,
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
		frame_system::CheckNonce::<Runtime>::from(nonce),
		frame_system::CheckWeight::<Runtime>::new(),
		ReversibleTransactionExtension::<Runtime>::new(),
		WormholeProofRecorderExtension::<Runtime>::new(),
		pallet_transaction_payment::ChargeTransactionPayment::<Runtime>::from(0),
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

/// Produce one QPoW block: `Executive::initialize_block` (pre-runtime digest =
/// miner preimage) → timestamp inherent → `Executive::finalize_block`
/// (`MiningRewards::on_finalize` mints + `record_transfer_proof`).
fn produce_qpow_block(number: u32, parent: H256, preimage: [u8; 32]) -> H256 {
	debug_assert_eq!(POW_ENGINE_ID, QPOW_ID);
	let header = Header::new(number, H256::zero(), H256::zero(), parent, miner_digest(preimage));
	Executive::initialize_block(&header);
	assert_ok!(pallet_timestamp::Pallet::<Runtime>::set(
		frame_system::RawOrigin::None.into(),
		TARGET_BLOCK_TIME_MS.saturating_mul(number as u64),
	));
	let finalized = Executive::finalize_block();
	finalized.hash()
}

fn last_miner_reward_event() -> (AccountId32, u128) {
	System::events()
		.into_iter()
		.rev()
		.find_map(|r| match r.event {
			RuntimeEvent::MiningRewards(pallet_mining_rewards::Event::MinerRewarded {
				miner,
				reward,
			}) => Some((miner, reward)),
			_ => None,
		})
		.expect("MiningRewards::on_finalize must emit MinerRewarded")
}

/// `on_finalize` mints miner then treasury, so the last leaf is the treasury
/// credit. Locate the miner-reward leaf by `(to, amount)`.
fn miner_reward_leaf(miner: &AccountId32, amount: u128) -> (u64, ZkLeaf<AccountId32, u32, u128>) {
	let count = pallet_zk_tree::Pallet::<Runtime>::leaf_count();
	assert!(count > 0, "mint_reward must insert a ZK leaf");
	for idx in (0..count).rev() {
		if let Some(leaf) = pallet_zk_tree::Pallet::<Runtime>::leaf(idx) {
			if &leaf.to == miner && leaf.amount == amount {
				return (idx, leaf);
			}
		}
	}
	panic!("no ZK leaf for miner with amount {amount}; leaf_count={count}");
}

fn count_miner_leaves(miner: &AccountId32) -> u64 {
	let count = pallet_zk_tree::Pallet::<Runtime>::leaf_count();
	(0..count)
		.filter(|&idx| {
			pallet_zk_tree::Pallet::<Runtime>::leaf(idx)
				.map(|leaf| &leaf.to == miner)
				.unwrap_or(false)
		})
		.count() as u64
}

#[cfg(test)]
mod tests {
	use super::*;

	#[test]
	fn prove_miner_remainder_frozen() {
		let _ = env_logger::try_init();

		let control_pair = Dilithium65Pair::from_seed_slice(&[42u8; 32]).expect("valid seed");
		let control = control_pair.public().into_account();
		let dest = AccountId32::new([9u8; 32]);

		let test_miner = TestMiner(1);
		let miner_preimage = test_miner.preimage();
		let miner = test_miner.account_id();

		let mut ext = test_ext(&control, &dest);
		ext.execute_with(|| {
			println!();
			println!("================================================================================");
			println!("H-1 PoC · Mining reward remainder permanently frozen on Poseidon miner");
			println!("================================================================================");
			println!();

			// ------------------------------------------------------------------
			// STEP 1 — miner identity is a wormhole address, not ML-DSA
			// ------------------------------------------------------------------
			println!("[STEP 1] Identify the QPoW miner as a Poseidon wormhole address (not ML-DSA)");
			println!("    QPoW preimage (32 bytes) : 0x{}", hex32(&miner_preimage));
			println!("    Miner AccountId (SS58)   : {}", ss58(&miner));
			println!("    Miner AccountId (hex)    : 0x{}", hex32(&account_bytes(&miner)));
			println!("    Dilithium control signer : {}", ss58(&control));
			println!("    Control AccountId (hex)  : 0x{}", hex32(&account_bytes(&control)));
			let control_pk_hash = hash_bytes(control_pair.public().as_ref());
			assert_eq!(
				AccountId32::from(control_pk_hash),
				control,
				"control AccountId must be hash_bytes(dilithium_pk)"
			);
			assert_ne!(
				miner, control,
				"Poseidon(preimage) must not equal hash_bytes(dilithium_pk)"
			);
			assert_eq!(
				miner,
				AccountId32::from(derive_wormhole_address(miner_preimage).unwrap()),
				"miner must be derive_wormhole_address(preimage)"
			);
			println!("    [OK] miner = Poseidon(preimage); control = hash(Dilithium pk); they differ");
			println!();

			// ------------------------------------------------------------------
			// STEP 2 — production constants + live issuance
			// ------------------------------------------------------------------
			let issuance_before = <Balances as Currency<AccountId32>>::total_issuance();
			println!("[STEP 2] Production runtime constants (quantus-runtime)");
			println!("    MAX_SUPPLY               : {} QUAN ({} planck)", MAX_SUPPLY / UNIT, MAX_SUPPLY);
			println!("    EmissionDivisor          : {EMISSION_DIVISOR}");
			println!("    Treasury portion         : 50% (genesis)");
			println!("    Circuit quantum          : {QUANTUM} planck = 0.01 QUAN");
			println!("    ExistentialDeposit       : {EXISTENTIAL_DEPOSIT} planck = 0.001 QUAN");
			println!("    total_issuance() before  : {} planck ({})", issuance_before, quan(issuance_before));
			assert_eq!(QUANTUM, pallet_wormhole::SCALE_DOWN_FACTOR);
			assert!(QUANTUM > EXISTENTIAL_DEPOSIT);
			let remaining_before = MAX_SUPPLY.saturating_sub(issuance_before);
			let expected_total = remaining_before / EMISSION_DIVISOR;
			let expected_treasury = expected_total / 2; // Permill 50% mul_floor
			let expected_miner = expected_total - expected_treasury;
			println!("    remaining supply         : {} planck ({})", remaining_before, quan(remaining_before));
			println!("    expected total reward    : {} planck ({})", expected_total, quan(expected_total));
			println!("    expected miner share     : {} planck ({})", expected_miner, quan(expected_miner));
			println!();

			// ------------------------------------------------------------------
			// STEP 3 — produce block 1 through Executive (real on_finalize path)
			// ------------------------------------------------------------------
			println!("[STEP 3] Produce block 1 via Executive::initialize_block + finalize_block");
			println!("    digest                   : PreRuntime(POW_ENGINE_ID, miner_preimage)");
			println!("    extrinsics               : none (isolates the block-reward leaf from tx fees)");
			let parent = produce_qpow_block(1, H256::zero(), miner_preimage);
			let (event_miner, miner_reward) = last_miner_reward_event();
			assert_eq!(event_miner, miner);
			assert_eq!(
				miner_reward, expected_miner,
				"MinerRewarded must equal (remaining / EmissionDivisor) after the 50% treasury split"
			);
			let miner_balance = Balances::free_balance(&miner);
			let issuance_after = <Balances as Currency<AccountId32>>::total_issuance();
			let committed_quanta = miner_reward / QUANTUM;
			let remainder = miner_reward % QUANTUM;
			let committed_planck = committed_quanta * QUANTUM;

			println!("    MinerRewarded.reward     : {miner_reward} planck ({})", quan(miner_reward));
			println!("    miner free_balance       : {miner_balance} planck ({})", quan(miner_balance));
			println!("    total_issuance() after   : {issuance_after} planck ({})", quan(issuance_after));
			println!("    floor(reward / quantum)  : {committed_quanta} quanta = {committed_planck} planck ({})", quan(committed_planck));
			println!("    remainder (FROZEN)       : {remainder} planck ({})", quan(remainder));
			println!(
				"    remainder / reward        : {:.4}%",
				(remainder as f64) * 100.0 / (miner_reward as f64)
			);
			assert_eq!(miner_balance, miner_reward, "first mint creates the miner account at exactly the reward");
			assert!(remainder > 0, "this PoC requires a non-zero remainder");
			assert!(
				remainder > EXISTENTIAL_DEPOSIT,
				"remainder is above ED so Substrate will not reap it — it is stuck, not dust"
			);
			assert!(
				committed_quanta >= 1,
				"this is NOT known issue #3 (whole reward below one quantum); committed quanta = {committed_quanta}"
			);
			println!("    [OK] full planck credited; remainder > ED and committed quanta >= 1");
			println!();

			// ------------------------------------------------------------------
			// STEP 4 — ZK leaf stores full planck, hash commits only quanta
			// ------------------------------------------------------------------
			println!("[STEP 4] ZK-tree leaf vs hash_leaf quantization");
			let (leaf_idx, leaf) = miner_reward_leaf(&miner, miner_reward);
			println!("    leaf_index               : {leaf_idx}");
			println!("    leaf.to                  : {}", ss58(&leaf.to));
			println!("    leaf.transfer_count      : {}", leaf.transfer_count);
			println!("    leaf.asset_id            : {}", leaf.asset_id);
			println!("    leaf.amount (storage)    : {} planck ({})", leaf.amount, quan(leaf.amount));
			println!("    zk-tree leaf_count       : {}", pallet_zk_tree::Pallet::<Runtime>::leaf_count());
			println!("    zk-tree root             : 0x{}", hex32(&pallet_zk_tree::Pallet::<Runtime>::root()));
			assert_eq!(leaf.to, miner);
			assert_eq!(leaf.amount, miner_reward, "record_transfer_proof stores the FULL mint");
			assert_eq!(leaf.asset_id, 0);

			let hash_full = hash_leaf::<Runtime>(&leaf);
			let hash_trunc = hash_leaf::<Runtime>(&ZkLeaf { amount: committed_planck, ..leaf.clone() });
			let hash_next = hash_leaf::<Runtime>(&ZkLeaf {
				amount: (committed_quanta + 1) * QUANTUM,
				..leaf.clone()
			});
			println!("    hash_leaf(full mint)     : 0x{}", hex32(&hash_full));
			println!("    hash_leaf(mint-remainder): 0x{}", hex32(&hash_trunc));
			println!("    hash_leaf(next quantum)  : 0x{}", hex32(&hash_next));
			assert_eq!(
				hash_full, hash_trunc,
				"the sub-quantum remainder must not change the committed leaf hash"
			);
			assert_ne!(
				hash_full, hash_next,
				"claiming one extra quantum would require a different Merkle leaf"
			);
			println!("    [OK] Merkle commitment is {committed_quanta} quanta; {remainder} planck are hash-invisible");
			println!("         A wormhole proof is bound to this hash, so the circuit's");
			println!("         private input_amount can only ever be {committed_quanta}, never {committed_quanta}+remainder.");
			println!();

			// ------------------------------------------------------------------
			// STEP 5 — Dilithium control CAN spend; the Poseidon miner CANNOT
			// ------------------------------------------------------------------
			println!("[STEP 5] Full TxExtension pipeline: Dilithium spend vs miner spend");
			let header2 =
				Header::new(2, H256::zero(), H256::zero(), parent, miner_digest(miner_preimage));
			Executive::initialize_block(&header2);
			assert_ok!(pallet_timestamp::Pallet::<Runtime>::set(
				frame_system::RawOrigin::None.into(),
				TARGET_BLOCK_TIME_MS * 2,
			));

			let control_before = Balances::free_balance(&control);
			let dest_before = Balances::free_balance(&dest);
			let xt_ok = signed_transfer(&control_pair, control.clone(), dest.clone(), 10 * UNIT, 0);
			let ok_outcome = Executive::apply_extrinsic(xt_ok);
			println!("    control Dilithium transfer 10 UNIT → dest : {:?}", ok_outcome);
			assert_ok!(ok_outcome.expect("signature + extensions must validate"));
			assert_eq!(Balances::free_balance(&dest), dest_before + 10 * UNIT);
			assert!(Balances::free_balance(&control) < control_before);
			println!("    [OK] hash_bytes(Dilithium pk) account spends through Executive::apply_extrinsic");

			let miner_before_attempt = Balances::free_balance(&miner);
			let xt_bad = signed_transfer(
				&control_pair,
				miner.clone(),
				dest.clone(),
				remainder.max(EXISTENTIAL_DEPOSIT),
				0,
			);
			let bad_outcome = Executive::apply_extrinsic(xt_bad);
			println!("    Dilithium signature claiming miner origin : {:?}", bad_outcome);
			assert_eq!(
				bad_outcome,
				Err(TransactionValidityError::Invalid(InvalidTransaction::BadProof)),
				"Poseidon miner must not be spendable with an ML-DSA key"
			);
			assert_eq!(
				Balances::free_balance(&miner),
				miner_before_attempt,
				"failed spend must not move miner funds"
			);
			println!("    miner balance unchanged    : {miner_before_attempt} planck");
			println!("    [OK] Verify binds AccountId = hash(pk); miner is Poseidon(preimage) → BadProof");
			println!("         There is no other spend path. Wormhole exit can only prove the HASH,");
			println!("         which does not contain the remainder.");
			println!();

			let parent = Executive::finalize_block().hash();
			let (_m2, reward2) = last_miner_reward_event();
			let remainder2 = reward2 % QUANTUM;
			println!(
				"    (block 2 also minted miner_reward={} planck ({}); remainder={} planck ({}))",
				reward2,
				quan(reward2),
				remainder2,
				quan(remainder2)
			);
			println!();

			// ------------------------------------------------------------------
			// STEP 6 — remainder repeats; leaves cannot be summed
			// ------------------------------------------------------------------
			println!(
				"[STEP 6] Mine {} further blocks; remainders accrue on separate leaves",
				ACCUMULATION_BLOCKS - 2
			);
			let mut frozen = remainder + remainder2;
			let mut reward_sum = miner_reward + reward2;
			let mut parent = parent;
			let miner_leaves_after_2 = count_miner_leaves(&miner);
			for n in 3..=ACCUMULATION_BLOCKS {
				parent = produce_qpow_block(n, parent, miner_preimage);
				let (_m, reward) = last_miner_reward_event();
				let r = reward % QUANTUM;
				frozen += r;
				reward_sum += reward;
				if n <= 5 || n == ACCUMULATION_BLOCKS || n % 50 == 0 {
					println!(
						"    block {:>3}: reward={} remainder={} frozen_so_far={} ({})",
						n,
						reward,
						r,
						frozen,
						quan(frozen)
					);
				}
			}
			let miner_leaves_end = count_miner_leaves(&miner);
			let miner_end = Balances::free_balance(&miner);
			let new_miner_leaves = miner_leaves_end.saturating_sub(miner_leaves_after_2);
			println!("    blocks produced          : {ACCUMULATION_BLOCKS}");
			println!("    miner free_balance end   : {miner_end} planck ({})", quan(miner_end));
			println!("    miner-addressed ZK leaves: {miner_leaves_end} (one block-reward leaf per block; block 2 also has a fee leaf)");
			println!("    ZK leaves added (blk3..): {new_miner_leaves}");
			println!("    SUM of block rewards     : {reward_sum} planck ({})", quan(reward_sum));
			println!("    SUM of remainders        : {frozen} planck ({})", quan(frozen));
			println!(
				"    remainders / rewards      : {:.4}%",
				(frozen as f64) * 100.0 / (reward_sum as f64)
			);
			assert!(
				new_miner_leaves >= (ACCUMULATION_BLOCKS as u64 - 2),
				"each block inserts its own miner-reward leaf; remainders cannot merge into one quantum"
			);
			assert!(frozen > remainder, "accumulation must exceed the first remainder");
			println!("    [OK] remainders live on distinct (to, transfer_count) leaves; they cannot be combined");
			println!();

			// ------------------------------------------------------------------
			// STEP 7 — not the listed known issues + loss envelope
			// ------------------------------------------------------------------
			println!("[STEP 7] Known-issue split and loss envelope");
			println!("    known issue #4           : volume fee uses increase_balance and SKIPS record_transfer");
			println!("                               this path mint_into + record_transfer_proof (leaf EXISTS)");
			println!("    known issue #3           : whole reward < 1 quantum at end of emission (planned upgrade)");
			println!("                               block 1 committed {committed_quanta} quanta, not 0");
			let remaining_now =
				MAX_SUPPLY.saturating_sub(<Balances as Currency<AccountId32>>::total_issuance());
			// miner_share < QUANTUM  ⇔  0.5 * remaining / D < Q  ⇔  remaining < 2*Q*D
			let remaining_when_dust = 2u128.saturating_mul(QUANTUM).saturating_mul(EMISSION_DIVISOR);
			let blocks_per_day = 86_400u64 / 12;
			let genesis_year = remainder.saturating_mul(blocks_per_day as u128).saturating_mul(365);
			println!("    remaining supply now     : {} planck ({})", remaining_now, quan(remaining_now));
			println!("    remaining when miner < 1q: {} planck ({})", remaining_when_dust, quan(remaining_when_dust));
			println!("    {ACCUMULATION_BLOCKS}-block frozen (measured): {} planck ({})", frozen, quan(frozen));
			println!("    if remainder stayed at block-1 rate for 1 year of 12s blocks:");
			println!(
				"        {blocks_per_day} blocks/day × 365 × {} planck = {} planck ({})",
				remainder,
				genesis_year,
				quan(genesis_year)
			);
			println!("    [OK] freeze is live from genesis-scale rewards, not the late-emission dust case");
			println!();

			println!("================================================================================");
			println!(
				"RESULT: {} planck ({}) of miner yield is credited on-chain but absent from",
				remainder,
				quan(remainder)
			);
			println!("        the ZK leaf hash on every block of this shape. The Poseidon miner");
			println!("        cannot sign it out. A wormhole proof cannot claim it. That is");
			println!("        permanent freezing of unclaimed yield (High #6).");
			println!(
				"        Measured over {ACCUMULATION_BLOCKS} blocks: {} planck ({}) frozen.",
				frozen,
				quan(frozen)
			);
			println!("================================================================================");
			println!();
		});
	}
}
```

The copy above is the complete test. If markdown rendering ever disturbs whitespace, use the on-disk file `chain/runtime/tests/transactions/h1_mining_reward_remainder.rs` — that is what was compiled and run.

## 3. Run command

From the `chain/` directory (Rust 1.93.0 via `chain/rust-toolchain`):

```
cargo test -p quantus-runtime --test mod -- --nocapture --exact \
  transactions::h1_mining_reward_remainder::tests::prove_miner_remainder_frozen
```

The integration-test binary is named `mod` because tests live at `runtime/tests/mod.rs`. No extra features, env vars, or network access. First compile of `pallet-wormhole` generates circuit binaries; the test itself is \~1.4s after that.

Expected: `test result: ok. 1 passed; 0 failed; 0 ignored; 0 measured; 42 filtered out`.

## 4. Full log (actual stdout of the run above)

Compiler warnings from `frame-benchmarking` / `pallet-wormhole` circuit generation are omitted. This is the test body:

```
running 1 test

================================================================================
H-1 PoC · Mining reward remainder permanently frozen on Poseidon miner
================================================================================

[STEP 1] Identify the QPoW miner as a Poseidon wormhole address (not ML-DSA)
    QPoW preimage (32 bytes) : 0x0100000000000000000000000000000000000000000000000000000000000000
    Miner AccountId (SS58)   : qznNecZsSzdkmwHDoNL541q6eaYUeMTYxeH9w7ExN5mLBgk2F
    Miner AccountId (hex)    : 0x813530fa0bfc5f9be516d78977bbcdc87e084a5c03da0d43dcfda1981b363cb6
    Dilithium control signer : qzpNwVGFBbe1hG3A7Z5Xmrhv9mvuuSv9pmGDrQrQbF9Ev7cMq
    Control AccountId (hex)  : 0xd9e6761ed141e22fdcfabe0bb599610568db4a7e7befac32956195846d3e6ce7
    [OK] miner = Poseidon(preimage); control = hash(Dilithium pk); they differ

[STEP 2] Production runtime constants (quantus-runtime)
    MAX_SUPPLY               : 21000000 QUAN (21000000000000000000 planck)
    EmissionDivisor          : 15163560
    Treasury portion         : 50% (genesis)
    Circuit quantum          : 10000000000 planck = 0.01 QUAN
    ExistentialDeposit       : 1000000000 planck = 0.001 QUAN
    total_issuance() before  : 2000001000000000 planck (2000.001000000000)
    remaining supply         : 20997999999000000000 planck (20997999.999000000000)
    expected total reward    : 1384767165428 planck (1.384767165428)
    expected miner share     : 692383582714 planck (0.692383582714)

[STEP 3] Produce block 1 via Executive::initialize_block + finalize_block
    digest                   : PreRuntime(POW_ENGINE_ID, miner_preimage)
    extrinsics               : none (isolates the block-reward leaf from tx fees)
    MinerRewarded.reward     : 692383582714 planck (0.692383582714)
    miner free_balance       : 692383582714 planck (0.692383582714)
    total_issuance() after   : 2001385767165428 planck (2001.385767165428)
    floor(reward / quantum)  : 69 quanta = 690000000000 planck (0.690000000000)
    remainder (FROZEN)       : 2383582714 planck (0.002383582714)
    remainder / reward        : 0.3443%
    [OK] full planck credited; remainder > ED and committed quanta >= 1

[STEP 4] ZK-tree leaf vs hash_leaf quantization
    leaf_index               : 3
    leaf.to                  : qznNecZsSzdkmwHDoNL541q6eaYUeMTYxeH9w7ExN5mLBgk2F
    leaf.transfer_count      : 0
    leaf.asset_id            : 0
    leaf.amount (storage)    : 692383582714 planck (0.692383582714)
    zk-tree leaf_count       : 5
    zk-tree root             : 0x90d0139eee08387a597d51045d93750e2fff26c44fb1fdf29359f73e6752ad23
    hash_leaf(full mint)     : 0x67adaeb8ac79476541af6d99196da48ac26d2587551b8c4e36ac06b8b58fc2e8
    hash_leaf(mint-remainder): 0x67adaeb8ac79476541af6d99196da48ac26d2587551b8c4e36ac06b8b58fc2e8
    hash_leaf(next quantum)  : 0x68408fce9cfadc125dc5b941dd948593569676b50dc6c788de94368e47394008
    [OK] Merkle commitment is 69 quanta; 2383582714 planck are hash-invisible
         A wormhole proof is bound to this hash, so the circuit's
         private input_amount can only ever be 69, never 69+remainder.

[STEP 5] Full TxExtension pipeline: Dilithium spend vs miner spend
    control Dilithium transfer 10 UNIT → dest : Ok(Ok(()))
    [OK] hash_bytes(Dilithium pk) account spends through Executive::apply_extrinsic
    Dilithium signature claiming miner origin : Err(TransactionValidityError::Invalid(InvalidTransaction::BadProof))
    miner balance unchanged    : 692383582714 planck
    [OK] Verify binds AccountId = hash(pk); miner is Poseidon(preimage) → BadProof
         There is no other spend path. Wormhole exit can only prove the HASH,
         which does not contain the remainder.

    (block 2 also minted miner_reward=692383537053 planck (0.692383537053); remainder=2383537053 planck (0.002383537053))

[STEP 6] Mine 198 further blocks; remainders accrue on separate leaves
    block   3: reward=692383491392 remainder=2383491392 frozen_so_far=7150611159 (0.007150611159)
    block   4: reward=692383445731 remainder=2383445731 frozen_so_far=9534056890 (0.009534056890)
    block   5: reward=692383400070 remainder=2383400070 frozen_so_far=11917456960 (0.011917456960)
    block  50: reward=692381345328 remainder=2381345328 frozen_so_far=119123201015 (0.119123201015)
    block 100: reward=692379062288 remainder=2379062288 frozen_so_far=238132249858 (0.238132249858)
    block 150: reward=692376779256 remainder=2376779256 frozen_so_far=357027146898 (0.357027146898)
    block 200: reward=692374496231 remainder=2374496231 frozen_so_far=475807892516 (0.475807892516)
    blocks produced          : 200
    miner free_balance end   : 138486803679516 planck (138.486803679516)
    miner-addressed ZK leaves: 201 (one block-reward leaf per block; block 2 also has a fee leaf)
    ZK leaves added (blk3..): 198
    SUM of block rewards     : 138475807892516 planck (138.475807892516)
    SUM of remainders        : 475807892516 planck (0.475807892516)
    remainders / rewards      : 0.3436%
    [OK] remainders live on distinct (to, transfer_count) leaves; they cannot be combined

[STEP 7] Known-issue split and loss envelope
    known issue #4           : volume fee uses increase_balance and SKIPS record_transfer
                               this path mint_into + record_transfer_proof (leaf EXISTS)
    known issue #3           : whole reward < 1 quantum at end of emission (planned upgrade)
                               block 1 committed 69 quanta, not 0
    remaining supply now     : 20997723047384215057 planck (20997723.047384215057)
    remaining when miner < 1q: 303271200000000000 planck (303271.200000000000)
    200-block frozen (measured): 475807892516 planck (0.475807892516)
    if remainder stayed at block-1 rate for 1 year of 12s blocks:
        7200 blocks/day × 365 × 2383582714 planck = 6264055372392000 planck (6264.055372392000)
    [OK] freeze is live from genesis-scale rewards, not the late-emission dust case

================================================================================
RESULT: 2383582714 planck (0.002383582714) of miner yield is credited on-chain but absent from
        the ZK leaf hash on every block of this shape. The Poseidon miner
        cannot sign it out. A wormhole proof cannot claim it. That is
        permanent freezing of unclaimed yield (High #6).
        Measured over 200 blocks: 475807892516 planck (0.475807892516) frozen.
================================================================================

test transactions::h1_mining_reward_remainder::tests::prove_miner_remainder_frozen ... ok

test result: ok. 1 passed; 0 failed; 0 ignored; 0 measured; 42 filtered out; finished in 1.38s
```

### What this log proves

* **Identity:** miner `0x813530fa…363cb6` = Poseidon(preimage); control `0xd9e6761e…3e6ce7` = `hash_bytes(Dilithium pk)`. They differ, so the miner cannot sign. After block 1 `leaf_count` is 5 because genesis endowments and the treasury mint also insert native leaves; the miner-reward leaf is identified by matching `to` and `amount` to `MinerRewarded` (index 3, `transfer_count` 0).
* **Full mint vs truncated hash:** `MinerRewarded` and `free_balance` are `692_383_582_714`. `hash_leaf(full)` = `hash_leaf(truncated)` = `0x67adaeb8…8fc2e8`. `hash_leaf(70 quanta)` = `0x68408fce…4008` (different). `2_383_582_714` planck are credited and stored but not in the Merkle commitment.
* **No Dilithium spend:** control transfer returns `Ok(Ok(()))`. The same key claiming miner origin returns `InvalidTransaction::BadProof`. Miner balance unchanged at `692_383_582_714`.
* **Loss:** `0.002383582714 QUAN` frozen on block 1 (`> ED`). Over 200 production-runtime blocks the remainders sum to `475_807_892_516` planck = `0.475807892516 QUAN` (\~0.34% of miner block rewards). Genesis-rate year envelope at 12s blocks: `6_264.055372392 QUAN`.
* **Not the listed known issues or the Eiger leftover finding:** 69 committed quanta (not known issue #3). A leaf exists (not known issue #4). The remainder is missing from the leaf hash before any exit (not Eiger `EQ-QNT-DOC-F-01` user-controlled partial withdrawal).
