# #49390 \[SC-Low] \`UInt64SetLib#remove\_item\` would revert if the item is empty

**Submitted on Jul 15th 2025 at 12:24:21 UTC by @pks271 for** [**Audit Comp | Folks Smart Contract Library**](https://immunefi.com/audit-competition/folks-sc-library)

* **Report ID:** #49390
* **Report Type:** Smart Contract
* **Report severity:** Low
* **Target:** <https://github.com/Folks-Finance/algorand-smart-contract-library/blob/main/contracts/library/UInt64SetLib.py>
* **Impacts:**
  * Contract fails to deliver promised returns, but doesn't lose value

## Description

## Bug Description

When `items` is empty, the `last_idx = items.length - 1 = -1` will revert because the vm seems dont support negative value, but the `remove_item` function don't handle such case.

## Impact

The `remove_item` function would revert instead of return `false` if the item is empty.

## Recommendation

Check the `item` is empty or not before `last_idx`.

## Proof of Concept

## Proof of Concept

Insert the case into `tests/library/UInt64SetLib.test.ts`:

```javascript
test("removes from empty array", async () => {
    expect(await client.removeItem({ args: [873099n, []] })).toEqual([false, []]);
    expect(await client.removeItem({ args: [34n, []] })).toEqual([false, []]);
});
```

Result:

```
● UInt64SetLib › remove item › removes from empty array

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
