# #37583 \[SC-Low] Incorrect For Annotation Parsing

**Submitted on Dec 9th 2024 at 20:30:36 UTC by @anatomist for** [**Attackathon | Ethereum Protocol**](https://immunefi.com/audit-competition/ethereum-protocol-attackathon)

* **Report ID:** #37583
* **Report Type:** Smart Contract
* **Report severity:** Low
* **Target:** <https://github.com/vyperlang/vyper>
* **Impacts:**
  * (Compiler) Unexpected behavior

## Description

## Brief/Intro

For loop extracts annotation parsing and constructs ast separately for it. However, insufficient validation allows otherwise illegal code to compile, and create ambiguity in the intended behavior of contracts.

## Vulnerability Details

Python for loops does not support type annotation for iterator variable. However, since vyper is a strongly typed language, it requires the type to be specified. This introduces the need for pre-parsing of for loops to extract out the type annotation, "fix" the code.

```
class ForParser:
    def __init__(self, code):
        self._code = code
        self.annotations = {}
        self._current_annotation = None

        self._state = ParserState.NOT_RUNNING
        self._current_for_loop = None

    def consume(self, token):
        # state machine: we can start slurping tokens soon
        if token.type == NAME and token.string == "for":
            # note: self._state should be NOT_RUNNING here, but we don't sanity
            # check here as that should be an error the parser will handle.
            self._state = ParserState.START_SOON
            self._current_for_loop = token.start

        if self._state == ParserState.NOT_RUNNING:
            return False

        # state machine: start slurping tokens
        if token.type == OP and token.string == ":":
            self._state = ParserState.RUNNING

            # sanity check -- this should never really happen, but if it does,
            # try to raise an exception which pinpoints the source.
            if self._current_annotation is not None:
                raise SyntaxException(
                    "for loop parse error", self._code, token.start[0], token.start[1]
                )

            self._current_annotation = []
            return True  # do not add ":" to tokens.

        # state machine: end slurping tokens
        if token.type == NAME and token.string == "in":
            self._state = ParserState.NOT_RUNNING
            self.annotations[self._current_for_loop] = self._current_annotation or []
            self._current_annotation = None
            return False

        if self._state != ParserState.RUNNING:
            return False

        # slurp the token
        self._current_annotation.append(token)
        return True
```

The extracted types are later manually injected back into the python parsed ast.

```
def visit_For(self, node):
    key = (node.lineno, node.col_offset)
    annotation_tokens = self._pre_parser.for_loop_annotations.pop(key)

    if not annotation_tokens:
        ...

    annotation_str = tokenize.untokenize(annotation_tokens)
    annotation_str = "dummy_target:" + annotation_str

    try:
        fake_node = python_ast.parse(annotation_str).body[0]
    except SyntaxError as e:
        ...

    self._tokens.mark_tokens(fake_node)

    fake_node.target = node.target
    node.target = fake_node

    return self.generic_visit(node)
```

However, the implementation does not check that the extracted code is purely an annotation. For example, the following code extracts `uint256 = 1` as the annotation, and when parsed along with `dummy_target: uint256 = 1`, it also appears to be valid python syntax. However, only the type is used in code generation, and the value associated with it will be discarded.

```
for i: uint256 = 1 in range(1, 2):
    pass
```

This means users may specify arbitrary "legal looking" python code as part of the annotation and the code will still compile, which may create ambiguity in what the contract is intended to do. For example, the following code will confuse users on whether the internal function is called.

```
x: uint256

@deploy
def __init__(init_x: uint256):
    #set_x won't be called since it's ignored in codegen
    for i: uint256 = self.set_x(init_x) in range(5):
        pass
        
@internal
def set_x(init_x: uint256) -> uint256:
    self.x = init_x
    return init_x
```

To make things worse, while the value is ignored in codegen, it is actually respected in certain semantic analysis checks (such as self recursion), which adds to the ambiguity on what the code does.

```
@internal
def recursion(a: uint256) -> uint256:
    for i: uint256 = self.recursion(a) in range(5):
        pass
```

## Impact Details

1. Ambiguity in contract behavior
2. Allow illegal code to compile

## References

* `https://github.com/vyperlang/vyper/blob/e98e004235961613c3d769d4c652884b2a242608/vyper/ast/pre_parser.py#L97`
* `https://github.com/vyperlang/vyper/blob/e98e004235961613c3d769d4c652884b2a242608/vyper/ast/parse.py#L308`

## Proof of Concept

## Proof of Concept

Already shown in Vulnerability Details section.
