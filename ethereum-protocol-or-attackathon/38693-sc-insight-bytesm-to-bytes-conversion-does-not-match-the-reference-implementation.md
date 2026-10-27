# #38693 \[SC-Insight] BytesM to Bytes conversion does not match the reference implementation

**Submitted on Jan 10th 2025 at 08:24:12 UTC by @impermanentW for** [**Attackathon | Ethereum Protocol**](https://immunefi.com/audit-competition/ethereum-protocol-attackathon)

* **Report ID:** #38693
* **Report Type:** Smart Contract
* **Report severity:** Insight
* **Target:** <https://github.com/vyperlang/vyper>
* **Impacts:**
  * (Compiler) Unexpected behavior

## Description

## Brief/Intro

According to the Vyper doc (see ref 1) a small Python reference implementation for conversion logic is maintained as part of Vyper’s test suite (see ref 2)

However behavior of the compiler does not match the reference when it comes to BytesM to Bytes\[] conversion

## Vulnerability Details

The reference specifies the `BytesM_T` -> `Bytes_T` conversion should be possible if the size of the output type is at least as big as the in type (see ref 3)

But the compiler does not implement that behavior correctly, this is done in `vyper/builtins/_convert.py`(see ref 4) which does not allow bytesM as an input type for `to_bytes()`

## Impact Details

Conversions that should work according to the reference are throwing an error at compile time.

## References

1. <https://docs.vyperlang.org/en/stable/types.html#type-conversions>
2. <https://github.com/vyperlang/vyper/blob/v0.4.0/tests/functional/builtins/codegen/test\\_convert.py>
3. <https://github.com/vyperlang/vyper/blob/e9db8d9f7486eae38f5b86531629019ad28f514e/tests/functional/builtins/codegen/test\\_convert.py#L83-L86>
4. <https://github.com/vyperlang/vyper/blob/e9db8d9f7486eae38f5b86531629019ad28f514e/vyper/builtins/\\_convert.py#L447-L449>

## Proof of Concept

## Proof of Concept

Compile the following code using Vyper v0.4.0

```
#pragma version ^0.4.0

def test(test: bytes1) -> Bytes[1]:
    x: Bytes[1] = convert(test, Bytes[1])
    return x
```

Fails with `vyper.exceptions.TypeMismatch: Can't convert bytes1 to Bytes[1]` even though both `_bits_of_type()` are equal so the conversion should be doable according to the reference
