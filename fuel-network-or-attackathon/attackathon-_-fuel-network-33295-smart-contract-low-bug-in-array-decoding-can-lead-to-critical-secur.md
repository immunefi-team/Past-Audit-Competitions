# Attackathon \_ Fuel Network 33295 - \[Smart Contract - Low] Bug in array decoding can lead to critical

Submitted on Wed Jul 17 2024 06:58:04 GMT-0400 (Atlantic Standard Time) by @Solosync6 for [Attackathon | Fuel Network](https://immunefi.com/bounty/fuel-network-attackathon/)

Report ID: #33295

Report type: Smart Contract

Report severity: Low

Target: <https://github.com/FuelLabs/sway/tree/v0.61.2>

Impacts:

* Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield

## Description

## Brief/Intro

A vulnerability in the array decoding mechanism allows an attacker to manipulate array lengths during decoding. This can be exploited to set critical protocol parameters to zero, potentially leading to serious financial outcomes (for eg. liquidations in Defi).

## Vulnerability Details

The vulnerability arises from the improper validation of array lengths during decoding. Specifically, the decoding logic does **NOT** ensure that the length of the data being read matches the expected length for the array. This allows an attacker to provide a maliciously crafted encoded array of a **smaller size** but with a manipulated length, resulting in undefined behavior and potential security risks.

In sway-lib-core -> codec::AbiDecode, let's look at a following case where an array of size 3 is being decoded. Critically, the logic assumes that the input array is of size 3 without checking its actual length (see below).

```rust
impl<T> AbiDecode for [T; 3]
where
    T: AbiDecode,
{
    fn abi_decode(ref mut buffer: BufferReader) -> [T; 3] {
        let first: T = buffer.decode::<T>();
        let mut array = [first; 3];
        let mut i = 1;
        while i < 3 {
            array[i] = buffer.decode::<T>();
            i += 1;
        }
        array
    }
}

```

The output is an array of size 3. If a malicious attacker sends an encoded array with size 1, then the last 2 elements are decoded as 0 (ideally this should revert but it goes through).This error in decoding can cause serious issues for applications that use core library's encoding/decoding. I listed a practical scenario in impact section where this can be exploited.

**Any under-sized arrays after decoding will have 0's at the end of the array. Presence of these 0's can cause un-intended behavior downstream**

## Impact Details

This vulnerability can be exploited in various ways, leading to severe financial exploitation. Here is a possible practical example:

Imagine a DeFi protocol that maintains an array of parameters \[u64; 3] where the three parameters are:

**Collateral Ratio**: Minimum collateral required as a percentage of the loan. **Interest Rate**: Interest rate applied to the loan. **Liquidation Threshold**: The collateral value below which the loan can be liquidated.

An attacker can manipulate the decoding process to set the last two elements (Interest Rate and Liquidation Threshold) to zero. This could allow the attacker to:

*Set Liquidation Threshold to Zero*: Trigger liquidations at will, potentially leading to unauthorized liquidation of all loans.

*Borrow interest free loans* - by setting interest rate to 0, users can enjoy interest free loans

In either of the above case, borrowers or protocol will end up with permanent losses.

Array decoding is a basic primitive that can be used by every application built on Sway core libraries. Since the potential use is widespread, a small error in primitive can cause potentially huge losses downstream. Considering the potential impact, I'm rating this as CRITICAL risk.

## References

<https://github.com/FuelLabs/sway/blob/e1b1c2bee73e0ba825e07736cefa6c0abd079595/sway-lib-core/src/codec.sw#L3144>

## Proof of concept

Copy the POC into codec.sw file and run with the following command inside the sway-lib-core folder:

> forc test --logs --filter-exact test\_array\_decode\_vulnerability

## Proof of Concept

```
#[test]
fn test_array_decode_vulnerability() {
    // Create our actual array data of size 1
    let actual_array: [u8; 1] = [42];

    // Create a raw_slice using assert_encoding
    let encoded_slice = encode(actual_array);

    // Malicious length (much larger than our actual data)
    let malicious_length: u64 = 3; 

    // Create a BufferReader with the actual slice but a malicious length
    let mut reader = BufferReader::from_parts(encoded_slice.ptr(), malicious_length);

    // Attempt to decode the array using the BufferReader
    // This should trigger an out-of-bounds read
    let decoded_result = reader.decode::<[u8; 3]>(); //@audit this is decoding an under-sized array without check

    // Attempt to access the out-of-bounds elements
    let first_element = decoded_result[0];
    let second_element = decoded_result[1]; // this is 0
    let third_element = decoded_result[2];  // this is 0

    // Log the results
    __log(first_element);
    __log(second_element);
    __log(third_element);

    assert(first_element == 42);
    assert(second_element == 0);
    assert(third_element == 0);
}

fn alloc<T>(count: u64) -> raw_ptr {
    asm(size: __size_of::<T>() * count, ptr) {
        aloc size;
        move ptr hp;
        ptr: raw_ptr
    }
}
```
