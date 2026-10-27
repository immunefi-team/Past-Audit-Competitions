# #49938 \[SC-Low] Underflow Revert in \`remove\_item\` When Removing from an Empty Array

**Submitted on Jul 20th 2025 at 16:30:46 UTC by @Afriauditor for** [**Audit Comp | Folks Smart Contract Library**](https://immunefi.com/audit-competition/folks-sc-library)

* **Report ID:** #49938
* **Report Type:** Smart Contract
* **Report severity:** Low
* **Target:** <https://github.com/Folks-Finance/algorand-smart-contract-library/blob/main/contracts/library/UInt64SetLib.py>
* **Impacts:**
  * Contract fails to deliver promised returns, but doesn't lose value

## Description

## Brief/Intro

The `remove_item` subroutine in `UInt64SetLib` unconditionally computes `last_idx = items.length - 1,` which underflows when the array is empty and causes the transaction to revert instead of returning `(false, items)`, leading to unexpected failures or DoS in production.

## Vulnerability Details

In the first line of `remove_item` the code does:

```
last_idx = items.length - 1

```

Because there is no check for an empty array before this subtraction, any call to `remove_item(x, [])` will immediately abort the TEAL execution at that line, bypassing the intended “not present” return path. So the function violates its specification and any upstream contract logic reverts

## Impact Details

Because `remove_item aborts` instead of returning the promised `(false, items)` when called on an empty array, any contract logic that depends on that return value will itself revert or misbehave breaking workflows

## References

## Proof of Concept

## Proof of Concept

1. **Deploy the `UInt64SetLibExposed` application** so you have an ABI method `dynamicRemoveItem` that calls `remove_item` on a box-backed array.
2. **Reset the box** to an empty array by invoking its `dynamicReset` method (or equivalent), ensuring `items.length == 0`.
3. **Attempt to remove** any value (e.g. `123`) by calling `dynamicRemoveItem({ args: [123], boxReferences: ["uint64_set"] })`.
4. **Watch the transaction revert** with an underflow or “invalid subtract” error at the very first line of `remove_item`.
5. **Confirm that no `(false, items)`** return is ever emitted and that the fallback return path is never reached.
