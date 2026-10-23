# #47291 \[SC-Insight] Serveal bugs in function set\_prices\_and\_funding\_snapshot

**Submitted on Jun 12th 2025 at 10:24:08 UTC by @Catchme for** [**IOP | Paradex**](https://immunefi.com/audit-competition/iop-paradex)

* **Report ID:** #47291
* **Report Type:** Smart Contract
* **Report severity:** Insight
* **Target:** <https://github.com/tradeparadex/audit-competition-may-2025/tree/main/oracle>
* **Impacts:**
  * Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield

## Description

## Brief/Intro

The `set_prices_and_funding_snapshot` function in ParaclearOracle contains multiple critical vulnerabilities related to timestamp handling, allowing executors to update price data with inconsistent or stale timestamps. This creates a significant risk of price manipulation, which could lead to improper liquidations, incorrect funding payments, and potential theft of user funds through manipulated settlement prices in this DeFi protocol.

## Vulnerability Details

The function contains three major vulnerabilities:

1. **Timestamp Regression**: The function allows updating oracle data with older timestamps than what's currently stored:

```cairo
// Current code just overwrites without checks
if new_prices.len() > 0 {
    self.latest_updated_timestamp.write((*new_prices.at(0)).last_updated_timestamp);
}
```

This enables a malicious executor to roll back prices to previous, more favorable values.

2. **Timestamp Inconsistency**: The function does not verify that all price/index entries have consistent timestamps:

```cairo
// No validation that these timestamps match
for tick in new_prices {
    self._set_value(@tick);
}
for tick in new_indices {
    self._set_funding_index(@tick);
}
```

This allows providing different timestamps for different assets, creating state confusion.

3. **Empty Update Handling**: If both arrays are empty, `latest_snapshot_id` is still updated without updating `latest_updated_timestamp`, creating an inconsistent state.

These issues violate oracle security fundamentals by permitting:

* Updates with older timestamps than current state
* Updates with inconsistent timestamps across different assets
* Empty updates that modify snapshot ID without timestamps

The vulnerabilities are particularly severe because the `get_value` function returns the global timestamp for all asset data:

```cairo
fn get_value(self: @ContractState, market: felt252) -> TickData {
    // ...
    let timestamp = self.latest_updated_timestamp.read();
    // ...
}
```

This means a single inconsistent update affects the validity of all price data.

## Impact Details

1. **Price Manipulation for Liquidations**:
   * An executor could update oracle with stale prices during market volatility
   * This could prevent legitimate liquidations that should occur (if prices are manipulated to appear healthier)
   * Or trigger improper liquidations of otherwise healthy positions (if prices are manipulated to appear worse)
   * Direct financial loss to users whose positions are incorrectly liquidated
2. **Settlement Price Manipulation**:
   * By reverting to older, more favorable prices, an attacker could manipulate settlement values
   * Users closing positions would receive incorrect payouts
   * Example: A position that should be underwater could be made profitable through stale data

## Proof of Concept

## Proof of Concept

```
#[test]
fn test_POC() {
    let mut state = setup_dispatcher();
    let market: felt252 = 'BTC-USD';
    let value = state.get_value(market);
    assert_eq!(
        value, TickData { asset_key: 0, asset_value: 0, decimals: 0, last_updated_timestamp: 0 },
    );
    let tick = TickData {
        asset_key: market, asset_value: 100_000_000, decimals: 8, last_updated_timestamp: 123,
    };
    start_cheat_caller_address(state.contract_address, EXECUTOR());
    state.set_value(tick);
    let mut  newprice : Array<TickData> =ArrayTrait::new();
    let mut  new_indices : Array<TickData> =ArrayTrait::new();

    let tick1= TickData {
        asset_key: market, asset_value: 200_000_000, decimals: 8, last_updated_timestamp: 120,
    };
    newprice.append(tick);
    newprice.append(tick1);
    new_indices.append(tick);
    new_indices.append(tick1);

    state.set_prices_and_funding_snapshot(
        1, newprice, new_indices,
    );
    stop_cheat_caller_address(state.contract_address);
    let value = state.get_value(market);
    assert_ne!(value, tick);
}
```
