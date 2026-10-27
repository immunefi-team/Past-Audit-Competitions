# #47318 \[SC-Insight] If the counterparty happens to be their own referrer, the protocol does not take the referral fee into account during the risk check.

**Submitted on Jun 12th 2025 at 15:04:43 UTC by @shaflow1 for** [**IOP | Paradex**](https://immunefi.com/audit-competition/iop-paradex)

* **Report ID:** #47318
* **Report Type:** Smart Contract
* **Report severity:** Insight
* **Target:** <https://github.com/tradeparadex/audit-competition-may-2025/tree/main/paraclear>
* **Impacts:**
  * Smart contract unable to operate due to lack of token funds

## Description

## Brief/Intro

Suppose in a trade, the taker's referrer happens to be the maker, or the maker's referrer happens to be the taker. Then at the end of the trade, the taker or maker will receive a referral fee from the counterparty. However, this fee is not considered in the `is_risky` check, which can cause some otherwise valid trades to fail.

## Vulnerability Details

```rust
            let maker_token_balance = *maker_state
                .token_balances
                .at(maker_state.settlement_token_index);
            let taker_token_balance = *taker_state
                .token_balances
                .at(taker_state.settlement_token_index);
            // Note: _fee_payments will also write maker and taker token balance into the storage to
            // include pnl and funding previously calculated
            self
                ._fee_payments(
                    maker_account,
                    @maker_token_balance,
                    maker_trade_fee_in_settlement_token,
                    maker_referrer,
                    maker_fee_commission_in_settlement_token,
                    taker_account,
                    @taker_token_balance,
                    taker_trade_fee_in_settlement_token,
                    taker_referrer,
                    taker_fee_commission_in_settlement_token,
                );
```

Suppose in a trade, the maker happens to be the taker's referrer. Then, in the `fee_payment` step, the maker will receive a referral fee from the taker. However, this referral fee is not considered in the earlier `is_risky` check. Since a potential source of funds is omitted from the check, this may cause a trade that should be executable to be incorrectly rejected.

## Impact Details

A trade that could have been executed may become non-executable due to the referral fee being omitted from consideration.

## References

<https://github.com/tradeparadex/audit-competition-may-2025/blob/0eb81b26a67666c399b4e16b39a96c19848ab7fd/paraclear/src/paraclear/paraclear.cairo#L1911>

## Proof of Concept

## Proof of Concept

1. In a trade, the taker's referrer is the maker.
2. The maker is supposed to pay a total fee of `fee1`, and the taker is supposed to pay `fee2`. Since the maker is the taker's referrer, the taker needs to pay an additional referral fee `fee3` to the maker.
3. However, in the `is_risky` check for the maker, the protocol only uses `account_value - fee1 - margin_requirement`, whereas it should actually use `account_value - fee1 + fee3 - margin_requirement`.
