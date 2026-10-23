# #49687 \[SC-Low] An underflow in \`remove\_item\` function in \`Uint64SetLib\` Contract.

**Submitted on Jul 18th 2025 at 11:41:38 UTC by @Immanux2160 for** [**Audit Comp | Folks Smart Contract Library**](https://immunefi.com/audit-competition/folks-sc-library)

* **Report ID:** #49687
* **Report Type:** Smart Contract
* **Report severity:** Low
* **Target:** <https://github.com/Folks-Finance/algorand-smart-contract-library/blob/main/contracts/library/UInt64SetLib.py>
* **Impacts:**
  * Temporary denial of service (smart contract is made unable to operate for one block, functionality is restored in the next block)

## Description

## Brief/Intro

The `remove_item` function in Uint64SetLib first line compute `last_idx = items.length - 1`, this line does not check if the array it is computing is empty(0) and this can lead to an underflow in the system which does revert.

## Vulnerability Details

The `remove_item` function in Uint64SetLib.py compute `last_idx = items.length - 1` the first line does not check if the array is empty, which can lead to an underflow. While the implementation is okay, there should be a check for empty(0) array, so as to avoid underflow, and it should also return a revert error, so the user knows that the array is empty instead of being clueless about what is wrong.

## Impact Details

Unexpected revert, without knowing what is wrong, the user or developer may be confused whenever they use the `remove_item` function.

Lack of an error feedback when the user or developer uses the remove\_item when the array is empty.

## References

We can check how the standard Openzeppelin EnumerableSet Library works:

<https://github.com/OpenZeppelin/openzeppelin-contracts/blob/6cfb6b5051a77a700593f7c7790bbad877139e38/contracts/utils/structs/EnumerableSet.sol>

## Proof of Concept

## Proof of Concept

This is their current implementation of the remove functionality:

```python
def remove_item(to_remove: UInt64, items: DynamicArray[ARC4UInt64]) -> Tuple[Bool, DynamicArray[ARC4UInt64]]:
    `last_idx = items.length - 1`
    for idx, item in uenumerate(items):
        if item.native == to_remove:
            # remove last item to replace the "to_remove" item or remove entirely if it's the match
            last_item = items.pop()
            if idx != last_idx:
                items[idx] = last_item
            # return with the item removed
            return Bool(True), items.copy()

    # if here then item is not present
    return Bool(False), items.copy()
```

I will recommend we should check that the length of the items is not zero and we handle such case gracefully increasing the robustness of the library.

```python
if len(items) == 0: 
     return Bool(False), items.copy()
else:
    for idx, item in uenumerate(items):
        if item.native == to_remove:
            # remove last item to replace the "to_remove" item or remove entirely if it's the match
            last_item = items.pop()
            if idx != last_idx:
                items[idx] = last_item
            # return with the item removed
            return Bool(True), items.copy()
```
