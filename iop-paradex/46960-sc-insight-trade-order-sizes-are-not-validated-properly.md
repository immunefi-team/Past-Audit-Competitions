# #46960 \[SC-Insight] trade order sizes are not validated properly

**Submitted on Jun 6th 2025 at 18:09:03 UTC by @gln for** [**IOP | Paradex**](https://immunefi.com/audit-competition/iop-paradex)

* **Report ID:** #46960
* **Report Type:** Smart Contract
* **Report severity:** Insight
* **Target:** <https://github.com/tradeparadex/audit-competition-may-2025/tree/main/paraclear>
* **Impacts:**

## Description

## Brief/Intro

The function settle\_trade\_v2 does not verify maker\_order.size and taker\_order.size fields.

## Vulnerability Details

Function settle\_order\_v2 takes a TradeRequestV2 struct as an argument, struct is defined like this:

```
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

#[derive(Copy, Drop, Serde, Debug, PartialEq)]
pub struct TradeRequestV2 {
    pub id: felt252,
    pub size: felt252,
    pub price: felt252,
    pub traded_at: felt252,
    pub maker_order: OrderV2,
    pub taker_order: OrderV2,
}

```

Let's see at the code of this function from paraclear.cairo:

```
  fn settle_trade_v2(ref self: ContractState, trade: TradeRequestV2) -> felt252 {
            self._assert_only_executor();

            let is_maker_order_reduce_only = trade.maker_order.is_reduce_only;
            let is_taker_order_reduce_only = trade.taker_order.is_reduce_only;

1)          assert(trade.size.into() > 1_u256, Errors::TRADE_SIZE_TOO_SMALL);
            let traded_at: u64 = trade.traded_at.try_into().unwrap();
            assert(traded_at > 0_u64, Errors::TRADE_TIME_INVALID);
            assert(
                trade.maker_order.market == trade.taker_order.market, Errors::TRADE_MARKET_MISMATCH,
            );
            assert(
                trade.maker_order.account != trade.taker_order.account, Errors::TRADE_SAME_ACCOUNT,
            );
            assert(trade.maker_order.side != trade.taker_order.side, Errors::TRADE_SAME_SIDE);

            let delegate = self.Paraclear_market_delegate.read(trade.maker_order.market);
            if (delegate.is_non_zero()) {
                let res = IParaclearDispatcher { contract_address: delegate }
                    .settle_trade_v2(trade);
                return res;
            }
		...
 }
```

1. This the only line where trade.size is checked.

Notice that trade.maker\_order.size and trade.taker\_order.size fields are not checked at all.

## Impact Details

Orders with invalid internal size fields might be processed by Paradex. It may introduce inconsistency between UI and smart contract.

## Proof of Concept

## Proof of Concept

How to reproduce:

To reproduce, we create a malformed trade request with invalid maker\_order.size and taker\_order.size fields and verify settle\_trade\_v2() call was successfull.

1. add test to tests/test\_paraclear\_trades.cairo, see gist link - <https://gist.github.com/gln7/fabee7f76d681c64ba5892c168d339c7>
2. run test:

```
$ snforge test test_settle_trade_issue
...
Collected 1 test(s) from paradex_paraclear package
Running 1 test(s) from tests/
XXXXKE settle_trade_v2 result 1

```
