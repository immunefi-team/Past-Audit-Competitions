# #38581 \[SC-Insight] Incorrect unwrap on Bytes and String

**Submitted on Jan 7th 2025 at 09:19:54 UTC by @anatomist for** [**Attackathon | Ethereum Protocol**](https://immunefi.com/audit-competition/ethereum-protocol-attackathon)

* **Report ID:** #38581
* **Report Type:** Smart Contract
* **Report severity:** Insight
* **Target:** <https://github.com/vyperlang/vyper>
* **Impacts:**
  * (Compiler) Unexpected behavior
  * Codegen errors

## Description

## Brief/Intro

During codegen stage, Vyper compiler does a incorrect unwrap to constant Bytes and String variables, later causing the compiler to panic.

## Vulnerability Details

While parsing Name and Attribute expressions in codegen stage, if the referenced variable is mark as a constant, then `Expr.parse_value_expr` will be called to directly setup IRnode from the expression assigned to the constant variable.

```
def parse_Name(self):
	varname = self.expr.id

	if varname == "self":
		return IRnode.from_list(["address"], typ=AddressT())

	varinfo = self.expr._expr_info.var_info
	assert varinfo is not None

	...
	if varinfo.is_constant:
		return Expr.parse_value_expr(varinfo.decl_node.value, self.context)

	...
```

After the IRnode is initialized, it calls `unwrap_location` to unwrap the variable to its value (as opposed to a pointer).

```
# Parse an expression that results in a value
@classmethod
def parse_value_expr(cls, expr, context):
	return unwrap_location(cls(expr, context).ir_node)
```

The constant Bytes / String is initialized by `Expr._make_bytelike`, and their IR node location will be set to `MEMORY`. Therefore, if a constant Bytes / String is provided to `Expr.parse_Name`, an incorrect IRnode will be generated, as there is no valid reason to directly unwrap a Bytes / String into single value.

```
def _make_bytelike(self, btype, bytez, bytez_length):
	placeholder = self.context.new_internal_variable(btype)
	seq = []
	seq.append(["mstore", placeholder, bytez_length])
	for i in range(0, len(bytez), 32):
		seq.append(
			[
				"mstore",
				["add", placeholder, i + 32],
				bytes_to_int((bytez + b"\x00" * 31)[i : i + 32]),
			]
		)
	return IRnode.from_list(
		["seq"] + seq + [placeholder],
		typ=btype,
		location=MEMORY,
		annotation=f"Create {btype}: {bytez}",
	)
```

Fortunately, the incorrect unwraps will never make it's way into compiled bytecodes. `unwrap_location` will clean up the `.location` field of IRnode, and any access to the Bytes / String variable will either call `bytes_data_ptr` or `get_bytearray_length`. Both will panic if the variable passed to them does not have `.location` field.

## Impact Details

Compiler panics when it shouldn't.

## References

<https://github.com/vyperlang/vyper/blob/a29b49d422f6979be2b9c6c80aa583a60b1ccb7f/vyper/codegen/expr.py#L197\\>
<https://github.com/vyperlang/vyper/blob/a29b49d422f6979be2b9c6c80aa583a60b1ccb7f/vyper/codegen/expr.py#L820>

## Proof of Concept

## Proof of Concept

```
struct X:
    a: uint256
    b: Bytes[64]

x: constant(X) = X(a = 1, b = b'1234')
b: constant(Bytes[64]) = x.b

@deploy
def __init__():
    a: Bytes[64] = b
```
