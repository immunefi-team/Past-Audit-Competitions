# #48885 \[SC-Low] No items length check in remove\_item leads to a revert with an underflow

**Submitted on Jul 8th 2025 at 18:32:15 UTC by @j3x for** [**Audit Comp | Folks Smart Contract Library**](https://immunefi.com/audit-competition/folks-sc-library)

* **Report ID:** #48885
* **Report Type:** Smart Contract
* **Report severity:** Low
* **Target:** <https://github.com/Folks-Finance/algorand-smart-contract-library/blob/main/contracts/library/UInt64SetLib.py>
* **Impacts:**

## Description

## Brief/Intro

No items length check in remove\_item leads to a revert with an underflow.

## Vulnerability Details

In `UInt64SetLib::remove_item()`, the items array length is not verified to be a non-zero value, and the function will directly proceed to subtract from it:

```python
def remove_item(to_remove: UInt64, items: DynamicArray[ARC4UInt64]) -> Tuple[Bool, DynamicArray[ARC4UInt64]]:
    last_idx = items.length - 1
...
```

If the items array is empty, this will lead to a subtration from a 0 uint64, which causes a panic.

## Impact Details

Contradicts with the system's best design, as it proceeds to directly subtracts from a length of 0, causing a revert with an underflow.

## References

<https://github.com/Folks-Finance/algorand-smart-contract-library/blob/main/contracts/library/UInt64SetLib.py#L32C1-L33C32>

## Proof of Concept

## Proof of Concept

* The function `remove_item()` is called with an empty `items` array.
* This will lead to an underflow, causing a panic, which reverts the transaction

## Mitigation

add the following lines before subtracting:

```python
    if items.length == 0:
        return Bool(False), items.copy()
```

This will return peacefully in case on an empty set.
