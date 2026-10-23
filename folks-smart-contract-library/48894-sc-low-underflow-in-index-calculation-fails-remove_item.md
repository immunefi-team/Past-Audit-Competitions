# #48894 \[SC-Low] Underflow in index calculation fails remove\_item

**Submitted on Jul 8th 2025 at 20:29:20 UTC by @uhudo for** [**Audit Comp | Folks Smart Contract Library**](https://immunefi.com/audit-competition/folks-sc-library)

* **Report ID:** #48894
* **Report Type:** Smart Contract
* **Report severity:** Low
* **Target:** <https://github.com/Folks-Finance/algorand-smart-contract-library/blob/main/contracts/library/UInt64SetLib.py>
* **Impacts:**
  * Permanent denial of service of a smart contract functionality
  * Temporary denial of service for more than one block
  * Temporary denial of service (smart contract is made unable to operate for one block, functionality is restored in the next block)

## Description

## Brief/Intro

Underflow in index calculation in `remove_item` causes the subroutine call to fail if `items` is an empty dynamic array. As this is a library, the impact can vary depending on use. In worst case, the contract could become blocked.

## Vulnerability Details

In subroutine `remove_item`, the `items` input can be an empty DynamicArray, which has a length of 0. Calculation of last\_idx (L33) causes an underflow in this case. As the code description for the library states that the subroutine "Removes an item from the set. Returns whether the item was removed and the new set.", it would be expected that the call succeeds but returns false and the same array.

## Impact Details

Impact of this error can vary greatly as this is a library. In the worst case, it could lead to contract being blocked due to unexpected call failure. However, if developers are careful and understand the library inner workings, they can protect against this by checking for empty array before calling the remove\_item. However, as the purpose of the library is to simplify development, it would be prudent to include this check in the library and let the developers handle the result based on whether the item was removed or not. Or at least a disclaimer should be added, similar to the maximum number of items supported by the library.

## References

Underflow happens at: <https://github.com/Folks-Finance/algorand-smart-contract-library/blob/7673a43fa5183af736b65f17d1a297fdea672059/contracts/library/UInt64SetLib.py#L33>

## Proof of Concept

## Proof of Concept

In contract UInt64SetLibExposed, the following method can be added to demonstrate the issue:

```
    @abimethod
    def empty_remove_item(self, to_remove: UInt64) -> Tuple[Bool, DynamicArray[ARC4UInt64]]:
        items = DynamicArray[ARC4UInt64]()
        return UInt64SetLib.remove_item(to_remove, items.copy())
```

Calling the method produces: `Error resolving execution info via simulate in transaction 0: transaction KUA2Y7DHE2LJEDFFZUAHRRZO2MHHJ3WURIGFEKORO7EPRPFMQ75Q: logic eval error: - would result negative. Details: app=28046, pc=584, opcodes=dup; intc_0 // 1; -`
