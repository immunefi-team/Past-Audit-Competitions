# Attackathon \_ Fuel Network 33303 - \[Smart Contract - Medium] Incorrect sign change

Submitted on Wed Jul 17 2024 10:37:38 GMT-0400 (Atlantic Standard Time) by @shadowHunter for [Attackathon | Fuel Network](https://immunefi.com/bounty/fuel-network-attackathon/)

Report ID: #33303

Report type: Smart Contract

Report severity: Medium

Target: <https://github.com/FuelLabs/sway-libs/tree/0f47d33d6e5da25f782fc117d4be15b7b12d291b>

Impacts:

* Compiler bug

## Description

This is a compiler bug since resulting value 0 sign is changed to negative. This library are meant to be used by third party apps as mentioned by Fuel team in github page

## Brief/Intro

It seems that if you are adding same number with different sign then resulting 0 sign is changed which is not correct

## Vulnerability Details

1. If `add` is called using number 3 and -3 then result should be 0 with positive sign
2. But since 3>3 is false so condition move to else condition which changes the sign even though it was not required

```
if self.underlying > other.underlying {
                underlying = self.underlying - other.underlying;
            } else {
                underlying = other.underlying - self.underlying;
                non_negative = false;
            }
```

4. Ideally the condition should be `if self.underlying >= other.underlying`

## Impact Details

Result will be with incorrect sign

## References

<https://github.com/FuelLabs/sway-libs/blob/2a869c583d2ab9fbe8de17a3301d928b224062c7/libs/src/fixed\\_point/ifp64.sw#L239C13-L244C14>

## Proof of concept

## Proof of Concept

```
script;

use sway_libs::fixed_point::ifp64::IFP64;
use std::assert::assert;

fn main() -> bool {
	let num = IFP64::from_uint(42_u32);
    let num2 = IFP64::from_uint(42_u32);
	
	let mut res = num + num2.sign_reverse();
	// fails since resulting 0 has -ve sign
	assert(res.non_negative());

    true
}
```
