# 89139 bc high non quantized credits to wormhole addresses permanently lock sub quantum value

**Submitted on Aug 17th 2026 at 12:47:05 UTC by @Kangaroo59250 for** [**Audit Comp | Quantus**](https://immunefi.com/audit-competition/audit-comp-quantus)

* **Report ID:** #89139
* **Report Type:** Blockchain/DLT
* **Report severity:** High
* **Target:** <https://github.com/immunefi-team/audit-comp-quantus-chain>
* **Impacts:**
  * Permanent freezing of funds

## Description

## Summary

Quantus tracks native balances with 12 decimal places, but wormhole ZK-tree leaves commit amounts with only two decimal places:

```
committed_amount = floor(amount / 10^10)
```

The transfer-proof recorder accepts every nonzero credit, including amounts that are not multiples of the `0.01 QUAN` leaf quantum. The full balance reaches the recipient, but the leaf hash commits only the truncated amount.

When the recipient is a Poseidon-derived wormhole address, the omitted remainder cannot be spent by signature and is not covered by any valid ZK leaf. It is therefore permanently unwithdrawable.

A sub-quantum credit such as `0.009 QUAN` is especially severe: it is above the runtime existential deposit, so the transfer succeeds, but its leaf commits to an input amount of zero. The entire credit becomes unexitable.

## Vulnerability Details

Native balance transfers emit events that are scanned by `WormholeProofRecorderExtension`. The extension passes each recorded credit to:

```rust
record_transfer_proof(asset_id, from, to, amount)
```

In `pallets/wormhole/src/lib.rs:1341-1390`, the recorder rejects only an exact zero:

```rust
if amount.is_zero() {
    return false;
}
```

All other native credits reach `record_transfer`, which stores the full-precision amount, advances the recipient’s transfer count, and inserts a ZK-tree leaf.

However, `pallets/zk-tree/src/tree.rs:135-149` hashes the amount as:

```rust
let amount_quantized =
    u32::try_from(amount_u128 / AMOUNT_SCALE_DOWN_FACTOR)
        .unwrap_or(u32::MAX);
```

`AMOUNT_SCALE_DOWN_FACTOR` is `10^10`, making one circuit quantum equal to `0.01 QUAN`. Floor division silently removes any smaller remainder from the Merkle commitment.

The wormhole circuit uses the resulting quantized `u32` as its private `input_amount`. For a credit of:

```
amount = n × 0.01 QUAN + remainder
```

only `n × 0.01 QUAN` is represented by the leaf hash. No valid membership witness can cover the remainder.

This is permanent when the recipient is a wormhole address. Such an address is derived from a Poseidon preimage rather than a Dilithium signing key, so the holder must use the ZK exit mechanism to recover its balance.

The relevant runtime units are:

```
1 QUAN              = 10^12 base units
Leaf quantum        = 10^10 base units = 0.01 QUAN
Existential deposit = 10^9 base units  = 0.001 QUAN
```

A fresh wormhole address can therefore receive a valid balance that quantizes to zero in the ZK tree.

## Impact

For every non-aligned wormhole credit, the amount omitted from the commitment is:

```
locked_remainder = amount mod 10^10
```

Examples:

| Credit                        |          Leaf input | Result                                                                       |
| ----------------------------- | ------------------: | ---------------------------------------------------------------------------- |
| `0.009 QUAN`                  |                 `0` | Entire `0.009 QUAN` is locked                                                |
| `1.009 QUAN`                  |         `1.00 QUAN` | `0.009 QUAN` is absent from the commitment                                   |
| `10.005 QUAN`                 |        `10.00 QUAN` | `0.005 QUAN` is absent from the commitment                                   |
| Exact multiple of `0.01 QUAN` | Full aligned amount | No quantization remainder; the circuit’s volume-fee constraint still applies |

Exact multiples do not suffer truncation, but they remain subject to the circuit’s fee constraint:

```
output × 10000 ≤ input × 9996
```

At the configured four-basis-point rate, a `0.01 QUAN` input has a maximum output of zero, while a `0.02 QUAN` input can output only `0.01 QUAN`. Deposits below `0.02 QUAN` are therefore fully unexitable, and a deposit of exactly `0.02 QUAN` loses 50%.

This small-deposit behavior amplifies the impact but is separate from the quantization remainder that is the primary root cause of this finding.

The unintended loss per larger credit is less than `0.01 QUAN`, but credits between the `0.001 QUAN` existential deposit and `0.01 QUAN` leaf quantum can lose 100% of their value.

The failure is silent:

* The transfer succeeds.
* The recipient’s balance increases by the full amount.
* A transfer event and ZK-tree leaf are created.
* No warning indicates that some or all of the value is unexitable.

Sub-quantum credits also insert a leaf whose hash commits to a zero amount, unnecessarily advancing the recipient’s transfer count and permanently growing the ZK tree.

## Recommendation

Prevent unrepresentable value from reaching keyless wormhole addresses.

Explicit wormhole-deposit and protocol-payout paths should validate, before transferring, that:

```
amount >= minimum usable input
amount % AMOUNT_SCALE_DOWN_FACTOR == 0
```

Official wallet tooling should align wormhole deposits to the `0.01 QUAN` quantum and leave any remainder in the sender’s signature-controlled account.

Ordinary accounts and wormhole addresses share the same `AccountId` type, so the generic recorder cannot reliably distinguish them. If arbitrary transfers to wormhole addresses must remain supported, the protocol should either:

* increase the circuit’s amount precision; or
* accumulate per-recipient remainders and create leaves only for complete quanta, with an explicit policy for terminal dust.

At minimum, a credit that quantizes to zero should not insert a worthless leaf, and the runtime should emit a dedicated warning event. This would prevent unnecessary state growth and make the loss visible, although it would not recover value already transferred.

## References

* `runtime/src/transaction_extensions.rs:309-398` — balance-event proof recording
* `pallets/wormhole/src/lib.rs:1280-1330` — transfer count and leaf insertion
* `pallets/wormhole/src/lib.rs:1341-1390` — nonzero credit acceptance
* `pallets/zk-tree/src/tree.rs:135-149` — amount quantization
* `wormhole/circuit/src/zk_merkle_proof.rs:246-255` — quantized private input
* `wormhole/circuit/src/zk_merkle_proof.rs:491-499` — exit fee constraint
* `runtime/src/lib.rs:97-102` — unit and existential deposit
* Commit: `3b243b870a5442a6e5f443056f67bbba87d472f5`

## Proof of Concept

Add the following test to `pallets/wormhole/src/tests.rs`:

```rust
#[test]
fn poc_fractional_deposits_are_permanently_locked() {
    use qp_wormhole::TransferProofRecorder;

    new_test_ext().execute_with(|| {
        let from = account_id(1);
        let to = account_id(9100);
        let fee_bps = VolumeFeeRateBps::get() as u128;
        let quantum =
            pallet_zk_tree::tree::AMOUNT_SCALE_DOWN_FACTOR;

        // Case A: 0.009 QUAN is above the runtime ED but below
        // the 0.01 QUAN leaf quantum.
        let dust: u128 = 9_000_000_000;

        assert!(
            <Wormhole as TransferProofRecorder<
                AccountId,
                u32,
                u128,
            >>::record_transfer_proof(
                None,
                from.clone(),
                to.clone(),
                dust,
            ),
            "the recorder accepts a sub-quantum credit"
        );

        let leaf =
            ZkTree::leaf(0).expect("the credit must insert a leaf");

        assert_eq!(
            leaf.amount,
            dust,
            "storage retains the full-precision amount"
        );

        let input_quanta_a = dust / quantum;
        assert_eq!(
            input_quanta_a,
            0,
            "the leaf hash commits to zero input quanta"
        );

        let max_output_a =
            input_quanta_a * (10_000 - fee_bps) / 10_000;

        assert_eq!(
            max_output_a,
            0,
            "the accepted credit has no exitable value"
        );

        assert_eq!(
            crate::TransferCount::<Test>::get(to),
            1,
            "the worthless leaf still advances the transfer count"
        );

        // Case B: exactly one quantum is aligned, but the fee
        // constraint still makes its maximum output zero.
        let input_quanta_b = quantum / quantum;
        let max_output_b =
            input_quanta_b * (10_000 - fee_bps) / 10_000;

        assert_eq!(
            max_output_b,
            0,
            "a one-quantum input cannot produce a nonzero output"
        );

        // Case C: the fractional part of 1.009 QUAN is omitted.
        let amount_c: u128 = UNIT + 9_000_000_000;
        let input_quanta_c = amount_c / quantum;

        assert_eq!(
            input_quanta_c,
            100,
            "the fractional part is truncated"
        );

        let locked_remainder =
            amount_c - input_quanta_c * quantum;

        assert_eq!(
            locked_remainder,
            9_000_000_000,
            "the 0.009 QUAN remainder is not committed"
        );
    });
}
```

Run:

```
cargo test -p pallet-wormhole --lib poc_fractional_deposits_are_permanently_locked
```

Verified on commit `3b243b870a5442a6e5f443056f67bbba87d472f5`:

```
test tests::private_batch_proof_tests::poc_fractional_deposits_are_permanently_locked ... ok

test result: ok. 1 passed; 0 failed
```

The PoC demonstrates that the recorder accepts a sub-quantum amount, inserts a leaf, and advances the transfer count even though the committed input amount and maximum circuit output are both zero.
