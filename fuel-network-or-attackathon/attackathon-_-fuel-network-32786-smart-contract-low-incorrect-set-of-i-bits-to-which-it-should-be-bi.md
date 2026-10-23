# Attackathon \_ Fuel Network 32786 - \[Smart Contract - Low] incorrect set of i bits to which it should

Submitted on Tue Jul 02 2024 13:13:27 GMT-0400 (Atlantic Standard Time) by @zeroK for [Attackathon | Fuel Network](https://immunefi.com/bounty/fuel-network-attackathon/)

Report ID: #32786

Report type: Smart Contract

Report severity: Low

Target: <https://github.com/FuelLabs/sway-libs/tree/0f47d33d6e5da25f782fc117d4be15b7b12d291b>

Impacts:

* Contract fails to deliver promised returns, but doesn't lose value

## Description

## Brief/Intro

the i256 is used to create/refer to signed value for u256 while sway can not handle negative value, however the bits function return incorrect bit value which is 128 bits, this is not true as i128 return 128 bits and i256 should return 256 bits same as shown in the `sway-core/primitives.sw` std.

## Vulnerability Details

the bits sets to 128 for i256 as shown below:

````sway
impl I256 {
    /// The size of this type in bits.
    ///
    /// # Returns
    ///
    /// [u64] - The defined size of the `I256` type.
    ///
    /// # Examples
    ///
    /// ``sway
    /// use sway_libs::signed_integers::i256::I256;
    ///
    /// fn foo() {
    ///     let bits = I256::bits();
    ///     assert(bits == 128);
    /// }
    /// ```
    pub fn bits() -> u64 {
        128
    }


````

but this is not true as i128 return 128 bits too:

````sway
impl I128 {
    /// The size of this type in bits.
    ///
    /// # Returns
    ///
    /// [u64] - The defined size of the `I128` type.
    ///
    /// # Examples
    ///
    /// ``sway
    /// use sway_libs::signed_integers::i128::I128;
    ///
    /// fn foo() {
    ///     let bits = I128::bits();
    ///     assert(bits == 128);
    /// }
    /// ```
    pub fn bits() -> u64 {
        128
    }


````

while in the primitives its clear that u256 should return 256 bit:

````sway
    /// The size of this integer type in bits.
    ///
    /// # Returns
    ///
    /// * [u32] - The number of bits for a `u256`.
    ///
    /// # Examples
    ///
    /// ```sway
    /// fn foo() {
    ///     let bits = u256::bits();
    ///     assert(bits == 256);
    /// }
    /// ```
    pub fn bits() -> u64 {
        256
    }
````

## Impact Details

the i256.sw bit function return incorrect bit number.

## References

return 256 bit in the bit function for i256.sw lib

## Proof of concept

## Proof of Concept

run the test right below the bit function in i256.sw:

```sway

#[test] 
fn test_bits() {
    let bitI = I256::bits();
    let u256Bit = u256::bits();

    assert(bitI != u256Bit);
}

```
