# #37191 \[BC-Insight] Unvalidated Field Names in Tuple ABI Parsing Causes Runtime Panic via reflect.StructOf

**Submitted on Nov 28th 2024 at 03:23:27 UTC by @CertiK for** [**Attackathon | Ethereum Protocol**](https://immunefi.com/audit-competition/ethereum-protocol-attackathon)

* **Report ID:** #37191
* **Report Type:** Blockchain/DLT
* **Report severity:** Insight
* **Target:** <https://github.com/ledgerwatch/erigon>
* **Impacts:**
  * (Specifications) A bug in specifications with no direct impact on client implementations

## Description

## Brief/Intro

A vulnerability exists in the accounts/abi package where unvalidated fieldName from user-provided ABI definitions are used to construct struct types using reflect.StructOf. This can cause a panic when the invalid characters are included in the fieldName.

## Vulnerability Details

In the type.go file of the accounts/abi package, the code dynamically constructs tuple types based on ABI component definitions as follow:

<https://github.com/erigontech/erigon/blob/v2.60.10/accounts/abi/type.go#L158>

```go
	case "tuple":
		var (
			fields     []reflect.StructField
			elems      []*Type
			names      []string
			expression string // canonical parameter expression
		)
		expression += "("
		overloadedNames := make(map[string]string)
		for idx, c := range components {
			cType, err := NewType(c.Type, c.InternalType, c.Components)
			if err != nil {
				return Type{}, err
			}
			fieldName, err := overloadedArgName(c.Name, overloadedNames)
			if err != nil {
				return Type{}, err
			}
			overloadedNames[fieldName] = fieldName
			fields = append(fields, reflect.StructField{
				Name: fieldName, // reflect.StructOf will panic for any exported field.
				Type: cType.GetType(),
				Tag:  reflect.StructTag("json:\"" + c.Name + "\""),
			})
			elems = append(elems, &cType)
			names = append(names, c.Name)
			expression += cType.stringKind
			if idx != len(components)-1 {
				expression += ","
			}
		}
		expression += ")"

		typ.TupleType = reflect.StructOf(fields)
		typ.TupleElems = elems
		typ.TupleRawNames = names
		typ.T = TupleTy
		typ.stringKind = expression

```

The fieldName is taken directly from the ABI component's Name without validation. However, the reflect.StructOf function requires that all field names be valid with the following check. Otherwise, the panic will rise.

<https://github.com/golang/go/blob/master/src/reflect/type.go#L2205>

```
		if !isValidFieldName(field.Name) {
			panic("reflect.StructOf: field " + strconv.Itoa(i) + " has invalid name")
		}

---

// isValidFieldName checks if a string is a valid (struct) field name or not.
//
// According to the language spec, a field name should be an identifier.
//

// identifier = letter { letter | unicode_digit } .
// letter = unicode_letter | "_" .
func isValidFieldName(fieldName string) bool {
	for i, c := range fieldName {
		if i == 0 && !isLetter(c) {
			return false
		}

		if !(isLetter(c) || unicode.IsDigit(c)) {
			return false
		}
	}

	return len(fieldName) > 0
}

```

It is worth noted a similar issue has been fixed in go-ethereum: <https://github.com/ethereum/go-ethereum/pull/24932>

## Impact Details

Any program that invokes the vulnerable code will panic with an invalid field name.

## References

* <https://github.com/golang/go/blob/master/src/reflect/type.go#L2205>
* <https://github.com/ethereum/go-ethereum/pull/24932>

## Proof of Concept

## Proof of Concept

We can reuse the test from go-ethereum to verify the issue:

```
// TestCrashers contains some strings which previously caused the abi codec to crash.
func TestCrashers(t *testing.T) {
	abi.JSON(strings.NewReader(`[{"inputs":[{"type":"tuple[]","components":[{"type":"bool","name":"_1"}]}]}]`))
	abi.JSON(strings.NewReader(`[{"inputs":[{"type":"tuple[]","components":[{"type":"bool","name":"&"}]}]}]`))
	abi.JSON(strings.NewReader(`[{"inputs":[{"type":"tuple[]","components":[{"type":"bool","name":"----"}]}]}]`))
	abi.JSON(strings.NewReader(`[{"inputs":[{"type":"tuple[]","components":[{"type":"bool","name":"foo.Bar"}]}]}]`))
}
```

Output:

```
--- FAIL: TestCrashers (0.00s)
panic: reflect.StructOf: field 0 has invalid name [recovered]
	panic: reflect.StructOf: field 0 has invalid name

goroutine 54 [running]:
testing.tRunner.func1.2({0x105c183c0, 0x140011f8710})
	/Users/xxx/.goenv/versions/1.22.5/src/testing/testing.go:1631 +0x1c4
testing.tRunner.func1()
	/Users/xxx/.goenv/versions/1.22.5/src/testing/testing.go:1634 +0x33c
panic({0x105c183c0?, 0x140011f8710?})
	/Users/xxx/.goenv/versions/1.22.5/src/runtime/panic.go:770 +0x124
reflect.StructOf({0x14000ad80e0, 0x1, 0x140016e4102?})
	/Users/xxx/.goenv/versions/1.22.5/src/reflect/type.go:2184 +0x1f14
github.com/erigontech/erigon/accounts/abi.NewType({0x140016e4118, 0x5}, {0x0, 0x0}, {0x140006d05a0, 0x1, 0x1})
	/Users/xxx/hackathon/erigon/accounts/abi/type.go:195 +0x1180
github.com/erigontech/erigon/accounts/abi.NewType({0x140016e4118, 0x7}, {0x0, 0x0}, {0x140006d05a0, 0x1, 0x1})
	/Users/xxx/hackathon/erigon/accounts/abi/type.go:90 +0x144
github.com/erigontech/erigon/accounts/abi.(*Argument).UnmarshalJSON(0x14000a36090, {0x14000ba020c, 0x3d, 0x1f4})
	/Users/xxx/hackathon/erigon/accounts/abi/argument.go:56 +0xc0
encoding/json.(*decodeState).object(0x14000a36000, {0x105d8f0e0?, 0x14000a36090?, 0x1046bd148?})
	/Users/xxx/.goenv/versions/1.22.5/src/encoding/json/decode.go:604 +0x5c4
encoding/json.(*decodeState).value(0x14000a36000, {0x105d8f0e0?, 0x14000a36090?, 0x1?})
	/Users/xxx/.goenv/versions/1.22.5/src/encoding/json/decode.go:374 +0x40
encoding/json.(*decodeState).array(0x14000a36000, {0x105bf55e0?, 0x14000ad8020?, 0x105c183c0?})
	/Users/xxx/.goenv/versions/1.22.5/src/encoding/json/decode.go:555 +0x48c
encoding/json.(*decodeState).value(0x14000a36000, {0x105bf55e0?, 0x14000ad8020?, 0x6?})
	/Users/xxx/.goenv/versions/1.22.5/src/encoding/json/decode.go:364 +0x70
encoding/json.(*decodeState).object(0x14000a36000, {0x105e51e80?, 0x14000ad8000?, 0x1046bd148?})
	/Users/xxx/.goenv/versions/1.22.5/src/encoding/json/decode.go:755 +0xa94
encoding/json.(*decodeState).value(0x14000a36000, {0x105e51e80?, 0x14000ad8000?, 0x1?})
	/Users/xxx/.goenv/versions/1.22.5/src/encoding/json/decode.go:374 +0x40
encoding/json.(*decodeState).array(0x14000a36000, {0x105bfb800?, 0x140016d9578?, 0x1046ca1ac?})
	/Users/xxx/.goenv/versions/1.22.5/src/encoding/json/decode.go:555 +0x48c
encoding/json.(*decodeState).value(0x14000a36000, {0x105bfb800?, 0x140016d9578?, 0x1046c9770?})
	/Users/xxx/.goenv/versions/1.22.5/src/encoding/json/decode.go:364 +0x70
encoding/json.(*decodeState).unmarshal(0x14000a36000, {0x105bfb800?, 0x140016d9578?})
	/Users/xxx/.goenv/versions/1.22.5/src/encoding/json/decode.go:181 +0x104
encoding/json.Unmarshal({0x14000ba0200, 0x4c, 0x200}, {0x105bfb800, 0x140016d9578})
	/Users/xxx/.goenv/versions/1.22.5/src/encoding/json/decode.go:108 +0xe4
github.com/erigontech/erigon/accounts/abi.(*ABI).UnmarshalJSON(0x14000cca008, {0x14000ba0200, 0x4c, 0x200})
	/Users/xxx/hackathon/erigon/accounts/abi/abi.go:160 +0x60
encoding/json.(*decodeState).array(0x14000a5e028, {0x105e44000?, 0x14000cca008?, 0x1046ccfe4?})
	/Users/xxx/.goenv/versions/1.22.5/src/encoding/json/decode.go:507 +0x370
encoding/json.(*decodeState).value(0x14000a5e028, {0x105e44000?, 0x14000cca008?, 0x1046cccec?})
	/Users/xxx/.goenv/versions/1.22.5/src/encoding/json/decode.go:364 +0x70
encoding/json.(*decodeState).unmarshal(0x14000a5e028, {0x105e44000?, 0x14000cca008?})
	/Users/xxx/.goenv/versions/1.22.5/src/encoding/json/decode.go:181 +0x104
encoding/json.(*Decoder).Decode(0x14000a5e000, {0x105e44000, 0x14000cca008})
	/Users/xxx/.goenv/versions/1.22.5/src/encoding/json/stream.go:73 +0x118
github.com/erigontech/erigon/accounts/abi.JSON({_, _})
	/Users/xxx/hackathon/erigon/accounts/abi/abi.go:55 +0x9c
github.com/erigontech/erigon/accounts/abi/bind_test.TestCrashers(0x140001b8820?)
	/Users/xxx/hackathon/erigon/accounts/abi/bind/base_test.go:265 +0x60
testing.tRunner(0x140001b8820, 0x105ef6850)
	/Users/xxx/.goenv/versions/1.22.5/src/testing/testing.go:1689 +0xec
created by testing.(*T).Run in goroutine 1
	/Users/xxx/.goenv/versions/1.22.5/src/testing/testing.go:1742 +0x318
FAIL	github.com/erigontech/erigon/accounts/abi/bind	0.929s
FAIL
```
