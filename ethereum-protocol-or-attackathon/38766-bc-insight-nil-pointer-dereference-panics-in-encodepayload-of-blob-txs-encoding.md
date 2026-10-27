# #38766 \[BC-Insight] Nil Pointer Dereference Panics in encodePayload() of Blob Tx’s Encoding

**Submitted on Jan 13th 2025 at 03:44:37 UTC by @CertiK for** [**Attackathon | Ethereum Protocol**](https://immunefi.com/audit-competition/ethereum-protocol-attackathon)

* **Report ID:** #38766
* **Report Type:** Blockchain/DLT
* **Report severity:** Insight
* **Target:** <https://github.com/ledgerwatch/erigon>
* **Impacts:**
  * (Specifications) A bug in specifications with no direct impact on client implementations

## Description

## Brief/Intro

`EIP-4844` (<https://github.com/ethereum/EIPs/blob/master/EIPS/eip-4844.md> ) introduces a new transaction type, blob tx, which does not allow the `To` address to be nil (i.e., it cannot be used to create contract).

In Erigon (<https://github.com/erigontech/erigon/> ), the blob tx allows the `To` address to be nil. In case that the blob tx includes a nil `To` address, the tx encoding causes a nil pointer dereference panics due to a flaw in function `encodePayload()`.

## Vulnerability Details

Affected Codebase:\
<https://github.com/erigontech/erigon/tree/v2.61.0>

The blob tx is defined via embedding the `DynamicFeeTransaction`, which allows the To address to be nil.

<https://github.com/erigontech/erigon/blob/v2.61.0/core/types/blob\\_tx.go#L20C1-L24C2>

```
type BlobTx struct {
	DynamicFeeTransaction
	MaxFeePerBlobGas    *uint256.Int
	BlobVersionedHashes []libcommon.Hash
}
```

<https://github.com/erigontech/erigon/blob/v2.61.0/core/types/dynamic\\_fee\\_tx.go#L35>

```
type DynamicFeeTransaction struct {
	CommonTx
	ChainID    *uint256.Int
	Tip        *uint256.Int
	FeeCap     *uint256.Int
	AccessList types2.AccessList
}
```

<https://github.com/erigontech/erigon/blob/v2.61.0/core/types/legacy\\_tx.go#L34>

```
type CommonTx struct {
	TransactionMisc

	Nonce   uint64             // nonce of sender account
	Gas     uint64             // gas limit
	To      *libcommon.Address `rlp:"nil"` // nil means contract creation
	Value   *uint256.Int       // wei amount
	Data    []byte             // contract invocation input data
	V, R, S uint256.Int        // signature values
}
```

However, this violates the `EIP-4844` (<https://github.com/ethereum/EIPs/blob/master/EIPS/eip-4844.md> )

> The field to deviates slightly from the semantics with the exception that it MUST NOT be nil and therefore must always represent a 20-byte address. This means that blob transactions cannot have the form of a create transaction.

In case that the `To` address is set to be nil in a blob tx, the encoding with two functions `EncodeRLP()` and `MarshalBinary()` calls the `encodePayload()` to perform the encoding of blob tx:

<https://github.com/erigontech/erigon/blob/v2.61.0/core/types/blob\\_tx.go#L243>

```
func (stx *BlobTx) EncodeRLP(w io.Writer) error {
	payloadSize, nonceLen, gasLen, accessListLen, blobHashesLen := stx.payloadSize()
	// size of struct prefix and TxType
	envelopeSize := 1 + rlp.ListPrefixLen(payloadSize) + payloadSize
	b := newEncodingBuf()
	defer pooledBuf.Put(b)
	// envelope
	if err := rlp.EncodeStringSizePrefix(envelopeSize, w, b[:]); err != nil {
		return err
	}
	// encode TxType
	b[0] = BlobTxType
	if _, err := w.Write(b[:1]); err != nil {
		return err
	}
	if err := stx.encodePayload(w, b[:], payloadSize, nonceLen, gasLen, accessListLen, blobHashesLen); err != nil {
		return err
	}
	return nil
}

func (stx *BlobTx) MarshalBinary(w io.Writer) error {
	payloadSize, nonceLen, gasLen, accessListLen, blobHashesLen := stx.payloadSize()
	b := newEncodingBuf()
	defer pooledBuf.Put(b)
	// encode TxType
	b[0] = BlobTxType
	if _, err := w.Write(b[:1]); err != nil {
		return err
	}
	if err := stx.encodePayload(w, b[:], payloadSize, nonceLen, gasLen, accessListLen, blobHashesLen); err != nil {
		return err
	}
	return nil
}
```

<https://github.com/erigontech/erigon/blob/v2.61.0/core/types/blob\\_tx.go#L167>

```
func (stx *BlobTx) encodePayload(w io.Writer, b []byte, payloadSize, nonceLen, gasLen, accessListLen, blobHashesLen int) error {
	// prefix
	if err := rlp.EncodeStructSizePrefix(payloadSize, w, b); err != nil {
		return err
	}
	// encode ChainID
	if err := rlp.EncodeUint256(stx.ChainID, w, b); err != nil {
		return err
	}
	// encode Nonce
	if err := rlp.EncodeInt(stx.Nonce, w, b); err != nil {
		return err
	}
	// encode MaxPriorityFeePerGas
	if err := rlp.EncodeUint256(stx.Tip, w, b); err != nil {
		return err
	}
	// encode MaxFeePerGas
	if err := rlp.EncodeUint256(stx.FeeCap, w, b); err != nil {
		return err
	}
	// encode Gas
	if err := rlp.EncodeInt(stx.Gas, w, b); err != nil {
		return err
	}
	// encode To
	b[0] = 128 + 20
	if _, err := w.Write(b[:1]); err != nil {
		return err
	}
	if _, err := w.Write(stx.To[:]); err != nil {
		return err
	}
	// encode Value
	if err := rlp.EncodeUint256(stx.Value, w, b); err != nil {
		return err
	}
	// encode Data
	if err := rlp.EncodeString(stx.Data, w, b); err != nil {
		return err
	}
	// prefix
	if err := rlp.EncodeStructSizePrefix(accessListLen, w, b); err != nil {
		return err
	}
	// encode AccessList
	if err := encodeAccessList(stx.AccessList, w, b); err != nil {
		return err
	}
	// encode MaxFeePerBlobGas
	if err := rlp.EncodeUint256(stx.MaxFeePerBlobGas, w, b); err != nil {
		return err
	}
	// prefix
	if err := rlp.EncodeStructSizePrefix(blobHashesLen, w, b); err != nil {
		return err
	}
	// encode BlobVersionedHashes
	if err := encodeBlobVersionedHashes(stx.BlobVersionedHashes, w, b); err != nil {
		return err
	}
	// encode V
	if err := rlp.EncodeUint256(&stx.V, w, b); err != nil {
		return err
	}
	// encode R
	if err := rlp.EncodeUint256(&stx.R, w, b); err != nil {
		return err
	}
	// encode S
	if err := rlp.EncodeUint256(&stx.S, w, b); err != nil {
		return err
	}
	return nil
}
```

At line 197, the `stx.To.Bytes()` directly invokes the `Bytes()` method without checking if the `To` is nil or not. This overlook would lead to the nil pointer dereference panics.

The latest commit makes slight changes to the affected code, but it still cause nil pointer deference panics when writing nil `To` address. (Demonstrated in the Proof of Concept)

<https://github.com/erigontech/erigon/blob/main/core/types/blob\\_tx.go#L218>

```
	// encode To
	b[0] = 128 + 20
	if _, err := w.Write(b[:1]); err != nil {
		return err
	}
	if _, err := w.Write(stx.To[:]); err != nil {
		return err
	}
```

## Impact Details

Since the encoding of tx is used frequently in the codebase, through users, p2p or consensus layer, any invocation of the blob tx encoding with nil `To` address would crash the Erigon node.

## References

* <https://github.com/erigontech/erigon/tree/v2.61.0>
* <https://github.com/ethereum/EIPs/blob/master/EIPS/eip-4844.md>

## Proof of Concept

## Proof of Concept

For simplicity, using the existing tests in file <https://github.com/erigontech/erigon/blob/v2.61.0/core/types/transaction\\_test.go#L600\\>
we create the following unit test to demonstrate the nil pointer dereference panics with MarshalBinary() of the blob tx.

1. Set the `To` address as nil:

```
func newRandBlobTx() *BlobTx {
	stx := &BlobTx{DynamicFeeTransaction: DynamicFeeTransaction{
		CommonTx: CommonTx{
			Nonce: rand.Uint64(),
			Gas:   rand.Uint64(),
			//To:    randAddr(),
			To:    nil,
			Value: uint256.NewInt(rand.Uint64()),
			Data:  randData(),
			V:     *uint256.NewInt(0),
			R:     *uint256.NewInt(rand.Uint64()),
			S:     *uint256.NewInt(rand.Uint64()),
		},
		ChainID:    uint256.NewInt(rand.Uint64()),
		Tip:        uint256.NewInt(rand.Uint64()),
		FeeCap:     uint256.NewInt(rand.Uint64()),
		AccessList: randAccessList(),
	},
		MaxFeePerBlobGas:    uint256.NewInt(rand.Uint64()),
		BlobVersionedHashes: randHashes(randIntInRange(1, 6)),
	}
	return stx
}
```

2. Run the following unit test in the same file:

```
func TestBlobTxEncodeDecode(t *testing.T) {
	rand.Seed(time.Now().UnixNano())
	populateBlobTxs()


	for i := 0; i < N; i++ {
		tx, err := encodeDecodeBinary(dummyBlobTxs[i])
		if err != nil {
			t.Fatal(err)
		}
		if err := assertEqual(dummyBlobTxs[i], tx); err != nil {
			t.Fatal(err)
		}
	}
}

func encodeDecodeBinary(tx Transaction) (Transaction, error) {
	var buf bytes.Buffer
	var err error
	if err = tx.MarshalBinary(&buf); err != nil {
		return nil, fmt.Errorf("rlp encoding failed: %w", err)
	}
	var parsedTx Transaction
	if parsedTx, err = UnmarshalTransactionFromBinary(buf.Bytes(), false /* blobTxnsAreWrappedWithBlobs */); err != nil {
		return nil, fmt.Errorf("rlp decoding failed: %w", err)
	}
	return parsedTx, nil
}
```

3. The test result shows an nil pointer dereference panics occur due to the method invocation `To.Bytes()`:

```
=== RUN   TestBlobTxEncodeDecode
--- FAIL: TestBlobTxEncodeDecode (0.01s)
panic: runtime error: invalid memory address or nil pointer dereference [recovered]
	panic: runtime error: invalid memory address or nil pointer dereference
[signal SIGSEGV: segmentation violation code=0x1 addr=0x0 pc=0x10232483e]

goroutine 25 [running]:
testing.tRunner.func1.2({0x1026be860, 0x102c99f30})
	/Users/***/go/pkg/mod/golang.org/toolchain@v0.0.1-go1.23.0.darwin-amd64/src/testing/testing.go:1632 +0x230
testing.tRunner.func1()
	/Users/***/go/pkg/mod/golang.org/toolchain@v0.0.1-go1.23.0.darwin-amd64/src/testing/testing.go:1635 +0x35e
panic({0x1026be860?, 0x102c99f30?})
	/Users/***/go/pkg/mod/golang.org/toolchain@v0.0.1-go1.23.0.darwin-amd64/src/runtime/panic.go:785 +0x132
github.com/erigontech/erigon/core/types.(*BlobTx).encodePayload(0xc0001c0500, {0x1027acba0, 0xc0005ae1e0}, {0xc0001267b0, 0x21, 0x21}, 0xc0005ae1e0?, 0x0?, 0x0?, 0x118, ...)
	/Users/***/immunefi/erigon/core/types/blob_tx.go:197 +0x17e
github.com/erigontech/erigon/core/types.(*BlobTx).MarshalBinary(0xc0001c0500, {0x1027acba0, 0xc0005ae1e0})
	/Users/***/immunefi/erigon/core/types/blob_tx.go:274 +0xde
github.com/erigontech/erigon/core/types.encodeDecodeBinary({0x1027bf260, 0xc0001c0500})
	/Users/***/immunefi/erigon/core/types/transaction_test.go:591 +0x50
github.com/erigontech/erigon/core/types.TestBlobTxEncodeDecode(0xc000142680)
	/Users/***/immunefi/erigon/core/types/transaction_test.go:871 +0xf0
testing.tRunner(0xc000142680, 0x1027a6508)
	/Users/***/go/pkg/mod/golang.org/toolchain@v0.0.1-go1.23.0.darwin-amd64/src/testing/testing.go:1690 +0xf4
created by testing.(*T).Run in goroutine 1
	/Users/***/go/pkg/mod/golang.org/toolchain@v0.0.1-go1.23.0.darwin-amd64/src/testing/testing.go:1743 +0x390


Process finished with the exit code 1
```

4. Nil Pointer dereference also occurs in the latest commit (<https://github.com/erigontech/erigon/blob/main/core/types/blob\\_tx.go#L218> ) when writing nil `To` address:

```
=== RUN   TestBlobTxEncodeDecode
--- FAIL: TestBlobTxEncodeDecode (0.00s)
panic: runtime error: invalid memory address or nil pointer dereference [recovered]
	panic: runtime error: invalid memory address or nil pointer dereference
[signal SIGSEGV: segmentation violation code=0x1 addr=0x0 pc=0x10c83485b]

goroutine 11 [running]:
testing.tRunner.func1.2({0x10cbce860, 0x10d1a9f30})
	/Users/***/go/pkg/mod/golang.org/toolchain@v0.0.1-go1.23.0.darwin-amd64/src/testing/testing.go:1632 +0x230
testing.tRunner.func1()
	/Users/***/go/pkg/mod/golang.org/toolchain@v0.0.1-go1.23.0.darwin-amd64/src/testing/testing.go:1635 +0x35e
panic({0x10cbce860?, 0x10d1a9f30?})
	/Users/***/go/pkg/mod/golang.org/toolchain@v0.0.1-go1.23.0.darwin-amd64/src/runtime/panic.go:785 +0x132
github.com/erigontech/erigon/core/types.(*BlobTx).encodePayload(0xc000256100, {0x10ccbcba0, 0xc00063ced0}, {0xc000126630, 0x21, 0x21}, 0xc00063ced0?, 0x0?, 0x0?, 0x188, ...)
	/Users/***/immunefi/erigon/core/types/blob_tx.go:200 +0x17b
github.com/erigontech/erigon/core/types.(*BlobTx).MarshalBinary(0xc000256100, {0x10ccbcba0, 0xc00063ced0})
	/Users/***/immunefi/erigon/core/types/blob_tx.go:274 +0xde
github.com/erigontech/erigon/core/types.encodeDecodeBinary({0x10cccf260, 0xc000256100})
	/Users/***/immunefi/erigon/core/types/transaction_test.go:591 +0x50
github.com/erigontech/erigon/core/types.TestBlobTxEncodeDecode(0xc0001424e0)
	/Users/***/immunefi/erigon/core/types/transaction_test.go:871 +0xf0
testing.tRunner(0xc0001424e0, 0x10ccb6508)
	/Users/***/go/pkg/mod/golang.org/toolchain@v0.0.1-go1.23.0.darwin-amd64/src/testing/testing.go:1690 +0xf4
created by testing.(*T).Run in goroutine 1
	/Users/***/go/pkg/mod/golang.org/toolchain@v0.0.1-go1.23.0.darwin-amd64/src/testing/testing.go:1743 +0x390


Process finished with the exit code 1
```
