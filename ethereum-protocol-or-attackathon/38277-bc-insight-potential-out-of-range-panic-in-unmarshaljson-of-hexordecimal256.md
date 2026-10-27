# #38277 \[BC-Insight] Potential Out-of-Range Panic in \`UnmarshalJSON()\` of \`HexOrDecimal256\`

**Submitted on Dec 29th 2024 at 23:39:47 UTC by @CertiK for** [**Attackathon | Ethereum Protocol**](https://immunefi.com/audit-competition/ethereum-protocol-attackathon)

* **Report ID:** #38277
* **Report Type:** Blockchain/DLT
* **Report severity:** Insight
* **Target:** <https://github.com/ledgerwatch/erigon>
* **Impacts:**
  * (Specifications) A bug in specifications with no direct impact on client implementations

## Description

## Brief/Intro

In the erigon-lib/common/math package, the type `HexOrDecimal256` marshals big.Int into hex or decimal strings. Due to a mishandling of the slice index in the method `UnmarshalJSON()`, it would possibly lead to an out-of-range panic if this method is invoked with a certain value.

## Vulnerability Details

Affected Codebase: <https://github.com/erigontech/erigon/tree/v3.0.0-alpha7>

The function `UnmarshalJSON()` is utilized to parse hex or decimal string into big.Int.

<https://github.com/erigontech/erigon/blob/v3.0.0-alpha7/erigon-lib/common/math/big.go#L61>

```
func (i *HexOrDecimal256) UnmarshalJSON(input []byte) error {
	if len(input) > 0 && input[0] == '"' {
		input = input[1 : len(input)-1]
	}
	return i.UnmarshalText(input)
}
```

The condition `len(input) > 0 && input[0] == ' " '` is intended to ignore the first quote ' " ' if it exists as the first element of the input.

However, this check is not effective. In case that the input only contains the quote ' " ' , then the input length is 1, so the condition is satisfied, which leads to the out-of-range panic when taking the slice `input[1:0]`.

In fact, the check should be `len(input) > 1 && input[0] == ' " '` .

## Impact Details

Though the type HexOrDecimal256 has been utilized in multiple places of the current codebase, for example,

<https://github.com/erigontech/erigon/blob/v3.0.0-alpha7/core/blockchain.go#L71>

```
type EphemeralExecResult struct {
	StateRoot        libcommon.Hash        `json:"stateRoot"`
	TxRoot           libcommon.Hash        `json:"txRoot"`
	ReceiptRoot      libcommon.Hash        `json:"receiptsRoot"`
	LogsHash         libcommon.Hash        `json:"logsHash"`
	Bloom            types.Bloom           `json:"logsBloom"        gencodec:"required"`
	Receipts         types.Receipts        `json:"receipts"`
	Rejected         RejectedTxs           `json:"rejected,omitempty"`
	Difficulty       *math.HexOrDecimal256 `json:"currentDifficulty" gencodec:"required"`
	GasUsed          math.HexOrDecimal64   `json:"gasUsed"`
	StateSyncReceipt *types.Receipt        `json:"-"`
}
```

we are not aware of the potential attack vector and it may not be exploitable at this moment. Due to the potential node crash if it’s triggered implicitly or by future update, it’s recommended to fix it.

## References

* <https://github.com/erigontech/erigon/tree/v3.0.0-alpha7>

## Proof of Concept

## Proof of Concept

We provide the following simple test case by setting the input as the quote ' " ' .

```
package math


import (
	"bytes"
	"encoding/hex"
	"math/big"
	"testing"


	"github.com/erigontech/erigon-lib/common"
)


func TestHexOrDecimal256UnmarshalJSON(t *testing.T) {
	input := []byte{'"'}
	var num HexOrDecimal256
	_ = num.UnmarshalJSON(input)
}
```

The test result shows the out-of-range panic could be triggered in the method `UnmarshalJSON()` with input ' " ' .

```
=== RUN   TestHexOrDecimal256UnmarshalJSON
--- FAIL: TestHexOrDecimal256UnmarshalJSON (0.00s)
panic: runtime error: slice bounds out of range [1:0] [recovered]
	panic: runtime error: slice bounds out of range [1:0]


goroutine 21 [running]:
testing.tRunner.func1.2({0x5a8b880, 0xc0000b8198})
	/Users/***/go/pkg/mod/golang.org/toolchain@v0.0.1-go1.23.0.darwin-amd64/src/testing/testing.go:1632 +0x230
testing.tRunner.func1()
	/Users/***/go/pkg/mod/golang.org/toolchain@v0.0.1-go1.23.0.darwin-amd64/src/testing/testing.go:1635 +0x35e
panic({0x5a8b880?, 0xc0000b8198?})
	/Users/***/go/pkg/mod/golang.org/toolchain@v0.0.1-go1.23.0.darwin-amd64/src/runtime/panic.go:785 +0x132
github.com/erigontech/erigon-lib/common/math.(*HexOrDecimal256).UnmarshalJSON(0xc000055f50?, {0xc00009ab28?, 0x151219ae47ae7?, 0x2a697f88?})
	/Users/***/immunefi/erigon/erigon-lib/common/math/big.go:63 +0x48
github.com/erigontech/erigon-lib/common/math.TestHexOrDecimal256UnmarshalJSON(0xc0000bc9c0?)
	/Users/***/immunefi/erigon/erigon-lib/common/math/big_test.go:34 +0x46
testing.tRunner(0xc0000bc9c0, 0x5aa1418)
	/Users/***/go/pkg/mod/golang.org/toolchain@v0.0.1-go1.23.0.darwin-amd64/src/testing/testing.go:1690 +0xf4
created by testing.(*T).Run in goroutine 1
	/Users/***/go/pkg/mod/golang.org/toolchain@v0.0.1-go1.23.0.darwin-amd64/src/testing/testing.go:1743 +0x390


Process finished with the exit code 1
```
