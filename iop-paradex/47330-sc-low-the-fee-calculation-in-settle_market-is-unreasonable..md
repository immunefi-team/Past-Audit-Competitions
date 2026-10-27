# #47330 \[SC-Low] The fee calculation in \`settle\_market\` is unreasonable.

**Submitted on Jun 12th 2025 at 16:57:58 UTC by @shaflow1 for** [**IOP | Paradex**](https://immunefi.com/audit-competition/iop-paradex)

* **Report ID:** #47330
* **Report Type:** Smart Contract
* **Report severity:** Low
* **Target:** <https://github.com/tradeparadex/audit-competition-may-2025/tree/main/paraclear>
* **Impacts:**
  * Theft of unclaimed yield

## Description

## Brief/Intro

In the `settle_market` function, only the taker fee from the account is charged, which is unreasonable:

1. Charging only the taker fee may result in a negative total fee, causing a loss to the fee account.
2. In a settle trade, the user acts as both the taker and the maker, so both sides' fees should be charged to ensure that the total fee is not negative and to prevent losses to the fee account.

## Vulnerability Details

```rust
        fn _settlement_fee_payments(
            ref self: ContractState,
            account: ContractAddress,
            account_state: @AccountState,
            pending_token_balance: i128, // usdc 
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
            let settlement_token_address = self.getSettlementTokenAsset();

            let fee_in_settlement_token = div_128(fee, settlement_token_price);
            if fee_commission == 0 {
                self
                    .token
                    .upsert_asset_balance(
                        fee_account, settlement_token_address, fee_in_settlement_token.into(),
                    );
            } else {
                let fee_commission_in_settlement_token = div_128(
                    fee_commission, settlement_token_price,
                );
                self
                    .token
                    .upsert_asset_balance(
                        referrer,
                        settlement_token_address,
                        fee_commission_in_settlement_token.into(),
                    );
                self
                    .token
                    .upsert_asset_balance(
                        fee_account,
                        settlement_token_address,
                        (fee_in_settlement_token - fee_commission_in_settlement_token).into(),
                    );
            }

            self.emit(AccountComponent::Event::Fee(Fee { account: account, fee: fee.into() }));
            return balance_after_fee;
        }
```

`_settlement_fee` only charges the taker fee, which is inappropriate.

In a settlement trade, the user acts as both the taker and the maker, so charging only the taker fee is unreasonable.

Moreover, if one of the fee rates (taker or maker) is negative, for example, if the taker fee rate is negative, then the settlement operation would result in a negative fee, causing the `feeAccount` to lose funds.

## Impact Details

The rewards from `fee_account` can be stolen under certain conditions

## References

<https://github.com/tradeparadex/audit-competition-may-2025/blob/0eb81b26a67666c399b4e16b39a96c19848ab7fd/paraclear/src/paraclear/paraclear.cairo#L1981>

## Proof of Concept

## Proof of Concept

1. Suppose the user's `taker_fee_rate` is -1% and the `maker_fee_rate` is 1.2%. A malicious actor can place an order and immediately settle it to steal funds from the `fee_account`.
2. Assume that when placing the order, the user acts as the taker — in this case, the taker *receives* a 1% fee. If the user acts as the maker, they *pay* a 1.2% fee.
3. Then the user immediately settles the trade, earning another 1% fee from the settlement operation.
4. Summary:
   * If the user is the **taker** when placing the order:
     * User profit = 1% (order) + 1% (settlement) = **2%**
     * `fee_account` loss = 1.2% (maker fee) - 1% (taker fee rebate) = **0.2%**
   * If the user is the **maker** when placing the order:
     * User loss = 1.2% (maker fee) - 1% (settlement rebate) = **0.2%**
     * `fee_account` gains 1.2% maker fee

In the first case, the user gains a large profit (2%). If the second case occurs, the loss is limited to only 0.2%. By repeatedly placing orders (e.g., 10 times), the user only needs to be the taker once to offset all losses and still profit, effectively draining value from the `fee_account`.
