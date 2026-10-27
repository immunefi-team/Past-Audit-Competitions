# #37985 \[SC-Low] Incorrectly Eliminate Code With Side Effect In Slice Args

**Submitted on Dec 20th 2024 at 19:11:37 UTC by @anatomist for** [**Attackathon | Ethereum Protocol**](https://immunefi.com/audit-competition/ethereum-protocol-attackathon)

* **Report ID:** #37985
* **Report Type:** Smart Contract
* **Report severity:** Low
* **Target:** <https://github.com/vyperlang/vyper>
* **Impacts:**
  * (Compiler) Incorrect bytecode generation leading to incorrect behavior
  * (Compiler) Unexpected behavior

## Description

## Brief/Intro

The `slice` precompile skips `length != 0` check when the first argument is an `is_adhoc_slice`. However, if `length == 0`, the `make_setter` function will discard all operations except `mstore dst_size_field 0` regardless of any potential side effect, leading to incorrect compilation.

## Vulnerability Details

The slice buildin skips a lot of checks when the src to be slices is an `adhoc_slice` (`msg.data` or `(address).code`). Notably, the `length_literal < 1` check is skipped.

```
class Slice(BuiltinFunctionT):
    ...
    def fetch_call_return(self, node):
        ...
        arg = node.args[0]
        ...
        length_expr = node.args[2].reduced()

        # CMC 2022-03-22 NOTE slight code duplication with semantics/analysis/local
        is_adhoc_slice = arg.get("attr") == "code" or (
            arg.get("value.id") == "msg" and arg.get("attr") == "data"
        )

        start_literal = start_expr.value if isinstance(start_expr, vy_ast.Int) else None
        length_literal = length_expr.value if isinstance(length_expr, vy_ast.Int) else None

        if not is_adhoc_slice:
            if length_literal is not None:
                if length_literal < 1:
                    raise ArgumentException("Length cannot be less than 1", length_expr)

                if length_literal > arg_type.length:
                    raise ArgumentException(f"slice out of bounds for {arg_type}", length_expr)

            if start_literal is not None:
                if start_literal > arg_type.length:
                    raise ArgumentException(f"slice out of bounds for {arg_type}", start_expr)
                if length_literal is not None and start_literal + length_literal > arg_type.length:
                    raise ArgumentException(f"slice out of bounds for {arg_type}", node)
        ...
```

Thus the following code will compile

```
a: public(uint256)

@external
def test():
    b: Bytes[10]= slice(msg.data, self.test(), 0)

def side_effect() -> uint256:
    self.a += 1
    return 0 
```

During code generation, the IR for the slice builtin is created by the following snippet

```
def _build_adhoc_slice_node(sub: IRnode, start: IRnode, length: IRnode, context: Context) -> IRnode:
    assert length.is_literal, "typechecker failed"
    assert isinstance(length.value, int)  # mypy hint

    dst_typ = BytesT(length.value)
    # allocate a buffer for the return value
    buf = context.new_internal_variable(dst_typ)

    with scope_multi((start, length), ("start", "length")) as (b1, (start, length)):
        # `msg.data` by `calldatacopy`
        if sub.value == "~calldata":
            node = [
                "seq",
                check_buffer_overflow_ir(start, length, "calldatasize"),
                ["mstore", buf, length],
                ["calldatacopy", add_ofst(buf, 32), start, length],
                buf,
            ]
        elif sub.value == "~selfcode":
            ...
        else:
            ...
        ret = IRnode.from_list(node, typ=BytesT(length.value), location=MEMORY)
        return b1.resolve(ret)
```

And then returned to `parse_AnnAssign` as `rhs`

```
class Stmt:
    ...
    def parse_AnnAssign(self):
        ltyp = self.stmt.target._metadata["type"]
        varname = self.stmt.target.id
        lhs = self.context.new_variable(varname, ltyp)

        assert self.stmt.value is not None
        rhs = Expr(self.stmt.value, self.context).ir_node

        return make_setter(lhs, rhs)
    ...
```

Then `make_setter` is called to generate the final ir for this statement.

```
def make_setter(left, right, hi=None):
    check_assign(left, right)

    if potential_overlap(left, right):
        raise CompilerPanic("overlap between src and dst!")

    assert (hi is not None) == _dirty_read_risk(right)

    if left.typ._is_prim_word:
        ...

    elif isinstance(left.typ, _BytestringT):
        # TODO rethink/streamline the clamp_basetype logic
        if needs_clamp(right.typ, right.encoding):
            ...
        else:
            ret = make_byte_array_copier(left, right)

        return IRnode.from_list(ret)

    elif isinstance(left.typ, DArrayT):
        ...

    ...

def make_byte_array_copier(dst, src):
    assert isinstance(src.typ, _BytestringT)
    assert isinstance(dst.typ, _BytestringT)

    _check_assign_bytes(dst, src)

    if src.value == "~empty" or src.typ.maxlen == 0:
        # set length word to 0.
        return STORE(dst, 0)                #drops all rhs (src) ir if src.typ.maxlen == 0

    ...
```

Unfortunately, `rhs` (`slice` result) ir is completely dropped when its `src.typ.maxlen == 0`, along with any side effects that it may have. This in turn leads to incorrect compilation.

While we use `slice` as an entry point for this issue, we consider the actual root cause to be `make_setter` and `make_byte_array_copier` which eliminates ir without checking whether it contains side effect. There are potentially other entry points that we haven't explored yet, but can lead to the same issue.

Additionally, the `is_adhoc_slice` checker is also not precise, since it doesn't check whether the `code` attribute comes from an `address`, so if user defined a struct that contains a field calls `code`, then it'll also be treated as an `is_adhoc_slice` in `Slice.fetch_call_return`.

```
        is_adhoc_slice = arg.get("attr") == "code" or (
            arg.get("value.id") == "msg" and arg.get("attr") == "data"
        )
```

## Impact Details

Incorrectly optimizing out side effects can result in all kinds of unexpected behaviors in a contract, such as skipping critical checks performed in the incorrectly eliminated code, skipping important business logic, and more. This exposes contract developers to substantial risks.

## References

* `https://github.com/vyperlang/vyper/blob/12ab4919cc4618fcac4f5d24d45a0e7fdbc4a48c/vyper/builtins/functions.py#L320`
* `https://github.com/vyperlang/vyper/blob/12ab4919cc4618fcac4f5d24d45a0e7fdbc4a48c/vyper/builtins/functions.py#L247`
* `https://github.com/vyperlang/vyper/blob/12ab4919cc4618fcac4f5d24d45a0e7fdbc4a48c/vyper/codegen/stmt.py#L63`
* `https://github.com/vyperlang/vyper/blob/12ab4919cc4618fcac4f5d24d45a0e7fdbc4a48c/vyper/codegen/core.py#L983`
* `https://github.com/vyperlang/vyper/blob/12ab4919cc4618fcac4f5d24d45a0e7fdbc4a48c/vyper/codegen/core.py#L160`

## Proof of Concept

## Proof of Concept

Already shown in Vulnerability Details section.
