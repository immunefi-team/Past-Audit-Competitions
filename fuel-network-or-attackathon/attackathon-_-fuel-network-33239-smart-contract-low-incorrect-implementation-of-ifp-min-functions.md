# Attackathon \_ Fuel Network 33239 - \[Smart Contract - Low] Incorrect Implementation of IFP Min Functi

## Incorrect Implementation of IFP Min Functions

Submitted on Mon Jul 15 2024 20:07:46 GMT-0400 (Atlantic Standard Time) by @Blockian for [Attackathon | Fuel Network](https://immunefi.com/bounty/fuel-network-attackathon/)

Report ID: #33239

Report type: Smart Contract

Report severity: Low

Target: <https://github.com/FuelLabs/sway-libs/tree/0f47d33d6e5da25f782fc117d4be15b7b12d291b>

Impacts:

* Contract fails to deliver promised returns, but doesn't lose value

### Description

## Fuel Network bug report

### Incorrect Implementation of IFP Min Functions

#### Description

The current implementation of the min function in `sway-libs` for the signed Fixed Point numbers is flawed. Instead of returning the smallest number, the min function returns zero.

### Root Cause

The `min` function from the `IFP64` implementation is as follows:

```rs
    pub fn min() -> Self {
        Self {
            underlying: UFP32::min(),
            non_negative: false,
        }
    }
```

However, this implementation returns `-0`, which is not the smallest value within the range of `-u32::max` to `u32::max`.

***NOTE***: This error affects all `IFP` types.

### Impact

Relying on the `min` function to return the smallest number can lead to incorrect results and potential mistakes for users.

### Proposed fix

To correct this, the `min` function should use the underlying `max` function instead of `min`:

```rs
    pub fn min() -> Self {
        Self {
            underlying: UFP32::max(),
            non_negative: false,
        }
    }
```

### Proof of concept

## Proof of Concept

Run the POC's with `forc test`

```rs
contract;

use sway_libs::fixed_point::ifp64::IFP64;

abi Tester { 
    fn smaller_than_min() -> bool;
}

impl Tester for Contract {
    fn smaller_than_min() -> bool {
        let min_ifp64 = IFP64::min();
        let smaller = IFP64::from_uint(1u32) - IFP64::from_uint(2u32); // minus 1
        smaller < min_ifp64
    }
}

#[test]
fn test() {
    let caller = abi(Tester, CONTRACT_ID);
    assert(caller.smaller_than_min() == true); // There is a number smaller than the minimum number
}
```
