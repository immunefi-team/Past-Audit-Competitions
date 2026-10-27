# Attackathon \_ Fuel Network 33233 - \[Smart Contract - Medium] Incorrect Implementation of Unsigned -b

## Incorrect Implementation of Unsigned 32-bit Fixed Point Fractional Function

Submitted on Mon Jul 15 2024 16:51:42 GMT-0400 (Atlantic Standard Time) by @Blockian for [Attackathon | Fuel Network](https://immunefi.com/bounty/fuel-network-attackathon/)

Report ID: #33233

Report type: Smart Contract

Report severity: Medium

Target: <https://github.com/FuelLabs/sway-libs/tree/0f47d33d6e5da25f782fc117d4be15b7b12d291b>

Impacts:

* Contract fails to deliver promised returns, but doesn't lose value
* Incorrect math
* Permanent freezing of funds
* Temporary freezing of funds for at least 1 hour

### Description

## Fuel Network bug report

### Incorrect Implementation of Unsigned 32-bit Fixed Point Fractional Function

#### Description

The current implementation of the fractional function in `sway-libs` for the unsigned 32-bit Fixed Point and signed 64-bit Fixed Point is flawed. This issue causes the function to revert every time it's called, potentially leading to problems for projects built on the Fuel platform.

### Root Cause

The `fract` function from the `UFP32` implementation is as follows:

```rs
    pub fn fract(self) -> Self {
        Self {
            // first move to the left (multiply by the denominator)
            // to get rid of integer part, than move to the
            // right (divide by the denominator), to ensure 
            // fixed-point structure
            underlying: ((self.underlying << 16) - u32::max() - 1u32) >> 16,
        }
    }
```

The issue arises after the left shift operation, where the function subtracts `u32::max` and then subtracts `1`. Since left shifting doesn't change the operand's type (i.e., `a << 16` keeps a as `u32`), `(self.underlying << 16)` remains within the bounds of `u32`. Specifically, `(self.underlying << 16)` is constrained by `4294901760` (`0b11111111111111110000000000000000`), with the left shift ensuring the 16 rightmost bits are zero.

Consequently, for any `self.underlying`, `(self.underlying << 16)` is always less than `u32::max`, leading to an underflow and causing a revert.

### Impact

Every usage of `UFP32.fract` results in a revert, affecting:

1. `UFP32.ceil` - relies on the fract function.
2. `IFP64.fract` - internally uses UFP32, causing it to revert.
3. `IFP64.ceil` - also reverts since it uses both UFP32 and the fract function.

Any function in a contract that relies on one of those functions will revert, possibly locking user funds.

### Proposed fix

Remove the unnecessary subtraction:

```rs
    pub fn fract(self) -> Self {
        Self {
            underlying: (self.underlying << 16) >> 16,
        }
    }
```

This modification ensures the function operates correctly without causing reverts.

### Proof of concept

## Proof of Concept

Run the POC's with `forc test`

```rs
contract;

use sway_libs::fixed_point::ufp32::UFP32;

abi Tester { 
    fn frac_crashing() -> UFP32;
    fn new_frac_not_crashing() -> bool;
}

pub fn new_fract(num: UFP32) -> UFP32 {
    UFP32::from((num.underlying() << 16) >> 16)
}
 
impl Tester for Contract {
    fn frac_crashing() -> UFP32 {
        let num = UFP32::from(u32::max());
        num.fract()
    }

    fn new_frac_not_crashing() -> bool {
        let num = UFP32::from(u32::max());
        new_fract(num) == UFP32::from(65535u32) // 65535 == 0b1111111111111111, which is the fractional part of u32::max
    }
}

#[test]
fn test_sub() {
    let caller = abi(Tester, CONTRACT_ID);
    assert(caller.new_frac_not_crashing() == true); // the test passes this point!
    let _ = caller.frac_crashing(); // crashes here with ArithmeticOverflow at the SUB instruction
}
```
