# #38855 \[SC-Low] Evaluation order is not respected in \`log\` function

**Submitted on Jan 16th 2025 at 01:26:18 UTC by @anatomist for** [**Attackathon | Ethereum Protocol**](https://immunefi.com/audit-competition/ethereum-protocol-attackathon)

* **Report ID:** #38855
* **Report Type:** Smart Contract
* **Report severity:** Low
* **Target:** <https://github.com/vyperlang/vyper>
* **Impacts:**
  * (Compiler) Incorrect bytecode generation leading to incorrect behavior
  * (Compiler) Unexpected behavior

## Description

## Brief/Intro

The left-to-right argument evaluation order is not respected when Event definition contains indexed fields.

## Vulnerability Details

There are two evaluation order issues in the current implementation of log function:

* indexed field always evaluate first
* indexed field evaluate in reverse order

During log expression parsing, IRnodes of indexed fields in the Event definition will be collected separately from normal data field. It will then be passed into `ir_node_for_log()` function, the function will also encode both of them separately and make them as the arguments of the output IRnode.

```python
def parse_Log(self):
	...

	args = [Expr(arg, self.context).ir_node for arg in to_compile]

	topic_ir = []
	data_ir = []
	for arg, is_indexed in zip(args, event.indexed):
		if is_indexed:
			topic_ir.append(arg) # saved into different list
		else:
			data_ir.append(arg)

	return events.ir_node_for_log(self.stmt, event, topic_ir, data_ir, self.context)

def ir_node_for_log(expr, event, topic_nodes, data_nodes, context):
    topics = _encode_log_topics(expr, event.event_id, topic_nodes, context)

    data = ir_tuple_from_args(data_nodes)

    bufsz = data.typ.abi_type.size_bound()
    buf = context.new_internal_variable(get_type_for_exact_size(bufsz))

    # encode_data is an IRnode which, cleverly, both encodes the data
    # and returns the length of the encoded data as a stack item.
    encode_data = abi_encode(buf, data, context, returns_len=True, bufsz=bufsz)

    assert len(topics) <= 4, "too many topics"  # sanity check
    log_opcode = "log" + str(len(topics))

    return IRnode.from_list(
        [log_opcode, buf, encode_data] + topics,
        add_gas_estimate=_gas_bound(len(topics), bufsz),
        annotation=f"LOG event {event.signature}",
    )
```

`ir_node_for_log()` will output an `LOGN` IRnode with arguments looks like below:

```
[buf, ABI_encoded_data, event, topic1, ..., topicN]
```

Since `topic` IRnodes are loaded and put into the argument array separately, during the final compilation from IRnodes to EVM assembly, the arguments of `LOGN` opcode will be translated into assembly in a reversed order, making the indexed fields always being evaluated first but in a reversed order.

```python
def _compile_to_assembly(code, withargs=None, existing_labels=None, break_dest=None, height=0):
	...

	if isinstance(code.value, str) and code.value.upper() in get_opcodes():
		o = []
		for i, c in enumerate(code.args[::-1]):
			o.extend(_compile_to_assembly(c, withargs, existing_labels, break_dest, height + i))
		o.append(code.value.upper())
		return o
	...
```

This issue is an interesting intersection of multiple root cause, from the separate parsing of different fields within an event to assembly emission order of irnode args. So naturally part of it would overlap some of the known issues such as [reversed order of side effects](https://github.com/vyperlang/vyper/security/advisories/GHSA-g2xh-c426-v8mf).

Aside from the issue itself, we especially want to highlight vyper currently lacks a clear definition of evaluation order (left-to-right is the rule of thumb, but there are quite a few exceptions). For example, judging from the code, assign should evaluated rhs (value) before lhs (assigned var), and augassign should evaluate lhs (augmented value) before rhs (delta). When it comes to subscript, the rules are even more complex, where `parent` should be evaluated before `key`, but in the case where `parent` resolves to a storage variable, the evaluation will be done lazily (the code below logs ("sideeffect1", "sideeffect2", 1) since the `self.hm1` returned from the `ifexp` is merely a reference to storage address, and only dereferenced after `key` is evaluated).

```python
event S:
    s: String

event I:
    i: uint256

hm1: HashMap[uint256, uint256]
hm2: HashMap[uint256, uint256]

@external
def test():
    self.hm1[0] = 0
    self.hm2[0] = 0
    i: uint256 = (self.hm1 if self.sideeffect1() else self.hm2)[self.sideeffect2()]
    log I(i)

def sideeffect1() -> bool:
    log S("sideeffect1")
    return True

def sideeffect2() -> uint256:
    self.hm1[0] += 1
    log S("sideeffect2")
    return 0

```

Without a proper definition of evaluation order, developers will often make their own assumptions (which may or may not be correct) while writing code, and it also makes it harder for auditors to decide whether a specific implementation should be considered buggy. Thus we would suggest vyper compiler devs to develop rules for evaluation orders to further improve the language.

## Impact Details

Unforeseen evaluation orders of arguments may lead to unexpected code behaviors.

## References

* `https://github.com/vyperlang/vyper/blob/c208b954564e8fffdd4c86cc3c497e0c3df1aeec/vyper/codegen/stmt.py#L106`
* `https://github.com/vyperlang/vyper/blob/c208b954564e8fffdd4c86cc3c497e0c3df1aeec/vyper/codegen/events.py#L63`
* `https://github.com/vyperlang/vyper/blob/c208b954564e8fffdd4c86cc3c497e0c3df1aeec/vyper/ir/compile_ir.py#L241`

## Proof of Concept

## Proof of Concept

In this case, the evaluation order is not left-to-right, instead, the order is `l -> j -> i -> k`

```
event I:
    i: uint256
    j: indexed(uint256)
    k: uint256
    l: indexed(uint256)

x: uint256

@deploy
def __init__():
    log I(self.sideeffect(), self.sideeffect(), self.sideeffect(), self.sideeffect())

def sideeffect() -> uint256:
    self.x += 1
    return self.x
```
