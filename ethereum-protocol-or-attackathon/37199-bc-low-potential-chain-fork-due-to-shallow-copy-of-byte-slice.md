# #37199 \[BC-Low] Potential Chain Fork Due to Shallow Copy of Byte Slice

**Submitted on Nov 28th 2024 at 17:16:42 UTC by @CertiK for** [**Attackathon | Ethereum Protocol**](https://immunefi.com/audit-competition/ethereum-protocol-attackathon)

* **Report ID:** #37199
* **Report Type:** Blockchain/DLT
* **Report severity:** Low
* **Target:** <https://github.com/ledgerwatch/erigon>
* **Impacts:**
  * Unintended chain split affecting less than 25% of the network (Network partition)

## Description

## Brief/Intro

A potential chain fork has been discovered in the Ethereum client Erigon ( <https://github.com/erigontech/erigon> ) due to the shallow copy of the byte slice in precompile contract dataCopy.

## Vulnerability Details

The issue outlined in this report pertains to the precompile contract dataCopy, detailed as follows:

Affected Codebase: <https://github.com/erigontech/erigon/tree/v2.61.0-beta1>

The precompile contract dataCopy is utilized to copy the input (byte slice):

<https://github.com/erigontech/erigon/blob/v2.61.0-beta1/core/vm/contracts.go#L303>

```
// data copy implemented as a native contract.
type dataCopy struct{}


// RequiredGas returns the gas required to execute the pre-compiled contract.
//
// This method does not require any overflow checking as the input size gas costs
// required for anything significant is so high it's impossible to pay for.
func (c *dataCopy) RequiredGas(input []byte) uint64 {
   return uint64(len(input)+31)/32*params.IdentityPerWordGas + params.IdentityBaseGas
}
func (c *dataCopy) Run(in []byte) ([]byte, error) {
   return in, nil
}
```

Which directly returns the input as a shallow copy of the input, which does not align with other Ethereum clients, for example, in Go Ethereum:

<https://github.com/ethereum/go-ethereum/blob/v1.14.12/core/vm/contracts.go#L315>

```
func (c *dataCopy) Run(in []byte) ([]byte, error) {
	return common.CopyBytes(in), nil
}
```

It performs a deep copy of the byte slice

<https://github.com/ethereum/go-ethereum/blob/v1.14.12/common/bytes.go#L40>

```
// CopyBytes returns an exact copy of the provided bytes.
func CopyBytes(b []byte) (copiedBytes []byte) {
   if b == nil {
      return nil
   }
   copiedBytes = make([]byte, len(b))
   copy(copiedBytes, b)


   return
}
```

Deep copy is also applied in REVM (<https://github.com/bluealloy/revm>), which is used in the Reth Ethereum client:

<https://github.com/bluealloy/revm/blob/main/crates/precompile/src/identity.rs#L19>

```
pub fn identity_run(input: &Bytes, gas_limit: u64) -> PrecompileResult {
    let gas_used = calc_linear_cost_u32(input.len(), IDENTITY_BASE, IDENTITY_PER_WORD);
    if gas_used > gas_limit {
        return Err(PrecompileError::OutOfGas.into());
    }
    Ok(PrecompileOutput::new(gas_used, input.clone()))
}
```

This discrepancy in implementation could lead to chain fork as observed in go-ethereum clients in the post mortem: <https://gist.github.com/karalabe/e1891c8a99fdc16c4e60d9713c35401f>

## Impact Details

This discrepancy in implementation of shallow copy and deep copy could lead to chain fork.

## References

* <https://gist.github.com/karalabe/e1891c8a99fdc16c4e60d9713c35401f>
* <https://github.com/erigontech/erigon>
* <https://github.com/bluealloy/revm>

## Proof of Concept

## Proof of Concept

This attack scenario has been observed in two opcodes RETURNDATASIZE and RETURNDATACOPY in go-ethereum as described in the post mortem: <https://gist.github.com/karalabe/e1891c8a99fdc16c4e60d9713c35401f>

Here we provide a unit test to show the difference between the implementation of data copy precompile contract with shallow copy and deep copy:

```
package vm

import (
	"bytes"
	"encoding/json"
	"fmt"
	"os"
	"testing"
	"time"

	libcommon "github.com/ledgerwatch/erigon-lib/common"

	"github.com/ledgerwatch/erigon/common"
)

func CopyBytes(b []byte) (copiedBytes []byte) {
	if b == nil {
		return nil
	}
	copiedBytes = make([]byte, len(b))
	copy(copiedBytes, b)

	return
}

func TestDatacopyPrecompile(t *testing.T) {
	dataCopyContract := PrecompiledContractsCancun[libcommon.BytesToAddress([]byte{0x04})]

	input1 := []byte{0x01, 0x02, 0x03, 0x04}
	maxGas := uint64(10000)

	//////////////shallow copy/////////

	output1, suppliedGas, err := RunPrecompiledContract(dataCopyContract, input1, maxGas)

	if err == nil {
		fmt.Printf("Output is: %x, supplied gas is: %d\n", output1, suppliedGas)
	}

	output1[0] = 0xff
	fmt.Println("//////////////shallow copy/////////")
	fmt.Printf("Input is changed from %x, to: %x\n", input1, output1)
	fmt.Printf("Output is: %x\n", output1)

	//////////////deep copy/////////
	input2 := []byte{0x01, 0x02, 0x03, 0x04}
	output2 := CopyBytes(input2)
	output2[0] = 0xff
	fmt.Println("//////////////deep copy/////////")
	fmt.Printf("Input is changed from %x, to: %x\n", input2, output2)
	fmt.Printf("Output is: %x\n", output2)
}
```

Test result:

```
=== RUN   TestDatacopyPrecompile
Output is: 01020304, supplied gas is: 9982
//////////////shallow copy/////////
Input is changed from ff020304, to: ff020304
Output is: ff020304
//////////////deep copy/////////
Input is changed from 01020304, to: ff020304
Output is: ff020304
--- PASS: TestDatacopyPrecompile (0.00s)
PASS
```

The result shows that the shallow copy modifies the original input while deep copy does not.

Since Erigon uses the shallow copy in the data copy precompile contract, once the input is modified, it would lead to inconsistent data with other Ethereum clients, potentially lead to chain fork/split.
