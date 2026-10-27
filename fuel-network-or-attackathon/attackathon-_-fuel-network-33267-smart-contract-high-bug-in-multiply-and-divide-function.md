# Attackathon \_ Fuel Network 33267 - \[Smart Contract - High] Bug in Multiply and Divide function

Submitted on Tue Jul 16 2024 16:36:28 GMT-0400 (Atlantic Standard Time) by @shadowHunter for [Attackathon | Fuel Network](https://immunefi.com/bounty/fuel-network-attackathon/)

Report ID: #33267

Report type: Smart Contract

Report severity: High

Target: <https://github.com/FuelLabs/sway-libs/tree/0f47d33d6e5da25f782fc117d4be15b7b12d291b>

Impacts:

* Griefing (e.g. no profit motive for an attacker, but damage to the users or the protocol)

## Description

## Brief/Intro

It seems that both multiply and divide function in `ifp64.sw`,`ifp128.sw`,`ifp256.sw` will not work correctly if any one of the number is negative as shown in below poc

## Location

<https://github.com/FuelLabs/sway-libs/blob/0f47d33d6e5da25f782fc117d4be15b7b12d291b/libs/src/fixed\\_point/ifp64.sw#L273-L276> <https://github.com/FuelLabs/sway-libs/blob/0f47d33d6e5da25f782fc117d4be15b7b12d291b/libs/src/fixed\\_point/ifp64.sw#L292-L295> Similarly for `ifp128.sw`,`ifp256.sw`

## Vulnerability Details

1. Lets see how resulting `non_negative` is calculated while multiplying and dividing

```sway
let non_negative = if (self.non_negative
            && !self.non_negative)
            || (!self.non_negative
            && self.non_negative)
        {
            false
        } else {
            true
        };
```

2. As we can see it is only checking `non_negative` param for 1st argument and not on the `other.non_negative`
3. So if we multiply -A \* B then `non_negative` becomes true since `(self.non_negative && !self.non_negative) || (!self.non_negative && self.non_negative)` always remain false
4. So result will be AB instead of -AB

## Impact Details

User who is trusting this library for arithmetic operation can bear huge losses since this will return resulting negative value as positive while multiplying and dividing

## Proof of concept

## Proof of Concept

```sway
script;

use sway_libs::fixed_point::ifp64::IFP64;
use std::assert::assert;

fn main() -> bool {
	// one is negative
    let one = IFP64::min();
    let two = IFP64::from_uint(2u32);
    let mut res = one * two;
	// Due to bug below check fails
	assert(res.non_negative() == false);

    // three is negative
    let three = IFP64::min();
    let four = IFP64::from_uint(2u32);
    let mut res = three / four;
	// Due to bug below check fails
	assert(res.non_negative() == false);

    true
}
```
