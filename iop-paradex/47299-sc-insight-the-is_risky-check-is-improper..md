# #47299 \[SC-Insight] The \`is\_risky\` check is improper.

**Submitted on Jun 12th 2025 at 13:24:06 UTC by @shaflow1 for** [**IOP | Paradex**](https://immunefi.com/audit-competition/iop-paradex)

* **Report ID:** #47299
* **Report Type:** Smart Contract
* **Report severity:** Insight
* **Target:** <https://github.com/tradeparadex/audit-competition-may-2025/tree/main/paraclear>
* **Impacts:**
  * Griefing (e.g. no profit motive for an attacker, but damage to the users or the protocol)

## Description

## Brief/Intro

The `is_risky` check allows a trade to reduce position size even when the account has insufficient margin, as long as `account_value >= 0`. However, the protocol does not take `trade_fee` into account, which is deducted at the end of the trade. As a result, if the fee is not considered during the check, the post-trade `account_value` might fall below zero, potentially causing bad debt.

## Vulnerability Details

```rust
        fn is_risky(
            self: @AccountState,
            position_size_before: felt252,
            position_size_after: felt252,
            trade_fee: i128,
        ) -> bool {
            let asset_value = self.get_asset_value();
            let (margin_requirement, total_upnl) = self
                .margin_requirement_and_total_upnl(MARGIN_CHECK_INITIAL);
            let account_value: i128 = asset_value + total_upnl;
            let free_balance = account_value - margin_requirement - trade_fee;
            if free_balance >= 0 {
                return false;
            }
            let abs_pos_before = abs_128(position_size_before.try_into().unwrap());
            let abs_pos_after = abs_128(position_size_after.try_into().unwrap());
            let position_decrease = abs_pos_before - abs_pos_after;
            if position_decrease >= 0 && account_value >= 0 {
                return false;
            }
```

The `is_risky` function allows reducing position size even when the account's health is insufficient, but it does not allow the trade to result in bad debt (i.e., account value < 0 after the trade). However, during the check, it does not take into account the trading fee that is immediately deducted from the account balance at the end of the trade. This means the account balance after the trade could likely fall below zero.

Therefore, fees should be taken into account to prevent bad debt from occurring after the trade.

```diff
-           if position_decrease >= 0 && account_value >= 0 {
+           if position_decrease >= 0 && account_value + trade_fee>= 0 {
                return false;
            }
```

## Impact Details

The `is_risky` function may fail in certain cases and cannot prevent bad debt from occurring after a trade.

## References

<https://github.com/tradeparadex/audit-competition-may-2025/blob/0eb81b26a67666c399b4e16b39a96c19848ab7fd/paraclear/src/paraclear/paraclear.cairo#L599> <https://github.com/tradeparadex/audit-competition-may-2025/blob/0eb81b26a67666c399b4e16b39a96c19848ab7fd/paraclear/src/account/account.cairo#L635>

## Proof of Concept

## Proof of Concept

A user can deliberately choose specific trades under certain conditions to cause the protocol to incur bad debt.

1. Position 1 of user1 is unhealthy, and user1 places an order to reduce the position.
2. During the `is_risky` risk check, `account_value < trade_fee`, but the check still passes.
3. At the end of the trade, `trade_fee` is deducted from user1’s account, causing `account_value` to fall below 0 and resulting in bad debt.
