# Attackathon \_ Fuel Network 33240 - \[Smart Contract - Insight] Incorrect Bitness in IFP Types

## Incorrect Bitness in IFP Types

Submitted on Mon Jul 15 2024 20:11:00 GMT-0400 (Atlantic Standard Time) by @Blockian for [Attackathon | Fuel Network](https://immunefi.com/bounty/fuel-network-attackathon/)

Report ID: #33240

Report type: Smart Contract

Report severity: Insight

Target: <https://github.com/FuelLabs/sway-libs/tree/0f47d33d6e5da25f782fc117d4be15b7b12d291b>

Impacts:

* Contract fails to deliver promised returns, but doesn't lose value

### Description

## Fuel Network bug report

### Incorrect Bitness in IFP Types

#### Description

The bitness of the `IFP` types is incorrectly defined. Specifically:

`IFP64` is actually 33 bits. `IFP128` is actually 65 bits. `IFP256` is actually 129 bits.

### Root Cause

Examining the definition of `IFP64` as an example:

```rs
/// The 64-bit signed fixed point number type.
///
/// # Additional Information
///
/// Represented by an underlying `UFP32` number and a boolean.
pub struct IFP64 {
    /// The underlying value representing the `IFP64` type.
    underlying: UFP32,
    /// The underlying boolean representing a negative value for the `IFP64` type.
    non_negative: bool,
}
```

The `IFP64` type internally uses a `UFP32`, which occupies `32` bits, along with a single bit for the `non_negative` boolean. This totals `33` bits, not `64` as the name suggests.

### Impact

The primary impact is the misleading and confusing naming of these types. This issue is of low severity but can cause misunderstanding.

### Proposed fix

Correct the names of the IFP types to accurately reflect their bitness.

### Proof of concept

## Proof of Concept

Run the POC's with `forc test`

```rs
contract;

use sway_libs::fixed_point::ifp64::IFP64;

abi Tester { 
    fn wrong_bitness() -> bool;
}

impl Tester for Contract {
    fn wrong_bitness() -> bool {
        IFP64::bits() != 33
    }
}

#[test]
fn test() {
    let caller = abi(Tester, CONTRACT_ID);
    assert(caller.wrong_bitness() == true); // the bitness is wrong
}
```
