# #49559 \[SC-Low] The remove functionality in \`UInt64SetLib::remove\_item\` underflows on empty array

**Submitted on Jul 17th 2025 at 08:37:56 UTC by @NHristov for** [**Audit Comp | Folks Smart Contract Library**](https://immunefi.com/audit-competition/folks-sc-library)

* **Report ID:** #49559
* **Report Type:** Smart Contract
* **Report severity:** Low
* **Target:** <https://github.com/Folks-Finance/algorand-smart-contract-library/blob/main/contracts/library/UInt64SetLib.py>
* **Impacts:**
  * Temporary denial of service for more than one block

## Description

## Brief/Intro

The `UInt64SetLib::remove_item` computes the last index as

```python
last_idx = items.length - 1
```

without first checking if the passed array is empty. If you call `remove_item` with an empty array, `items.length - 1` underflows to a negative value and immediately aborts the TEAL execution, reverting the transaction.

## Vulnerability Details

In `contracts/library/UInt64SetLib.py` we have:

```python
# filepath: contracts/library/UInt64SetLib.py
@subroutine
def remove_item(
    to_remove: UInt64,
    items: DynamicArray[ARC4UInt64]
) -> Tuple[Bool, DynamicArray[ARC4UInt64]]:
    # ← no guard for empty `items`
    last_idx = items.length - 1
    for idx, item in uenumerate(items):
        if item.native == to_remove.native:
            last_item = items.pop()
            if idx != last_idx:
                items[idx] = last_item
            return Bool(True), items.copy()
    return Bool(False), items.copy()
```

and if we pass an empty array, `items.length` is `0`, so `last_idx` becomes `-1`. This leads to an immediate TEAL abort with the error message copied from the PoC test below as follows:

```
frame_dig -1

    intc_0 // 0
    extract_uint16
    dupn 2
    intc_1 // 1
    - <--- Error
    swap
    // contracts/library/UInt64SetLib.py:34
    // if items.length == 0:
    bnz remove_item_after_if_else@2
```

That “frame dig ‑1” error is coming from this line:

```python
last_idx = items.length - 1
```

When `items.length` is 0, you end up emitting TEAL that does:

```teal
// push items.length (0)
extract_uint16
dup
intc 1
-
swap
…
```

Which effectively computes `0 - 1` in unsigned land, underflows, and blows up with “frame dig ‑1”.

## Impact Details

Any consumer of `remove_item` that passes an empty array will experience an immediate revert. This can lead to:

* Unexpected Denial-of-Service in higher-level logic that relies on safe removal.
* Forcing callers to add additional pre-checks, undermining the library’s usability guarantees.

## Remediation

Add an explicit guard against an empty array at the top of `remove_item`, for example:

```python
if items.length == 0:
    return Bool(False), items.copy()
last_idx = items.length - 1
```

## References

* [Vulnerable code: `contracts/library/UInt64SetLib.py: remove_item`](https://github.com/Folks-Finance/algorand-smart-contract-library/blob/main/contracts/library/UInt64SetLib.py)

## Proof of Concept

## Proof of Concept

In `tests/library/UInt64SetLib.test.ts` append the following test under the `describe("remove item")` block:

```typescript
// filepath: tests/library/UInt64SetLib.test.ts
describe("remove item", () => {
  // …

  test("fails when removing from empty array", async () => {
    // calling removeItem on [] should revert
    await expect(
      client.removeItem({ args: [34n, []] })
    ).toThrow();
  });
});
```
