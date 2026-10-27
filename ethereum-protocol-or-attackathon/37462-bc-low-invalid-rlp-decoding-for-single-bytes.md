# #37462 \[BC-Low] Invalid RLP decoding for single bytes

**Submitted on Dec 5th 2024 at 10:45:39 UTC by @Franfran for** [**Attackathon | Ethereum Protocol**](https://immunefi.com/audit-competition/ethereum-protocol-attackathon)

* **Report ID:** #37462
* **Report Type:** Blockchain/DLT
* **Report severity:** Low
* **Target:** <https://github.com/hyperledger/besu>
* **Impacts:**
  * (Specifications) A bug in specifications with no direct impact on client implementations

## Description

## Brief/Intro

The `decodeOne` function in the `RLP` class has a bug which makes it accept invalid inputs for an encoded value that has the same prefix as a single byte and could trigger consensus failures if the MPT contains some invalid state.

## Vulnerability Details

There is a corectness bug in the implementation of the `RLP.decodeOne` function.\
Indeed, if we look at the [Ethereum official documentation about the RLP format](https://ethereum.org/en/developers/docs/data-structures-and-encoding/rlp/), it is stated that

```
RLP encoding is defined as follows:
...
- For a single byte whose value is in the `[0x00, 0x7f]` (decimal `[0, 127]`) range, that byte is its own RLP encoding.
...
```

By looking at the `decodeOne` function code, we can see the case of the single byte as:

```java
public static Bytes decodeOne(final Bytes encodedValue) {  
  if (encodedValue.size() == 0) {  
    throw new RLPException("Invalid empty input for RLP decoding");  
  }  
  
  final int prefix = encodedValue.get(0) & 0xFF;  
  final RLPDecodingHelpers.Kind kind = RLPDecodingHelpers.Kind.of(prefix);  
  if (kind.isList()) {  
    throw new RLPException(format("Invalid input: value %s is an RLP list", encodedValue));  
  }  
  
  if (kind == RLPDecodingHelpers.Kind.BYTE_ELEMENT) {  
    return encodedValue;  
  }

  /* ... */
```

We can see that if the prefix is a single byte, the full **encodedValue** is returned while it is not checked that this value is indeed a single byte.\
Let's write a test to confirm this behavior, which can be done by extending `RLPTest.java`

## Impact Details

This function is called twice in the codebase.

1. <https://github.com/hyperledger/besu/blob/e5c9f55f8b52d7a15d3b37db9beb076fb2f1c121/ethereum/p2p/src/main/java/org/hyperledger/besu/ethereum/p2p/rlpx/framing/Framer.java#L281>
2. <https://github.com/hyperledger/besu/blob/472357f118832ae4a0374a31bea71d33d2408259/ethereum/core/src/main/java/org/hyperledger/besu/ethereum/trie/diffbased/common/worldview/accumulator/DiffBasedWorldStateUpdateAccumulator.java#L402\\>
   The only relevant one is 2. because in the case of 1., only a single byte can be passed to the `decodeOne` function. In the case of 2., if an invalid RLP value would be stored in the Merkle Patricia Trie (MPT) that would wrongly be decoded by this function such that it starts with a `SINGLE_BYTE` and ends up with other bytes, while other clients with the correct implementation would disagree with the world view of Besu, triggering a consensus failure. Because of this assumption, the chosen vulnerability level was set to `LOW`. Nonetheless, it is important to consider this report in order to fix the implementation since future code may rely on the assumption that any RLP-encoded value whose prefix is the single byte will error like other implementations.\
   Let's for instance consider `go-ethereum`, which has a different implementation of the decoding of an RLP value whose prefix is a single byte: <https://github.com/ethereum/go-ethereum/blob/67a3b087951a3f3a8e341ae32b6ec18f3553e5cc/rlp/decode.go#L1056-L1058>

```go
func (s *Stream) readKind() (kind Kind, size uint64, err error) {
	b, err := s.readByte()
	/* ... */
    s.byteval = 0
	switch {
	case b < 0x80:
		// For a single byte whose value is in the [0x00, 0x7F] range, that byte
		// is its own RLP encoding.
		s.byteval = b
		return Byte, 0, nil
	/* ... */
```

Then, the prefix is of type `Byte` and is validated correctly: <https://github.com/ethereum/go-ethereum/blob/67a3b087951a3f3a8e341ae32b6ec18f3553e5cc/rlp/decode.go#L376-L381>

```go
func decodeByteArray(s *Stream, val reflect.Value) error {•••••••••••••
	kind, size, err := s.Kind()
	if err != nil {
		return err
	}
	slice := byteArrayBytes(val, val.Len())
	switch kind {
	case Byte:
		if len(slice) == 0 {
			return &decodeError{msg: "input string too long", typ: val.Type()}
		} else if len(slice) > 1 {
			return &decodeError{msg: "input string too short", typ: val.Type()}
		}
		slice[0] = s.byteval
		s.kind = -1
```

Since it's going to return a `decodeError` if the length of the slice is not 1.

## References

<https://ethereum.org/en/developers/docs/data-structures-and-encoding/rlp\\>
<https://github.com/hyperledger/besu/blob/472357f118832ae4a0374a31bea71d33d2408259/ethereum/rlp/src/main/java/org/hyperledger/besu/ethereum/rlp/RLP.java#L150-L152>

## Proof of Concept

## Proof of Concept

```java
@Test
public void decodeOneSuffixBytes() {  
  Assertions.assertThrows(RLPException.class, () -> RLP.decodeOne(Bytes.fromHexString("0x7f1234"))); // this is invalid RLP
}
```

As expected, this test fails because the function doesn't throw as it should.
