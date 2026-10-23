# Attackathon \_ Fuel Network 33487 - \[Smart Contract - Insight] Flags Do Not Affect Types Less Than u

## Flags Do Not Affect Types Less Than u64

Submitted on Sun Jul 21 2024 20:43:48 GMT-0400 (Atlantic Standard Time) by @Blockian for [Attackathon | Fuel Network](https://immunefi.com/bounty/fuel-network-attackathon/)

Report ID: #33487

Report type: Smart Contract

Report severity: Insight

Target: <https://github.com/FuelLabs/sway/tree/v0.61.2>

Impacts:

* Contract fails to deliver promised returns, but doesn't lose value

### Description

## Fuel Network bug report

### Flags Do Not Affect Types Less Than u64

#### Description

Types smaller than `u64` (`u32`, `u16`, and `u8`) are not influenced by user-set flags, leading to unintended behavior.

### Root Cause

Since non-64-bit values are compiled to `u64` under-the-hood, the ALU does not detect overflows. Therefore, every mathematical operation should manually perform overflow checks.

For instance, the add implementation includes such checks:

```rs
// Emulate overflowing arithmetic for non-64-bit integer types
impl Add for u32 {
    fn add(self, other: Self) -> Self {
        // any non-64-bit value is compiled to a u64 value under-the-hood
        // constants (like Self::max() below) are also automatically promoted to u64
        let res = __add(self, other);
        if __gt(res, Self::max()) {
            // integer overflow
            __revert(0)
        } else {
            // no overflow
            res
        }
    }
}
```

Flags exist to indicate whether an overflow is allowed, such as the `disable_panic_on_overflow` function. However, since `disable_panic_on_overflow` disables panics caused by the ALU, it does not disable the panics triggered by types that manually check for overflows.

### Impact

This issue affects the `u32`, `u16`, and `u8` types in the Fuel ecosystem. Any project utilizing these types may experience unintended behavior in their contracts.

### Proposed fix

Incorporate flag checks in mathematical operations involving `u32`, `u16`, and `u8` types.

### Proof of concept

## Proof of Concept

Run the POC with `forc test`

```rs
contract;

use std::flags::{ disable_panic_on_overflow, enable_panic_on_overflow };

abi Tester {
    fn add_with_overflow_u8(a: u8, b: u8) -> u8;
    fn add_with_overflow_u64(a: u64, b: u64) -> u64;
}
 
impl Tester for Contract {
    fn add_with_overflow_u8(a: u8, b: u8) -> u8 {
        let _ = disable_panic_on_overflow();
        let c = a + b;
        let _ = enable_panic_on_overflow();
        c
    }

    fn add_with_overflow_u64(a: u64, b: u64) -> u64 {
        let _ = disable_panic_on_overflow();
        let c = a + b;
        let _ = enable_panic_on_overflow();
        c
    }
}

#[test]
fn test_increment_success() {
    let caller = abi(Tester, CONTRACT_ID);
    assert(caller.add_with_overflow_u64(u64::max(), 1) == 0); // works fine for u64
    assert(caller.add_with_overflow_u8(u8::max(), 1u8) == 0u8); // reverts
}
```
