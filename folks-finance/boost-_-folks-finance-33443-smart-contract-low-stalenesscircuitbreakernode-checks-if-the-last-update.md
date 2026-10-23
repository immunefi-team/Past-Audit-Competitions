# Boost \_ Folks Finance 33443 - \[Smart Contract - Low] StalenessCircuitBreakerNode checks if the last

Submitted on Sat Jul 20 2024 14:42:29 GMT-0400 (Atlantic Standard Time) by @Tripathi for [Boost | Folks Finance](https://immunefi.com/bounty/folksfinance-boost/)

Report ID: #33443

Report type: Smart Contract

Report severity: Low

Target: <https://testnet.snowtrace.io/address/0xA758c321DF6Cd949A8E074B22362a4366DB1b725>

Impacts:

* Contract fails to deliver promised returns, but doesn't lose value

## Description

## Brief/Intro

StalenessCircuitBreakerNode checks if the last update time of the parent node is less than the threshold but the `publicTime` could be greater than current `block.timestamp` .

`PythNode::Process()` calls `pyth.getEmaPriceUnsafe()` and `pyth.getPriceUnsafe()` for fetching price.

Issue is In `StalenessCircuitBreakerNode` `stalenessTolerance`is conceived as the maximum number of seconds that the price can be in the past(compared to block.timestamp) but in reality the price could also be in future

## Vulnerability Details

This fact is corroborated by the logic inside Pyth SDK that performs an abs delta between the `price.publishTime` in `[getPriceNoOlderThan](https://github.com/pyth-network/pyth-crosschain/blob/main/target_chains/ethereum/sdk/solidity/AbstractPyth.sol#L48-L58)`. In the [near SDK](https://github.com/pyth-network/pyth-crosschain/blob/main/target_chains/near/receiver/src/lib.rs#L503-L505) the check is even more explicit

Let's analyse `getPriceUnsafe()`

```js
  pub fn get_price_unsafe(&self, price_identifier: PriceIdentifier) -> Option<Price> {
        self.get_price_no_older_than(price_identifier, u64::MAX)
    }

    /// Get the latest available price cached for the given price identifier, if that price is
    /// no older than the given age.
    pub fn get_price_no_older_than(
        &self,
        price_id: PriceIdentifier,
        age: Seconds,
    ) -> Option<Price> {
        self.prices.get(&price_id).and_then(|feed| {
            let block_timestamp = env::block_timestamp() / 1_000_000_000;
            let price_timestamp = feed.price.publish_time;

            // - If Price older than STALENESS_THRESHOLD, set status to Unknown.
            // - If Price newer than now by more than STALENESS_THRESHOLD, set status to Unknown.
            // - Any other price around the current time is considered valid.
            if u64::abs_diff(block_timestamp, price_timestamp.try_into().unwrap()) > age {
                return None;
            }

            Some(feed.price)
        })
    }
```

<https://github.com/pyth-network/pyth-crosschain/blob/main/target\\_chains/near/receiver/src/lib.rs#L491>

we can see that absolute difference is taken between`price.publishtime` and age which shows that `price.publishtime` could be greater than `block.timestamp` but during staleness check `stalenessTolerance` is used for only previous prices

## Impact Details

`price.publishtime` could be greater than block.timestamp. In such case fetching price will revert due to underflow since `block.timestamp < price.publishtime`

## References

<https://github.com/pyth-network/pyth-crosschain/blob/main/target\\_chains/near/receiver/src/lib.rs#L509>

## Proof of concept

## Proof of Concept

call `getPriceNoOlderThan()` on any `Pyth` priceFeed and check `price.publishtime`

For more clarifications

Check 5.4.13 of <https://github.com/euler-xyz/euler-price-oracle/blob/master/audits/Euler\\_Price\\_Oracle\\_Spearbit\\_Report\\_DRAFT.pdf>
