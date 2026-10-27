# Attackathon \_ Fuel Network 33401 - \[Smart Contract - Insight] insight compiler crash - trait dummy m

Submitted on Fri Jul 19 2024 13:27:40 GMT-0400 (Atlantic Standard Time) by @cyberthirst for [Attackathon | Fuel Network](https://immunefi.com/bounty/fuel-network-attackathon/)

Report ID: #33401

Report type: Smart Contract

Report severity: Insight

Target: <https://github.com/FuelLabs/sway/tree/v0.61.2>

Impacts:

* Compiler bug

## Description

## Brief/Intro

Certain classes of contracts utilizing `abi_decode` crash the compiler.

## Vulnerability Details

The compiler crashes with the following message:

```
2527 | pub trait AbiDecode {
2528 |     fn abi_decode(ref mut buffer: BufferReader) -> Self;
     |        ^^^^^^^^^^ Internal compiler error: Method abi_decode_37 is a trait method dummy and was not properly replaced.
Please file an issue on the repository and include the code that triggered this error.
2529 | }
     |
```

## Impact Details

Certain classes of programs are not possible to compile due to a compiler bug.

## Proof of concept

## Proof of Concept

src/main.sw

```
contract;

use std::bytes::*;

abi MyAbi {
    fn test() -> u64;
}

abi FakeAbi {
    fn test() -> Bytes;
}

impl MyAbi for Contract {
    fn test() -> u64 {
        64
    }
}

#[test]
fn test() {
    let caller = abi(FakeAbi, CONTRACT_ID);
    let res  = caller.test();
    assert(res.len() == 64);
    let s: str[30] = abi_decode(res.as_raw_slice());
}
```

forc.toml:

```
[project]
authors = ["cyberthirst"]
entry = "main.sw"
license = "Apache-2.0"
name = "abi-decode-poc"

[dependencies]
```
