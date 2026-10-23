# Attackathon \_ Fuel Network 33450 - \[Blockchain\_DLT - Insight] fuel\_gas\_price\_algorithm AlgorithmV ma

Submitted on Sat Jul 20 2024 17:22:45 GMT-0400 (Atlantic Standard Time) by @Crab for [Attackathon | Fuel Network](https://immunefi.com/bounty/fuel-network-attackathon/)

Report ID: #33450

Report type: Blockchain/DLT

Report severity: Insight

Target: <https://github.com/FuelLabs/fuel-core/tree/v0.31.0>

Impacts:

* Causing network processing nodes to process transactions from the mempool beyond set parameters (e.g. prevents processing transactions from the mempool)

## Description

## Brief/Intro

The `fuel_gas_price_algorithm::AlgorithmV1` `calculate` function may panic if it is not initialized correctly. This would result in the halting of nodes on mainnet and would stop the network from working correctly.

## Vulnerability Details

The [`AlgorithmV1`](https://github.com/FuelLabs/fuel-core/blob/v0.31.0/crates/fuel-gas-price-algorithm/src/lib.rs#L50) is currently unused but would be used to determine the price of the current gas price on Fuel. It is featuring a PID controller (without the I component though) that takes in the block profit target and adjust the P (proportional) and D (derivative) parameters in order to tend to reach it. There is a bug though that gets triggered when the [`abs`](https://github.com/FuelLabs/fuel-core/blob/v0.31.0/crates/fuel-gas-price-algorithm/src/lib.rs#L119) function is called. It triggers a panic with the message "attempt to negate with overflow". This will panic whenever [`p + d == i64::MIN`](https://github.com/FuelLabs/fuel-core/blob/v0.31.0/crates/fuel-gas-price-algorithm/src/lib.rs#L113) because `-i64::MIN` is undefined on the range of the integers.

This has been found with fuzzing, with the following input:

```sh
AlgorithmV1 {
    min_da_gas_price: 10199964370168811003,
    new_exec_price: 10199964370168810893,
    last_da_price: 10199964370168810929,
    max_change_percent: 141,
    latest_da_cost_per_byte: 10199964370168810893,
    total_rewards: 10199964370168810893,
    total_costs: 18446744073709551615,
    da_p_factor: -1,
    da_d_factor: -1,
    avg_profit: 92685846460760063,
    avg_window: 1,
}
```

And here is the fuzzing target:

```rust
#![no_main]

use fuel_gas_price_algorithm::AlgorithmV1;
use libfuzzer_sys::arbitrary::Arbitrary;
use libfuzzer_sys::fuzz_target;
use libfuzzer_sys::{arbitrary, Corpus};

#[derive(Debug)]
struct ExtAlgorithmV1(AlgorithmV1, u64);

impl<'a> Arbitrary<'a> for ExtAlgorithmV1 {
    fn arbitrary(u: &mut arbitrary::Unstructured<'a>) -> arbitrary::Result<Self> {
        let algo = AlgorithmV1 {
            min_da_gas_price: u.arbitrary()?,
            new_exec_price: u.arbitrary()?,
            last_da_price: u.arbitrary()?,
            max_change_percent: u.arbitrary()?,
            latest_da_cost_per_byte: u.arbitrary()?,
            total_rewards: u.arbitrary()?,
            total_costs: u.arbitrary()?,
            da_p_factor: u.arbitrary()?,
            da_d_factor: u.arbitrary()?,
            avg_profit: u.arbitrary()?,
            avg_window: u.arbitrary()?,
        };

        Ok(ExtAlgorithmV1(algo, u.arbitrary()?))
    }
}

fuzz_target!(|input: ExtAlgorithmV1| -> Corpus {
    let ExtAlgorithmV1(algo, block_bytes) = input;

    let _ = algo.calculate(block_bytes);

    Corpus::Keep
});
```

You can adjust the target so that it will reject invalid p and d factors and won't trigger the bug anymore:

```rust
fuzz_target!(|input: ExtAlgorithmV1| -> Corpus {
    let ExtAlgorithmV1(algo, block_bytes) = input;

    if algo.da_p_factor < 0 {
        return Corpus::Reject;
    } else if algo.da_d_factor < 0 {
        return Corpus::Reject;
    } else if algo.avg_profit < 0 {
        return Corpus::Reject;
    }

    let _ = algo.calculate(block_bytes);

    Corpus::Keep
});
```

The code had to be instrumented to be able to import the `AlgorithmV1` and the fields have been made public to build this harness.

You can look at the bug more closely with the following reduced code

```rust
fn main() {
    let num: i64 = -9223372036854775808;
    assert_eq!(num, i64::MIN);
    dbg!(&num.abs());
}
```

## Impact Details

The `AlgorithmV1` does not seem to be used right now, but if it is and that the parameters are not checked correctly, it will panic on mainnet and halt all of the nodes of the network since they should agree on the same parameters.

## References

## Proof of concept

## Proof of Concept

```rust
#[test]
#[should_panic]
fn algo_v1_panics() {
    let algo = AlgorithmV1 {
        min_da_gas_price: 0,
        new_exec_price: 0,
        last_da_price: 0,
        max_change_percent: 0,
        latest_da_cost_per_byte: 0,
        total_rewards: 0,
        total_costs: 0,
        da_p_factor: 0,
        da_d_factor: 0,
        avg_profit: 0,
        avg_window: 0,
    };

    algo.change(i64::MIN / 2, i64::MIN / 2);
}
```
