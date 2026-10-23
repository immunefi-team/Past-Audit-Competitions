# 88943 bc high residue truncation and two leaf split in pallet mining rewards lead to permanent freezing of unclaimed miner yield

## #88943 \[BC-High] Residue truncation and two-leaf split in pallet-mining-rewards lead to permanent freezing of unclaimed miner yield

**Submitted on Aug 15th 2026 at 20:15:43 UTC by @v4rvl for** [**Audit Comp | Quantus**](https://immunefi.com/audit-competition/audit-comp-quantus)

* **Report ID:** #88943
* **Report Type:** Blockchain/DLT
* **Report severity:** High
* **Target:** <https://github.com/immunefi-team/audit-comp-quantus-chain>
* **Impacts:**
  * Permanent freezing of unclaimed yield

### Description

### Summary

`pallet-mining-rewards::on_finalize` pays the block author at a **keyless** wormhole address. That account has no private key by construction, so its **only** spend path is a ZK wormhole exit against a zk-tree leaf. The leaf, however, does not commit the credited amount: it commits `amount / 10^10` truncated to `u32` (`pallets/zk-tree/src/tree.rs:146-148`). Every planck below the `10^10` quantum is therefore minted into an account that can never spend it.

The pallet's only guard on a payout is `if reward.is_zero() { return; }` (`pallets/mining-rewards/src/lib.rs:220-223`). There is no quantum alignment and no minimum anywhere in the pallet. It then makes the loss worse by splitting one block's income into **two** independently-quantized credits (`lib.rs:193` fees, `lib.rs:196` emission), so the block pays the truncation twice.

The sibling pallet in the same repository already carries exactly this guard, with a comment naming exactly this failure mode.

**The lead claim is the quantization residue.** `leg mod 1e10` planck sits below the leaf quantum, so it is absent from every commitment: no leaf holds it, no exit can mint it, no fee collects it and no burn accounts for it. It is paid to nobody at all. This is the half that no documented design decision covers, and it is the half the report leads on. Measured at production constants on the runtime harness: **3,246,474,598 planck per block** (994,787,000 on the fee leg plus 2,251,687,598 on the reward leg), which is **0.4616% of that block's entire miner income of 703,246,474,598 planck**.

**The two-leaf split is the second claimed mechanism, and it is probabilistic rather than per-block.** `on_finalize` issues one block's income as two independently quantized credits (`lib.rs:193` fees, `lib.rs:196` emission), so the block pays the truncation twice. A further whole quantum is destroyed only on the blocks where the two residues sum past the quantum. The harness swept real production blocks carrying 1 to 8 signed ML-DSA-65 transfers each and measured the split strictly worse on **1 of the 8 sampled blocks**, with the split never better than one combined credit on any of them. It is stated as that measured ratio, not as an unconditional per-block cost.

**Amplification, reported as amplification: the dead one-quantum fee leg.** At production constants one signed ML-DSA-65 transfer costs 10,994,787,000 planck, that is 1.099 quanta, so the fee leg does **not** commit a zero-value leaf: it commits **one quantum**. The impact survives by a different mechanism. The circuit's fee relation `(output_1 + output_2) * 10000 <= input * (10000 - 4)` drives a one-quantum input to output `0`, and `process_exit_bundle` then skips a zero-output slot outright (`pallets/wormhole/src/lib.rs:787-789`), so the fee leg's single committed quantum, **10,000,000,000 planck**, can never be exited either. Total permanently frozen on the measured block: **13,246,474,598 planck, 1.8836% of miner income**. The legitimate 4 bps volume fee taken off the exitable reward leg (also 10,000,000,000 planck on this block) is reported separately and is **not** counted as stranding anywhere in this report.

The distinct future case where the emission reward itself falls below the quantum is conceded to Official Known Issue 3 (`SCOPE.md:78`) and is **not** claimed here.

### Root Cause

**In-scope trigger, reached without any elevated role.** The root cause is reached by `on_finalize` (`pallets/mining-rewards/src/lib.rs:129`), a FRAME hook with no origin, no dispatchable and no caller. There is no pre-existing administrator anywhere on this path, and nothing on it waits on an administrative step - it is a non-privileged path end to end, and it is not a self-inflicted one either, since the account that loses the value is the honest block author's. This finding does not rely on, and is independent of, every one of the following, each ruled out on its own terms - it needs no administrator, no contract owner, no governance vote, no multisig, no protocol team intervention, no operator, no sequencer, no guardian, no off-chain infrastructure, no leaked key, no leaked credential, and no test double standing in for a verifier. The loss fires under fully honest operation of the shipped runtime with every secret intact. The only actor is the node that authors the block, and the only configuration required is the one documented deployment mode (`node/src/service.rs:488`).

Three in-scope facts compose:

{% stepper %}
{% step %}

### The block author's account is keyless

`on_finalize` resolves the miner via `Self::extract_miner_from_digest()` (`pallets/mining-rewards/src/lib.rs:165`), which is `qp_wormhole::extract_author_from_digest` (`pallets/mining-rewards/src/lib.rs:205-208`). That function takes the miner-supplied 32-byte pre-runtime preimage and returns `derive_wormhole_address(preimage)` (`primitives/wormhole/src/lib.rs:147-149`), which is `rehash_to_bytes(&inner_digest)` - i.e. `address = Poseidon(Poseidon(salt || secret))` (`primitives/wormhole/src/lib.rs:97-111`). No ML-DSA keypair maps to that AccountId32. The node treats the operator-supplied value as the *preimage*, not as an address (`node/src/service.rs:488`: `let rewards_preimage: [u8; 32] = rewards_address.into();`), so a miner cannot opt out and be paid at a signable account.
{% endstep %}

{% step %}

### The only spend path commits a truncated amount

Each `mint_reward` calls `T::ProofRecorder::record_transfer_proof` (`pallets/mining-rewards/src/lib.rs:232-237`); `ProofRecorder = Wormhole` (`runtime/src/configs/mod.rs:141`). That lands in `Wormhole::record_transfer` (`pallets/wormhole/src/lib.rs:1280-1310`) which calls `T::ZkTree::record_transfer(...)`, and the leaf preimage commits

```rust
// pallets/zk-tree/src/tree.rs:146-148
let amount_u128: u128 = leaf.amount.into();
let amount_quantized =
    u32::try_from(amount_u128 / AMOUNT_SCALE_DOWN_FACTOR).unwrap_or(u32::MAX);
```

with `AMOUNT_SCALE_DOWN_FACTOR = 10_000_000_000` (`pallets/zk-tree/src/tree.rs:34`).
{% endstep %}

{% step %}

### A leaf under two quanta yields nothing on exit

The circuit's fee/balance relation is

```rust
// quantus-zk-circuits/wormhole/circuit/src/zk_merkle_proof.rs:491-497
// Fee constraint: (output_1 + output_2) * 10000 <= input * (10000 - fee_bps)
```

and the chain pins `bundle.volume_fee_bps == T::VolumeFeeRateBps::get()` (`pallets/wormhole/src/lib.rs:1100`) with `VolumeFeeRateBps: u32 = 4` (`runtime/src/configs/mod.rs:734`). For `input = 1` quantum: `out * 10000 <= 9996`, so `out = 0`; and `process_exit_bundle` then drops the slot outright:

```rust
// pallets/wormhole/src/lib.rs:787-789
if exit_account_bytes == [0u8; 32] || account_data.summed_output_amount == 0 {
    continue;
}
```

The minimum spendable leaf is therefore **2 quanta = 2e10 planck = 0.02 UNIT** (`UNIT = 1_000_000_000_000`, `runtime/src/lib.rs:97`).
{% endstep %}
{% endstepper %}

**The omission.** `mint_reward` guards nothing but zero:

```rust
// pallets/mining-rewards/src/lib.rs:220-223
fn mint_reward(maybe_miner: Option<&T::AccountId>, reward: BalanceOf<T>) {
    if reward.is_zero() {
        return;
    }
```

**The amplifier.** One block's author income is issued as two independent leaves:

```rust
// pallets/mining-rewards/src/lib.rs:192-196
// Send fees to miner if any
Self::mint_reward(miner.as_ref(), tx_fees);

// Send block rewards to miner
Self::mint_reward(miner.as_ref(), miner_reward);
```

Since `floor(a/q) + floor(b/q) <= floor((a+b)/q)`, the split destroys an additional quantum whenever `frac(a) + frac(b) >= q`. That is a **probabilistic** condition, not a per-block one, and it is reported as measured rather than as modelled: sweeping real production blocks carrying 1 to 8 signed ML-DSA-65 transfers each, the split was never better than a single combined credit and was **strictly worse on 1 of the 8 sampled blocks** (the 8-transfer block, where the fee-leg residue 7,958,296,000 and the reward-leg residue 2,251,687,598 sum to 10,209,983,598 planck, past the 1e10 quantum). The pallet already owns the helper that would fix it: `retain_unminted` rolls a residue back into `CollectedFees` for the next finalize (`pallets/mining-rewards/src/lib.rs:301-306`).

### Attack Path

There is no attacker. This is a spontaneous, unconditional protocol defect that fires on **every block** with a block author. The sequence is:

{% stepper %}
{% step %}

### A miner runs a node

A miner runs a node with `REWARDS_INNER_HASH` set (the only supported configuration - `node/src/service.rs:488`). The runtime pays them at `Poseidon(Poseidon(salt||secret))`.
{% endstep %}

{% step %}

### Users submit transactions

Users submit transactions; `TransactionFeesCollector` accumulates them into `CollectedFees` (`pallets/mining-rewards/src/lib.rs:210-217, 309-320`).
{% endstep %}

{% step %}

### `on_finalize` issues two credits

`on_finalize` takes the fees (`lib.rs:132`), computes the emission split (`lib.rs:143-162`) and issues **two** credits to the keyless miner (`lib.rs:193`, `lib.rs:196`).
{% endstep %}

{% step %}

### Both credits succeed

`T::Currency::mint_into` raises `free_balance` and `total_issuance`.
{% endstep %}

{% step %}

### Both credits create a zk-tree leaf

Both credits create a zk-tree leaf committing `amount / 1e10`.
{% endstep %}

{% step %}

### Sub-quantum planck becomes unreachable

Any planck below the quantum on either leg is now in an account with no key and no representable leaf. It is unreachable forever. This is the claimed residue: at production constants, 994,787,000 planck on the fee leg plus 2,251,687,598 on the reward leg, 3,246,474,598 planck for the block.
{% endstep %}

{% step %}

### One-quantum legs are also unreachable

Amplification on top of step 6, and reported as amplification: whenever the truncated leaf value of a whole leg is `0` or `1`, that entire leg is unreachable too, because the circuit forces `output = 0` and `process_exit_bundle` skips the slot. At production constants the fee leg lands exactly here: one signed ML-DSA-65 transfer costs 10,994,787,000 planck, so the leg commits **one quantum, not zero**, and that one committed quantum is driven to zero output by the exit fee constraint. A further 10,000,000,000 planck is therefore dead on this block. The distinct future case where the emission reward itself falls below the quantum is **conceded to Official Known Issue 3**.
{% endstep %}
{% endstepper %}

## Impact

Permanent loss of unclaimed miner yield, measured at production constants on a real runtime block. The report leads with the mechanism that no documented design decision covers, and presents the rest as amplification.

**(i) LEAD CLAIM - the quantization residue, value paid to nobody at all.** `leg mod 1e10` planck is below the leaf quantum, so it is absent from every commitment. There is no leaf holding it, no exit that can mint it, no fee that collects it and no burn that accounts for it. It simply has no recipient. Measured on the executed production block: **3,246,474,598 planck**, decomposing as 994,787,000 on the fee leg plus 2,251,687,598 on the reward leg. That is **0.4616%** of that block's miner income of 703,246,474,598 planck. There is no magnitude at which this stops: the emission credit today is 69 quanta, 69x above the leaf quantum, and its residue is still destroyed; a reward of 1,000 quanta strands its residue exactly the same way. At `TARGET_BLOCK_TIME_MS = 12_000` (`runtime/src/lib.rs:87`) that is 7,200 blocks/day, so **\~23.37 UNIT/day, \~8,532 UNIT/year, \~0.0406%** of the 21,000,000 UNIT cap per year on this block profile. The arithmetic behind every one of those figures is shown in Funds At Risk And Reward Bound.

**(ii) The two-leaf split, which pays the truncation twice - probabilistic, measured, not per-block.** `on_finalize` issues one block's author income as two independently quantized credits (`pallets/mining-rewards/src/lib.rs:193`, `:196`). Since `floor(a/q) + floor(b/q) <= floor((a+b)/q)`, a whole extra quantum (1e10 planck) is destroyed whenever the two legs' fractions sum to at least one quantum. The harness swept real production blocks carrying 1 to 8 signed ML-DSA-65 transfers each and measured: the split is **never** better than one combined credit, and it is **strictly worse on 1 of the 8 sampled blocks**. The report claims that measured ratio and nothing stronger. The 8-transfer block is the strictly-worse one: two leaves commit 77 quanta where one combined credit commits 78. The mutation control flips exactly that assertion, proving it is load-bearing on `:193`/`:196` and on nothing else. This is not a quantization necessity; it is an avoidable implementation choice, and the pallet already owns the helper that fixes it (`retain_unminted`, `lib.rs:301-306`).

**(iii) AMPLIFICATION, stated as amplification - the dead one-quantum fee leg.** At production constants one signed ML-DSA-65 transfer costs 10,994,787,000 planck, so the fee leg is 1.099 quanta and commits **one quantum, not zero**. That single committed quantum is nevertheless unexitable: the circuit's fee relation `(output_1 + output_2) * 10000 <= input * (10000 - 4)` admits no output above zero for a one-quantum input, and `process_exit_bundle` skips a zero-output slot at `pallets/wormhole/src/lib.rs:787-789`. A further **10,000,000,000 planck** is therefore dead on the measured block. This is presented as amplification rather than as the lead because `runtime/src/configs/mod.rs:729-733` documents the flat one-quantum minimum exit fee as intended, so a triager can reasonably argue the dead fee leaf is by design. No such comment covers the residue in (i), which is why (i) leads.

**Total on the measured block**: **13,246,474,598 planck permanently frozen, 1.8836% of miner income**, that is \~95.37 UNIT/day, \~34,812 UNIT/year, \~0.1658% of the 21,000,000 UNIT cap per year. The legitimate 4 bps volume fee taken off the exitable reward leg (10,000,000,000 planck on this block) is a correctly-charged protocol fee with a recipient and is reported separately; it is **not** counted as stranding anywhere in this report.

No claimed loss is recoverable in-protocol (see Victim Recovery Enumeration).

### Funds At Risk And Reward Bound

* **Asset identity**: the native Quantus token, minted by `T::Currency::mint_into` in `pallets/mining-rewards/src/lib.rs:230` with `type Currency = Balances` (`runtime/src/configs/mod.rs:138-141`). Denomination fixed at `pub const UNIT: Balance = 1_000_000_000_000;` (`runtime/src/lib.rs:97`). Total cap `MAX_SUPPLY = 21_000_000 * UNIT` (`runtime/src/lib.rs:105`).
* **Unit -> USD**: 1 UNIT = 1e12 planck (`runtime/src/lib.rs:97`). **No USD figure is claimed.** The chain is pre-launch at this pin: the program publishes no token price and there is no TVL figure to convert against, so asserting a dollar number here would be fabricated. The loss is quantified in native token and as a percentage of author income instead, and the honest statement is that the USD figure scales linearly with whatever the launch price is.
* **Market-size scenario**: 7,200 blocks/day (`TARGET_BLOCK_TIME_MS = 12_000`, `runtime/src/lib.rs:87`); block emission `remaining_supply / EmissionDivisor` with `EmissionDivisor = 15_163_560` (`runtime/src/configs/mod.rs:144`), treasury share 50% (`pallets/treasury/src/lib.rs:60`, v1 migration). On the executed production block the miner reward leg is **692,251,687,598 planck** and the fee leg is **10,994,787,000 planck**, so gross miner income is **703,246,474,598 planck**. Traffic scenario: 1 signed transfer/block, which is the realistic launch regime for a newly launched base-layer chain and is the regime the program's own 12-second block target implies.
* **The production transaction fee, decomposed**:

  ```
  5,343,000,000  length fee     5,343 encoded bytes x LENGTH_FEE_MULTIPLIER 1e6   (configs/mod.rs:445)
  5,543,630,000  weight fee     5,543,630,000 ps ref_time via IdentityFee          (configs/mod.rs:483)
    108,157,000  ExtrinsicBaseWeight                                               (extrinsic_weights.rs:55-56)
  --------------
  10,994,787,000  planck per signed ML-DSA-65 transfer_keep_alive  =  1.0994787 quanta
  ```
* **Per-block loss, with the arithmetic shown.** Quantum `q = 1e10` planck.

  ```
  fee leg      10,994,787,000 = 1 x q + 994,787,000          -> commits 1 quantum, residue     994,787,000
  reward leg  692,251,687,598 = 69 x q + 2,251,687,598       -> commits 69 quanta, residue   2,251,687,598
  quantization residue (no recipient at all)                                                3,246,474,598
  dead whole-leaf value: the fee leg's 1 committed quantum, driven to output 0              10,000,000,000
  ------------------------------------------------------------------------------------------------------
  PERMANENTLY FROZEN this block                                                             13,246,474,598
  (reported separately, NOT stranding: legitimate 4 bps volume fee on the exitable
   reward leg, 69 quanta in -> 68 quanta out)                                               10,000,000,000
  ```
* **Percentage of author income, from the production numbers**:

  ```
  residue share    3,246,474,598 / 703,246,474,598 = 0.00461641  ->  0.4616 %
    of which fee-leg residue      994,787,000 / 703,246,474,598 =    0.1415 %
    of which reward-leg residue 2,251,687,598 / 703,246,474,598 =    0.3202 %
  dead-leaf share 10,000,000,000 / 703,246,474,598 = 0.01421977  ->  1.4220 %
  total frozen    13,246,474,598 / 703,246,474,598 = 0.01883617  ->  1.8836 %
  ```
* **Per-day and per-year** (7,200 blocks/day, 365 days, 1 UNIT = 1e12 planck):

  ```
  residue   3,246,474,598 x 7,200 =  23,374,617,105,600 planck/day =    23.3746 UNIT/day
            23,374,617,105,600 x 365 = 8,531,735,243,544,000 planck =  8,531.74 UNIT/year
            8,531.74 / 21,000,000 = 0.00040627                      ->   0.0406 % of cap per year
  total    13,246,474,598 x 7,200 =  95,374,617,105,600 planck/day =    95.3746 UNIT/day
            95,374,617,105,600 x 365 = 34,811,735,243,544,000 planck = 34,811.74 UNIT/year
            34,811.74 / 21,000,000 = 0.00165770                     ->   0.1658 % of cap per year
  ```
* **Which of these numbers are stable and which are not - stated rather than left to be found.**
  * **Issuance-INDEPENDENT** (fixed by the fee constants and the quantum, not by supply): the transaction fee 10,994,787,000 planck, its residue 994,787,000 planck, and the fee leg's 1 dead committed quantum 10,000,000,000 planck. Their sum, **10,994,787,000 planck per block, is the stable floor** of the per-block loss at 1 transfer/block, because the entire fee leg is unexitable at that traffic level.
  * **Issuance-DEPENDENT**: the reward-leg residue 2,251,687,598 planck. The reward leg is `(MAX_SUPPLY - total_issuance) / EmissionDivisor` net of the treasury share, so its residue is a function of `total_issuance`. The runtime-test genesis seeds roughly 6,000 UNIT, far below any realistic mainnet genesis, so this component is **not** a stable figure: it varies block to block across the whole range `0 .. 1e10` planck with an expected value of about 5e9 planck. Gross miner income, and therefore every percentage above, moves with it for the same reason.
  * The honest consequence: the **shape** of the finding, the residue having no recipient and a one-quantum leg being unexitable, is issuance-independent; the **exact reward-leg planck figure and the exact percentages** are one measured block and are labelled as such throughout.
* **Absolute figure vs floor**: the loss is unbounded in time (it never stops) and monotonically accumulating. \*\*Per-residue magnitude is strictly under 1e10 planck (0.01 UNIT), and the severity claim does not rest on per-credit magnitude.
* **Attacker cost, capital lock, profit and net loss - stated in the expected terms, and stated honestly as zero where they are zero.** This finding has **no attacker**, so several standard economic fields have no value to report. They are recorded as not applicable rather than filled in with an invented figure:
  * **Attacker cost**: not applicable, and the honest value is zero. There is no attack transaction, so there is no attacker cost to price: no gas, no fee, no bond, no stake and no bribe is spent by anybody to cause this loss. `on_finalize` (`pallets/mining-rewards/src/lib.rs:129`) runs by itself on every block.
  * **Capital lock**: none, and the honest value is zero. No capital is locked, posted, borrowed, flash-loaned, bonded or staked at any point on this path. The required capital for the finding to fire is zero because there is nothing to fund.
  * **Attacker economics**: **there is no attacker, nobody gains anything from this finding, and no gain is claimed.** Nobody receives the frozen planck. It is minted into a keyless account (`pallets/mining-rewards/src/lib.rs:230`) and simultaneously made unrepresentable in the only instrument that can ever claim it (`pallets/zk-tree/src/tree.rs:146-148`), so the value is **destroyed, not transferred**. Because the gain side is structurally empty, no profitability test, no cost-versus-gain comparison and no return-on-capital figure applies here, and this report asserts none. The finding stands on destruction of the miner's income, not on anyone's extraction of it.
  * **Net loss**: the net loss is the protocol's and the block author's, with no offsetting counterparty gain anywhere: **13,246,474,598 planck per block** on the executed production-constant block, accruing every block with an author, forever.
  * **Affected amount and amount basis**: the affected amount is the frozen 13,246,474,598 planck per block, decomposed above into 3,246,474,598 planck of quantization residue that has no recipient at all and the fee leg's single committed quantum of 10,000,000,000 planck that the exit relation drives to zero output. The amount basis is that block's gross miner income of 703,246,474,598 planck, which puts the affected amount at 1.8836% of author income on the measured block. Every figure here is read off the executed transcripts and the production constants cited above; none is modelled, extrapolated to a token price, or assumed.
  * **Why an economics section exists at all on a no-attacker finding**: the fields are answered rather than skipped so that a triager does not have to infer whether they were overlooked or are genuinely empty. They are genuinely empty, and saying so is the accurate answer.

### Proof of Concept

### Proof Of Concept

**Harness A (primary) - production-constant runtime integration harness.** It is not a mock. It runs inside the `quantus-runtime` crate's own integration-test directory using the crate's existing `TestCommons::new_test_ext()` harness, with real `frame_system` and `pallet_treasury` genesis. The transaction fee is produced by a **real ML-DSA-65 signed extrinsic driven through `Executive::apply_extrinsic`**, so nothing about the fee is assumed. Because the runtime wires `ProofRecorder = Wormhole` (`runtime/src/configs/mod.rs:141`), a miner credit walks the **real** `pallet_wormhole::record_transfer` into `pallet_zk_tree::insert_leaf`; the leaves are then read back out of live storage, and the quantization is proven by `ZkTree::verify_proof` **accepting the quantized leaf against the same live on-chain root** as the real one. Every constant is read off the runtime at run time, never hard coded.

The pinned source tree was not modified by either harness beyond the test file itself; each run was made against the pinned tree, which was restored afterwards.

#### Harness A - production-constant runtime integration harness (PRIMARY EVIDENCE)

Run command:

```bash
SKIP_WASM_BUILD=1 cargo test -p quantus-runtime --test quantum_freeze_production_constants -- --nocapture --test-threads=1
```

**Harness A source (inline, complete)**

```rust
//! Lane `PROD-CONSTANTS-POC` - the sub-quantum mining-reward freeze at PRODUCTION
//! constants, driven through the real `quantus_runtime::Runtime` rather than the
//! pallet mock.
//!
//! TO RUN: copy this file to `src/quantus-chain/runtime/tests/quantum_freeze_production_constants.rs`
//! (cargo auto-discovers `runtime/tests/*.rs` as integration-test targets; the crate
//! declares no `[[test]]` sections), then from `src/quantus-chain`:
//!
//!     SKIP_WASM_BUILD=1 cargo test -p quantus-runtime \
//!         --test quantum_freeze_production_constants -- --nocapture --test-threads=1
//!
//! Every constant below is read from the real runtime at run time, never hard coded:
//!   AMOUNT_SCALE_DOWN_FACTOR   pallets/zk-tree/src/tree.rs:34
//!   VolumeFeeRateBps           runtime/src/configs/mod.rs:734
//!                              pinned on chain at pallets/wormhole/src/lib.rs:1100
//!   MILLI_UNIT/EXISTENTIAL_DEPOSIT  runtime/src/lib.rs:98,102
//!   WeightToFee = IdentityFee  runtime/src/configs/mod.rs:483
//!   Balance = u128             runtime/src/lib.rs:120
//!   MaxSupply / EmissionDivisor / Treasury portion
//!                              runtime/src/configs/mod.rs:138-148
//!
//! Unlike the pallet mock, `ProofRecorder = Wormhole` here (configs/mod.rs:140), so a
//! miner credit walks the REAL recorder: `pallet_wormhole::record_transfer` ->
//! `pallet_zk_tree::insert_leaf`, and the leaf that lands in the live tree is the
//! authoritative one. The transaction fee is produced by a real ML-DSA-65 signed
//! extrinsic through `Executive::apply_extrinsic`, so nothing about the fee is assumed.
//!
//! Source anchors re-read for this harness:
//!   mining-rewards/src/lib.rs:193   mint_reward(miner, tx_fees)      leg A
//!   mining-rewards/src/lib.rs:196   mint_reward(miner, miner_reward) leg B
//!   mining-rewards/src/lib.rs:220   the only guard is `reward.is_zero()`
//!   zk-tree/src/tree.rs:146-148     leaf commits u32(amount / 10^10)
//!   wormhole/src/lib.rs:787         exit slot skipped when summed_output_amount == 0
//!   zk-circuits wormhole/circuit/src/zk_merkle_proof.rs:491-497
//!                                   (out_1 + out_2) * 10000 <= input * (10000 - bps)

#[allow(dead_code)]
mod common;

use codec::Encode;
use common::TestCommons;
use frame_support::{
	assert_ok,
	dispatch::GetDispatchInfo,
	traits::{Currency, Get, Hooks},
	weights::WeightToFee as WeightToFeeT,
};
use qp_dilithium_crypto::Dilithium65Pair;
use quantus_runtime::{
	transaction_extensions::{ReversibleTransactionExtension, WormholeProofRecorderExtension},
	Balances, BalancesCall, Executive, MiningRewards, Runtime, RuntimeCall, Signature,
	SignedPayload, System, TreasuryPallet, TxExtension, UncheckedExtrinsic, ZkTree,
	EXISTENTIAL_DEPOSIT, MILLI_UNIT, UNIT, VERSION,
};
use sp_core::Pair;
use sp_runtime::{generic::Era, generic::DigestItem, traits::IdentifyAccount, AccountId32,
	MultiAddress};

type Leaf = pallet_zk_tree::ZkLeaf<AccountId32, quantus_runtime::AssetId, u128>;

/// zk-tree/src/tree.rs:34 - the leaf amount quantum, read from the real crate.
const Q: u128 = pallet_zk_tree::tree::AMOUNT_SCALE_DOWN_FACTOR;

/// What the ZK-tree leaf actually commits for a raw planck `amount`
/// (zk-tree/src/tree.rs:146-148).
fn leaf_quanta(amount: u128) -> u128 {
	amount / Q
}

/// Largest `output_amount_1 + output_amount_2` the wormhole circuit admits for a leaf
/// holding `input_quanta`, from quantus-zk-circuits
/// `wormhole/circuit/src/zk_merkle_proof.rs:491-497`, using the fee rate the chain
/// actually pins at `pallets/wormhole/src/lib.rs:1100`.
fn max_circuit_output_quanta(input_quanta: u128, bps: u128) -> u128 {
	input_quanta * (10_000 - bps) / 10_000
}

/// `process_exit_bundle` drops the slot when the summed output is zero
/// (pallets/wormhole/src/lib.rs:787-789), so nothing is ever minted for it.
fn exit_mints_planck(input_quanta: u128, bps: u128) -> u128 {
	let out = max_circuit_output_quanta(input_quanta, bps);
	if out == 0 {
		0
	} else {
		out.saturating_mul(pallet_wormhole::SCALE_DOWN_FACTOR)
	}
}

/// The production volume fee rate as the runtime configures it
/// (runtime/src/configs/mod.rs:734).
fn production_fee_bps() -> u128 {
	<Runtime as pallet_wormhole::Config>::VolumeFeeRateBps::get() as u128
}

/// runtime/src/configs/mod.rs:143 - `MaxSupply = ConstU128<{ MAX_SUPPLY }>`.
fn max_supply() -> u128 {
	<<Runtime as pallet_mining_rewards::Config>::MaxSupply as Get<u128>>::get()
}

/// runtime/src/configs/mod.rs:144 - `EmissionDivisor = ConstU128<15_163_560>`.
fn emission_divisor() -> u128 {
	<<Runtime as pallet_mining_rewards::Config>::EmissionDivisor as Get<u128>>::get()
}

/// Build a `transfer_keep_alive` extrinsic signed with ML-DSA-65, byte-identical in
/// shape to `runtime/tests/transactions/ml_dsa_65.rs::signed_transfer`.
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

/// Assert every production constant the finding leans on, straight off the runtime.
fn assert_production_constants() {
	// zk-tree/src/tree.rs:34
	assert_eq!(Q, 10_000_000_000, "AMOUNT_SCALE_DOWN_FACTOR");
	// wormhole/src/lib.rs:214 - the exit scale-up uses the same quantum.
	assert_eq!(pallet_wormhole::SCALE_DOWN_FACTOR, Q, "wormhole SCALE_DOWN_FACTOR");
	// runtime/src/configs/mod.rs:734, pinned on chain at wormhole/src/lib.rs:1100
	assert_eq!(production_fee_bps(), 4, "VolumeFeeRateBps");
	// runtime/src/lib.rs:97,98,102
	assert_eq!(UNIT, 1_000_000_000_000);
	assert_eq!(MILLI_UNIT, 1_000_000_000);
	assert_eq!(EXISTENTIAL_DEPOSIT, MILLI_UNIT);
	// runtime/src/lib.rs:120 - Balance is u128, so the compiler checks this line.
	let _balance_is_u128: u128 = <Runtime as pallet_balances::Config>::Balance::from(1u32);
	// runtime/src/configs/mod.rs:483 - IdentityFee maps ref_time 1:1 onto planck.
	let one_ms = frame_support::weights::Weight::from_parts(1_000_000_000, 0);
	assert_eq!(
		<<Runtime as pallet_transaction_payment::Config>::WeightToFee as WeightToFeeT>::weight_to_fee(&one_ms),
		1_000_000_000u128,
		"WeightToFee must be IdentityFee<Balance>"
	);
	// runtime/src/configs/mod.rs:143,144
	assert_eq!(max_supply(), 21_000_000 * UNIT);
	assert_eq!(emission_divisor(), 15_163_560u128);
	// The quantum is 10 milli-UNIT: ten times the existential deposit, so a credit can
	// be far above the ED (a live, non-reaped account) and still commit nothing.
	assert_eq!(Q, 10 * MILLI_UNIT);
}

/// The reward split the pallet computes at `mining-rewards/src/lib.rs:142-162`,
/// recomputed here from the live runtime values so nothing is hard coded.
fn expected_legs(tx_fees: u128) -> (u128, u128) {
	let current_supply: u128 = Balances::total_issuance().saturating_add(tx_fees);
	let total_reward = max_supply().saturating_sub(current_supply) / emission_divisor();
	let treasury_reward = TreasuryPallet::portion().mul_floor(total_reward);
	(tx_fees, total_reward.saturating_sub(treasury_reward))
}

/// Externalities: the runtime-tests genesis (`tests/common.rs`) plus a funded
/// ML-DSA-65 payer, so a real signed transfer can be applied.
fn production_ext(payer: &AccountId32) -> sp_io::TestExternalities {
	let mut ext = TestCommons::new_test_ext();
	ext.execute_with(|| {
		Balances::make_free_balance_be(payer, 1000 * UNIT);
		System::set_block_number(1);
	});
	ext
}

/// The wormhole miner address for a QPoW pre-runtime preimage
/// (primitives/wormhole/src/lib.rs:109,122-148). Deposits the digest too.
fn seat_miner(preimage: [u8; 32]) -> AccountId32 {
	System::deposit_log(DigestItem::PreRuntime(qp_wormhole::POW_ENGINE_ID, preimage.to_vec()));
	let bytes = qp_wormhole::derive_wormhole_address(preimage)
		.expect("derive_wormhole_address must succeed for a 32-byte preimage");
	AccountId32::new(bytes)
}

/// Apply `count` real ML-DSA-65 signed transfers and return the total fee the chain
/// charged for them. Nothing about the fee is assumed: it is read back out of the
/// pallet's own `CollectedFees` pot.
fn apply_signed_transfers(pair: &Dilithium65Pair, payer: &AccountId32, count: u32) -> u128 {
	let dest = AccountId32::new([9u8; 32]);
	let fees_before = collected_fees();
	for nonce in 0..count {
		let xt = signed_transfer(pair, payer.clone(), dest.clone(), UNIT, nonce);
		if nonce == 0 {
			let len = xt.encode().len() as u128;
			println!(
				"PRODUCTION CONSTANTS - ML-DSA-65 transfer_keep_alive: {} encoded bytes, \
				 length fee {} planck (LENGTH_FEE_MULTIPLIER 1e6, configs/mod.rs:445), \
				 ref_time {} ps -> weight fee {} planck (IdentityFee, configs/mod.rs:483)",
				len,
				len * 1_000_000,
				xt.get_dispatch_info().total_weight().ref_time(),
				<<Runtime as pallet_transaction_payment::Config>::WeightToFee as WeightToFeeT>::weight_to_fee(
					&xt.get_dispatch_info().total_weight()
				),
			);
		}
		let outcome = Executive::apply_extrinsic(xt).expect("ML-DSA-65 extrinsic must be valid");
		assert_ok!(outcome);
	}
	let charged = collected_fees() - fees_before;
	assert!(charged > 0, "the production fee must be non-zero");
	charged
}

fn apply_one_signed_transfer(pair: &Dilithium65Pair, payer: &AccountId32) -> u128 {
	apply_signed_transfers(pair, payer, 1)
}

/// The fee pot the miner will be paid from, as the chain actually holds it
/// (mining-rewards/src/lib.rs:39-41, filled by `TransactionFeesCollector`).
fn collected_fees() -> u128 {
	MiningRewards::collected_fees()
}

/// Planck credited to `who` in this block that no exit can ever mint back out:
/// the sub-quantum residue (absent from every commitment) plus any whole leaf whose
/// quantized value yields zero output under the production fee constraint. The
/// legitimate 4 bps volume fee on exitable leaves is deliberately NOT counted.
fn frozen_planck(first_index: u64, who: &AccountId32) -> u128 {
	let bps = production_fee_bps();
	leaves_for(first_index, who)
		.iter()
		.map(|(_, leaf)| {
			let quanta = leaf_quanta(leaf.amount);
			let residue = leaf.amount % Q;
			if exit_mints_planck(quanta, bps) == 0 {
				residue + quanta * Q
			} else {
				residue
			}
		})
		.sum()
}

/// Every leaf the live ZK tree gained from `first_index` onward that pays `who`.
fn leaves_for(first_index: u64, who: &AccountId32) -> Vec<(u64, Leaf)> {
	(first_index..ZkTree::leaf_count())
		.filter_map(|i| pallet_zk_tree::Leaves::<Runtime>::get(i).map(|leaf| (i, leaf)))
		.filter(|(_, leaf)| leaf.to == *who)
		.collect()
}

// ---------------------------------------------------------------------------------
// (a) two independent credits  +  (b) the leaf commits only floor(amount / 10^10)
// ---------------------------------------------------------------------------------

#[test]
fn production_miner_credit_lands_in_two_leaves_that_commit_only_whole_quanta() {
	let pair = Dilithium65Pair::from_seed_slice(&[42u8; 32]).expect("valid seed");
	let payer = pair.public().into_account();
	let mut ext = production_ext(&payer);

	ext.execute_with(|| {
		assert_production_constants();

		// A real signed transfer through the whole pipeline. The fee is routed into
		// `CollectedFees` by `TransactionFeesCollector` (configs/mod.rs:479-480).
		let fee = apply_one_signed_transfer(&pair, &payer);

		let miner = seat_miner([7u8; 32]);
		let miner_before = Balances::free_balance(&miner);
		let first_new_leaf = ZkTree::leaf_count();
		let (fee_leg, reward_leg) = expected_legs(fee);

		MiningRewards::on_finalize(1);

		// (a) the credit is issued as TWO independent mint_reward calls in ONE
		//     on_finalize: mining-rewards/src/lib.rs:193 and :196.
		let miner_leaves = leaves_for(first_new_leaf, &miner);
		let amounts: Vec<u128> = miner_leaves.iter().map(|(_, l)| l.amount).collect();
		assert_eq!(
			amounts,
			vec![fee_leg, reward_leg],
			"lib.rs:193 and :196 must produce two separate leaves, not one"
		);
		// The balance really moved, in full, to the keyless wormhole address.
		let credited = Balances::free_balance(&miner) - miner_before;
		assert_eq!(credited, fee_leg + reward_leg);

		// (b) each leaf commits floor(amount / 10^10). Proven against the LIVE tree:
		//     the same leaf with its sub-quantum tail shaved off verifies against the
		//     on-chain root exactly as the real one does, so the authoritative
		//     commitment cannot tell the two apart. The remainder is not represented.
		let bps = production_fee_bps();
		// Value that vanishes with no recipient at all: below the leaf quantum, so it
		// is not in the commitment and no exit, fee or burn ever accounts for it.
		let mut quantization_residue = 0u128;
		// Whole leaves that commit a non-zero quantum count yet can never mint anything
		// out, because the circuit fee constraint drives their output to 0 and
		// wormhole/src/lib.rs:787 then skips the slot.
		let mut dead_leaf_value = 0u128;
		// The legitimate 4 bps exit fee taken off leaves that CAN be exited.
		let mut volume_fee = 0u128;

		for (index, leaf) in &miner_leaves {
			let proof = ZkTree::get_merkle_proof(*index).expect("leaf must be in the tree");
			assert!(ZkTree::verify_proof(leaf, &proof), "the real leaf verifies");

			let quanta = leaf_quanta(leaf.amount);
			let quantized = Leaf { amount: quanta * Q, ..leaf.clone() };
			assert!(
				ZkTree::verify_proof(&quantized, &proof),
				"the quantized leaf must verify against the SAME on-chain root: the \
				 sub-quantum remainder of {} planck is absent from the commitment",
				leaf.amount % Q
			);
			quantization_residue += leaf.amount % Q;

			let exitable = exit_mints_planck(quanta, bps);
			assert!(exitable <= leaf.amount);
			if exitable == 0 {
				dead_leaf_value += quanta * Q;
			} else {
				volume_fee += quanta * Q - exitable;
			}
		}

		// At production constants ONE signed ML-DSA-65 transfer buys the miner a fee leg
		// of just over one quantum, and a one-quantum leaf mints ZERO on exit. So the
		// whole fee leg is dead, not merely its sub-quantum tail.
		assert_eq!(
			leaf_quanta(fee_leg),
			1,
			"one production transfer yields a one-quantum fee leg"
		);
		assert_eq!(exit_mints_planck(1, bps), 0);
		assert_eq!(dead_leaf_value, Q, "the fee leg's single quantum can never be exited");

		let frozen = quantization_residue + dead_leaf_value;
		println!(
			"PRODUCTION CONSTANTS - block income {} planck ({} fee leg + {} reward leg)",
			fee_leg + reward_leg,
			fee_leg,
			reward_leg
		);
		println!(
			"PRODUCTION CONSTANTS - quantization residue (no recipient at all): {} planck \
			 ({} fee-leg + {} reward-leg)",
			quantization_residue,
			fee_leg % Q,
			reward_leg % Q
		);
		println!(
			"PRODUCTION CONSTANTS - dead whole-leaf value (committed but zero output): {} planck",
			dead_leaf_value
		);
		println!(
			"PRODUCTION CONSTANTS - PERMANENTLY-FROZEN this block: {} planck (legitimate 4bps \
			 volume fee on the exitable leaf, shown separately: {} planck)",
			frozen, volume_fee
		);
		assert_eq!(quantization_residue, (fee_leg % Q) + (reward_leg % Q));
		assert!(quantization_residue > 0);
		assert!(frozen > 0);
	});
}

// ---------------------------------------------------------------------------------
// (c) a one-quantum leaf yields output 0 under the PRODUCTION fee constraint,
//     and wormhole/src/lib.rs:787 then skips the slot entirely.
// ---------------------------------------------------------------------------------

#[test]
fn production_one_quantum_leaf_yields_zero_output_and_is_skipped_on_exit() {
	let pair = Dilithium65Pair::from_seed_slice(&[42u8; 32]).expect("valid seed");
	let payer = pair.public().into_account();
	let mut ext = production_ext(&payer);

	ext.execute_with(|| {
		let bps = production_fee_bps();
		assert_eq!(bps, 4, "runtime/src/configs/mod.rs:734");

		// 1 * (10000 - 4) = 9996; out * 10000 <= 9996 => out = 0.
		assert_eq!(max_circuit_output_quanta(1, bps), 0);
		// summed_output_amount == 0 -> `continue` at wormhole/src/lib.rs:787, so the
		// slot mints nothing at all.
		assert_eq!(exit_mints_planck(1, bps), 0, "1 quantum in, 0 planck out");
		// 2 * 9996 = 19992 => out = 1. Two quanta is the real floor.
		assert_eq!(max_circuit_output_quanta(2, bps), 1);
		assert_eq!(exit_mints_planck(2, bps), Q);
		for q in 0..2u128 {
			assert_eq!(exit_mints_planck(q, bps), 0);
		}

		// So the minimum spendable leaf is 2 quanta = 2e10 planck = 0.02 UNIT, which is
		// twenty times the existential deposit: a wormhole account can be alive and
		// funded and still hold nothing it can move.
		assert_eq!(2 * Q, 20 * EXISTENTIAL_DEPOSIT);

		// And the whole block's fee take stays frozen until it crosses that floor.
		let fee = apply_one_signed_transfer(&pair, &payer);
		let mut n = 0u128;
		loop {
			n += 1;
			if exit_mints_planck(leaf_quanta(n * fee), bps) > 0 {
				break;
			}
			assert!(n < 1_000, "runaway");
		}
		println!(
			"PRODUCTION CONSTANTS - one signed ML-DSA-65 transfer costs {} planck; {} such \
			 transfers must land in ONE block before any fee income is exitable",
			fee, n
		);
		assert!(n > 1, "a single block's fee take is never exitable on its own");
	});
}

// ---------------------------------------------------------------------------------
// THE SPLIT: two mint_reward calls destroy value one call would have kept.
// This is the assertion the vesting-style fix flips.
//
// The extra quantum is lost on the blocks where the two remainders sum past the
// quantum, so this drives a sweep of real blocks (1..=8 signed transfers each) and
// requires (i) the split is NEVER better than one combined credit, and (ii) at least
// one production block where it is strictly worse.
// ---------------------------------------------------------------------------------

#[test]
fn production_split_into_two_leaves_destroys_an_extra_quantum() {
	let mut strictly_worse = 0u32;
	let mut sampled = 0u32;

	for tx_count in 1..=8u32 {
		let pair = Dilithium65Pair::from_seed_slice(&[42u8; 32]).expect("valid seed");
		let payer = pair.public().into_account();
		let mut ext = production_ext(&payer);

		ext.execute_with(|| {
			let fee = apply_signed_transfers(&pair, &payer, tx_count);
			let miner = seat_miner([7u8; 32]);
			let first_new_leaf = ZkTree::leaf_count();
			let (fee_leg, reward_leg) = expected_legs(fee);

			MiningRewards::on_finalize(1);

			let miner_leaves = leaves_for(first_new_leaf, &miner);
			assert_eq!(miner_leaves.len(), 2, "baseline: the pallet splits the credit");

			// What the chain actually commits, leaf by leaf.
			let committed_split: u128 =
				miner_leaves.iter().map(|(_, l)| leaf_quanta(l.amount)).sum();
			// What ONE combined credit (the vesting-style fix) would have committed.
			let committed_combined = leaf_quanta(fee_leg + reward_leg);

			println!(
				"PRODUCTION CONSTANTS - {} tx/block: committed as two leaves {} quanta, as one \
				 credit {} quanta (residues {} + {} planck)",
				tx_count,
				committed_split,
				committed_combined,
				fee_leg % Q,
				reward_leg % Q
			);
			assert!(
				committed_split <= committed_combined,
				"splitting can never commit MORE than one combined credit"
			);
			sampled += 1;
			if committed_split < committed_combined {
				strictly_worse += 1;
				assert_eq!(
					(committed_combined - committed_split) * Q,
					Q,
					"exactly one extra quantum (1e10 planck) burned by the split"
				);
			}
		});
	}

	println!(
		"PRODUCTION CONSTANTS - the split burned an EXTRA quantum on {}/{} sampled blocks",
		strictly_worse, sampled
	);
	assert!(
		strictly_worse > 0,
		"the split must destroy an extra quantum on at least one production block"
	);
}

// ---------------------------------------------------------------------------------
// RESTART-SURVIVAL (Rule 22) at production constants. The stranded planck is
// COMMITTED CHAIN STATE: the credit is written by `T::Currency::mint_into`
// (mining-rewards/src/lib.rs:230) into pallet_balances, and the leaf is written into
// the pallet-zk-tree storage. Commit the overlay, drain the backend to a raw
// snapshot (the format a node reloads from), drop the running instance, reopen a
// FRESH externalities from that snapshot, and re-read. Nothing is re-derived,
// re-credited, swept or healed.
// ---------------------------------------------------------------------------------

#[test]
fn production_stranded_planck_survives_a_node_restart() {
	let pair = Dilithium65Pair::from_seed_slice(&[42u8; 32]).expect("valid seed");
	let payer = pair.public().into_account();
	let mut ext = production_ext(&payer);

	let (miner, credited, frozen, first_new_leaf) = ext.execute_with(|| {
		let _fee = apply_one_signed_transfer(&pair, &payer);
		let miner = seat_miner([7u8; 32]);
		let miner_before = Balances::free_balance(&miner);
		let first_new_leaf = ZkTree::leaf_count();

		MiningRewards::on_finalize(1);

		let credited = Balances::free_balance(&miner) - miner_before;
		let frozen = frozen_planck(first_new_leaf, &miner);
		(miner, credited, frozen, first_new_leaf)
	});

	let stranded_pre_restart = frozen;
	assert!(stranded_pre_restart > 0);

	// --- the restart itself ----------------------------------------------------
	ext.commit_all().expect("commit the overlay into the storage backend");
	let state_version = ext.state_version;
	let (raw_storage, storage_root) = ext.into_raw_snapshot();
	// `ext` is consumed here: the old node process is gone.
	let mut restarted =
		sp_io::TestExternalities::from_raw_snapshot(raw_storage, storage_root, state_version);

	restarted.execute_with(|| {
		// The keyless miner's balance reloads byte-identical.
		assert_eq!(Balances::free_balance(&miner), credited);
		// The leaves reload too, and still commit only whole quanta.
		let still_stranded = frozen_planck(first_new_leaf, &miner);
		assert_eq!(
			still_stranded, stranded_pre_restart,
			"restart does not recover any of the stranded planck"
		);
		println!(
			"PRODUCTION CONSTANTS - POST-RESTART still permanently frozen: {} planck",
			still_stranded
		);
	});
}

// ---------------------------------------------------------------------------------
// FIX-VERIFYING TEST. FAILS on the CUT as shipped; PASSES once mint_reward combines
// the legs, quantizes down and rolls the residue back into CollectedFees (the pallet
// already owns that helper: mining-rewards/src/lib.rs:302-306 `retain_unminted`).
// This is the discipline pallet-vesting already applies (vesting/src/lib.rs:563-566).
// ---------------------------------------------------------------------------------

#[test]
fn production_no_miner_income_is_stranded_below_the_leaf_quantum() {
	let pair = Dilithium65Pair::from_seed_slice(&[42u8; 32]).expect("valid seed");
	let payer = pair.public().into_account();
	let mut ext = production_ext(&payer);

	ext.execute_with(|| {
		let _fee = apply_one_signed_transfer(&pair, &payer);
		let miner = seat_miner([7u8; 32]);
		let miner_before = Balances::free_balance(&miner);
		let first_new_leaf = ZkTree::leaf_count();

		MiningRewards::on_finalize(1);

		let credited = Balances::free_balance(&miner) - miner_before;
		let committed: u128 = leaves_for(first_new_leaf, &miner)
			.iter()
			.map(|(_, l)| leaf_quanta(l.amount) * Q)
			.sum();

		assert_eq!(
			credited, committed,
			"planck credited to the KEYLESS miner account but not representable in any \
			 ZK-tree leaf is frozen forever: {} planck stranded",
			credited - committed
		);
	});
}
```

**Transcript A1 - BASELINE, unmodified pinned source, production constants**

Verbatim from `production_baseline_transcript.txt`.

```
    Finished `test` profile [unoptimized + debuginfo] target(s) in 4.93s
warning: the following packages contain code that will be rejected by a future version of Rust: trie-db v0.30.0
note: to see what the problems were, use the option `--future-incompat-report`, or run `cargo report future-incompatibilities --id 1`
     Running tests/quantum_freeze_production_constants.rs (target/debug/deps/quantum_freeze_production_constants-c4ade414b605f633)

running 4 tests
test production_miner_credit_lands_in_two_leaves_that_commit_only_whole_quanta ... PRODUCTION CONSTANTS - ML-DSA-65 transfer_keep_alive: 5343 encoded bytes, length fee 5343000000 planck (LENGTH_FEE_MULTIPLIER 1e6, configs/mod.rs:445), ref_time 5543630000 ps -> weight fee 5543630000 planck (IdentityFee, configs/mod.rs:483)
PRODUCTION CONSTANTS - block income 703246474598 planck (10994787000 fee leg + 692251687598 reward leg)
PRODUCTION CONSTANTS - quantization residue (no recipient at all): 3246474598 planck (994787000 fee-leg + 2251687598 reward-leg)
PRODUCTION CONSTANTS - dead whole-leaf value (committed but zero output): 10000000000 planck
PRODUCTION CONSTANTS - PERMANENTLY-FROZEN this block: 13246474598 planck (legitimate 4bps volume fee on the exitable leaf, shown separately: 10000000000 planck)
ok
test production_one_quantum_leaf_yields_zero_output_and_is_skipped_on_exit ... PRODUCTION CONSTANTS - ML-DSA-65 transfer_keep_alive: 5343 encoded bytes, length fee 5343000000 planck (LENGTH_FEE_MULTIPLIER 1e6, configs/mod.rs:445), ref_time 5543630000 ps -> weight fee 5543630000 planck (IdentityFee, configs/mod.rs:483)
PRODUCTION CONSTANTS - one signed ML-DSA-65 transfer costs 10994787000 planck; 2 such transfers must land in ONE block before any fee income is exitable
ok
test production_split_into_two_leaves_destroys_an_extra_quantum ... PRODUCTION CONSTANTS - ML-DSA-65 transfer_keep_alive: 5343 encoded bytes, length fee 5343000000 planck (LENGTH_FEE_MULTIPLIER 1e6, configs/mod.rs:445), ref_time 5543630000 ps -> weight fee 5543630000 planck (IdentityFee, configs/mod.rs:483)
PRODUCTION CONSTANTS - 1 tx/block: committed as two leaves 70 quanta, as one credit 70 quanta (residues 994787000 + 2251687598 planck)
PRODUCTION CONSTANTS - ML-DSA-65 transfer_keep_alive: 5343 encoded bytes, length fee 5343000000 planck (LENGTH_FEE_MULTIPLIER 1e6, configs/mod.rs:445), ref_time 5543630000 ps -> weight fee 5543630000 planck (IdentityFee, configs/mod.rs:483)
PRODUCTION CONSTANTS - 2 tx/block: committed as two leaves 71 quanta, as one credit 71 quanta (residues 1989574000 + 2251687598 planck)
PRODUCTION CONSTANTS - ML-DSA-65 transfer_keep_alive: 5343 encoded bytes, length fee 5343000000 planck (LENGTH_FEE_MULTIPLIER 1e6, configs/mod.rs:445), ref_time 5543630000 ps -> weight fee 5543630000 planck (IdentityFee, configs/mod.rs:483)
PRODUCTION CONSTANTS - 3 tx/block: committed as two leaves 72 quanta, as one credit 72 quanta (residues 2984361000 + 2251687598 planck)
PRODUCTION CONSTANTS - ML-DSA-65 transfer_keep_alive: 5343 encoded bytes, length fee 5343000000 planck (LENGTH_FEE_MULTIPLIER 1e6, configs/mod.rs:445), ref_time 5543630000 ps -> weight fee 5543630000 planck (IdentityFee, configs/mod.rs:483)
PRODUCTION CONSTANTS - 4 tx/block: committed as two leaves 73 quanta, as one credit 73 quanta (residues 3979148000 + 2251687598 planck)
PRODUCTION CONSTANTS - ML-DSA-65 transfer_keep_alive: 5343 encoded bytes, length fee 5343000000 planck (LENGTH_FEE_MULTIPLIER 1e6, configs/mod.rs:445), ref_time 5543630000 ps -> weight fee 5543630000 planck (IdentityFee, configs/mod.rs:483)
PRODUCTION CONSTANTS - 5 tx/block: committed as two leaves 74 quanta, as one credit 74 quanta (residues 4973935000 + 2251687598 planck)
PRODUCTION CONSTANTS - ML-DSA-65 transfer_keep_alive: 5343 encoded bytes, length fee 5343000000 planck (LENGTH_FEE_MULTIPLIER 1e6, configs/mod.rs:445), ref_time 5543630000 ps -> weight fee 5543630000 planck (IdentityFee, configs/mod.rs:483)
PRODUCTION CONSTANTS - 6 tx/block: committed as two leaves 75 quanta, as one credit 75 quanta (residues 5968722000 + 2251687598 planck)
PRODUCTION CONSTANTS - ML-DSA-65 transfer_keep_alive: 5343 encoded bytes, length fee 5343000000 planck (LENGTH_FEE_MULTIPLIER 1e6, configs/mod.rs:445), ref_time 5543630000 ps -> weight fee 5543630000 planck (IdentityFee, configs/mod.rs:483)
PRODUCTION CONSTANTS - 7 tx/block: committed as two leaves 76 quanta, as one credit 76 quanta (residues 6963509000 + 2251687598 planck)
PRODUCTION CONSTANTS - ML-DSA-65 transfer_keep_alive: 5343 encoded bytes, length fee 5343000000 planck (LENGTH_FEE_MULTIPLIER 1e6, configs/mod.rs:445), ref_time 5543630000 ps -> weight fee 5543630000 planck (IdentityFee, configs/mod.rs:483)
PRODUCTION CONSTANTS - 8 tx/block: committed as two leaves 77 quanta, as one credit 78 quanta (residues 7958296000 + 2251687598 planck)
PRODUCTION CONSTANTS - the split burned an EXTRA quantum on 1/8 sampled blocks
ok
test production_stranded_planck_survives_a_node_restart ... PRODUCTION CONSTANTS - ML-DSA-65 transfer_keep_alive: 5343 encoded bytes, length fee 5343000000 planck (LENGTH_FEE_MULTIPLIER 1e6, configs/mod.rs:445), ref_time 5543630000 ps -> weight fee 5543630000 planck (IdentityFee, configs/mod.rs:483)
PRODUCTION CONSTANTS - POST-RESTART still permanently frozen: 13246474598 planck
ok

test result: ok. 4 passed; 0 failed; 0 ignored; 0 measured; 1 filtered out; finished in 0.58s
```

### Suggested Fix

Carry the vesting guard across. In `pallets/mining-rewards/src/lib.rs`:

{% stepper %}
{% step %}

### Add a payout quantum

Add a `PayoutQuantum` constant to `Config`, set to `pallet_wormhole::SCALE_DOWN_FACTOR` in the runtime, exactly as `VestingPayoutQuantum` is (`runtime/src/configs/mod.rs:567`).
{% endstep %}

{% step %}

### Issue one author credit

Issue the author's block income as **one** credit instead of two (`lib.rs:193` + `:196` -> a single `mint_reward`), so the truncation is paid once.
{% endstep %}

{% step %}

### Retain the residue

Quantize that credit down and roll the residue back into `CollectedFees` with the pallet's existing `retain_unminted` (`lib.rs:301-306`), so nothing sub-quantum is ever minted into the keyless account and the residue accrues until it is representable.
{% endstep %}

{% step %}

### Add an integrity test

Add an `integrity_test` mirroring `pallets/vesting/src/lib.rs:297-311` so the invariant cannot regress.
{% endstep %}
{% endstepper %}

The production fix mutant (`production_fix_mutant.patch`) is a working, executed implementation of steps 2 and 3: under it the fix-verifying test passes, that is the 3,246,474,598 planck of quantization residue goes to zero, and the two-leaf split disappears with it.

The treasury leg (`lib.rs:199`) needs no change: `T::Treasury::account_id()` is a pallet account spendable through the treasury pallet, not a keyless wormhole beneficiary.
