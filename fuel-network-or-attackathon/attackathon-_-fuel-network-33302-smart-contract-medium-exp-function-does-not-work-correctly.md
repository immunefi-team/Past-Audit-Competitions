# Attackathon \_ Fuel Network 33302 - \[Smart Contract - Medium] Exp function does not work correctly

Submitted on Wed Jul 17 2024 10:20:43 GMT-0400 (Atlantic Standard Time) by @shadowHunter for [Attackathon | Fuel Network](https://immunefi.com/bounty/fuel-network-attackathon/)

Report ID: #33302

Report type: Smart Contract

Report severity: Medium

Target: <https://github.com/FuelLabs/sway-libs/tree/0f47d33d6e5da25f782fc117d4be15b7b12d291b>

Impacts:

* Griefing (e.g. no profit motive for an attacker, but damage to the users or the protocol)
* Compiler bug

## Description

## Brief/Intro

It seems like `exp` function will always return `UFP128::from((1, 0))` no matter what argument is passed. This happens due to mistake in the return variable

```diff
-- let res = one;
++ let res = one + _res_minus_1;
```

## Vulnerability Details

1. Observe the `exp` function

```
impl Exponent for UFP128 {
    fn exp(exponent: Self) -> Self {
        let one = UFP128::from((1, 0));
        let p2 = one / UFP128::from((2, 0));
        let p3 = one / UFP128::from((6, 0));
        let p4 = one / UFP128::from((24, 0));
        let p5 = one / UFP128::from((120, 0));
        let p6 = one / UFP128::from((720, 0));
        let p7 = one / UFP128::from((5040, 0));

        // common technique to counter losing sugnifucant numbers in usual approximation
        let _res_minus_1 = exponent + exponent * exponent * (p2 + exponent * (p3 + exponent * (p4 + exponent * (p5 + exponent * (p6 + exponent * p7)))));
        let res = one;
        res
    }
}
```

2. Observe that res is simply returning `one` instead of `one + _res_minus_1` which will be correct exp

## Impact Details

Dapp relying on sway library will find itself with incorrect computation of exp. If the resulting was financial calculation then this will cause fund loss to the dapp user

## References

<https://github.com/FuelLabs/sway-libs/blob/0f47d33d6e5da25f782fc117d4be15b7b12d291b/libs/src/fixed\\_point/ufp128.sw#L480>

## Proof of concept

## Proof of Concept

```
 fn foo() {
	let mut one = UFP128::from((1, 0));
    let two = UFP128::from((2, 0));
    res = UFP128::exp(two);
    assert(res != one);
         
     }
```
