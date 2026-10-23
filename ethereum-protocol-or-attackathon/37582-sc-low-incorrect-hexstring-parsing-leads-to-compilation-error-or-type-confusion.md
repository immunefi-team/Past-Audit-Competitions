# #37582 \[SC-Low] Incorrect HexString Parsing Leads To Compilation Error Or Type Confusion

**Submitted on Dec 9th 2024 at 20:26:34 UTC by @anatomist for** [**Attackathon | Ethereum Protocol**](https://immunefi.com/audit-competition/ethereum-protocol-attackathon)

* **Report ID:** #37582
* **Report Type:** Smart Contract
* **Report severity:** Low
* **Target:** <https://github.com/vyperlang/vyper>
* **Impacts:**
  * (Compiler) Incorrect bytecode generation leading to incorrect behavior
  * (Compiler) Unexpected behavior
  * (Compiler) Semantic analysis errors

## Description

## Brief/Intro

Hex string pre-parsing without considering position adjustments may result in mismatched offsets, and eventually cause compiler failure or type confusions.

## Vulnerability Details

Vyper supports several custom syntax, and one of those are HexString, written in the `x'01234'` syntax, and treated as `Bytes[]` literal values. Since HexString is not legal python syntax, a pre-parsing pass must be made to "fix" the code before passing it to python.ast module for parsing. The fixing rule is relatively simple, it just strips the leading `x`, and track occurances in `hex_string_locations`.

```
class HexStringParser:
    def __init__(self):
        self.locations = []
        self._current_x = None
        self._state = ParserState.NOT_RUNNING

    def consume(self, token, result):
        # prepare to check if the next token is a STRING
        if token.type == NAME and token.string == "x":
            self._state = ParserState.RUNNING
            self._current_x = token
            return True

        if self._state == ParserState.NOT_RUNNING:
            return False

        if self._state == ParserState.RUNNING:
            current_x = self._current_x
            self._current_x = None
            self._state = ParserState.NOT_RUNNING

            toks = [current_x]

            # drop the leading x token if the next token is a STRING to avoid a python
            # parser error
            if token.type == STRING:
                self.locations.append(current_x.start)
                toks = [TokenInfo(STRING, token.string, current_x.start, token.end, token.line)]
                result.extend(toks)
                return True

            result.extend(toks)

        return False
```

Later when visiting ast nodes, the `hex_string_locations` are looked up and used to annotate ast nodes as `Bytes` type instead of `Str` type.

```
def visit_Constant(self, node):
    ...
    elif isinstance(node.value, str):
        key = (node.lineno, node.col_offset)
        if key in self._pre_parser.hex_string_locations:
            if len(node.value) % 2 != 0:
                raise SyntaxException(
                    "Hex string must have an even number of characters",
                    self._source_code,
                    node.lineno,
                    node.col_offset,
                )
            node.ast_type = "HexBytes"
        else:
            node.ast_type = "Str"
    ...
```

However, it is overlooked the pre-parsing of HexString may

1. Get placed in a location where there were prior code adjustments that changes "fixed" occurance locations
2. HexString consumes leading `x` and may change "fixed" code locations itself.

This has several implications. The immediate consequence is compilation failure. For example, the following code fails to compile while it should

```
event X:
    a: Bytes[2]

@deploy
def __init__():
    log X(a = x"6161")  #log changes offset of HexString, and the hex_string_locations tracked location is incorrect when visiting ast
```

Slightly more consequential, code that should fail to compile can now compile fine, and suffer a type confusion + incorrect value interpretation at the same time. For example, the following code shouldn't compile, but it does, and it incorrectly passes `"6161"` as second argument to `FooBar.test` instead of `b"\x61\x61"`.

```
interface FooBar:
    def test(a: Bytes[2], b: String[4]): payable

@deploy
def __init__(ext: FooBar):
    extcall ext.test(x'6161', x'6161')  #ext.test(b'\x61\61', '6161') gets called
```

Additionally, the state machine of HexString pre-parser is also slightly flawed, and allows code such as the following to compile.

```
@deploy
def __init__():
    a: Bytes[2] = x x x x x"6161"
```

## Impact Details

1. Compilation failure for legal code
2. Potential type confusion when coupled with a user mistake in type specification
3. Allow illegal code to compile

## References

* `https://github.com/vyperlang/vyper/blob/e98e004235961613c3d769d4c652884b2a242608/vyper/ast/pre_parser.py#L136`
* `https://github.com/vyperlang/vyper/blob/e98e004235961613c3d769d4c652884b2a242608/vyper/ast/pre_parser.py#L282`
* `https://github.com/vyperlang/vyper/blob/e98e004235961613c3d769d4c652884b2a242608/vyper/ast/parse.py#L398`

## Proof of Concept

## Proof of Concept

Already shown in Vulnerability Details section.
