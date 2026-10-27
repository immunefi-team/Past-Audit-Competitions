# #38682 \[SC-Medium] AugAssign evaluation order causing OOB write within the object

**Submitted on Jan 10th 2025 at 01:15:10 UTC by @anatomist for** [**Attackathon | Ethereum Protocol**](https://immunefi.com/audit-competition/ethereum-protocol-attackathon)

* **Report ID:** #38682
* **Report Type:** Smart Contract
* **Report severity:** Medium
* **Target:** <https://github.com/vyperlang/vyper>
* **Impacts:**
  * (Compiler) Incorrect bytecode generation leading to incorrect behavior
  * (Compiler) Unexpected behavior

## Description

## Brief/Intro

AugAssign (e.g. `+=`, `-=`, ...) statements does not consider the side effect caused by rhs, allowing out-of-bounds index write within the array.

## Vulnerability Details

Vyper parse `AugAssign` statements by first cache the target location to avoid double evaluation.\
However, in the case when target is an access to a `DynArray` and the right value modify the array, the cached `target` will evaluate first, following the modifications like `DynArray.pop()`, and eventually the `STORE` operation could store the computed output in an index that falls outside the array’s current length bounds.

```python
def parse_AugAssign(self):
	target = self._get_target(self.stmt.target)
	right = Expr.parse_value_expr(self.stmt.value, self.context)

	if not target.typ._is_prim_word:
		# because of this check, we do not need to check for
		# make_setter references lhs<->rhs as in parse_Assign -
		# single word load/stores are atomic.
		raise TypeCheckFailure("unreachable")

	with target.cache_when_complex("_loc") as (b, target):
		left = IRnode.from_list(LOAD(target), typ=target.typ)
		new_val = Expr.handle_binop(self.stmt.op, left, right, self.context)
		return b.resolve(STORE(target, new_val))
```

This is alike [this](https://github.com/vyperlang/vyper/security/advisories/GHSA-3p37-3636-q8wv) high severity issue where it discusses similar issues for `Assign`. However, it is not appropriate to apply the same mitigations in `parse_Assign` to `parse_AugAssign`, since the evaluation order of `Assign` is `rhs -> lhs`, while evaluation of `AugAssign` should be `dereference lhs -> rhs -> assign to lhs`, so assigning `rhs` to a temporary variable will actually change the evaluation order. We currently don't have a simple patch in mind, aside from reimplementing the entire `AugAssign`. Notably, care must be taken to not double evaluate `lhs` when fixing the bug.

## Impact Details

The direct impact is out-of-bounds index access within the array. But wrong IR code emission could also lead to various consequences.

## References

<https://github.com/vyperlang/vyper/blob/a29b49d422f6979be2b9c6c80aa583a60b1ccb7f/vyper/codegen/stmt.py#L291>

## Proof of Concept

## Proof of Concept

```
@deploy
def __init__():
    a: DynArray[uint256, 2] = [1, 2]
    a[1] += a.pop() # should revert here, but doesn't. rhs doesn't have to be pop, and can be other complex functions
```
