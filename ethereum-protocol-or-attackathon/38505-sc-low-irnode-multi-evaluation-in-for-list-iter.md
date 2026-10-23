# #38505 \[SC-Low] IRNode Multi-Evaluation In For List Iter

**Submitted on Jan 5th 2025 at 06:57:29 UTC by @anatomist for** [**Attackathon | Ethereum Protocol**](https://immunefi.com/audit-competition/ethereum-protocol-attackathon)

* **Report ID:** #38505
* **Report Type:** Smart Contract
* **Report severity:** Low
* **Target:** <https://github.com/vyperlang/vyper>
* **Impacts:**
  * (Compiler) Incorrect bytecode generation leading to incorrect behavior

## Description

## Brief/Intro

Following report #38504, we show that multi-evaluation of a single expression if possible for for list iter, and also discuss why other context check and defense in depth mechanisms such as `unique_symbol` failed to serve as the last line of defense.

## Vulnerability Details

Following report #38504, we want to demonstrate another case where multi-evaluation happens, and the `unique_symbol` defense in depth is not able to catch it at compile time. This will touch a lot of components, so bear with the lengthy explaination.

Vyper for loops allow two kinds of `iters`, namely `range` and an iterable type like `SArray` and `DArray`. We only care about iterable types for this report.

```
 def _analyse_list_iter(self, iter_node, target_type):
    # iteration over a variable or literal list
    iter_val = iter_node.reduced()

    if isinstance(iter_val, vy_ast.List):
        len_ = len(iter_val.elements)
        if len_ == 0:
            raise StructureException("For loop must have at least 1 iteration", iter_node)
        iter_type = SArrayT(target_type, len_)
    else:
        try:
            iter_type = get_exact_type_from_node(iter_node)
        except (InvalidType, StructureException):
            raise InvalidType("Not an iterable type", iter_node)

    if not isinstance(iter_type, (DArrayT, SArrayT)):
        raise InvalidType("Not an iterable type", iter_node)
    ...
```

Upon being passed to codegen, `iter_list` is required to not produce any side-effects through calls (the `range_scope` forces `iter_list` to be parsed in a constant context, and calls check against `is_constant`).

```
def _parse_For_list(self):
    with self.context.range_scope():
        iter_list = Expr(self.stmt.iter, self.context).ir_node
    ...
```

```
def range_scope(self):
    prev_value = self.in_range_expr
    self.in_range_expr = True
    yield
    self.in_range_expr = prev_value

def is_constant(self):
    return self.constancy is Constancy.Constant or self.in_range_expr
```

However, this does not prevent the `iter` from consuming side effects provided by other code. So multi-evaluation of `iter` may still be troublesome.

Let's start with establishing the fact that multi-evaluation is possible for `iter`. `_parse_For_list` parses out the `iter_list` from ast, and can be a complex `ir_node` including consumption of side effects. The `iter_list` `irnode` is not cached before usage, so if more than one usage exist, multi-evaluation will emerge.

```
def _parse_For_list(self):
    with self.context.range_scope():
        iter_list = Expr(self.stmt.iter, self.context).ir_node

    ...
    # list literal, force it to memory first
    if isinstance(self.stmt.iter, vy_ast.List):
        tmp_list = self.context.new_internal_variable(iter_list.typ)
        ret.append(make_setter(tmp_list, iter_list))                    #this branch works as a cache for `iter_list`, so multi-eval
        iter_list = tmp_list

    # set up the loop variable
    e = get_element_ptr(iter_list, i, array_bounds_check=False)                             #usage one
    body = ["seq", make_setter(loop_var, e), parse_body(self.stmt.body, self.context)]      #usage one is placed into body

    if isinstance(iter_list.typ, DArrayT):
        array_len = get_dyn_array_count(iter_list)                                          #usage two is placed into rounds (array_len)
    else:
        array_len = repeat_bound

    ret.append(["repeat", i, 0, array_len, repeat_bound, body])

    ...
```

It is immediately apparent double evaluation is possible for the `DArrayT`, and true enough, the following PoC confirms this and panics on `unique_symbol` check.

```
x: DynArray[uint256, 3]

@external
def test():
    for i: uint256 in (self.usesideeffect() if True else self.usesideeffect()):
        pass

def usesideeffect() -> DynArray[uint256, 3]:
    return self.x
```

So defense in depth caught a bug, crisis averted, everything good? Not so fast, let's look at the `SArrayT` case. In `SArrayT` case, there is only one instantiation of `iter_list`, but it happens in the `body` of a `repeat` ir, so it in fact, can be evaluated several times. The `unique_symbol` check is unable to catch this since it only recursively traverses the ir tree without considering whether an irnode or argument is actually executed more than once at runtime.

```
elif code.value == "unique_symbol":
    symbol = code.args[0].value
    assert isinstance(symbol, str)

    if symbol in existing_labels:
        raise Exception(f"symbol {symbol} already exists!")
    else:
        existing_labels.add(symbol)

    return []
```

How may this affect execution? Let's analyze it with 3 examples.

In the first example, the following test case pre-evaluates the iter `list` and stores the result to a `tmp_list`. So no multi-evaluation happens, and the log output will be `0, 0, 0`.

```
event I:
	i: uint256

x: uint256

@deploy
def __init__():
    self.x = 0

@external
def test():
	for i: uint256 in [self.usesideeffect(), self.usesideeffect(), self.usesideeffect()]:
		self.x += 1
		log I(i)

@view
def usesideeffect() -> uint256:
	return self.x
```

In the second example, the `iter_list` is an `ifexp`, thus it will only be evaluated lazily in the loop body. The log output will be `0, 1, 2` due to consumption of side effects.

```
event I:
	i: uint256

x: uint256

@deploy
def __init__():
    self.x = 0

@external
def test():
	for i: uint256 in ([self.usesideeffect(), self.usesideeffect(), self.usesideeffect()] if True else self.otherclause()):
		self.x += 1
		log I(i)

@view
def usesideeffect() -> uint256:
	return self.x

@view
def otherclause() -> uint256[3]:
	return [0, 0, 0]
```

In the third example, the `iter_list` is also an `ifexp`, thus it will only be evaluated lazily in the loop body. The log output will be `0, 1, 2` due to consumption of side effects.

```
event I:
	i: uint256

x: uint256[3]

@deploy
def __init__():
    self.x = [0, 0, 0]

@external
def test():
	for i: uint256 in (self.usesideeffect() if True else self.otherclause()):
		self.x[0] += 1
		self.x[1] += 1
		self.x[2] += 1
		log I(i)

@view
def usesideeffect() -> uint256[3]:
	return self.x

@view
def otherclause() -> uint256[3]:
	return [0, 0, 0]
```

This difference in evaluating `iter_list` only once at the very start and lazily within loop is already confusing. But since python itself is a pretty "interesting" language, if the behavior of vyper and python matches, we can still possibly justify it as a reasonable (but not ideal) design. In the following part, we show where the behavior differs between vyper and python to further argue that the current design should be considered faulty.

The matching python code for the first example also outputs `0, 0, 0` since `list` are pre-evaluated.

```
x = 0

def test():
    for i in [usesideeffect(), usesideeffect(), usesideeffect()]:
        global x
        x += 1
        print(i)

def usesideeffect():
    return x

test()
```

The matching python code for the second example pre-evaluates the list embedded in the `ifexp`, while `ifexp` itself is evaluated lazily, so the output is `0, 0, 0` and differs from vyper

```
x = 0

def test():
    for i in ([usesideeffect(), usesideeffect(), usesideeffect()] if True else otherclause()):
        global x
        x += 1 
        print(i)

def usesideeffect():
    return x

def otherclause():
    return [0, 0, 0]

test()
```

The matching python code for the third example evaluates `ifexp` lazily, so the output is the same `0, 1, 2` as vyper.

```
x = [0, 0, 0]

def test():
    for i in (usesideeffect() if True else otherclause()):
        global x
        x[0] += 1
        x[1] += 1
        x[2] += 1
        print(i)

def usesideeffect():
    return x

def otherclause():
    return [0, 0, 0]

test()
```

To be honest, we would say the python design is also pretty confusing, and could be a pitfall for developers, so it isn't necessarily a good idea to mimic their behavior. The cleanest way to handle this is probably to disallow complex expressions within the `iter_list` expression. Limiting it to a plain variable access or a literal will be sufficient to eliminate most confusion and allow us to stick to caching the `iter_list` expression before entering the loop.

If flexibility is desired over clarity, it would probably still be better to stick to a either always pre-evaluate (preferred) or lazy-evaluate (if lazy-evaluate is chosen, it is necessary to let developers know that sideeffect consumption may happen). This would reasonably reduce the chances of developers shooting themselves in their feet.

Finally, while we absolutely don't recommend this, if the decision is to stick to python behavior, then it should be made extremely clear in the docs and thoroughly tested.

We don't think the current implementation is justifiable, so hopefully we won't have to discuss the case where we stick to it.

While this report is a just another multi evaluation bug (and we believe this is currently the only case where multi evaluation may compile successfully into incorrect bytecode), it reveals insights into the constraints of `unique_symbol` in loops, as well as relatively limited considerations around side effect consumption compared to side effect generation. We believe these are topics worth exploring to improve the robustness of vyper.

## Impact Details

Incorrect multi-evaluations that may consume side effects might lead to unexpected program behavior and cause all kinds of problems for developers.

## References

* `https://github.com/vyperlang/vyper/blob/a29b49d422f6979be2b9c6c80aa583a60b1ccb7f/vyper/semantics/analysis/local.py#L572`
* `https://github.com/vyperlang/vyper/blob/a29b49d422f6979be2b9c6c80aa583a60b1ccb7f/vyper/semantics/analysis/local.py#L544`
* `https://github.com/vyperlang/vyper/blob/a29b49d422f6979be2b9c6c80aa583a60b1ccb7f/vyper/codegen/stmt.py#L261`
* `https://github.com/vyperlang/vyper/blob/a29b49d422f6979be2b9c6c80aa583a60b1ccb7f/vyper/codegen/stmt.py#L267`
* `https://github.com/vyperlang/vyper/blob/a29b49d422f6979be2b9c6c80aa583a60b1ccb7f/vyper/codegen/stmt.py#L272`
* `https://github.com/vyperlang/vyper/blob/a29b49d422f6979be2b9c6c80aa583a60b1ccb7f/vyper/ir/compile_ir.py#L756`

## Proof of Concept

## Proof of Concept

Already shown in Vulnerability Details section.
