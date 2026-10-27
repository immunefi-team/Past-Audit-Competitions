# Attackathon \_ Fuel Network 33039 - \[Smart Contract - High] The subtraction function is not correctly

Submitted on Tue Jul 09 2024 20:44:32 GMT-0400 (Atlantic Standard Time) by @Schnilch for [Attackathon | Fuel Network](https://immunefi.com/bounty/fuel-network-attackathon/)

Report ID: #33039

Report type: Smart Contract

Report severity: High

Target: <https://github.com/FuelLabs/sway-libs/tree/0f47d33d6e5da25f782fc117d4be15b7b12d291b>

Impacts:

* Griefing (e.g. no profit motive for an attacker, but damage to the users or the protocol)
* Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield

## Description

## Brief/Intro

In the subtraction functions for signed integers in the Sway libraries, the case when the `other` parameter is negative is handled incorrectly. When the `other` parameter is negative, it is still simply subtracted instead of being added, as would be mathematically correct.

## Vulnerability Details

In the sway-libs, within the signed integer library, every signed integer has a subtraction function (see 1. reference. I have linked to the I64 subtraction function as an example, but the subtraction functions of all signed integers have the same bug). The parts of the function that handle cases where the `other` parameter is negative are incorrectly implemented (see 2nd and 3rd references). When the `other` parameter is negative, it is subtracted, but it should actually be added.

```rust
305:             res = Self::from_uint(self.underlying - Self::indent() + other.underlying);
```

This part of the code is responsible for when `self` is positive and `other` is negative. Here you can see that `self` and `other` are being added, but the problem is that `other` is added as a negative value. Therefore, `other` is subtracted from `self`. For the result to be calculated correctly, the absolute value of other (so it is no longer negative) should be added to self. It would look like this:

```rust
305:             res = Self::from_uint(self.underlying + (Self::indent() - other.underlying));
```

A similar error occurs when self and other both are negative:

```rust
313:             if self.underlying < other.underlying {
314:                 res = Self::from_uint(other.underlying - self.underlying + Self::indent());
315:             } else {
316:                 res = Self::from_uint(self.underlying + other.underlying - Self::indent());
317:             }
```

This part could be fixed like this:

```rust
313:             if self.underlying < other.underlying {
314:                 res = Self::from_uint((Self::indent - other.underlying) + self.underlying);
315:             } else {
316:                 res = Self::from_uint(self.underlying + (Self::indent() - other.underlying));
317:             }
```

### An Example

self = 9223372036854775908 (which is 100 when you subtract 2^63); other = 9223372036854775608 (which is -200 when you subtract 2^63). This means the result should be 9223372036854776108 because 100 - (-200) = 300 and 300 + 2^63 = 9223372036854776108. However, the error occurs because instead of adding 200 as in the example, it is subtracted. And the resulting value would be 9223372036854775708 (which is -100).

## Impact Details

This bug can have various consequences depending on what the signed integer subtraction is used for. If this subtraction function is used to calculate token amounts on which transfers are then based, it can lead to a loss of tokens because the calculated amount may be incorrect. A user could lose money this way, or in the worst case, an attacker could deliberately exploit this vulnerability to steal money if he controls the `other` parameter.

## References

1. <https://github.com/FuelLabs/sway-libs/blob/2a869c583d2ab9fbe8de17a3301d928b224062c7/libs/src/signed\\_integers/i256.sw#L375-L406>
2. <https://github.com/FuelLabs/sway-libs/blob/2a869c583d2ab9fbe8de17a3301d928b224062c7/libs/src/signed\\_integers/i64.sw#L310-L318>
3. <https://github.com/FuelLabs/sway-libs/blob/2a869c583d2ab9fbe8de17a3301d928b224062c7/libs/src/signed\\_integers/i64.sw#L302-L305>

## Proof of concept

## Proof of Concept

For the POC of this bug, a sway project is required. This can be created with this command: `forc new substraction-bug`

After that, this dependency needs to be added to the Forc.toml: `sway_libs = { git = "https://github.com/FuelLabs/sway-libs", tag = "v0.22.0" }`

Then the following code for the PoC must be inserted into the main.sw file:

```rust
contract;

use sway_libs::signed_integers::i64::*; //The PoC is only an example with the i64, the other signed integer data types are also affected by the bug

abi MyContract {
    fn test_function(a: I64, b: I64) -> I64;
}

impl MyContract for Contract {
    fn test_function(a: I64, b: I64) -> I64 { //The function that subtracts two signed integers to show the bug in the subtraction
        a - b
    }
}

#[test]
fn test_bug() {
    let caller = abi(MyContract, CONTRACT_ID);
    let a = I64::from_uint(9223372036854775708); //That is -100
    let b = I64::from_uint(9223372036854775108); //That is -700

    let result: I64 = caller.test_function{}(a, b); //test_function will return a false result because a and b are negative
    assert(result == I64::from_uint(9223372036854775008)) //Shows that the result is wrong because 9223372036854775008 is -800 but -100-(-700) is actually 600 and therefore 9223372036854776408 would be the correct result
}
```

The PoC can then be started with `forc test`. This PoC demonstrates that the subtraction function of the signed integer library does not work. Although this is only a minimal PoC, if this function is used for the calculation of token amounts, it can easily lead to a loss of tokens.
