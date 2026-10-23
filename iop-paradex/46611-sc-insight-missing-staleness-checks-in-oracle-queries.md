# #46611 \[SC-Insight] Missing staleness checks in oracle queries

**Submitted on Jun 2nd 2025 at 11:33:50 UTC by @gln for** [**IOP | Paradex**](https://immunefi.com/audit-competition/iop-paradex)

* **Report ID:** #46611
* **Report Type:** Smart Contract
* **Report severity:** Insight
* **Target:** <https://github.com/tradeparadex/audit-competition-may-2025/tree/main/paraclear>
* **Impacts:**
  * Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield

## Description

## Brief/Intro

Current implementation does not check last\_updated\_timestamp when fetching oracle price ticks.

## Vulnerability Details

To query oracle price the following function is called from oracle/src/oracle.cairo:

```
   fn get_value(self: @ContractState, market: felt252) -> TickData {
            let tick_data_price = self.latest_tick_data.read(market);
            let timestamp = self.latest_updated_timestamp.read();
            TickData {
                asset_key: tick_data_price.asset_key,
                asset_value: tick_data_price.asset_value,
                decimals: tick_data_price.decimals,
                last_updated_timestamp: timestamp,
            }
        }

```

As you can see, it returns asset price and last\_updated\_timestamp.

Let's see how price data is actually used, code from paraclear/src/paraclear.cairo:

```
  fn getAccountUnrealizedPnlByMarket(
            self: @ContractState, account: ContractAddress, market: felt252,
        ) -> felt252 {
            let oracle_address = self.getOracleContract();
            let oracle_dispatcher = IParaclearOracleDispatcher { contract_address: oracle_address };
1)          let mark_price_tick = oracle_dispatcher.get_value(market);
            let settlement_token_tick = oracle_dispatcher.get_value(SETTLEMENT_TOKEN_ASSET_KEY);
3)          let mark_price: i128 = mark_price_tick.asset_value.try_into().unwrap();
            let settlement_token_price_i: i128 = settlement_token_tick
                .asset_value
                .try_into()
                .unwrap();
            self
                .perpetual_future
                .calculate_unrealized_pnl(
                    account, market, mark_price, settlement_token_price_i.try_into().unwrap(),
                )
                .into()
        }

```

1. Price tick is fetched
2. There is no verification that price tick can be stale, the tick's field 'last\_updated\_timestamp' is not verified

As a result, stale price might be used in critical calculations.

## Impact Details

Users will rely on asset price information, that is believed to be fresh. It could lead to erroneous decisions and potential loss of funds.

## Proof of Concept

## Proof of Concept

In this test, we set price tick, then advance timestamp to 2 hours.

Second call to getAccountUnrealizedPnlByMarket() thus should fail, as the price is outdated, but it does not.

How to reproduce:

1. add test to src/paraclear/tests/test\_paraclear.cairo

```
#[test]
fn test_oracle_stale_price_issue() {
    let (_, paraclear_dispatcher, oracle_dispatcher) = setup_paraclear_with_oracle();
    let futures_dispatcher = IPerpetualFutureDispatcher {
        contract_address: paraclear_dispatcher.contract_address,
    };

    let mut perpetual_asset = get_perpetual_asset();
    start_cheat_caller_address(futures_dispatcher.contract_address, CONFIGURATOR());
    futures_dispatcher.create_perpetual_asset(perpetual_asset);
    stop_cheat_caller_address(futures_dispatcher.contract_address);

    let market = perpetual_asset.market;
    let account: ContractAddress = 'TEST_ACCOUNT'.try_into().unwrap();
    let current_time = get_block_timestamp();
    
    start_cheat_caller_address(oracle_dispatcher.contract_address, EXECUTOR());
    oracle_dispatcher.set_value(
        TickData {
            asset_key: market,
            asset_value: 50000_00000000,
            decimals: DECIMALS.try_into().unwrap(),
            last_updated_timestamp: current_time.into(),
        }
    );
    stop_cheat_caller_address(oracle_dispatcher.contract_address);

    let mut unrealized_pnl = paraclear_dispatcher.getAccountUnrealizedPnlByMarket(account, market);
    println!("XXXXKE get pnl for timestamp {}, pnl {}", current_time, unrealized_pnl);

    let new_time = get_block_timestamp() + 7200; // + 2 hours
    start_cheat_block_timestamp_global(new_time);
    unrealized_pnl = paraclear_dispatcher.getAccountUnrealizedPnlByMarket(account, market);
    println!("XXXXKE get pnl for timestamp {}, pnl {}", get_block_timestamp(), unrealized_pnl);
}

```

2. run the test:

```
$ snforge test test_oracle_stale_price_issue
...
Running 1 test(s) from tests/
XXXXKE get pnl for timestamp 0, pnl 0
XXXXKE get pnl for timestamp 7200, pnl 0
```
