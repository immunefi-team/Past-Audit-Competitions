# Attackathon \_ Fuel Network 33242 - \[Smart Contract - High] Incorrect Implementation of IFP Multiply

## Incorrect Implementation of IFP Multiply and Divide Functions

Submitted on Mon Jul 15 2024 20:37:02 GMT-0400 (Atlantic Standard Time) by @Blockian for [Attackathon | Fuel Network](https://immunefi.com/bounty/fuel-network-attackathon/)

Report ID: #33242

Report type: Smart Contract

Report severity: High

Target: <https://github.com/FuelLabs/sway-libs/tree/0f47d33d6e5da25f782fc117d4be15b7b12d291b>

Impacts:

* Incorrect math
* Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield

### Description

## Fuel Network bug report

### Incorrect Implementation of IFP Multiply and Divide Functions

#### Description

The current implementation of the multiply and divide functions in `sway-libs` for the signed Fixed Point numbers is flawed. Specifically, the implementation always returns a positive number, even when a negative result is expected.

### Root Cause

The `multiply` function from the `IFP64` implementation is as follows:

```rs
    fn multiply(self, other: Self) -> Self {
        let non_negative = if (self.non_negative
            && !self.non_negative)
            || (!self.non_negative
            && self.non_negative)
        {
            false
        } else {
            true
        };
        Self {
            underlying: self.underlying * other.underlying,
            non_negative: non_negative,
        }
    }
```

Similarly, the `divide` function is implemented as:

```rs
    fn divide(self, divisor: Self) -> Self {
        let non_negative = if (self.non_negative
            && !self.non_negative)
            || (!self.non_negative
            && self.non_negative)
        {
            false
        } else {
            true
        };
        Self {
            underlying: self.underlying / divisor.underlying,
            non_negative: non_negative,
        }
    }
```

In both functions, when determining the value of `non_negative`, the code incorrectly compares `self` with `self` instead of `self` with `other` or `divisor`. This results in `non_negative` always being set to true, regardless of the values being operated on.

***NOTE***: This error affects all `IFP` types.

### Impact

This issue affects every implementation of signed fixed point numbers in the Fuel ecosystem. Any project utilizing these implementations will encounter incorrect calculations.

In the crypto space, even minor errors, like off by one, can lead to substantial financial losses, underscoring the critical nature of this bug.

### Proposed fix

Compare `self` and `other` in multiply and `self` and `divisor` in divide

By correctly comparing `self` with `other` or `divisor`, the functions will properly set the `non_negative` value, ensuring accurate results.

### Proof of concept

## Proof of Concept

Run the POC's with `forc test`

```rs
contract;

use sway_libs::fixed_point::ifp64::IFP64;

abi Tester { 
    fn wrong_mult() -> bool;
    fn wrong_div() -> bool;
}

impl Tester for Contract {
    fn wrong_mult() -> bool {
        let one = IFP64::from_uint(1u32);
        let minus_one = IFP64::from_uint(1u32) - IFP64::from_uint(2u32); // minus 1
        one * minus_one > IFP64::zero() // that is a mistake, 1 * -1 = -1 which is smaller that zero
    }

    fn wrong_div() -> bool {
        let one = IFP64::from_uint(1u32);
        let minus_one = IFP64::from_uint(1u32) - IFP64::from_uint(2u32); // minus 1
        one / minus_one > IFP64::zero() // that is a mistake, 1 / -1 = -1 which is smaller that zero
    }
}

#[test]
fn test() {
    let caller = abi(Tester, CONTRACT_ID);
    assert(caller.wrong_mult() == true); // the result is bigger than zero even though it should be negative
    assert(caller.wrong_div() == true); // the result is bigger than zero even though it should be negative
}
```
