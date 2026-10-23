# #49690 \[SC-Low] Integer Underflow in UInt64SetLib.py

**Submitted on Jul 18th 2025 at 12:17:53 UTC by @Opzteam for** [**Audit Comp | Folks Smart Contract Library**](https://immunefi.com/audit-competition/folks-sc-library)

* **Report ID:** #49690
* **Report Type:** Smart Contract
* **Report severity:** Low
* **Target:** <https://github.com/Folks-Finance/algorand-smart-contract-library/blob/main/contracts/library/UInt64SetLib.py>
* **Impacts:**

## Description

A critical integer underflow vulnerability exists in the `remove_item` function of the UInt64SetLib.py smart contract library. The vulnerability allows the function to revert unexpectedly when operating on empty arrays, potentially causing denial of service conditions.

The remove\_item function performs an unchecked subtraction operation without validating that the array contains at least one element. When items.length equals 0, the calculation items.length - 1 results in an underflow, producing a negative value that causes the function to revert.

```
  @subroutine
  def remove_item(to_remove: UInt64, items: DynamicArray[ARC4UInt64]) -> Tuple[Bool,
  DynamicArray[ARC4UInt64]]:
      last_idx = items.length - 1  # ← VULNERABLE LINE
      for idx, item in uenumerate(items):
          if item.native == to_remove:
              last_item = items.pop()
              if idx != last_idx:
                  items[idx] = last_item
              return Bool(True), items.copy()
      return Bool(False), items.copy()
```

## Proof of Concept

## Proof of Concept

1. Attacker calls remove\_item with any to\_remove value on an empty array
2. The function calculates last\_idx = 0 - 1 = -1
3. The underflow causes the function to revert
4. Any dependent operations fail, potentially causing system-wide disruption

## Recommended Fix

Add a length validation check before performing the subtraction:
