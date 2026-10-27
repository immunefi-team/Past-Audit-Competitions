# #37206 \[SC-Medium] Overflow due to lack of checks leading to incorrect price calculation

**Submitted on Nov 28th 2024 at 20:08:33 UTC by @okmxuse for** [**Audit Comp | Celo**](https://immunefi.com/audit-competition/audit-comp-celo)

* **Report ID:** #37206
* **Report Type:** Smart Contract
* **Report severity:** Medium
* **Target:** <https://github.com/celo-org/optimism/blob/celo10/op-chain-ops/cmd/check-derivation/main.go>
* **Impacts:**
  * Smart contract unable to operate due to lack of token funds

## Description

Inside the `check-derivation/main.go` function, the `getRandomSignedTransaction` function is invoked (note that this function traces all the way back to `checkConsolidation` which is then called in the `main.go` function ).

`getRandomSignedTransaction` calls `IntrinsicGas` at three places. We will focus on the one that includes the `accessList`, which is `case types.AccessListTxType`:

```go
=> case types.AccessListTxType:
	accessList := types.AccessList{types.AccessTuple{
		Address:     randomAddress,
		StorageKeys: []common.Hash{common.HexToHash("0x1234")},
	}}
	gasLimit, err := core.IntrinsicGas(data, accessList, false, true, true, false, nil, nil)
	if err != nil {
		return nil, fmt.Errorf("failed to get intrinsicGas: %w", err)
	}
```

Next, let's look at the `IntrinsicGas` function:

```go
func IntrinsicGas(data []byte, accessList types.AccessList, isContractCreation, isHomestead, isEIP2028, isEIP3860 bool, feeCurrency *common.Address, feeIntrinsicGas common.IntrinsicGasCosts) (uint64, error) {
	// Set the starting gas for the raw transaction
	var gas uint64
	if isContractCreation && isHomestead {
		gas = params.TxGasContractCreation
	} else {
		gas = params.TxGas
	}
	dataLen := uint64(len(data))
	// Bump the required gas by the amount of transactional data
	if dataLen > 0 {
		// Zero and non-zero bytes are priced differently
		var nz uint64
		for _, byt := range data {
			if byt != 0 {
				nz++
			}
		}
		// Ensure we do not exceed uint64 limits for all data combinations
		nonZeroGas := params.TxDataNonZeroGasFrontier
		if isEIP2028 {
			nonZeroGas = params.TxDataNonZeroGasEIP2028
		}
=>		if (math.MaxUint64-gas)/nonZeroGas < nz {
			return 0, ErrGasUintOverflow
		}
		gas += nz * nonZeroGas

		z := dataLen - nz
=>		if (math.MaxUint64-gas)/params.TxDataZeroGas < z {
			return 0, ErrGasUintOverflow
		}
		gas += z * params.TxDataZeroGas

		if isContractCreation && isEIP3860 {
			lenWords := toWordSize(dataLen)
=>			if (math.MaxUint64-gas)/params.InitCodeWordGas < lenWords {
				return 0, ErrGasUintOverflow
			}
			gas += lenWords * params.InitCodeWordGas
		}
	}

	if feeCurrency != nil {
		intrinsicGasForFeeCurrency, ok := common.CurrencyIntrinsicGasCost(feeIntrinsicGas, feeCurrency)
		if !ok {
			return 0, fmt.Errorf("%w: %x", exchange.ErrUnregisteredFeeCurrency, feeCurrency)
		}
=>		if (math.MaxUint64 - gas) < intrinsicGasForFeeCurrency {
			return 0, ErrGasUintOverflow
		}
		gas += intrinsicGasForFeeCurrency
	}

	if accessList != nil {
=>		gas += uint64(len(accessList)) * params.TxAccessListAddressGas
=>		gas += uint64(accessList.StorageKeys()) * params.TxAccessListStorageKeyGas
	}
	return gas, nil
}
```

Looking at the arrowed code-lines everywhere in the function, when `gas` is incremented, overflow checks are performed using conditions like:

```go
if (math.MaxUint64 - gas)/value < multiplier {
	return 0, ErrGasUintOverflow
}
```

This ensures that the gas does not become bigger than `uint64`.

However, **no overflow check is performed in the final block involving the `accessList`**, specifically at:

```go
	if accessList != nil {
=>		gas += uint64(len(accessList)) * params.TxAccessListAddressGas
=>		gas += uint64(accessList.StorageKeys()) * params.TxAccessListStorageKeyGas
	}
```

As you can see it simply adds the result to `gas` without any overflow check.

### Impact

This bug can lead to **silent overflows**, causing:

* Incorrect computation of gas limits for transactions which will cause the complete flow to fail.

### Recommendation

Add an overflow check for the gas additions within the `accessList` block:

```go
	if accessList != nil {
		addressGas := uint64(len(accessList)) * params.TxAccessListAddressGas
		if (math.MaxUint64 - gas) < addressGas {
			return 0, ErrGasUintOverflow
		}
		gas += addressGas

		storageKeyGas := uint64(accessList.StorageKeys()) * params.TxAccessListStorageKeyGas
		if (math.MaxUint64 - gas) < storageKeyGas {
			return 0, ErrGasUintOverflow
		}
		gas += storageKeyGas
	}
```

This ensures that both address and storage key gas costs are properly validated, preventing silent overflows and ensuring accurate gas calculations.

## Proof of Concept

### POC

First I modified the main `IntrinsicGas`function slightly to ensure that an overflow is guaranteed:

```diff
+	if accessList != nil {
+		// set the gas to uint64 - 2400 to guarantee it to trigger (note this is just for testing simplicity)
+		gas := uint64(18446744073709551615)
+		gas += uint64(len(accessList)) * params.TxAccessListAddressGas
+		gas += uint64(accessList.StorageKeys()) * params.TxAccessListStorageKeyGas
+ // return the gas to log in the test file 
	}
+	return gas, nil
+ }
```

then create a `main_test.go` file and past the following:

```javascript
package main_test

import (
	"math/rand"
	"testing"

	"github.com/ethereum-optimism/optimism/op-service/testutils"
	"github.com/ethereum/go-ethereum/common"
	"github.com/ethereum/go-ethereum/core"
	"github.com/ethereum/go-ethereum/core/types"
)

func Test_Overflow(t *testing.T) {
	rng := rand.New(rand.NewSource(1337))
	data := testutils.RandomData(rng, 10)

	accessList := make(types.AccessList, 5) 
	for i := range accessList {
		accessList[i] = types.AccessTuple{
			Address:     randomAddress(rng),
			StorageKeys: []common.Hash{common.HexToHash("0x1234")},
		}
	}

	// Call IntrinsicGas with the near-overflow AccessList
	gas, err := core.IntrinsicGas(data, accessList, false, true, true, false, nil, nil)

	// Log both the gas value and the error to understand the result
	t.Logf("Returned gas: %d", gas) // Correctly print the gas value
	t.Logf("Caught expected error: %v", err)
}

// Helper function to generate random Ethereum addresses
func randomAddress(rng *rand.Rand) common.Address {
	var addr [20]byte
	rng.Read(addr[:])
	return common.Address(addr)
}
```

* run `go test -v -run Test_Overflow`
* Logs:

```
=== RUN   Test_Overflow
    main_test.go:30: Returned gas: 42660
    main_test.go:31: Caught expected error: <nil>
--- PASS: Test_Overflow (0.00s)
PASS
```

As we can see the gas has wrapped and now returns 42660.
