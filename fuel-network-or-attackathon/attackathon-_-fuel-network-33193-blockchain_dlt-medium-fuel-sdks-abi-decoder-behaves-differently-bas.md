# Attackathon \_ Fuel Network 33193 - \[Blockchain\_DLT - Medium] Fuel SDKs ABI Decoder Behaves Different

Submitted on Sat Jul 13 2024 21:25:36 GMT-0400 (Atlantic Standard Time) by @savi0ur for [Attackathon | Fuel Network](https://immunefi.com/bounty/fuel-network-attackathon/)

Report ID: #33193

Report type: Blockchain/DLT

Report severity: Medium

Target: <https://github.com/fuellabs/fuels-rs/tree/d3ac1d3f8910cc12c662ccbe5ff51d9e9354ed1a>

Impacts:

* A bug in the respective layer 0/1/2 network code that results in unintended smart contract behavior with no concrete funds at direct risk

## Description

## Bug Description

Inside `fuels-rs` module, we have `fuels-core` module which defines a functionality to ABI encode and decode.

<https://github.com/FuelLabs/fuels-rs/blob/d3ac1d3f8910cc12c662ccbe5ff51d9e9354ed1a/packages/fuels-core/src/codec/abi\\_decoder/bounded\\_decoder.rs#L336-L342>

```rust
fn peek_length(bytes: &[u8]) -> Result<usize> {
    let slice = peek_fixed::<LENGTH_BYTES_SIZE>(bytes)?;

    u64::from_be_bytes(*slice)
        .try_into()//@audit-issue
        .map_err(|_| error!(Other, "could not convert `u64` to `usize`"))
}
```

While decoding data, it first finds the length of that data by reading first 8 bytes using `peek_length` function. This 8 byte data is converted to `usize`. Since the size of `usize` is platform dependent, this conversion behaves differently based on the platform architecture on which this code runs.

For example, When run on 64 bit machine, where `usize` is 64 bit wide, if length of data (`bytes argument`) is `> u32::MAX`, it will correctly returns length of the data by converting this length to `usize` using `try_into` function. Since `usize` is 64 bit wide, it will be able to store this result.

But when run on 32-bit machine, where `usize` is 32 bit wide, if length of data is `> u32::MAX`, `peek_length` function will revert with error - `"could not convert u64 to usize"`

As we have seen, based on the platform architecture, this function behaves differently.

## Impact

Fuels SDK's ABI encoding and decoding behaves differently based on architecture of the system.

## Recommendation

Avoid conversions that may behave differently across architectures. Use architecture independent data types for performing required operations.

## References

<https://github.com/FuelLabs/fuels-rs/blob/d3ac1d3f8910cc12c662ccbe5ff51d9e9354ed1a/packages/fuels-core/src/codec/abi\\_decoder/bounded\\_decoder.rs#L336-L342>

## Proof of concept

## Proof Of Concept

`peek_length` function is called by every `decode_XXX` function in `bounded_decoder.rs`. We have used `ParamType::String` to show an issue using `decode_std_string(bytes)` function.

**Steps to Run using Foundry:**

* Change directory to `fuels-rs` directory i.e., `cd fuels-rs`
* Copy following test case in `packages/fuels-core/src/codec/abi_decoder/decode_as_debug_str.rs`
* Open terminal and run `cargo test --package fuels-core --lib -- codec::abi_decoder::decode_as_debug_str::tests::test_param_type_decode_bug_diff_arch --exact --show-output`

```rust
#[test]
fn test_param_type_decode_bug_diff_arch() -> Result<()> {
    let decoder = ABIDecoder::default();
    
    // Generate a string with 4GB
    let size: usize = 4 * 1024 * 1024 * 1024 + 10;
    let vec = vec!['a'; size];
    let large_string: String = vec.into_iter().collect();

    let token = crate::types::Token::String(large_string.clone());
    let result = crate::codec::ABIEncoder::default().encode(&[token])?;

    assert_eq!(
        large_string,
        decoder.decode_as_debug_str(
            &String::param_type(),
            &result
        )?
    );

    Ok(())
}
```

**Console Output:**

* On 64-bit machine, this test case will execute successfully.
* On 32-bit machine, this test case will fail.
