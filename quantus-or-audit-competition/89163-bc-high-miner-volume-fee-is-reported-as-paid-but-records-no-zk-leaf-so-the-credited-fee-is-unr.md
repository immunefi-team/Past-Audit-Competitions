# 89163 bc high miner volume fee is reported as paid but records no zk leaf so the credited fee is unrealizable by the miner

**Submitted on Aug 17th 2026 at 15:30:34 UTC by @hunter67 for** [**Audit Comp | Quantus**](https://immunefi.com/audit-competition/audit-comp-quantus)

* **Report ID:** #89163
* **Report Type:** Blockchain/DLT
* **Report severity:** High
* **Target:** <https://github.com/immunefi-team/audit-comp-quantus-chain>
* **Impacts:**
  * Permanent freezing of funds

## Description

**Selected impact:** `Permanent freezing of funds` (Blockchain/DLT). Not the `(fix requires hardfork)` variant: Quantus is Substrate, so a forkless runtime upgrade fixes this, and a migration could record leaves for balances already stranded.

**Affected source:** `github.com/immunefi-team/audit-comp-quantus-chain` @ `3b243b870a5442a6e5f443056f67bbba87d472f5`, branch `audit-comp-ready`.

## Summary

When a settled wormhole exit bundle successfully credits the block author's share of the volume fee, `pallet-wormhole` credits it with `Unbalanced::increase_balance` and emits `MinerVolumeFeePaid`, so the protocol reports the miner as paid. (Where no author is found, or the credit returns `Err`, the pallet burns that share instead — this report concerns only the successful credit.)

The block author's address is not an ordinary account, and the miner cannot choose otherwise. The project's own `MINING.md` states it plainly: *"Mining rewards are automatically sent to wormhole addresses derived from your preimage… The system derives your wormhole address using Poseidon hashing"*, and the miner supplies only that preimage (`--rewards-inner-hash`). `extract_author_from_digest` performs exactly that derivation, so whenever a fee is successfully credited the recipient is necessarily a wormhole-derived address.

That address class cannot be spent by signature — `DilithiumSignatureScheme::verify` requires `hash_bytes(pubkey) == signer`, so signing as a Poseidon output would require a preimage attack — and the circuit crate names the class `UnspendableAccount`. Its only non-privileged spend path available to the miner is a ZK exit proved against a leaf, which `docs/wormhole-zk.md` confirms: *"One leaf proof = one user exit from a wormhole address."*

The miner therefore cannot avoid this by nominating an ordinary account: the node derives the payout address from the supplied preimage, and the derived address has no signing key.

The fee credit records no leaf. Unlike the exit credit fifty lines earlier in the same function, and unlike `pallet_mining_rewards::mint_reward` which pays the *same* address class, the volume-fee branch never calls `record_transfer`. Because a wormhole exit *mints* against a leaf rather than debiting the source account, an address's realizable value is the sum of its leaves — so the credited fee raises a balance that the miner has no way to spend or withdraw.

No attacker is involved and no attacker profits. This occurs under ordinary honest operation: a miner produces blocks, an aggregator settles a valid batch.

## Root cause

`pallets/wormhole/src/lib.rs`, `process_exit_bundle`:

* **Exit credit (correct):** `increase_balance(exit_account, exit_balance)` immediately followed by `record_transfer(asset, mint_account, exit_account, exit_balance)`.
* **Miner-fee credit (defective):** `increase_balance(author, miner_fee)`, then `deposit_event(MinerVolumeFeePaid { .. })`, and nothing else. No `record_transfer`.

`pallets/mining-rewards/src/lib.rs`, `mint_reward`, paying the same wormhole-derived miner address, does call `record_transfer_proof(None, mint_account, miner, reward)` — the codebase's own control case for how this address class must be paid.

## Why the credited value cannot be recovered by the miner

| Question                                                                      | Answer                                                                                                                                                                                                                                          |
| ----------------------------------------------------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Can the author sign a transaction from that address?                          | No. Signature verification requires an ML-DSA public key whose account hash equals the address; the address is a Poseidon output, so this is a preimage problem.                                                                                |
| Can a later recorder or sweep create a leaf for the already-credited balance? | No. `record_transfer` commits the *credit amount* passed to it together with `(canonical_recipient, transfer_count)`; it never reads the account's balance. The only balance-wide sweep is the genesis hook at block 1, which cannot run again. |
| Does the miner's next mining-reward leaf cover it?                            | No. That leaf commits the reward amount only.                                                                                                                                                                                                   |

## Impact and magnitude

Per settled bundle the lost amount is the miner's share of the volume fee: `miner_fee = total_fee − burn`, with `total_fee = ceil(exit_quanta · VolumeFeeRateBps / (10000 − VolumeFeeRateBps))` quanta and a 50% burn rate, at `VolumeFeeRateBps = 4` and a quantum of `10^10` planck. That is roughly **0.02% of wormhole exit volume**, credited to the miner but **unrealizable by the miner under the current protocol**, accumulating across settled exit bundles.

This is the miner's fee income rather than user deposits. The defect is deterministic and accumulates across settled bundles, but the amount per bundle is small; this report does not rely on a large per-event loss claim.

## What this report does not claim

* Not a theft. Nothing is stolen; no party gains the lost value.
* Not "all funds at wormhole addresses are inaccessible". Deposits and mining rewards each receive their own leaf and remain exitable. The defect is confined to the volume-fee credit.
* Not a claim that no privileged or future path could ever move the balance — Root retains `force_set_balance`. The claim is that the **miner** cannot realize a fee the protocol reports as paid.
* Not demonstrated end-to-end: a runtime-level test showing the miner realizing a mining reward `R` while the fee `F` stays stranded was **not** run. The pallet-level test demonstrates the mechanism; the unrealizability argument is the static chain above.

## Known-issue distinction

The program declares as a known issue: *"future zk-tree depth and mining reward scaling limitations requiring runtime upgrades"*. This report is **not** that issue. That known issue concerns capacity/scaling ceilings needing a future upgrade; this is a missing bookkeeping call on one credit path, present and effective today at current parameters, and independent of tree depth or reward scaling.

## Suggested fix

In the miner-fee branch of `process_exit_bundle`, mirror the exit-credit branch and the mining-rewards path: after a successful `increase_balance` for the author, call `record_transfer` for the same amount, so the credit is committed to the zk-tree and becomes exitable by the miner.

## Proof of Concept

Self-contained, reproduced from the frozen competition source. No private-repository dependency, no node build and no wasm build are needed.

```bash
git clone --branch audit-comp-ready --single-branch \
  https://github.com/immunefi-team/audit-comp-quantus-chain chain
cd chain
git rev-parse HEAD          # 3b243b870a5442a6e5f443056f67bbba87d472f5
git apply /path/to/H-A11-miner-fee-no-leaf.patch      # patch inlined at the end of this section
cargo test -p pallet-wormhole \
  miner_volume_fee_credits_balance_but_records_no_leaf -- --nocapture
```

The patch adds **one test** to `pallets/wormhole/src/tests.rs` and changes no production file (`1 file changed, 124 insertions(+)`). Observed result on the workspace-pinned toolchain `1.93.0` (`rust-toolchain`, not the host default):

```
running 1 test
test tests::private_batch_proof_tests::miner_volume_fee_credits_balance_but_records_no_leaf ... ok

test result: ok. 1 passed; 0 failed; 0 ignored; 0 measured; 71 filtered out; finished in 0.05s
```

## What the test asserts

{% stepper %}
{% step %}

### Miner payment is reported and credited

`MinerVolumeFeePaid` is emitted and the author's `free_balance` rises by exactly the expected fee — the protocol reports payment.
{% endstep %}

{% step %}

### No transfer count advances for the fee

The author's `TransferCount` does not advance, so the fee credit created no leaf keyed to it.
{% endstep %}

{% step %}

### The tree contains only exit-credit leaves

The tree grew by exactly one leaf per real exit credit — none for the fee.
{% endstep %}

{% step %}

### No leaf is keyed to the author

The number of leaves keyed to the author is unchanged by the settlement. This is measured as a delta rather than as an absolute zero, so it also holds for a miner that already owns earlier mining-reward leaves.
{% endstep %}
{% endstepper %}

Assertions 1 and 2 together are the finding: the balance rises while the bookkeeping that makes a wormhole-derived balance spendable does not.

The test deliberately contrasts with the existing `test_verify_private_batch_emits_miner_volume_fee_paid`, whose assertions stop at `free_balance` and therefore cannot distinguish "paid" from "credited into an address that can never spend it".

## Artifact integrity

| Artifact                                                 | sha256                                                             |
| -------------------------------------------------------- | ------------------------------------------------------------------ |
| `H-A11-miner-fee-no-leaf.patch` — inlined verbatim below | `0ecb65ed8bb7492ae640e3c302c0c049fd5a6fde2cf7650ec1de20390bef1ff5` |
| Full captured run log                                    | `48adaf011cdae0a7a598dd45dbd99149bef497e2f194d19d2bf931abf9bf7f3a` |

The decisive lines of the run log are quoted above. The full log is not reproduced here because it contains local host paths; it can be provided on request. Applying the patch below to the pinned SHA reproduces the result independently of that log.

## The patch

```diff
diff --git a/pallets/wormhole/src/tests.rs b/pallets/wormhole/src/tests.rs
index 4473b70..0702559 100644
--- a/pallets/wormhole/src/tests.rs
+++ b/pallets/wormhole/src/tests.rs
@@ -1092,6 +1092,130 @@ mod private_batch_proof_tests {
 		});
 	}
 
+	/// H-A11 PoC — the volume miner fee is credited to the block author's
+	/// wormhole-derived address, but **no ZK leaf is recorded for it**, so the
+	/// miner's realizable value does not rise with its balance.
+	///
+	/// Deliberate oracle contrast with
+	/// `test_verify_private_batch_emits_miner_volume_fee_paid` above, whose
+	/// assertions stop at `free_balance`: that oracle cannot distinguish "paid"
+	/// from "credited into an address that can never spend it". This test pins
+	/// the zk-tree state instead, which is what decides realizability.
+	///
+	/// Assertions (2) and (4) are the decisive ones. (3) additionally pins the
+	/// exact leaf delta and is the one that would need updating if a fixture
+	/// ever produced a failed exit mint.
+	#[test]
+	fn miner_volume_fee_credits_balance_but_records_no_leaf() {
+		new_test_ext().execute_with(|| {
+			let proof = deserialize_test_proof();
+			let inputs = parse_private_batch_public_inputs(&proof).expect("Should parse");
+
+			let block_number = inputs.block_data.block_number as u64;
+			let block_hash_bytes: [u8; 32] =
+				inputs.block_data.block_hash.as_ref().try_into().unwrap();
+			frame_system::BlockHash::<Test>::insert(block_number, H256::from(block_hash_bytes));
+			System::set_block_number(block_number + 10);
+
+			// Same production path as QPoW: the author is derived from the
+			// pre-runtime digest preimage, never supplied as a raw account.
+			let miner_preimage = [7u8; 32];
+			set_miner_preimage_digest(miner_preimage);
+			let expected_author = sp_core::crypto::AccountId32::from(
+				qp_wormhole::derive_wormhole_address(miner_preimage)
+					.expect("test preimage limbs are canonical"),
+			);
+
+			let exit_quanta: u128 = inputs
+				.account_data
+				.iter()
+				.filter(|a| a.summed_output_amount > 0)
+				.map(|a| a.summed_output_amount as u128)
+				.sum();
+			let fee_bps = VolumeFeeRateBps::get() as u128;
+			let total_fee = super::ceil_volume_fee(exit_quanta, fee_bps);
+			let expected_miner_fee = total_fee - total_fee / 2;
+			assert!(expected_miner_fee > 0, "fixture must produce a non-zero miner fee");
+
+			// One leaf per real exit credit is what a correct settlement records.
+			let expected_exit_credits = inputs
+				.account_data
+				.iter()
+				.filter(|a| {
+					let exit: [u8; 32] = (*a.exit_account)
+						.as_ref()
+						.try_into()
+						.expect("exit account is 32 bytes");
+					a.summed_output_amount > 0 && exit != [0u8; 32]
+				})
+				.count() as u64;
+
+			// Counts leaves keyed to `who`. Used before AND after so the assertion
+			// measures the *delta* rather than an absolute zero: a production miner
+			// may already hold legitimate mining-reward leaves, and this test must
+			// stay true in that case too.
+			let count_author_leaves = |who: &sp_core::crypto::AccountId32| {
+				(0..pallet_zk_tree::LeafCount::<Test>::get())
+					.filter_map(|i| pallet_zk_tree::Leaves::<Test>::get(i))
+					.filter(|leaf| &leaf.to == who)
+					.count()
+			};
+
+			let author_balance_before = Balances::free_balance(expected_author.clone());
+			let author_transfer_count_before = Wormhole::transfer_count(expected_author.clone());
+			let leaf_count_before = pallet_zk_tree::LeafCount::<Test>::get();
+			let author_leaves_before = count_author_leaves(&expected_author);
+
+			assert_ok!(Wormhole::verify_private_batch(
+				RawOrigin::None.into(),
+				get_test_proof_bytes()
+			));
+
+			// (1) The protocol reports the miner as paid, and the balance agrees.
+			System::assert_has_event(
+				crate::Event::<Test>::MinerVolumeFeePaid {
+					miner: expected_author.clone(),
+					amount: expected_miner_fee,
+				}
+				.into(),
+			);
+			assert_eq!(
+				Balances::free_balance(expected_author.clone()),
+				author_balance_before + expected_miner_fee,
+				"the miner's balance must rise by the volume fee"
+			);
+
+			// (2) DECISIVE: the fee credit advances no transfer count and creates
+			// no corresponding leaf, so the credited fee contributes no spendable
+			// leaf and no nullifier. Scoped deliberately: a LATER legitimate
+			// mining reward to the same author does create its own leaf, and this
+			// assertion says nothing about that - only about the fee credit.
+			assert_eq!(
+				Wormhole::transfer_count(expected_author.clone()),
+				author_transfer_count_before,
+				"H-A11: the miner fee advanced no transfer count, so no leaf commits it"
+			);
+
+			// (3) The tree grew by exactly one leaf per exit credit - none for the fee.
+			assert_eq!(
+				pallet_zk_tree::LeafCount::<Test>::get() - leaf_count_before,
+				expected_exit_credits,
+				"H-A11: leaves recorded == exit credits; the miner fee added none"
+			);
+
+			// (4) DECISIVE, exhaustive: the settlement added no leaf keyed to the
+			// author anywhere in the tree. Measured as a delta, so the assertion
+			// holds for a miner that already owns earlier reward leaves. A Poseidon
+			// output is already Goldilocks-canonical, so the stored (canonicalised)
+			// recipient equals the author verbatim.
+			assert_eq!(
+				count_author_leaves(&expected_author),
+				author_leaves_before,
+				"H-A11: the fee credit added no leaf for the author, so it added nothing spendable"
+			);
+		});
+	}
+
 	#[test]
 	fn test_verify_private_batch_cannot_replay() {
 		new_test_ext().execute_with(|| {
```

## Scope of the demonstration

The test is pallet-level and demonstrates the mechanism — credit without leaf. It does not attempt an end-to-end runtime demonstration of a miner realizing a mining reward while the fee stays stranded; that argument rests on the static chain given in the description (no signature path for a Poseidon-derived address, no later recorder or sweep that reads balances, and no balance-covering leaf), not on execution.
