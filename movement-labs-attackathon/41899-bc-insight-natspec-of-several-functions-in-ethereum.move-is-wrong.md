# #41899 \[BC-Insight] NatSpec of several functions in \`ethereum.move\` is wrong

**Submitted on Mar 19th 2025 at 09:07:07 UTC by @avoloder for** [**Attackathon | Movement Labs**](https://immunefi.com/audit-competition/movement-labs-attackathon)

* **Report ID:** #41899
* **Report Type:** Blockchain/DLT
* **Report severity:** Insight
* **Target:** <https://github.com/immunefi-team/attackathon-movement-aptos-core/tree/main>
* **Impacts:**

## Description

## Brief/Intro

NatSpec comments in several functions are wrong with regard when the functions should abort (`aptos-move/framework/aptos-framework /sources/ethereum.move`)

## Vulnerability Details

Functions `ethereum_address_no_eip55()` and `ethereum_address_20_bytes()` contain comments that they are expected to abort if the address does not conform to EIP-55 standards, which is not true.

`ethereum_address_no_eip55()` calls `assert_40_char_hex()` which checks if the address is a nonzero 40-character hexadecimal string, but not if the address is compliant with EIP-55 standard.

`ethereum_address_20_bytes()` only checks if the vector's length is 20, but also does not check if the address is a valid EIP-55 address.

## Impact Details

There is no security impact here, only documentation issue, which might cause confusion as to how the function should behave

## References

<https://github.com/immunefi-team/attackathon-movement-aptos-core/blob/627b4f9e0b63c33746fa5dae6cd672cbee3d8631/aptos-move/framework/aptos-framework/sources/ethereum.move#L31-L39>

<https://github.com/immunefi-team/attackathon-movement-aptos-core/blob/627b4f9e0b63c33746fa5dae6cd672cbee3d8631/aptos-move/framework/aptos-framework/sources/ethereum.move#L41-L49>

## Proof of Concept

Aligning NatSpec with the function's behavior prevents confusion about what the function is supposed to do, i.e., it clearly describes the function's expectations.

It is also a valuable enhancement for the following reasons:

* In collaborative development, it might be unclear whether these functions should abort if they are/are not EIP-55 compliant.
* It helps eliminate false assumptions that the function will abort if it’s not EIP-55 compliant, reducing the risk of errors elsewhere in the code
