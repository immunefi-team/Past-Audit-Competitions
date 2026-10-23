# #49075 \[SC-Low] \`SetLib.remove\_item()\` is not safe on empty Dynamic arrays

**Submitted on Jul 11th 2025 at 11:43:00 UTC by @danvinci\_20 for** [**Audit Comp | Folks Smart Contract Library**](https://immunefi.com/audit-competition/folks-sc-library)

* **Report ID:** #49075
* **Report Type:** Smart Contract
* **Report severity:** Low
* **Target:** <https://github.com/Folks-Finance/algorand-smart-contract-library/blob/main/contracts/library/UInt64SetLib.py>
* **Impacts:**
  * Permanent denial of service of a smart contract functionality
  * Contract fails to deliver promised returns, but doesn't lose value

## Description

## Brief/Intro

The Uint64SetLib library implements a `remove_item()` function that aims to provide efficient, linear-time unordered removal of an element from a dynamic array of `UInt64`. It does this by using a swap-and-pop mechanism, replacing the matched element with the last one and calling .pop() to reduce the array length.

However, the implementation assumes that the array always contains at least one item, and makes two critical unsafe operations:

```python
@>> last_idx = items.length - 1
@>> last_item = items.pop()
```

If `items.length == 0`, these operations will result in either underflow or an attempt to pop from an empty array both of which can raise runtime errors or exceptions, depending on the environment. This makes the library function unsafe in general-purpose or reusable contexts where empty arrays might be passed.

While the function is logically correct for non-empty arrays, a defensive library implementation like `Openzeppelin EnumerableSet` must safely handle all valid inputs, including edge cases such as an empty set. This is especially important in smart contracts and subroutines, where an unexpected revert or exception can cascade and disrupt critical processes.

## References

You can check how the Openzeppelin library handles it, it makes use of +1 offset to handle such cases making it more robust: <https://github.com/OpenZeppelin/openzeppelin-contracts/blob/master/contracts/utils/structs/EnumerableSet.sol>

## Recommendation

The function should explicitly check if the array is empty before attempting to remove an element. A simple and safe fix would be:

```python
@subroutine
def remove_item(to_remove: UInt64, items: DynamicArray[ARC4UInt64]) -> Tuple[Bool, DynamicArray[ARC4UInt64]]:
    if items.length == UInt64(0):
        return Bool(False), items.copy()

    last_idx = items.length - 1

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

Below is the original implementation of the function:

```
@subroutine
def remove_item(to_remove: UInt64, items: DynamicArray[ARC4UInt64]) -> Tuple[Bool, DynamicArray[ARC4UInt64]]:
    last_idx = items.length - 1

    for idx, item in uenumerate(items):
        if item.native == to_remove:
            last_item = items.pop()

            if idx != last_idx:
                items[idx] = last_item

            return Bool(True), items.copy()

    return Bool(False), items.copy()
```

If `items` is an empty array, `items.length - 1` underflows, and `items.pop()` causes a runtime error.
