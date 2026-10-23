# Attackathon \_ Fuel Network 33191 - \[Smart Contract - Insight] Sway Formatting Behaves Differently Ba

Submitted on Sat Jul 13 2024 19:27:00 GMT-0400 (Atlantic Standard Time) by @savi0ur for [Attackathon | Fuel Network](https://immunefi.com/bounty/fuel-network-attackathon/)

Report ID: #33191

Report type: Smart Contract

Report severity: Insight

Target: <https://github.com/FuelLabs/sway/tree/v0.61.2>

Impacts:

* Compiler bug

## Description

Inside `sway` module, we have `swayfmt` module which does a job of formatting `sway` language code, similar to what `rustfmt` do for `rust` lang.

<https://github.com/FuelLabs/sway/blob/e1b1c2bee73e0ba825e07736cefa6c0abd079595/swayfmt/src/utils/map/newline.rs#L144-L150>

```rust
fn calculate_offset(base: usize, offset: i64) -> usize {
    offset
        .checked_add(base as i64)
        .unwrap_or(base as i64)
        .try_into()//@audit-issue conversion discrepancies on 32/64 bit machine
        .unwrap_or(base)
}
```

It have a `calculate_offset` function which is behaving differently based on which platform it runs.

For example, When run on 64 bit machine, where `usize` is 64 bit wide, if `offset` is `> u32::MAX`, it will return new offset as `offset + base` by converting this result to `usize`. Since `usize` is 64 bit wide, it will be able to store this result.

But when run on 32-bit machine, where `usize` is 32 bit wide, if `offset` is `> u32::MAX`, and `base` is non-zero, `calculate_offset` function will return `base` instead of `offset + base`.

As we have seen, based on the platform architecture, this function returning different offset.

## Impact

Sway formatting behaves differently based on architecture of the system.

## Recommendation

Avoid conversions that may behave differently across architectures. Use architecture independent data types for performing required operations.

## References

<https://github.com/FuelLabs/sway/blob/e1b1c2bee73e0ba825e07736cefa6c0abd079595/swayfmt/src/utils/map/newline.rs#L144-L150>

## Proof of concept

## Proof Of Concept

**Steps to Run using Foundry:**

* Change directory to `sway` directory i.e., `cd sway`
* Copy following test case in `swayfmt/tests/mod.rs`
* Open terminal and run `cargo test --package swayfmt --test mod -- test_calculate_offset_bug_diff_arch --exact --show-output`

```rust
#[test]
fn test_calculate_offset_bug_diff_arch() {
    fn calculate_offset(base: usize, offset: i64) -> usize {
        offset
            .checked_add(base as i64)
            .unwrap_or(base as i64)
            .try_into()
            .unwrap_or(base)
    }

    let base: usize = 10000;
    let offset: i64 = u32::MAX as i64 + 10;
    let res = calculate_offset(base, offset);
    
    if cfg!(target_pointer_width = "64") {    
        assert!(res != base);
    } else {
        assert!(res == base);
    }
}
```

**Console Output:**

* On 64-bit machine, this test case will execute successfully with `res != base`.
* On 32-bit machine, this test case will execute successfully with `res == base`.
