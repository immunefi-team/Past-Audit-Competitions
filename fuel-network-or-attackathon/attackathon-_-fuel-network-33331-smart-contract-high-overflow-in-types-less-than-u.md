# Attackathon \_ Fuel Network 33331 - \[Smart Contract - High] Overflow in Types Less Than u

## Overflow in Types Less Than u64

Submitted on Wed Jul 17 2024 22:27:13 GMT-0400 (Atlantic Standard Time) by @Blockian for [Attackathon | Fuel Network](https://immunefi.com/bounty/fuel-network-attackathon/)

Report ID: #33331

Report type: Smart Contract

Report severity: High

Target: <https://github.com/FuelLabs/sway/tree/v0.61.2>

Impacts:

* Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield
* Incorrect math

### Description

## Fuel Network bug report

### Overflow in Types Less Than u64

#### Description

Since any non-64-bit value is compiled to a `u64` value under-the-hood, the ALU fails to detect overflows. Specifically, the `pow` function does not manually check for overflows, allowing overflows and undefined behavior in `u32`, `u16`, and `u8` types.

### Root Cause

Non-64-bit values are compiled to u64 under the hood, preventing the ALU from detecting overflows. Consequently, every mathematical operation should manually perform overflow checks.

For instance, the `add` implementation includes such checks:

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

As shown, the `add` function checks for overflows manually. However, the `pow` implementation lacks this check:

```rs
impl Power for u32 {
    fn pow(self, exponent: u32) -> Self {
        asm(r1: self, r2: exponent, r3) {
            exp r3 r1 r2;
            r3: Self
        }
    }
}
```

This oversight means any overflows not triggering the ALU overflow check (i.e., all values until `u64::max`) are missed.

### Impact

This issue affects the `u32`, `u16` and `u8` types in the Fuel ecosystem. Any project utilizing these types will encounter incorrect calculations.

In the crypto space, even minor errors, like off by one, can lead to substantial financial losses, underscoring the critical nature of this bug.

Additionally, since values are stored as `u64` under the hood, the actual stored value may differ from the displayed value. For instance, an overflown u8 may present as `0` but hold `0xffffffffffffff00` internally.

### Proposed fix

Introduce a check for overflows in the `pow` function:

```rs
    fn pow(self, exponent: u32) -> Self {
        let res = asm(r1: self, r2: exponent, r3) {
            exp r3 r1 r2;
            r3: Self
        };

        if __gt(res, Self::max()) {
            __revert(0)
        } else {
            res
        }
    }
```

### Proof of concept

## Proof of Concept

Run the POC with `forc test`

```rs
contract;

abi Tester { 
    fn u8_overflow() -> u8;
    fn u16_overflow() -> u16;
    fn u32_overflow() -> u32;
}

impl Tester for Contract {
    fn u8_overflow() -> u8 {
        let max = u8::max();
        max.pow(2)
    }

    fn u16_overflow() -> u16 {
        let max = u16::max();
        max.pow(2)
    }

    fn u32_overflow() -> u32 {
        let max = u32::max();
        max.pow(2)
    }
}

#[test]
fn test() {
    let caller = abi(Tester, CONTRACT_ID);
    assert(caller.u8_overflow() == 1u8); // overflown value, it's actually 0b1111111000000001 under the hood, but u8 reads only 0b00000001 part
    assert(caller.u16_overflow() == 1u16); // overflown value
    assert(caller.u32_overflow() == 1u32); // overflown value
}
```
