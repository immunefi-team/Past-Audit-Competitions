# #38169 \[SC-Insight] Deferred Evaluation Of \`Default\_Return\_Value\` May Skip Side Effect Execution

**Submitted on Dec 26th 2024 at 21:17:32 UTC by @anatomist for** [**Attackathon | Ethereum Protocol**](https://immunefi.com/audit-competition/ethereum-protocol-attackathon)

* **Report ID:** #38169
* **Report Type:** Smart Contract
* **Report severity:** Insight
* **Target:** <https://github.com/vyperlang/vyper>
* **Impacts:**
  * (Compiler) Incorrect bytecode generation leading to incorrect behavior
  * (Compiler) Unexpected behavior

## Description

## Brief/Intro

Calls to external contracts has four additional kwargs that allow user to tune the behavior of the call. Within those kwargs, `default_return_value` specifies what to use as return value when the callee contract does not return anything. However, deferred evaluation of the expression for `default_return_value` may result in it being executed at the wrong time, or worse, skipped altogether along with any side effects it has.

## Vulnerability Details

The `_unpack_returndata` is responsible for decoding data returned from an external call. It also injects the `default_return_value` as return data if the `returndatasize == 0`

```
def _unpack_returndata(buf, fn_type, call_kwargs, contract_address, context, expr):
    ...
    if call_kwargs.default_return_value is not None:
        ...

        override_value = wrap_value_for_external_return(call_kwargs.default_return_value)
        stomp_return_buffer = ["seq"]
        if not call_kwargs.skip_contract_check:
            stomp_return_buffer.append(_extcodesize_check(contract_address))
        stomp_return_buffer.append(make_setter(return_buf, override_value))
        unpacker = ["if", ["eq", "returndatasize", 0], stomp_return_buffer, unpacker]       #use default_return_value if returndatasize == 0

    unpacker = ["seq", unpacker, return_buf]

    return unpacker, ret_ofst, ret_len
```

However, since `default_return_value` is not cached with `cache_when_complex` prior to the external call, the evaluation is deferred until expression value is actually used. This creates an interesting scenario, if the expr for `default_return_value` contains side effects, it might be skipped. For example, in the PoC below, `get_default_id` only gets called if the external call doesn't return any data, and the side effect of increasing `external_call_counter` might be skipped.

```
interface FooBar:
	def test() -> uint256: payable

external_call_counter: uint256

@deploy
def __init__(ext: FooBar):
	res: uint256 = extcall ext.test(default_return_value = self.get_default_id())   #get_default_id might not run at all

def get_default_id() -> uint256:
    counter: uint256 = self.external_call_counter
	self.external_call_counter += 1
	return counter
```

This implementation is confusing from language users' point of view, since it is reasonable to expect all arguments of a function to be evaluated prior to the function call itself. We believe the "correct" language design would be to cache the `expr` of `default_return_value`, so it gets executed before the external call, regardless of the returned result.

Additionally, we also noticed another small "mistake" in the default value assignment code, where the following "legal" code fails to compile.

```
interface FooBar:
    def test() -> (uint256, uint256): payable

@deploy
def __init__(ext: FooBar):
    x: uint256 = 2
    a: (uint256, uint256) = (x, x)
    b: (uint256, uint256) = extcall ext.test(default_return_value = a)  #fails to compile
```

This is due to `make_setter` not able to assign values from VYPER encoding pointers to ABI encoding pointers

```
def _unpack_returndata(buf, fn_type, call_kwargs, contract_address, context, expr):
    ...
    encoding = Encoding.ABI
    ...
    buf.encoding = encoding
    ...
    if not needs_clamp(wrapped_return_t, encoding):
        ...
        return_buf = buf
    else:
        ...

    if call_kwargs.default_return_value is not None:
        ...
        override_value = wrap_value_for_external_return(call_kwargs.default_return_value)
        ...
        stomp_return_buffer.append(make_setter(return_buf, override_value))         #assign from VYPER encoding pointers to ABI encoding pointers
```

```
def make_setter(left, right, hi=None):
    ...
    with right.cache_when_complex("c_right") as (b1, right):
        ret = ["seq"]
        if hi is not None:
            item_end = add_ofst(right, right.typ.abi_type.static_size())
            len_check = ["assert", ["le", item_end, hi]]
            ret.append(len_check)

        ret.append(_complex_make_setter(left, right, hi=hi))
        return b1.resolve(IRnode.from_list(ret))


def _complex_make_setter(left, right, hi=None):
    ...
    if left.is_pointer and right.is_pointer and right.encoding == Encoding.VYPER:
        # both left and right are pointers, see if we want to batch copy
        # instead of unrolling the loop.
        assert left.encoding == Encoding.VYPER
        ...
```

## Impact Details

Unforeseen deferred evaluation or unevaluated code can result in all kinds of unexpected behaviors in a contract, such as skipping critical checks performed in the code, skipping important business logic, and more. This exposes language users to substantial risks.

## References

* `https://github.com/vyperlang/vyper/blob/9c98b3ed4a4fbb1a614e63f815617fc275a0d16a/vyper/codegen/external_call.py#L157`
* `https://github.com/vyperlang/vyper/blob/9c98b3ed4a4fbb1a614e63f815617fc275a0d16a/vyper/codegen/external_call.py#L156`
* `https://github.com/vyperlang/vyper/blob/9c98b3ed4a4fbb1a614e63f815617fc275a0d16a/vyper/codegen/core.py#L1013`
* `https://github.com/vyperlang/vyper/blob/9c98b3ed4a4fbb1a614e63f815617fc275a0d16a/vyper/codegen/core.py#L1044`

## Proof of Concept

## Proof of Concept

Already shown in Vulnerability Details section.
