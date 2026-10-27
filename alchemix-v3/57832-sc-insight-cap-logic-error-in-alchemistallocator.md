# 57832 sc insight cap logic error in alchemistallocator

**Submitted on Oct 29th 2025 at 05:52:35 UTC by @rbd3 for** [**Audit Comp | Alchemix V3**](https://immunefi.com/audit-competition/alchemix-v3-audit-competition)

* **Report ID:** #57832
* **Report Type:** Smart Contract
* **Report severity:** Insight
* **Target:** <https://github.com/alchemix-finance/v3-poc/blob/immunefi\\_audit/src/AlchemistAllocator.sol>
* **Impacts:**

## Description

## Brief/Intro

Unit mismatch error in the `allocation` and `deallocation` caps calculation compares asset units with BPS units directly without conversion, causing mathematically meaningless comparisons and severe artificial restrictions on fund withdrawals.

## Vulnerability Details

```solidity
// Current buggy code in AlchemistAllocator.sol:
uint256 absoluteCap = vault.absoluteCap(id);  // Returns: Asset units (wei)
uint256 relativeCap = vault.relativeCap(id);  // Returns: BPS units (WAD)
uint256 adjusted = absoluteCap < relativeCap ? absoluteCap : relativeCap;  // INCORRECT
```

The deallocate function in AlchemistAllocator incorrectly compares values in different units without conversion.

absoluteCap(id) → Asset units (e.g., 2000000000000000000 = 2 ETH in wei)

relativeCap(id) → BPS units (e.g., 500000000000000000 = 0.5e18 = 50%)

## Impact Details

Comparing these directly is mathematically meaningless () and creates severe artificial restrictions, like asking "Is 2 kilometers less than 50 percent?"

## References

<https://github.com/alchemix-finance/v3-poc/blob/a192ab313c81ba3ab621d9ca1ee000110fbdd1e9/src/AlchemistAllocator.sol#L29-L44>

<https://github.com/alchemix-finance/v3-poc/blob/a192ab313c81ba3ab621d9ca1ee000110fbdd1e9/src/AlchemistAllocator.sol#L51-L66>

## Proof of Concept

## Proof of Concept

In the actual code, cap isn't enforced. But if it is, this create a critical or high issue in MYT allocation and deallocation.
