# #46989 \[SC-Insight] Invalid trade side check

**Submitted on Jun 7th 2025 at 11:54:53 UTC by @shaflow1 for** [**IOP | Paradex**](https://immunefi.com/audit-competition/iop-paradex)

* **Report ID:** #46989
* **Report Type:** Smart Contract
* **Report severity:** Insight
* **Target:** <https://github.com/tradeparadex/audit-competition-may-2025/tree/main/paraclear>
* **Impacts:**

## Description

## Brief/Intro

The `settle_trade_v2` function checks whether the `side` values in the orders are opposite, but this check may be invalid. Since the `side` in the orders can be any value, the direction is only considered positive when `side` equals 1, and negative for all other values. As a result, two different `side` values that are not equal to 1 can still pass the check, even though their actual trading directions are the same.

## Vulnerability Details

```rust
#[derive(Copy, Drop, Serde, Debug, PartialEq)]
pub struct OrderV2 {
    pub account: ContractAddress,
    pub market: felt252,
    pub side: felt252,
    pub orderType: felt252,
    pub size: felt252,
    pub price: felt252,
    pub signature_timestamp: felt252,
    pub is_reduce_only: bool,
}

        fn settle_trade_v2(ref self: ContractState, trade: TradeRequestV2) -> felt252 {
            ...
            assert(trade.maker_order.side != trade.taker_order.side, Errors::TRADE_SAME_SIDE);
            ...
        }

    #[inline(always)]
    pub fn _get_sign_from_side(side: felt252) -> i128 {
        // Implement this function based on your side definitions
        if side == 1 {
            1
        } else {
            -1
        }
    }

```

The OrderV2.side field can take any value, but only 1 is treated as the positive direction, while all other values are treated as the negative direction.

The settle\_trade\_v2 function currently only checks whether the sides are unequal, which is insufficient. Instead, it should explicitly verify:

1. One side must be 1
2. The other side must be not 1

## Impact Details

Two orders with the same direction could be erroneously executed, leading to an imbalance between long and short positions in the system.

## References

<https://github.com/tradeparadex/audit-competition-may-2025/blob/0eb81b26a67666c399b4e16b39a96c19848ab7fd/paraclear/src/paraclear/paraclear.cairo#L492> <https://github.com/tradeparadex/audit-competition-may-2025/blob/0eb81b26a67666c399b4e16b39a96c19848ab7fd/paraclear/src/perpetual/future.cairo#L317>

## Proof of Concept

## Proof of Concept

1. There are two user orders with side values set to -1 and 2 respectively.
2. The executor calls settle\_trade\_v2 to process the orders.
3. Due to an ineffective trade side validation check, two short positions are opened instead of one short and one long position.
