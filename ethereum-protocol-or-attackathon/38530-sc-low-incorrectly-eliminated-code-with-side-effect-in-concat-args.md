# #38530 \[SC-Low] Incorrectly Eliminated Code With Side Effect In Concat Args

**Submitted on Jan 5th 2025 at 22:03:33 UTC by @anatomist for** [**Attackathon | Ethereum Protocol**](https://immunefi.com/audit-competition/ethereum-protocol-attackathon)

* **Report ID:** #38530
* **Report Type:** Smart Contract
* **Report severity:** Low
* **Target:** <https://github.com/vyperlang/vyper>
* **Impacts:**
  * (Compiler) Incorrect bytecode generation leading to incorrect behavior
  * (Compiler) Unexpected behavior

## Description

## Brief/Intro

If `length == 0` or `length_bound == 0`, the `copy_bytes` function will discard all operations regardless of any potential side effect, leading to incorrect compilation. This may lead to `concat` args being eliminated.

## Vulnerability Details

The Concat builtin utilizes `copy_bytes` to copy arguments to the destination buffer.

```
class Concat(BuiltinFunctionT):
    ...
    def build_IR(self, expr, context):
        ...
        for arg in args:
            dst_data = add_ofst(bytes_data_ptr(dst), ofst)

            if isinstance(arg.typ, _BytestringT):
                # Ignore empty strings
                if arg.typ.maxlen == 0:
                    continue

                with arg.cache_when_complex("arg") as (b1, arg):
                    argdata = bytes_data_ptr(arg)

                    with get_bytearray_length(arg).cache_when_complex("len") as (b2, arglen):
                        do_copy = [
                            "seq",
                            copy_bytes(dst_data, argdata, arglen, arg.typ.maxlen),              #utilize copy_bytes
                            ["set", ofst, ["add", ofst, arglen]],
                        ]
                        ret.append(b1.resolve(b2.resolve(do_copy)))

            ...
```

Notably, `copy_bytes` may short circuit and eliminate the entire argument when it decides that `length_bound` or `length` is a constant 0.

```
def copy_bytes(dst, src, length, length_bound):
    annotation = f"copy up to {length_bound} bytes from {src} to {dst}"

    ...

    with src.cache_when_complex("src") as (b1, src), length.cache_when_complex(
        "copy_bytes_count"
    ) as (b2, length), dst.cache_when_complex("dst") as (b3, dst):
        assert isinstance(length_bound, int) and length_bound >= 0

        # correctness: do not clobber dst
        if length_bound == 0:
            return IRnode.from_list(["seq"], annotation=annotation)                             #short circuit 1
        # performance: if we know that length is 0, do not copy anything
        if length.value == 0:
            return IRnode.from_list(["seq"], annotation=annotation)                             #short circuit 2
```

In most cases, length for `_BytestringT` should not be 0, since type annotations must have size greater than 0, however, this is not guaranteed for "literal-like arguments". For example, the expression `b"" if True else b""` will not have a fixed size and only have a `min_size = 0`. This is usually mitigated downstream when assigning the literal to a variable, since further usage of the variable will use the type annotation of the variable instead of the literal. Unfortunately, if we pass the "literal-like argument" directly to a function, this assumption breaks.

For example, the following code passes a 0 length `_BytestringT` to `copy_bytes`, and consequently, the `ifexp` is eliminated.

```
x: bool

def test():
    a: Bytes[256] = concat(b"" if self.sideeffect() else b"", b"aaaa")

def sideeffect() -> bool:
    self.x += 1
    return True
```

Similar to #37985, this is once again a premature optimization resulting in incorrect code emission. This category of bugs are probably not particularly impactful since they all require developers to write "strange code", but it has started to become a recurring scheme, so it's probably worth some further scrutiny as well as developing a way to systematically scan and fix similar issues.

We already did a quick scan over the codebase and are reasonably sure `concat` is the only use site of `copy_bytes` susceptible to the bug, but variants of #37985 and other short circuiting code still require more analysis.

## Impact Details

Incorrectly optimizing out side effects can result in all kinds of unexpected behaviors in a contract, such as skipping critical checks performed in the incorrectly eliminated code, skipping important business logic, and more. This exposes language users to substantial risks.

## References

* `https://github.com/vyperlang/vyper/blob/a29b49d422f6979be2b9c6c80aa583a60b1ccb7f/vyper/builtins/functions.py#L574`
* `https://github.com/vyperlang/vyper/blob/a29b49d422f6979be2b9c6c80aa583a60b1ccb7f/vyper/codegen/expr.py#L152`
* `https://github.com/vyperlang/vyper/blob/9c98b3ed4a4fbb1a614e63f815617fc275a0d16a/vyper/codegen/core.py#L307`

## Proof of Concept

## Proof of Concept

Already shown in Vulnerability Details section.
