# #48983 \[SC-Low] Potential Underflow in remove\_item() on Empty Array

**Submitted on Jul 10th 2025 at 08:05:30 UTC by @Oxenzo\_eth for** [**Audit Comp | Folks Smart Contract Library**](https://immunefi.com/audit-competition/folks-sc-library)

* **Report ID:** #48983
* **Report Type:** Smart Contract
* **Report severity:** Low
* **Target:** <https://github.com/Folks-Finance/algorand-smart-contract-library/blob/main/contracts/library/UInt64SetLib.py>
* **Impacts:**
  * Contract fails to deliver promised returns, but doesn't lose value

## Description

## Brief/Intro

When calling `remove_item(to_remove, items)` on an empty `DynamicArray`, the code computes `last_idx = items.length - 1`, which underflows the unsigned integer. Although the function still returns `(False, items.copy())`, this underflow can introduce unwanted behavior and should be addressed to maintain clarity and correctness.

## Vulnerability Details

The `remove_item` function which is here:

```python
@subroutine
def remove_item(to_remove: UInt64, items: DynamicArray[ARC4UInt64]) -> Tuple[Bool, DynamicArray[ARC4UInt64]]:
    last_idx = items.length - 1
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

begins by calculating -> `last_idx = items.length - 1`

For an empty array, `items.length == 0`, so `last_idx` wraps to the maximum UInt64 value (e.g., 2\*\*64 - 1), due to unsigned underflow. Although the subsequent loop does not fire and the code returns safely, the presence of an `invalid last_idx` may:

Consume TEAL stack unnecessarily or increase code complexity.

```python
# Potential underflow scenario
items = DynamicArray([])
last_idx = items.length - 1  # underflows -> UInt64 max
for idx, item in uenumerate(items):  # loop skipped
    ...
return Bool(False), items.copy()

```

## Impact Details

Contract fails to deliver promised behavior

## References

The function can be located here: <https://github.com/Folks-Finance/algorand-smart-contract-library/blob/7673a43fa5183af736b65f17d1a297fdea672059/contracts/library/UInt64SetLib.py#L32>

## Proof of Concept

## Proof of Concept

Kindly place this in the `UInt64SetLib.test.ts`

```javascript
describe("remove_item underflow behavior", () => {
  test("calling removeItem on empty items array should return false and empty array", async () => {
    const { appClient: client } = await factory.deploy();

    await localnet.algorand.send.payment({
      sender: creator,
      receiver: getApplicationAddress(client.appId),
      amount: (1).algo(),
    });

    // Confirm empty array behavior
    const result = await client.removeItem({ args: [0n, []] });

    expect(result).toEqual([false, []]);
  });
});
```
