# Attackathon \_ Fuel Network 33248 - \[Smart Contract - High] Incorrect Implementation of IFP Floor and

## Incorrect Implementation of IFP Floor and Ceil Functions

Submitted on Mon Jul 15 2024 23:00:54 GMT-0400 (Atlantic Standard Time) by @Blockian for [Attackathon | Fuel Network](https://immunefi.com/bounty/fuel-network-attackathon/)

Report ID: #33248

Report type: Smart Contract

Report severity: High

Target: <https://github.com/FuelLabs/sway-libs/tree/0f47d33d6e5da25f782fc117d4be15b7b12d291b>

Impacts:

* Temporary freezing of funds for at least 1 hour
* Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield
* Permanent freezing of funds
* Incorrect math

### Description

## Fuel Network bug report

### Incorrect Implementation of IFP Floor and Ceil Functions

#### Description

The current implementation of the floor and ceil functions in `sway-libs` for the signed Fixed Point numbers is flawed. The implementation returns the wrong number for every negative input.

### Root Cause

The `floor` function from the `IFP64` implementation is as follows:

```rs
    pub fn floor(self) -> Self {
        if self.non_negative {
            self.trunc()
        } else {
            let trunc = self.underlying.trunc();
            if trunc != UFP32::zero() {
                self.trunc() - Self::from(UFP32::from(1u32))
            } else {
                self.trunc()
            }
        }
    }
```

Similarly, the `ceil` function is implemented as:

```rs
    pub fn ceil(self) -> Self {
        let mut underlying = self.underlying;
        let mut non_negative = self.non_negative;

        if self.non_negative {
            underlying = self.underlying.ceil();
        } else {
            let ceil = self.underlying.ceil();
            if ceil != self.underlying {
                underlying = ceil + UFP32::from(1u32);
                if ceil == UFP32::from(1u32) {
                    non_negative = true;
                }
            } else {
                underlying = ceil;
            }
        }
        Self {
            underlying: underlying,
            non_negative: self.non_negative,
        }
    }
```

Several issues exist within these implementations, as detailed below:

#### Issues Identified

1. ***Incorrect Usage of `UFP32::from`***: The `from` function does not multiply the input number by the `denominator`, resulting in `UFP32::from(1u32)` being `0.0..01` instead of `1`.
2. ***Incorrect Subtraction in floor Function***: The floor function attempts to subtract `1` even when the number is already rounded, so for negative numbers different from zero, it subtracts `0.0..01` (covered by the first issue). Regardless if it's rounded already
3. ***Unreachable Branch in ceil Function***: The ceil function contains a branch that changes the sign of the number, which should not occur. Fortunately, this branch is unreachable.

***NOTE***: This error affects all `IFP` types.

### Impact

This issue affects every implementation of signed fixed point numbers in the Fuel ecosystem. Any project utilizing these implementations will encounter incorrect calculations.

In the crypto space, even minor errors, like off by one, can lead to substantial financial losses, underscoring the critical nature of this bug.

### Proposed fix

By examining the mathematical definitions of `floor` and `ceil`, we observe that for negative numbers, `ceil` acts as `floor` for positive numbers and vice versa. Thus, a simpler implementation can be adopted:

```rs
    pub fn floor(self) -> Self {
        if self.non_negative {
            self.trunc()
        } else {
            self.ceil()
        }
    }

    pub fn ceil(self) -> Self {
        if self.non_negative {
            self.ceil()
        } else {
            self.trunc()
        }
    }
```

This revised implementation ensures correct handling of both positive and negative fixed-point numbers.

### Proof of concept

## Proof of Concept

Run the POC's with `forc test`

```rs
contract;

use sway_libs::fixed_point::ifp64::IFP64;
use sway_libs::fixed_point::ufp32::UFP32;

abi Tester { 
    fn wrong_floor() -> bool;
    fn wrong_ceil() -> bool;
}

impl Tester for Contract {
    fn wrong_floor() -> bool {
        let minus_one = IFP64::from_uint(1u32) - IFP64::from_uint(2u32); // minus 1
        let minus_one_and_a_bit = IFP64::from(UFP32::from(1u32)) - IFP64::from(UFP32::from(65538u32)); // minus 0b10000000000000001
        minus_one.floor() == minus_one_and_a_bit // floor(-1) should equal -1, but in fact it results in -0b10000000000000001
    }

    fn wrong_ceil() -> bool {
        let minus_one_and_a_bit = IFP64::from(UFP32::from(1u32)) - IFP64::from(UFP32::from(65538u32)); // minus 0b10000000000000001
        let two_minus_one_bit = IFP64::from(UFP32::from(1u32)) - IFP64::from(UFP32::from(131072u32)); // minus 0b11111111111111111
        minus_one_and_a_bit.ceil() == two_minus_one_bit // ceil(-1.0..01) should equal -1, but in fact it results in -0b11111111111111111
    }
}

#[test]
fn test() {
    let caller = abi(Tester, CONTRACT_ID);
    assert(caller.wrong_floor() == true); // the result of floor is incorrect
    // assert(caller.wrong_ceil() == true); // don't uncomment since it revert's due to issue mentioned in 33233
}
```
