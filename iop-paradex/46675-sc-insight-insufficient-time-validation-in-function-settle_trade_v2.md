# #46675 \[SC-Insight] Insufficient Time Validation in function settle\_trade\_v2

**Submitted on Jun 3rd 2025 at 08:40:59 UTC by @Catchme for** [**IOP | Paradex**](https://immunefi.com/audit-competition/iop-paradex)

* **Report ID:** #46675
* **Report Type:** Smart Contract
* **Report severity:** Insight
* **Target:** <https://github.com/tradeparadex/audit-competition-may-2025/tree/main/paraclear>
* **Impacts:**
  * Griefing (e.g. no profit motive for an attacker, but damage to the users or the protocol)
  * Contract fails to deliver promised returns, but doesn't lose value

## Description

## Brief/Intro

The `settle_trade_v2` function in Paraclear has a timestamp validation vulnerability. The function only checks if the trade timestamp is greater than zero (`traded_at > 0_u64`), without verifying that it's recent or valid relative to the current block time. This enables various time-based attacks including trade replay attacks, stale trade execution, and price manipulation that could lead to direct financial losses for users through unfair trade execution and potential theft of funds.

## Vulnerability Details

The vulnerable code exists in the `settle_trade_v2` function:

```cairo
let traded_at: u64 = trade.traded_at.try_into().unwrap();
assert(traded_at > 0_u64, Errors::TRADE_TIME_INVALID);
```

This validation is severely inadequate for several reasons:

1. **No recency check**: The function doesn't compare `traded_at` with the current block timestamp to ensure the trade is recent. As long as `traded_at > 0`, any timestamp (even from years ago) would be accepted.
2. **No maximum age enforcement**: There is no mechanism to enforce a maximum staleness period for trades. This means executors can hold onto trades and execute them much later when market conditions are favorable to them.
3. **No protection against future timestamps**: Since there's no upper bound check, trade timestamps set in the future would also be accepted.

## Impact Details

1. **Trade Replay Attacks**: An attacker could potentially resubmit old trades, especially if IDs can be reused or manipulated. This would result in unauthorized duplicate trades.
2. **Price Manipulation**: By holding onto trades and executing them at opportune moments with manipulated oracle data, a malicious executor could generate artificial profits at the expense of users.

## Proof of Concept

## Proof of Concept

```
     let trade_request = TradeRequestV2 {
            id: 1000,
            size: 2_u128.into(),
            price: 800000_u128.into(),
            traded_at: 11111111111, // history timestamp
            maker_order: OrderV2 {
                account: maker_account,
                market: market,
                side: SELL_SIDE,
                orderType: 0,
                size: 2_u128.into(),
                price: 800000_u128.into(),
                signature_timestamp: get_block_timestamp().into(),
                is_reduce_only: false,
            },
            taker_order: OrderV2 {
                account: taker_account,
                market: market,
                side: BUY_SIDE,
                orderType: 0,
                size: 2_u128.into(),
                price: 800000_u128.into(),
                signature_timestamp: get_block_timestamp().into(),
                is_reduce_only: false,
            },
        };

```

The attacker calls settle\_trade\_v2 with this trade:

```
paraclear.settle_trade_v2(manipulated_trade);
```
