# #49527 \[SC-Low] Edge case Integer UInt64SetLib.py::remove\_item leads to int underflow

**Submitted on Jul 16th 2025 at 21:18:01 UTC by @hunter0xweb3 for** [**Audit Comp | Folks Smart Contract Library**](https://immunefi.com/audit-competition/folks-sc-library)

* **Report ID:** #49527
* **Report Type:** Smart Contract
* **Report severity:** Low
* **Target:** <https://github.com/Folks-Finance/algorand-smart-contract-library/blob/main/contracts/library/UInt64SetLib.py>
* **Impacts:**
  * Contract fails to deliver promised returns, but doesn't lose value

## Description

## Brief/Intro

UInt64SetLib.py::remove\_item doesnt checks that items parameter is not empty, leading to int underflow when an empty items set is used as a parameter\
removing an element from an empty set should return an empty set , it should not revert, this breaks the promised return of calling remove\_item from a set returns a set

## Vulnerability Details

UInt64SetLib.py::remove\_item edge case leads to int underflow when calling remove\_item with an empty set.\
The vulnerability occurs because function declares the variable last\_idx to be equal to array length - 1:\
<https://github.com/Folks-Finance/algorand-smart-contract-library/blob/7673a43fa5183af736b65f17d1a297fdea672059/contracts/library/UInt64SetLib.py#L32-L41>

```python
def remove_item(to_remove: UInt64, items: DynamicArray[ARC4UInt64]) -> Tuple[Bool, DynamicArray[ARC4UInt64]]:
@>    last_idx = items.length - 1
```

However if empty array is passed this leads to a frame dig -1 and call to be reverted, instead of returning an empty array\
This is significative because call parameters to this functions are dynamic and no data integrity assumptions should be made, so defense in depth should be enforced

## Impact Details

* Function remove\_item doesnt applies security in depth mechanism
* remove\_item is integrity inconsistent, removing an element from an empty set should return an empty set , it should not revert, this breaks the promised return of calling remove\_item from a set returns a set

## References

<https://github.com/Folks-Finance/algorand-smart-contract-library/blob/7673a43fa5183af736b65f17d1a297fdea672059/contracts/library/UInt64SetLib.py#L32-L41>

## Proof of Concept

## Proof of Concept

Create the following test case in\
tests/library/UInt64SetLib.test.ts inside remove item section:

```ts
    test("Removing from empty arrays reverts instead of returning empty array ", async () => {
      const items = [];
      expect(await client.removeItem({ args: [0n, items] })).toEqual([false, items]);
    });
```

Observe function remove\_item reverts but it should return an empty array

```bash
  ● UInt64SetLib › remove item › Removing from empty arrays reverts instead of returning empty array 

        frame_dig -1

          intc_0 // 0
          extract_uint16
          dup
          intc_1 // 1
          - <--- Error
          intc_0 // 0
      remove_item_for_header@1:
          // contracts/library/UInt64SetLib.py:34
```
