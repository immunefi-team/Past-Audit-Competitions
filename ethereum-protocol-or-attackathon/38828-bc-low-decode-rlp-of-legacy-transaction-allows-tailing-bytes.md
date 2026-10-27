# #38828 \[BC-Low] Decode RLP of Legacy Transaction Allows Tailing Bytes

**Submitted on Jan 14th 2025 at 19:33:55 UTC by @CertiK for** [**Attackathon | Ethereum Protocol**](https://immunefi.com/audit-competition/ethereum-protocol-attackathon)

* **Report ID:** #38828
* **Report Type:** Blockchain/DLT
* **Report severity:** Low
* **Target:** <https://github.com/ledgerwatch/erigon>
* **Impacts:**
  * (Specifications) A bug in specifications with no direct impact on client implementations

## Description

## Brief/Intro

RLP serialization is widely used in the Ethereum clients, which provides a standard format for the data transfer between nodes.

In Erigon (<https://github.com/erigontech/erigon/> ), the decode RLP of legacy tx misses check the tailing bytes, which allows extra tailing bytes during the RLP decoding. This could possibly be a consensus issue as other Ethereum clients reject such legacy transactions.

## Vulnerability Details

Affected Codebase:\
<https://github.com/erigontech/erigon/tree/v2.61.0>

The function `DecodeRLPTransaction()` is intended to decode the RLP encoding of transactions:

<https://github.com/erigontech/erigon/blob/v2.61.0/core/types/transaction.go#L116>

```
func DecodeRLPTransaction(s *rlp.Stream, blobTxnsAreWrappedWithBlobs bool) (Transaction, error) {
	kind, _, err := s.Kind()
	if err != nil {
		return nil, err
	}
	if rlp.List == kind {
		tx := &LegacyTx{}
		if err = tx.DecodeRLP(s); err != nil {
			return nil, err
		}
		return tx, nil
	}
	if rlp.String != kind {
		return nil, fmt.Errorf("not an RLP encoded transaction. If this is a canonical encoded transaction, use UnmarshalTransactionFromBinary instead. Got %v for kind, expected String", kind)
	}
	// Decode the EIP-2718 typed TX envelope.
	var b []byte
	if b, err = s.Bytes(); err != nil {
		return nil, err
	}
	if len(b) == 0 {
		return nil, rlp.EOL
	}
	return UnmarshalTransactionFromBinary(b, blobTxnsAreWrappedWithBlobs)
}
```

For non-legacy transactions, it invokes `UnmarshalTransactionFromBinary()`that handles the tailing bytes with the `check s.Remaining() != 0`:

<https://github.com/erigontech/erigon/blob/v2.61.0/core/types/transaction.go#L178C1-L211C2>

```
func UnmarshalTransactionFromBinary(data []byte, blobTxnsAreWrappedWithBlobs bool) (Transaction, error) {
	if len(data) <= 1 {
		return nil, fmt.Errorf("short input: %v", len(data))
	}
	s := rlp.NewStream(bytes.NewReader(data[1:]), uint64(len(data)-1))
	var t Transaction
	switch data[0] {
	case AccessListTxType:
		t = &AccessListTx{}
	case DynamicFeeTxType:
		t = &DynamicFeeTransaction{}
	case BlobTxType:
		if blobTxnsAreWrappedWithBlobs {
			t = &BlobTxWrapper{}
		} else {
			t = &BlobTx{}
		}
	case SetCodeTxType:
		t = &SetCodeTransaction{}
	default:
		if data[0] >= 0x80 {
			// Tx is type legacy which is RLP encoded
			return DecodeTransaction(data)
		}
		return nil, ErrTxTypeNotSupported
	}
	if err := t.DecodeRLP(s); err != nil {
		return nil, err
	}
	if s.Remaining() != 0 {
		return nil, fmt.Errorf("trailing bytes after rlp encoded transaction")
	}
	return t, nil
}
```

However, if it’s the legacy transaction, it calls `tx.DecodeRLP(s)` to directly decode the legacy transaction, in which the tailing bytes is unchecked. This oversight would allow tailing bytes to sneak into the legacy transaction when calling `DecodeRLPTransaction()` to decode the legacy transaction.

<https://github.com/erigontech/erigon/blob/v2.61.0/core/types/legacy\\_tx.go#L291C1-L340C2>

```
func (tx *LegacyTx) DecodeRLP(s *rlp.Stream) error {
	_, err := s.List()
	if err != nil {
		return fmt.Errorf("legacy tx must be a list: %w", err)
	}
	if tx.Nonce, err = s.Uint(); err != nil {
		return fmt.Errorf("read Nonce: %w", err)
	}
	var b []byte
	if b, err = s.Uint256Bytes(); err != nil {
		return fmt.Errorf("read GasPrice: %w", err)
	}
	tx.GasPrice = new(uint256.Int).SetBytes(b)
	if tx.Gas, err = s.Uint(); err != nil {
		return fmt.Errorf("read Gas: %w", err)
	}
	if b, err = s.Bytes(); err != nil {
		return fmt.Errorf("read To: %w", err)
	}
	if len(b) > 0 && len(b) != 20 {
		return fmt.Errorf("wrong size for To: %d", len(b))
	}
	if len(b) > 0 {
		tx.To = &libcommon.Address{}
		copy((*tx.To)[:], b)
	}
	if b, err = s.Uint256Bytes(); err != nil {
		return fmt.Errorf("read Value: %w", err)
	}
	tx.Value = new(uint256.Int).SetBytes(b)
	if tx.Data, err = s.Bytes(); err != nil {
		return fmt.Errorf("read Data: %w", err)
	}
	if b, err = s.Uint256Bytes(); err != nil {
		return fmt.Errorf("read V: %w", err)
	}
	tx.V.SetBytes(b)
	if b, err = s.Uint256Bytes(); err != nil {
		return fmt.Errorf("read R: %w", err)
	}
	tx.R.SetBytes(b)
	if b, err = s.Uint256Bytes(); err != nil {
		return fmt.Errorf("read S: %w", err)
	}
	tx.S.SetBytes(b)
	if err = s.ListEnd(); err != nil {
		return fmt.Errorf("close tx struct: %w", err)
	}
	return nil
}
```

## Impact Details

The function `DecodeRLPTransaction()` has been widely utilized in the Erigon codebase to decode the transactions from block, block body and user’s submitted raw transaction. Since tailing bytes of a transaction is disallowed in other Ethereum clients, it could introduce consensus issues with this flawed decoding function.

## References

* <https://github.com/erigontech/erigon/tree/v2.61.0>

## Proof of Concept

## Proof of Concept

For simplicity, we provide the following unit test to compare two transaction decoding functions, `DecodeRLPTransaction()` vs `UnmarshalTransactionFromBinary()`.

The encoded legacy transaction is `[]byte{248, 99, 128, 2, 1, 148, 9, 94, 123, 174, 166, 166, 199, 196, 194, 223, 235, 151, 126, 250, 195, 38, 175, 85, 45, 135, 128, 134, 97, 98, 99, 100, 101, 102, 37, 160, 142, 183, 96, 239, 234, 152, 124, 9, 98, 209, 245, 242, 175, 209, 122, 221, 51, 54, 23, 237, 233, 142, 83, 7, 50, 17, 119, 99, 15, 34, 57, 125, 160, 21, 141, 44, 195, 195, 220, 247, 64, 91, 79, 6, 37, 93, 226, 96, 69, 227, 240, 8, 168, 169, 112, 118, 124, 14, 250, 73, 19, 190, 7, 104, 57, 1}` that contains the last taling bytes `0x01`.

1. Run the following unit test:

```
func TestDecodeRLPTransactionTailing(t *testing.T) {
	input := []byte{248, 99, 128, 2, 1, 148, 9, 94, 123, 174, 166, 166, 199, 196, 194, 223, 235, 151, 126, 250, 195, 38, 175, 85, 45, 135, 128, 134, 97, 98, 99, 100, 101, 102, 37, 160, 142, 183, 96, 239, 234, 152, 124, 9, 98, 209, 245, 242, 175, 209, 122, 221, 51, 54, 23, 237, 233, 142, 83, 7, 50, 17, 119, 99, 15, 34, 57, 125, 160, 21, 141, 44, 195, 195, 220, 247, 64, 91, 79, 6, 37, 93, 226, 96, 69, 227, 240, 8, 168, 169, 112, 118, 124, 14, 250, 73, 19, 190, 7, 104, 57, 1}


	tx, err := UnmarshalTransactionFromBinary(input, false)
	if err != nil {
		fmt.Printf("Unmarshal Tx with UnmarshalTransactionFromBinary() fails: %+v\n", err)
	} else {
		fmt.Printf("Unmarshal Tx with UnmarshalTransactionFromBinary() succeeds: %+v\n", tx.Hash())
	}


	data := input
	s := rlp.NewStream(bytes.NewReader(data), uint64(len(data)))


	legacyTx, err := DecodeRLPTransaction(s, false)
	if err != nil {
		fmt.Printf("Decode RLP of Tx DecodeRLPTransaction() fails: %+v\n", err)
	} else {
		fmt.Printf("Decode RLP of Tx DecodeRLPTransaction() succeeds: %+v\n", legacyTx.Hash())
	}
}
```

2. The test results show that the decoding of this legacy transaction with tailing bytes succeeds with `DecodeRLPTransaction()` but fails with `UnmarshalTransactionFromBinary()`.

```
=== RUN   TestDecodeRLPTransactionTailing
Unmarshal Tx with UnmarshalTransactionFromBinary() fails: trailing bytes after rlp encoded transaction
Decode RLP of Tx DecodeRLPTransaction() succeeds: 0x74487f78cbc2fd511acd00d2d5ec65d9a19fb8eaad38c33b4e5737d780c7cc60
--- PASS: TestDecodeRLPTransactionTailing (0.00s)
PASS


Process finished with the exit code 0
```
