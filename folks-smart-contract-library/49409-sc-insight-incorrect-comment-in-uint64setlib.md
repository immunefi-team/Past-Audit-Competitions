# #49409 \[SC-Insight] Incorrect comment in UInt64SetLib

**Submitted on Jul 15th 2025 at 15:47:35 UTC by @j3x for** [**Audit Comp | Folks Smart Contract Library**](https://immunefi.com/audit-competition/folks-sc-library)

* **Report ID:** #49409
* **Report Type:** Smart Contract
* **Report severity:** Insight
* **Target:** <https://github.com/Folks-Finance/algorand-smart-contract-library/blob/main/contracts/library/UInt64SetLib.py>
* **Impacts:**

## Description

## Brief/Intro

Incorrect comment in `UInt64SetLib` library.

## Vulnerability Details

It is mentioned in [this comment](https://github.com/Folks-Finance/algorand-smart-contract-library/blob/main/contracts/library/UInt64SetLib.py#L10) that:

> item on stack cannot exceed 4098 bytes.

But this is incorrect, the correct value is 4096 bytes, according to the algorad documentation:

> If the stack depth is exceeded or if a byte-array element exceeds 4096 bytes, the program fails.

Source: <https://developer.algorand.org/docs/get-details/dapps/avm/teal/specification/#the-stack>

## Impact Details

Incorrect information to integrators, which may erode their trust.

## References

<https://github.com/Folks-Finance/algorand-smart-contract-library/blob/main/contracts/library/UInt64SetLib.py#L10>

<https://developer.algorand.org/docs/get-details/dapps/avm/teal/specification/#the-stack>

## Proof of Concept

## PoC

Change the 4098 to 4096, this will enhance the documentation and give confidence to readers.
