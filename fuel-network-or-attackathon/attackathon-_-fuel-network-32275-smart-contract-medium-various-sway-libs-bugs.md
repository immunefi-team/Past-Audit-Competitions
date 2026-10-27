# Attackathon \_ Fuel Network 32275 - \[Smart Contract - Medium] Various Sway Libs Bugs

Submitted on Mon Jun 17 2024 08:57:19 GMT-0400 (Atlantic Standard Time) by @anatomist for [Attackathon | Fuel Network](https://immunefi.com/bounty/fuel-network-attackathon/)

Report ID: #32275

Report type: Smart Contract

Report severity: Medium

Target: <https://github.com/FuelLabs/sway-libs/tree/0f47d33d6e5da25f782fc117d4be15b7b12d291b>

Impacts:

* Incorrect library function behaviors

## Description

## Brief/Intro

[sway-libs](https://github.com/FuelLabs/sway-libs/tree/0f47d33d6e5da25f782fc117d4be15b7b12d291b/libs) provides a lot of utility functions to help contract development. Thus its correctness is important to ensure developers don't suffer from hidden vulnerabilities. We've identified a large amount of bugs within the libraries, and will go through each bug class in this report.

## Vulnerability Details

### Incorrect bit width for `signed_integers` and `fixed_point`

* The `i256::bits` is incorrectly set to 128.
* Strictly speaking, `ifp64` has \~33 bits and not 64 bits. Similarly `ifp128` has \~64 bits and `ifp256` has \~129 bits

### Incorrect two's complement implementation for `signed_integers`

* The twos complement currently doesn't make sense. For instance, two's complement for -1i8 should be 0xff, but the implementation returns 0x81

### Incorrect indent for `i128` and `i256`

* `i128` indent should be `1<<127`, and `i256` indent should be `1<<255`

### Unnecessary subtract In `UFP32::fract`

* The subtraction of `u32::max` in `UFP32::fract` will lead to underflow and `panic`

### Incorrect usage of `IFP::from`

* `IFP::from` treats input as raw underlying instead of integers (different from `signed_integer`). It is misused in several places

### `IFP::ceil` overflow on values close to `min`

* `IFP::ceil takes` ceil`of underlying and then adjusts the values. Thus values near to`min\` might suffer from incorrect overflow

### Incorrect panic of `fixed_point::round`

* `fixed_point::round` takes both `ceil` / `floor` and use them to derive the rounded values. Thus even if the final result falls within the valid range, function would still `panic` if either `ceil` of `floor` does.

### Incorrect denominator in `UFP128::sqrt` and `UFP32::sqrt`

* The denominator for `UFP128::sqrt` should be `1<<32` instead of `2<<32`
* The denominator for `UFP32::sqrt` should be `1<<8` instead of `1<<16`

### `UFP::sqrt` precision loss

* The implementation of `UFP::sqrt` loses 1/4 of precision in terms of its size.

### UFP::pow\` premature overflow leads to revert

* `UFP::pow` directly calls `pow` on the underlying value, which will easily overflow for values >= 1.

### `fixed_point::exp` precision loss

* `UFP::exp` is wildly inaccurate. We haven't properly estimated to amount of precision lost. But the PoC attached should demonstrate the idea.

### `UFP32::exp` and `IFP256::exp` uses incorrect taylor series

* `UFP32::exp` uses the taylor series for `UFP64::exp`
* `IFP256::exp` uses the taylor series for `IFP128::exp`

### Incorrect `min` for `IFP`

* `min` for `IFP` should use `type::MAX` as underlying value

### Lack of consideration of negative zero for `IFP` compare functions

* `IFP::gt` and `IFP::lt` doesn't consider negative zeros. This might lead to incorrect results.
* `IFP::non_negative` doesn't consider negative zeroes. And might return `false` for those.

### Incorrect adjustment for `IFP::ceil`

* `IFP::ceil` increases `underlying` when the value is negative, while the correct implementation should be to decrease `underlying`.

## Impact Details

It is hard to provide a concrete impact for library functions. But let's just say users may not be able to predict its behavior and might be caught of guard when those functions do unexpected stuff. And given the abundance of bugs, it seems extremely likely that users would run into those.

## References

### Incorrect bit width for `signed_integers` and `fixed_point`

* `https://github.com/FuelLabs/sway-libs/blob/0f47d33d6e5da25f782fc117d4be15b7b12d291b/libs/src/signed_integers/i256.sw#L88`
* `https://github.com/FuelLabs/sway-libs/blob/0f47d33d6e5da25f782fc117d4be15b7b12d291b/libs/src/fixed_point/ifp64.sw#L46`
* `https://github.com/FuelLabs/sway-libs/blob/0f47d33d6e5da25f782fc117d4be15b7b12d291b/libs/src/fixed_point/ifp128.sw#L46`
* `https://github.com/FuelLabs/sway-libs/blob/0f47d33d6e5da25f782fc117d4be15b7b12d291b/libs/src/fixed_point/ifp256.sw#L46`

### Incorrect two's complement implementation for `signed_integers`

* `https://github.com/FuelLabs/sway-libs/blob/0f47d33d6e5da25f782fc117d4be15b7b12d291b/libs/src/signed_integers/i8.sw#L398`
* `https://github.com/FuelLabs/sway-libs/blob/0f47d33d6e5da25f782fc117d4be15b7b12d291b/libs/src/signed_integers/i16.sw#L399`
* `https://github.com/FuelLabs/sway-libs/blob/0f47d33d6e5da25f782fc117d4be15b7b12d291b/libs/src/signed_integers/i32.sw#L398`
* `https://github.com/FuelLabs/sway-libs/blob/0f47d33d6e5da25f782fc117d4be15b7b12d291b/libs/src/signed_integers/i64.sw#L399`
* `https://github.com/FuelLabs/sway-libs/blob/0f47d33d6e5da25f782fc117d4be15b7b12d291b/libs/src/signed_integers/i128.sw#L423`
* `https://github.com/FuelLabs/sway-libs/blob/0f47d33d6e5da25f782fc117d4be15b7b12d291b/libs/src/signed_integers/i256.sw#L419`

### Incorrect indent for `i128` and `i256`

* `https://github.com/FuelLabs/sway-libs/blob/0f47d33d6e5da25f782fc117d4be15b7b12d291b/libs/src/signed_integers/i128.sw#L38`
* `https://github.com/FuelLabs/sway-libs/blob/0f47d33d6e5da25f782fc117d4be15b7b12d291b/libs/src/signed_integers/i256.sw#L39`

### Unnecessary subtract In `UFP32::fract`

* `https://github.com/FuelLabs/sway-libs/blob/0f47d33d6e5da25f782fc117d4be15b7b12d291b/libs/src/fixed_point/ufp32.sw#L391`

### Incorrect usage of `IFP::from`

* `https://github.com/FuelLabs/sway-libs/blob/0f47d33d6e5da25f782fc117d4be15b7b12d291b/libs/src/fixed_point/ifp64.sw#L416`
* `https://github.com/FuelLabs/sway-libs/blob/0f47d33d6e5da25f782fc117d4be15b7b12d291b/libs/src/fixed_point/ifp64.sw#L475`
* `https://github.com/FuelLabs/sway-libs/blob/0f47d33d6e5da25f782fc117d4be15b7b12d291b/libs/src/fixed_point/ifp64.sw#L476`
* `https://github.com/FuelLabs/sway-libs/blob/0f47d33d6e5da25f782fc117d4be15b7b12d291b/libs/src/fixed_point/ifp128.sw#L416`
* `https://github.com/FuelLabs/sway-libs/blob/0f47d33d6e5da25f782fc117d4be15b7b12d291b/libs/src/fixed_point/ifp128.sw#L475`
* `https://github.com/FuelLabs/sway-libs/blob/0f47d33d6e5da25f782fc117d4be15b7b12d291b/libs/src/fixed_point/ifp128.sw#L476`
* `https://github.com/FuelLabs/sway-libs/blob/0f47d33d6e5da25f782fc117d4be15b7b12d291b/libs/src/fixed_point/ifp256.sw#L554`

### `IFP::ceil` overflow on values close to `min`

* `https://github.com/FuelLabs/sway-libs/blob/0f47d33d6e5da25f782fc117d4be15b7b12d291b/libs/src/fixed_point/ifp64.sw#L473`
* `https://github.com/FuelLabs/sway-libs/blob/0f47d33d6e5da25f782fc117d4be15b7b12d291b/libs/src/fixed_point/ifp128.sw#L473`
* `https://github.com/FuelLabs/sway-libs/blob/0f47d33d6e5da25f782fc117d4be15b7b12d291b/libs/src/fixed_point/ifp256.sw#L473`

### Incorrect panic of `fixed_point::round`

* `https://github.com/FuelLabs/sway-libs/blob/0f47d33d6e5da25f782fc117d4be15b7b12d291b/libs/src/fixed_point/ufp32.sw#L443`
* `https://github.com/FuelLabs/sway-libs/blob/0f47d33d6e5da25f782fc117d4be15b7b12d291b/libs/src/fixed_point/ufp64.sw#L437`
* `https://github.com/FuelLabs/sway-libs/blob/0f47d33d6e5da25f782fc117d4be15b7b12d291b/libs/src/fixed_point/ufp128.sw#L383`
* `https://github.com/FuelLabs/sway-libs/blob/0f47d33d6e5da25f782fc117d4be15b7b12d291b/libs/src/fixed_point/ifp64.sw#L515`
* `https://github.com/FuelLabs/sway-libs/blob/0f47d33d6e5da25f782fc117d4be15b7b12d291b/libs/src/fixed_point/ifp128.sw#L515`
* `https://github.com/FuelLabs/sway-libs/blob/0f47d33d6e5da25f782fc117d4be15b7b12d291b/libs/src/fixed_point/ifp256.sw#L515`

### Incorrect denominator in `UFP128::sqrt` and `UFP32::sqrt`

* `https://github.com/FuelLabs/sway-libs/blob/0f47d33d6e5da25f782fc117d4be15b7b12d291b/libs/src/fixed_point/ufp128.sw#L443`
* `https://github.com/FuelLabs/sway-libs/blob/0f47d33d6e5da25f782fc117d4be15b7b12d291b/libs/src/fixed_point/ufp32.sw#L462`

### `UFP::sqrt` precision loss

* `https://github.com/FuelLabs/sway-libs/blob/0f47d33d6e5da25f782fc117d4be15b7b12d291b/libs/src/fixed_point/ufp32.sw#L462`
* `https://github.com/FuelLabs/sway-libs/blob/0f47d33d6e5da25f782fc117d4be15b7b12d291b/libs/src/fixed_point/ufp64.sw#L456`
* `https://github.com/FuelLabs/sway-libs/blob/0f47d33d6e5da25f782fc117d4be15b7b12d291b/libs/src/fixed_point/ufp128.sw#L442`

### UFP::pow\` premature overflow leads to revert

* `https://github.com/FuelLabs/sway-libs/blob/0f47d33d6e5da25f782fc117d4be15b7b12d291b/libs/src/fixed_point/ufp32.sw#L492`
* `https://github.com/FuelLabs/sway-libs/blob/0f47d33d6e5da25f782fc117d4be15b7b12d291b/libs/src/fixed_point/ufp32.sw#L497`
* `https://github.com/FuelLabs/sway-libs/blob/0f47d33d6e5da25f782fc117d4be15b7b12d291b/libs/src/fixed_point/ufp64.sw#L487`
* `https://github.com/FuelLabs/sway-libs/blob/0f47d33d6e5da25f782fc117d4be15b7b12d291b/libs/src/fixed_point/ufp128.sw#L450`
* `https://github.com/FuelLabs/sway-libs/blob/0f47d33d6e5da25f782fc117d4be15b7b12d291b/libs/src/fixed_point/ufp128.sw#L452`

### `fixed_point::exp` precision loss

* `https://github.com/FuelLabs/sway-libs/blob/0f47d33d6e5da25f782fc117d4be15b7b12d291b/libs/src/fixed_point/ufp32.sw#L471`
* `https://github.com/FuelLabs/sway-libs/blob/0f47d33d6e5da25f782fc117d4be15b7b12d291b/libs/src/fixed_point/ufp64.sw#L465`
* `https://github.com/FuelLabs/sway-libs/blob/0f47d33d6e5da25f782fc117d4be15b7b12d291b/libs/src/fixed_point/ufp128.sw#L469`
* `https://github.com/FuelLabs/sway-libs/blob/0f47d33d6e5da25f782fc117d4be15b7b12d291b/libs/src/fixed_point/ifp64.sw#L533`
* `https://github.com/FuelLabs/sway-libs/blob/0f47d33d6e5da25f782fc117d4be15b7b12d291b/libs/src/fixed_point/ifp128.sw#L533`
* `https://github.com/FuelLabs/sway-libs/blob/0f47d33d6e5da25f782fc117d4be15b7b12d291b/libs/src/fixed_point/ifp256.sw#L533`

### `UFP32::exp` and `IFP256::exp` uses incorrect taylor series

* `https://github.com/FuelLabs/sway-libs/blob/0f47d33d6e5da25f782fc117d4be15b7b12d291b/libs/src/fixed_point/ufp32.sw#L475`
* `https://github.com/FuelLabs/sway-libs/blob/0f47d33d6e5da25f782fc117d4be15b7b12d291b/libs/src/fixed_point/ifp256.sw#L537`

### Incorrect `min` for `IFP`

* `https://github.com/FuelLabs/sway-libs/blob/0f47d33d6e5da25f782fc117d4be15b7b12d291b/libs/src/fixed_point/ifp64.sw#L87`
* `https://github.com/FuelLabs/sway-libs/blob/0f47d33d6e5da25f782fc117d4be15b7b12d291b/libs/src/fixed_point/ifp128.sw#L87`
* `https://github.com/FuelLabs/sway-libs/blob/0f47d33d6e5da25f782fc117d4be15b7b12d291b/libs/src/fixed_point/ifp256.sw#L87`

### Lack of consideration of negative zero for `IFP` compare functions

* `https://github.com/FuelLabs/sway-libs/blob/0f47d33d6e5da25f782fc117d4be15b7b12d291b/libs/src/fixed_point/ifp64.sw#L208`
* `https://github.com/FuelLabs/sway-libs/blob/0f47d33d6e5da25f782fc117d4be15b7b12d291b/libs/src/fixed_point/ifp64.sw#L220`
* `https://github.com/FuelLabs/sway-libs/blob/0f47d33d6e5da25f782fc117d4be15b7b12d291b/libs/src/fixed_point/ifp128.sw#L208`
* `https://github.com/FuelLabs/sway-libs/blob/0f47d33d6e5da25f782fc117d4be15b7b12d291b/libs/src/fixed_point/ifp128.sw#L220`
* `https://github.com/FuelLabs/sway-libs/blob/0f47d33d6e5da25f782fc117d4be15b7b12d291b/libs/src/fixed_point/ifp256.sw#L208`
* `https://github.com/FuelLabs/sway-libs/blob/0f47d33d6e5da25f782fc117d4be15b7b12d291b/libs/src/fixed_point/ifp256.sw#L220`
* `https://github.com/FuelLabs/sway-libs/blob/0f47d33d6e5da25f782fc117d4be15b7b12d291b/libs/src/fixed_point/ifp64.sw#L193`
* `https://github.com/FuelLabs/sway-libs/blob/0f47d33d6e5da25f782fc117d4be15b7b12d291b/libs/src/fixed_point/ifp128.sw#L193`
* `https://github.com/FuelLabs/sway-libs/blob/0f47d33d6e5da25f782fc117d4be15b7b12d291b/libs/src/fixed_point/ifp256.sw#L193`

### Incorrect adjustment for `IFP::ceil`

* `https://github.com/FuelLabs/sway-libs/blob/0f47d33d6e5da25f782fc117d4be15b7b12d291b/libs/src/fixed_point/ifp64.sw#L475`
* `https://github.com/FuelLabs/sway-libs/blob/0f47d33d6e5da25f782fc117d4be15b7b12d291b/libs/src/fixed_point/ifp128.sw#L475`
* `https://github.com/FuelLabs/sway-libs/blob/0f47d33d6e5da25f782fc117d4be15b7b12d291b/libs/src/fixed_point/ifp256.sw#L475`

## Proof of concept

## Proof of Concept

### Incorrect bit width for `signed_integers` and `fixed_point`

```
#[test]
fn i256_bits() -> () {
    assert(I256::bits() == 256);
    ()
}
```

### Incorrect two's complement implementation for `signed_integers`

```
#[test]
fn i8_twos_compliment() -> () {
    let val = I8::neg_from(1);
    let complement = val.twos_complement();
    assert(complement.underlying() == 255);
}
```

### Incorrect indent for `i128` and `i256`

```
#[test]
fn i128_indent() -> () {
    // -2^65 should be in range
    let u128_val = U128::from((2, 0));
    let i128_val = I128::neg_from(u128_val);
    ()
}
```

### Unnecessary subtract In `UFP32::fract`

```
#[test]
fn ufp32_fract_incorrect_underflow() -> () {
    let zero = UFP32::from_uint(0);
    let one = UFP32::from_uint(1);
    assert(one.fract() == zero);
    ()
}
```

### Incorrect usage of `IFP::from`

```
#[test]
fn ifp64_incorrect_usage_of_from() -> () {
    let ifp64_zero = IFP64::zero();
    let ifp64_val = ifp64_zero - IFP64::from(UFP32::from(65537u32));                    //-0x1.0001

    assert((ifp64_zero - IFP64::from_uint(2u32)) == ifp64_val.floor());
    ()
}
```

### `IFP::ceil` overflow on values close to `min`

```
#[test]
fn ifp128_ceil_incorrect_overflow() -> () {
    let ifp128_zero = IFP128::zero();
    let ifp128_val = ifp128_zero - IFP128::from(UFP64::from(18446744069414584321));     //-0xffffffff.00000001
    let expected = ifp128_zero - IFP128::from(UFP64::from(18446744069414584320));       //-0xffffffff.00000000

    assert(expected == ifp64_val.ceil());
    ()
}
```

### Incorrect panic of `fixed_point::round`

```
#[test]
fn ufp64_round_incorrect_overflow() -> () {
    let val = UFP64::from(18446744069414584321);        //0xffffffff.00000001
    assert(UFP64::from(18446744069414584320) == val.round());
    ()
}
```

### Incorrect denominator in `UFP128::sqrt` and `UFP32::sqrt`

```
#[test]
fn ufp128_sqrt_incorrect_denom() -> () {
    let val = UFP128::from((0, 1));                             //0x0000000000000000.0000000000000001
    assert(UFP128::from((0, 4294967296)) == val.sqrt());        //0x0000000000000000.0000000100000000
    ()
}
```

### `UFP::sqrt` precision loss

```
#[test]
fn ufp64_sqrt_precision_loss() -> () {
    let val = UFP64::from(3);                           //0x00000000.00000002
    assert(UFP64::from(92681) == val.sqrt());           //0x00000000.00016a09
    ()
}
```

### UFP::pow\` premature overflow leads to revert

```
#[test]
fn ufp64_pow_premature_overflow() -> () {
    let val = UFP64::from_uint(1);                      //0x00000001.00000000
    assert(val == val.pow(2u32));
    ()
}
```

### `fixed_point::exp` precision loss

```
#[test]
fn u64_imprecise_exponent() -> () {
    let val = UFP64::from_uint(1);
    let val = UFP64::exp(val);
    assert(val == UFP64::from(11674931554));            //0x2.b7e15162
    ()
}
```

### `UFP32::exp` and `IFP256::exp` uses incorrect taylor series

### Incorrect `min` for `IFP`

```
#[test]
fn ifp128_incorrect_min() -> () {
    let ifp128_zero = IFP128::zero();
    let ifp128_min = IFP128::min();
    let ifp128_max = IFP128::max();

    assert(ifp128_zero - ifp128_max == ifp128_min);
    ()
}
```

### Lack of consideration of negative zero for `IFP` compare functions

```
#[test]
fn ifp128_negative_zero() -> () {
    let val = IFP128::from(UFP64::from(1));
    let zero = IFP128::zero();
    let neg_zero = (zero - val).fract();
    assert((zero == neg_zero) != (zero > neg_zero));
    assert(neg_zero.non_negative());
    ()
}
```

### Incorrect adjustment for `IFP::ceil`

```
#[test]
fn ifp128_incorrect_ceil() -> () {
    let ifp128_zero = IFP128::zero();
    let ifp128_val = ifp128_zero - IFP128::from(UFP64::from(1));

    assert(ifp128_val.ceil() > ifp128_val);
    ()
}
```
