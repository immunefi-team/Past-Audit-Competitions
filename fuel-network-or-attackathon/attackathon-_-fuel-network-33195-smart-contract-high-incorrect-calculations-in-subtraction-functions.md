# Attackathon \_ Fuel Network 33195 - \[Smart Contract - High] Incorrect Calculations in Subtraction Fun

## Incorrect Calculations in Subtraction Functions for Signed Integers

Submitted on Sat Jul 13 2024 22:57:56 GMT-0400 (Atlantic Standard Time) by @Blockian for [Attackathon | Fuel Network](https://immunefi.com/bounty/fuel-network-attackathon/)

Report ID: #33195

Report type: Smart Contract

Report severity: High

Target: <https://github.com/FuelLabs/sway-libs/tree/0f47d33d6e5da25f782fc117d4be15b7b12d291b>

Impacts:

* Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield
* Incorrect math

### Description

## Fuel Network bug report

### Incorrect Calculations in Subtraction Functions for Signed Integers

#### Description

The current implementation of the subtraction function in the `sway-libs` for signed integers is incorrect. This can lead to erroneous calculations, potentially causing critical vulnerabilities in projects built on the Fuel platform.

### Root Cause

The way the signed numbers work in sway is by taking the indent of the unsigned counterparts and anything above it is positive while anything below is negative, so for example: The u8 range is `0-255` and the indent is `128` so 128 becomes 0 and `5 == 133`, `-5 == 123` and so on.

So to generalize, for every `x` the I8 underlying value will be `x + 128`.

Let's generalize even more, for every `x` the signed underlying value will be `x + indent`

Lets look at how a naive sub function would look:

The mechanism for handling signed integers in Sway involves using an offset (indent) based on the unsigned counterparts. For example, the range for `u8` is `0-255`, with an indent of `128`. Thus, `128` is mapped to `0`, `133` to `5`, and `123` to `-5`, and so forth.

In general terms, for any signed integer `x`, the underlying value is calculated as `x + indent`.

A naive subtraction function might be written as:

```rs
sub(a, b) => a - b => (a + indent) - (b + indent) => a - b
```

This approach loses the indent offset. Therefore, the correct general function should be:

```rs
sub(a, b) => a.underlying - b.underlyting + indent
```

However, due to overflow and underflow issues, the operations need to be ordered correctly. This is what the Sway function attempts, but it still falls short. Ultimately, every calculation should follow the form `a.underlying - b.underlying + indent`, albeit with different order of operations.

Examining the Sway implementation (using `I8` as an example, applicable to all signed integers):

```rs
impl core::ops::Subtract for I8 { // should be of the form a.underlying - b.underlyting + 128
    /// Subtract a I8 from a I8. Panics of overflow.
    fn subtract(self, other: Self) -> Self {
        let mut res = Self::new();
        if self.underlying >= Self::indent()
            && other.underlying >= Self::indent()
        {
            if self.underlying > other.underlying {
                res = Self::from_uint(self.underlying - other.underlying + Self::indent());
            } else {
                res = Self::from_uint(self.underlying - (other.underlying - Self::indent()));
            }
        } else if self.underlying >= Self::indent()
            && other.underlying < Self::indent()
        {
            res = Self::from_uint(self.underlying - Self::indent() + other.underlying); // wrong here
        } else if self.underlying < Self::indent()
            && other.underlying >= Self::indent()
        {
            res = Self::from_uint(self.underlying - (other.underlying - Self::indent()));
        } else if self.underlying < Self::indent()
            && other.underlying < Self::indent()
        {
            if self.underlying < other.underlying {
                res = Self::from_uint(other.underlying - self.underlying + Self::indent()); // wrong here this returns b - a
            } else {
                res = Self::from_uint(self.underlying + other.underlying - Self::indent()); // wrong here
            }
        }
        res
    }
}
```

#### Incorrect Calculations Identified

1. When self.`underlying >= indent && other.underlying < indent`, the calculation is `a.underlying + b.underlying - 128`.
2. When `self.underlying < indent && other.underlying < indent`, if `self.underlying < other.underlying`, the calculation is `b.underlying - a.underlying - 128`.
3. When `self.underlying < indent && other.underlying < indent` and `self.underlying >= other.underlying`, the calculation is `a.underlying + b.underlying - 128`.

### Impact

This issue affects every implementation of signed integers in the Fuel ecosystem. Any project utilizing these implementations will encounter incorrect calculations.

In the crypto space, even minor errors, like off by one, can lead to substantial financial losses, underscoring the critical nature of this bug.

### Proposed fix

Correct the erroneous branches in the subtraction functions to ensure they follow the format `a.underlying - b.underlying + indent`.

### Proof of concept

## Proof of Concept

Two proofs of concept are provided. The first is a simple demonstration to illustrate the issue, while the second is a more comprehensive example covering all possible branches for all signed integers.

Run the POC's with `forc test`

Simple Test:

```rs
contract;

use sway_libs::signed_integers::i8::I8;

abi Tester { 
    fn sub_negative_nums() -> bool;
}
 
impl Tester for Contract {
    fn sub_negative_nums() -> bool {
        let a = I8::neg_from(5u8);
        let b = I8::neg_from(5u8);
        return (a - b) == I8::zero(); // should be true since -5 - (-5) == 0
    }
}

#[test]
fn test_sub() {
    let caller = abi(Tester, CONTRACT_ID);
    // assert(caller.sub_negative_nums() == true); // this is what we expect, but the test fails here
    assert(caller.sub_negative_nums() == false);
}
```

Long Test:

```rs
contract;

use std::u128::U128;

use sway_libs::signed_integers::i8::I8;
use sway_libs::signed_integers::i16::I16;
use sway_libs::signed_integers::i32::I32;
use sway_libs::signed_integers::i64::I64;
use sway_libs::signed_integers::i128::I128;
use sway_libs::signed_integers::i256::I256;

abi Tester { 
    fn sub_poc_i8_flow_1() -> bool;
    fn sub_poc_i8_flow_2() -> bool;
    fn sub_poc_i8_flow_3() -> bool;
    fn sub_poc_i16_flow_1() -> bool;
    fn sub_poc_i16_flow_2() -> bool;
    fn sub_poc_i16_flow_3() -> bool;
    fn sub_poc_i32_flow_1() -> bool;
    fn sub_poc_i32_flow_2() -> bool;
    fn sub_poc_i32_flow_3() -> bool;
    fn sub_poc_i64_flow_1() -> bool;
    fn sub_poc_i64_flow_2() -> bool;
    fn sub_poc_i64_flow_3() -> bool;
    fn sub_poc_i128_flow_1() -> bool;
    fn sub_poc_i128_flow_2() -> bool;
    fn sub_poc_i128_flow_3() -> bool;
    fn sub_poc_i256_flow_1() -> bool;
    fn sub_poc_i256_flow_2() -> bool;
    fn sub_poc_i256_flow_3() -> bool;
}
 
impl Tester for Contract {
    fn sub_poc_i8_flow_1() -> bool { // a >= indent, b < indent
        let a = I8::from(5u8);
        let b = I8::neg_from(5u8);
        return (a - b) == I8::from(10u8); // should be true since 5 - (-5) = 10
    }

    fn sub_poc_i8_flow_2() -> bool { // a < b < indent
        let a = I8::neg_from(6u8);
        let b = I8::neg_from(5u8);
        return (a - b) == I8::neg_from(1u8); // should be true since -6 - (-5) == -1
    }

    fn sub_poc_i8_flow_3() -> bool { // a < indent, b < indent
        let a = I8::neg_from(5u8);
        let b = I8::neg_from(5u8);
        return (a - b) == I8::zero(); // should be true since -5 - (-5) == 0
    }

    fn sub_poc_i16_flow_1() -> bool { // a >= indent, b < indent
        let a = I16::from(5u16);
        let b = I16::neg_from(5u16);
        return (a - b) == I16::from(10u16); // should be true since 5 - (-5) = 10
    }

    fn sub_poc_i16_flow_2() -> bool { // a < b < indent
        let a = I16::neg_from(6u16);
        let b = I16::neg_from(5u16);
        return (a - b) == I16::neg_from(1u16); // should be true since -6 - (-5) == -1
    }

    fn sub_poc_i16_flow_3() -> bool { // a < indent, b < indent
        let a = I16::neg_from(5u16);
        let b = I16::neg_from(5u16);
        return (a - b) == I16::zero(); // should be true since -5 - (-5) == 0
    }

    fn sub_poc_i32_flow_1() -> bool { // a >= indent, b < indent
        let a = I32::from(5u32);
        let b = I32::neg_from(5u32);
        return (a - b) == I32::from(10u32); // should be true since 5 - (-5) == 10
    }

    fn sub_poc_i32_flow_2() -> bool { // a < b < indent
        let a = I32::neg_from(6u32);
        let b = I32::neg_from(5u32);
        return (a - b) == I32::neg_from(1u32); // should be true since -6 - (-5) == -1
    }

    fn sub_poc_i32_flow_3() -> bool { // a < indent, b < indent
        let a = I32::neg_from(5u32);
        let b = I32::neg_from(5u32);
        return (a - b) == I32::zero(); // should be true since -5 - (-5) == 0
    }

    fn sub_poc_i64_flow_1() -> bool { // a >= indent, b < indent
        let a = I64::from(5u64);
        let b = I64::neg_from(5u64);
        return (a - b) == I64::from(10u64); // should be true since 5 - (-5) == 10
    }

    fn sub_poc_i64_flow_2() -> bool { // a < b < indent
        let a = I64::neg_from(6u64);
        let b = I64::neg_from(5u64);
        return (a - b) == I64::neg_from(1u64); // should be true since -6 - (-5) == -1
    }

    fn sub_poc_i64_flow_3() -> bool { // a < indent, b < indent
        let a = I64::neg_from(5u64);
        let b = I64::neg_from(5u64);
        return (a - b) == I64::zero(); // should be true since -5 - (-5) == 0
    }

    fn sub_poc_i128_flow_1() -> bool { // a >= indent, b < indent
        let a = I128::from(U128::from(5u8));
        let b = I128::neg_from(U128::from(5u8));
        return (a - b) == I128::from(U128::from(10u8)); // should be true since 5 - (-5) == 10
    }

    fn sub_poc_i128_flow_2() -> bool { // a < b < indent
        let a = I128::neg_from(U128::from(6u8));
        let b = I128::neg_from(U128::from(5u8));
        return (a - b) == I128::neg_from(U128::from(1u8)); // should be true since -6 - (-5) == -1
    }

    fn sub_poc_i128_flow_3() -> bool { // a < indent, b < indent
        let a = I128::neg_from(U128::from(5u8));
        let b = I128::neg_from(U128::from(5u8));
        return (a - b) == I128::zero(); // should be true since -5 - (-5) == 0
    }
    
    fn sub_poc_i256_flow_1() -> bool { // a >= indent, b < indent
        let a = I256::from(0x5u256);
        let b = I256::neg_from(0x5u256);
        return (a - b) == I256::from(0x10u256); // should be true since 5 - (-5) == 10
    }

    fn sub_poc_i256_flow_2() -> bool { // a < b < indent
        let a = I256::neg_from(0x6u256);
        let b = I256::neg_from(0x5u256);
        return (a - b) == I256::neg_from(0x1u256); // should be true since -6 - (-5) == -1
    }

    fn sub_poc_i256_flow_3() -> bool { // a < indent, b < indent
        let a = I256::neg_from(0x5u256);
        let b = I256::neg_from(0x5u256);
        return (a - b) == I256::zero(); // should be true since -5 - (-5) == 0
    }
}

#[test]
fn test_sub() {
    let caller = abi(Tester, CONTRACT_ID);
    assert(caller.sub_poc_i8_flow_1() == false);
    assert(caller.sub_poc_i8_flow_2() == false);
    assert(caller.sub_poc_i8_flow_3() == false);
    assert(caller.sub_poc_i16_flow_1() == false);
    assert(caller.sub_poc_i16_flow_2() == false);
    assert(caller.sub_poc_i16_flow_3() == false);
    assert(caller.sub_poc_i32_flow_1() == false);
    assert(caller.sub_poc_i32_flow_2() == false);
    assert(caller.sub_poc_i32_flow_3() == false);
    assert(caller.sub_poc_i64_flow_1() == false);
    assert(caller.sub_poc_i64_flow_2() == false);
    assert(caller.sub_poc_i64_flow_3() == false);
    assert(caller.sub_poc_i128_flow_1() == false);
    assert(caller.sub_poc_i128_flow_2() == false);
    assert(caller.sub_poc_i128_flow_3() == false);
    assert(caller.sub_poc_i256_flow_1() == false);
    assert(caller.sub_poc_i256_flow_2() == false);
    assert(caller.sub_poc_i256_flow_3() == false);
}
```
