# #46639 \[SC-Low] The \`\_settlement\_fee\_payments\` function contains a calculation error that leads to abnormal user balances.

**Submitted on Jun 2nd 2025 at 18:31:46 UTC by @shaflow1 for** [**IOP | Paradex**](https://immunefi.com/audit-competition/iop-paradex)

* **Report ID:** #46639
* **Report Type:** Smart Contract
* **Report severity:** Low
* **Target:** <https://github.com/tradeparadex/audit-competition-may-2025/tree/main/paraclear>
* **Impacts:**
  * Permanent freezing of funds

## Description

## Brief/Intro

The `_settlement_fee_payments` function contains an error in the calculation of `balance_after_fee` because the fee is not normalized from value terms to the settlement token (USDC). This leads to incorrect user balance calculations when the `settlement_token_price` is not equal to 1.

## Vulnerability Details

```rust
        fn _settlement_fee_payments(
            ref self: ContractState,
            account: ContractAddress,
            account_state: @AccountState,
            pending_token_balance: i128,
            token_balance_address: ContractAddress,
            trade_size: i128,
            trade_price: i128,
            settlement_token_price: NonZero<i128>,
            asset: @PerpetualMarketAsset,
        ) -> i128 {
            let fee_account = self.getFeeAccount();

            let base_fee = asset
                .calculate_fee(
                    *account_state.asset_data,
                    trade_size,
                    trade_price,
                    false,
                    account_state.fee_rates,
                );
            let (fee, referrer, fee_commission) = account_state
                .get_trade_fee_and_referral_commission(base_fee);
            let balance_after_fee = pending_token_balance - fee;

            self
                .token
                .write_asset_balance(account, token_balance_address, balance_after_fee.into());
```

In the `_settlement_fee_payments` function, `pending_token_balance` represents the amount of settlement tokens (USDC) the user should hold after settling funding and PnL, and it is denominated in USDC with 8 decimals.

However, the `fee` is calculated using `trade_price` and `trade_size`, and its unit is in value terms (i.e., not yet normalized to USDC).

These two values (`pending_token_balance` and `fee`) cannot be directly subtracted. The `fee` must first be divided by the `settlement_token_price` to convert it into settlement token units (USDC) before it can be subtracted from `pending_token_balance`.

Failing to do this results in an incorrect calculation of `balance_after_fee`, which is then incorrectly written to the user's balance.

## Impact Details

The direction of the impact depends on the `settlement_token_price`:

* **If `settlement_token_price` > 1**, then `balance_after_fee` will be **underestimated**, resulting in **excessive fees being deducted from the user**. Moreover, the **overcharged portion is not credited to the `fee_account`**, because later, when updating the `fee_account` balance, the `fee` is divided by `settlement_token_price` to convert it to USDC. As a result, this **extra deducted amount is effectively lost and permanently locked** in the system.
* **If `settlement_token_price` < 1**, then `balance_after_fee` will be **overestimated**, meaning the **user pays less fee than intended**. However, the **full fee amount (in USDC terms) is still added to the `fee_account`**, leading to the **total USDC balance recorded in the system exceeding the actual amount of USDC held by the contract**. This **can potentially cause withdrawal failures** later, as the system assumes it holds more USDC than it actually does.

Since this deviation occurs every time the settlement function is called — unless `settlement_token_price` is exactly 1 — it will **accumulate over time**. In practice, the price of USDC is usually slightly below 1, so the **system tends to over-credit users**, which makes the risk of withdrawal failures even more likely. During periods when USDC depegs, the issue becomes even more pronounced and dangerous.

## References

<https://github.com/tradeparadex/audit-competition-may-2025/blob/0eb81b26a67666c399b4e16b39a96c19848ab7fd/paraclear/src/paraclear/paraclear.cairo#L1991>

## Proof of Concept

## Proof of Concept

Every time a settlement occurs with `settlement_token_price` not equal to 1, `balance_after_fee` cannot be accurately calculated:

1. The user calls `settlement` to close a position in a market.
2. Call `_settlement_fee_payments` processes the fee. After settling PnL and funding, the user's `pending_token_balance` is `100 * 10^8`, and the `settlement_token_price` is 1.001.
3. Based on `trade_size`, `trade_price`, and `fee_rates`, the fee is calculated to be `10^8` in value units. There is no referral fee.
4. The function calculates `balance_after_fee = 100 * 10^8 - 10^8 = 99 * 10^8`, and the user's USDC balance is updated to `99 * 10^8`.
5. The `fee_account` balance is increased by `fee_in_settlement_token = 10^8 / 1.001 ≈ 0.999 * 10^8`.
6. The user is overcharged by approximately `10^6` USDC, and this excess fee is **not** credited to the `fee_account`, resulting in the funds being **permanently locked**.
