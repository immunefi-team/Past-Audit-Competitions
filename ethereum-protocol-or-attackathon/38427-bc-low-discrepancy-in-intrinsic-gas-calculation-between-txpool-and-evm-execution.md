# #38427 \[BC-Low] Discrepancy in Intrinsic Gas Calculation between Txpool and EVM Execution

**Submitted on Jan 3rd 2025 at 00:46:07 UTC by @CertiK for** [**Attackathon | Ethereum Protocol**](https://immunefi.com/audit-competition/ethereum-protocol-attackathon)

* **Report ID:** #38427
* **Report Type:** Blockchain/DLT
* **Report severity:** Low
* **Target:** <https://github.com/ledgerwatch/erigon>
* **Impacts:**
  * (Specifications) A bug in specifications with no direct impact on client implementations

## Description

## Brief/Intro

The intrinsic gas amount is the minimal amount of gas required for a transaction calculated solely based on the transaction structure before the actual transaction execution. Both transaction validation in txpool and evm execution enforce the gas limit provided in the transaction larger than the intrinsic gas to be included into a block and execution.

In Erigon (<https://github.com/erigontech/erigon> ), the intrinsic gas amount calculated in txpool is less than the transaction execution for transaction type **AccessListTxType** (**they should be the same**), which could possibly lead to DoS attack with many spam transactions entered into the txpool or user accidentally provides gas limit larger than the intrinsic gas amount in txpool but less than the execution, but the transaction ultimately will not be included in a block for execution.

## Vulnerability Details

Affected Codebase:\
<https://github.com/erigontech/erigon/releases/tag/v3.0.0-alpha7>

The function `validateTx()` is intended to validate the transaction when entering the mempool (txpool):

<https://github.com/erigontech/erigon/blob/v3.0.0-alpha7/txnprovider/txpool/pool.go#L815>

```
func (p *TxPool) validateTx(txn *TxnSlot, isLocal bool, stateCache kvcache.CacheView) txpoolcfg.DiscardReason {
	isShanghai := p.isShanghai() || p.isAgra()
	if isShanghai && txn.Creation && txn.DataLen > fixedgas.MaxInitCodeSize {
		return txpoolcfg.InitCodeTooLarge // EIP-3860
	}
	if txn.Type == BlobTxnType {
		if !p.isCancun() {
			return txpoolcfg.TypeNotActivated
		}
		if txn.Creation {
			return txpoolcfg.InvalidCreateTxn
		}
		blobCount := uint64(len(txn.BlobHashes))
		if blobCount == 0 {
			return txpoolcfg.NoBlobs
		}
		if blobCount > p.maxBlobsPerBlock {
			return txpoolcfg.TooManyBlobs
		}
		equalNumber := len(txn.BlobHashes) == len(txn.Blobs) &&
			len(txn.Blobs) == len(txn.Commitments) &&
			len(txn.Commitments) == len(txn.Proofs)


		if !equalNumber {
			return txpoolcfg.UnequalBlobTxExt
		}


		for i := 0; i < len(txn.Commitments); i++ {
			if libkzg.KZGToVersionedHash(txn.Commitments[i]) != libkzg.VersionedHash(txn.BlobHashes[i]) {
				return txpoolcfg.BlobHashCheckFail
			}
		}


		// https://github.com/ethereum/consensus-specs/blob/017a8495f7671f5fff2075a9bfc9238c1a0982f8/specs/deneb/polynomial-commitments.md#verify_blob_kzg_proof_batch
		kzgCtx := libkzg.Ctx()
		err := kzgCtx.VerifyBlobKZGProofBatch(toBlobs(txn.Blobs), txn.Commitments, txn.Proofs)
		if err != nil {
			return txpoolcfg.UnmatchedBlobTxExt
		}


		if !isLocal && (p.all.blobCount(txn.SenderID)+uint64(len(txn.BlobHashes))) > p.cfg.BlobSlots {
			if txn.Traced {
				p.logger.Info(fmt.Sprintf("TX TRACING: validateTx marked as spamming (too many blobs) idHash=%x slots=%d, limit=%d", txn.IDHash, p.all.count(txn.SenderID), p.cfg.AccountSlots))
			}
			return txpoolcfg.Spammer
		}
		if p.totalBlobsInPool.Load() >= p.cfg.TotalBlobPoolLimit {
			if txn.Traced {
				p.logger.Info(fmt.Sprintf("TX TRACING: validateTx total blobs limit reached in pool limit=%x current blobs=%d", p.cfg.TotalBlobPoolLimit, p.totalBlobsInPool.Load()))
			}
			return txpoolcfg.BlobPoolOverflow
		}
	}


	authorizationLen := len(txn.Authorizations)
	if txn.Type == SetCodeTxnType {
		if !p.isPrague() {
			return txpoolcfg.TypeNotActivated
		}
		if txn.Creation {
			return txpoolcfg.InvalidCreateTxn
		}
		if authorizationLen == 0 {
			return txpoolcfg.NoAuthorizations
		}
	}


	// Drop non-local transactions under our own minimal accepted gas price or tip
	if !isLocal && uint256.NewInt(p.cfg.MinFeeCap).Cmp(&txn.FeeCap) == 1 {
		if txn.Traced {
			p.logger.Info(fmt.Sprintf("TX TRACING: validateTx underpriced idHash=%x local=%t, feeCap=%d, cfg.MinFeeCap=%d", txn.IDHash, isLocal, txn.FeeCap, p.cfg.MinFeeCap))
		}
		return txpoolcfg.UnderPriced
	}
	gas, reason := txpoolcfg.CalcIntrinsicGas(uint64(txn.DataLen), uint64(txn.DataNonZeroLen), uint64(authorizationLen), nil, txn.Creation, true, true, isShanghai)
	if txn.Traced {
		p.logger.Info(fmt.Sprintf("TX TRACING: validateTx intrinsic gas idHash=%x gas=%d", txn.IDHash, gas))
	}
	if reason != txpoolcfg.Success {
		if txn.Traced {
			p.logger.Info(fmt.Sprintf("TX TRACING: validateTx intrinsic gas calculated failed idHash=%x reason=%s", txn.IDHash, reason))
		}
		return reason
	}
	if gas > txn.Gas {
		if txn.Traced {
			p.logger.Info(fmt.Sprintf("TX TRACING: validateTx intrinsic gas > txn.gas idHash=%x gas=%d, txn.gas=%d", txn.IDHash, gas, txn.Gas))
		}
		return txpoolcfg.IntrinsicGas
	}
	if !isLocal && uint64(p.all.count(txn.SenderID)) > p.cfg.AccountSlots {
		if txn.Traced {
			p.logger.Info(fmt.Sprintf("TX TRACING: validateTx marked as spamming idHash=%x slots=%d, limit=%d", txn.IDHash, p.all.count(txn.SenderID), p.cfg.AccountSlots))
		}
		return txpoolcfg.Spammer
	}


	// Check nonce and balance
	senderNonce, senderBalance, _ := p.senders.info(stateCache, txn.SenderID)
	if senderNonce > txn.Nonce {
		if txn.Traced {
			p.logger.Info(fmt.Sprintf("TX TRACING: validateTx nonce too low idHash=%x nonce in state=%d, txn.nonce=%d", txn.IDHash, senderNonce, txn.Nonce))
		}
		return txpoolcfg.NonceTooLow
	}
	// Transactor should have enough funds to cover the costs
	total := requiredBalance(txn)
	if senderBalance.Cmp(total) < 0 {
		if txn.Traced {
			p.logger.Info(fmt.Sprintf("TX TRACING: validateTx insufficient funds idHash=%x balance in state=%d, txn.gas*txn.tip=%d", txn.IDHash, senderBalance, total))
		}
		return txpoolcfg.InsufficientFunds
	}
	return txpoolcfg.Success
}
```

Which calls the function `CalcIntrinsicGas()` to compute the intrinsic gas amount for the transaction.

<https://github.com/erigontech/erigon/blob/v3.0.0-alpha7/txnprovider/txpool/txpoolcfg/txpoolcfg.go#L194>

```
func CalcIntrinsicGas(dataLen, dataNonZeroLen, authorizationsLen uint64, accessList types.AccessList, isContractCreation, isHomestead, isEIP2028, isShanghai bool) (uint64, DiscardReason) {
	// Set the starting gas for the raw transaction
	var gas uint64
	if isContractCreation && isHomestead {
		gas = fixedgas.TxGasContractCreation
	} else {
		gas = fixedgas.TxGas
	}
	// Bump the required gas by the amount of transactional data
	if dataLen > 0 {
		// Zero and non-zero bytes are priced differently
		nz := dataNonZeroLen
		// Make sure we don't exceed uint64 for all data combinations
		nonZeroGas := fixedgas.TxDataNonZeroGasFrontier
		if isEIP2028 {
			nonZeroGas = fixedgas.TxDataNonZeroGasEIP2028
		}

		product, overflow := emath.SafeMul(nz, nonZeroGas)
		if overflow {
			return 0, GasUintOverflow
		}
		gas, overflow = emath.SafeAdd(gas, product)
		if overflow {
			return 0, GasUintOverflow
		}

		z := dataLen - nz

		product, overflow = emath.SafeMul(z, fixedgas.TxDataZeroGas)
		if overflow {
			return 0, GasUintOverflow
		}
		gas, overflow = emath.SafeAdd(gas, product)
		if overflow {
			return 0, GasUintOverflow
		}

		if isContractCreation && isShanghai {
			numWords := toWordSize(dataLen)
			product, overflow = emath.SafeMul(numWords, fixedgas.InitCodeWordGas)
			if overflow {
				return 0, GasUintOverflow
			}
			gas, overflow = emath.SafeAdd(gas, product)
			if overflow {
				return 0, GasUintOverflow
			}
		}
	}
	if accessList != nil {
		product, overflow := emath.SafeMul(uint64(len(accessList)), fixedgas.TxAccessListAddressGas)
		if overflow {
			return 0, GasUintOverflow
		}
		gas, overflow = emath.SafeAdd(gas, product)
		if overflow {
			return 0, GasUintOverflow
		}

		product, overflow = emath.SafeMul(uint64(accessList.StorageKeys()), fixedgas.TxAccessListStorageKeyGas)
		if overflow {
			return 0, GasUintOverflow
		}
		gas, overflow = emath.SafeAdd(gas, product)
		if overflow {
			return 0, GasUintOverflow
		}
	}

	// Add the cost of authorizations
	product, overflow := emath.SafeMul(authorizationsLen, fixedgas.PerEmptyAccountCost)
	if overflow {
		return 0, GasUintOverflow
	}

	gas, overflow = emath.SafeAdd(gas, product)
	if overflow {
		return 0, GasUintOverflow
	}

	return gas, Success
}
```

In case that the gas limit provided in the transaction is less than the intrinsic gas amount, the transaction will be discarded immediately.

However, the inputs of the function miss the accesslist of the transaction:

```
gas, reason := txpoolcfg.CalcIntrinsicGas(uint64(txn.DataLen), uint64(txn.DataNonZeroLen), uint64(authorizationLen), nil, txn.Creation, true, true, isShanghai)
```

During the execution of the transaction, a similar computation is performed within in function `IntrinsicGas()` :

<https://github.com/erigontech/erigon/blob/v3.0.0-alpha7/core/state\\_transition.go#L111C1-L126C2>

```
func IntrinsicGas(data []byte, accessList types.AccessList, isContractCreation bool, isHomestead, isEIP2028, isEIP3860 bool, authorizationsLen uint64) (uint64, error) {
	// Zero and non-zero bytes are priced differently
	dataLen := uint64(len(data))
	dataNonZeroLen := uint64(0)
	for _, byt := range data {
		if byt != 0 {
			dataNonZeroLen++
		}
	}

	gas, status := txpoolcfg.CalcIntrinsicGas(dataLen, dataNonZeroLen, authorizationsLen, accessList, isContractCreation, isHomestead, isEIP2028, isEIP3860)
	if status != txpoolcfg.Success {
		return 0, ErrGasUintOverflow
	}
	return gas, nil
}
```

In this case, the access list of the transaction is provided for the intrinsic gas. Consequently, the intrinsic gas of a transaction with access list calculated in the txpool (mempool) is less than that in the execution

<https://github.com/erigontech/erigon/blob/v3.0.0-alpha7/core/state\\_transition.go#L463>

```
	// Check clauses 4-5, subtract intrinsic gas if everything is correct
	gas, err := IntrinsicGas(st.data, accessTuples, contractCreation, rules.IsHomestead, rules.IsIstanbul, isEIP3860, uint64(len(auths)))
	if err != nil {
		return nil, err
	}
	if st.gasRemaining < gas {
		return nil, fmt.Errorf("%w: have %d, want %d", ErrIntrinsicGas, st.gasRemaining, gas)
	}
```

This would allow spam transactions to flood into the txpool (mempool) with a gas limit larger than the intrinsic gas in mempool but less than the execution, which could potentially lead to DoS attack to the txpool.

## Impact Details

The difference in the intrinsic gas computation in the txpool (mempool) and execution would allow spam transactions to flood into the mempool, which could potentially lead to DoS attack to the txpool (mempool).

## References

* <https://github.com/erigontech/erigon/releases/tag/v3.0.0-alpha7>

## Proof of Concept

## Proof of Concept

We create the following unit test to show that the intrinsic gas computed in txpool is less than the intrinsic gas in execution. Moreover, the gas limit lies in between them.

1. Create a transaction of type AccessListTxType of non-empty access list and compute the intrinsic gas in txpool and execution:

```
package core

import (
	"bytes"
	"fmt"
	libcommon "github.com/erigontech/erigon-lib/common"
	"github.com/erigontech/erigon-lib/crypto"
	"github.com/erigontech/erigon/core/types"
	"github.com/erigontech/erigon/txnprovider/txpool/txpoolcfg"
	"github.com/holiman/uint256"
	"testing"
)

func TestIntrinsicGasCalculation(t *testing.T) {
	var (
		signer    = types.LatestSignerForChainID(libcommon.Big1)
		addr      = libcommon.HexToAddress("0x0000000000000000000000000000000000000001")
		recipient = libcommon.HexToAddress("095e7baea6a6c7c4c2dfeb977efac326af552d87")
		accesses  = types.AccessList{{Address: addr, StorageKeys: []libcommon.Hash{{0}}}}
	)

	key, err := crypto.GenerateKey()
	if err != nil {
		t.Fatalf("could not generate key: %v", err)
	}
	txdata := &types.AccessListTx{
		ChainID: uint256.NewInt(1),
		LegacyTx: types.LegacyTx{
			CommonTx: types.CommonTx{
				To:    &recipient,
				Nonce: 1,
				Gas:   25000,
			},
			GasPrice: uint256.NewInt(10),
		},
		AccessList: accesses,
	}
	tx, err := types.SignNewTx(key, *signer, txdata)
	if err != nil {
		t.Fatalf("could not sign transaction: %v", err)
	}
	// RLP
	parsedTx, err := encodeDecodeBinary(tx)

	data := parsedTx.GetData()
	dataLen := uint64(len(data))
	dataNonZeroLen := uint64(0)
	for _, byt := range data {
		if byt != 0 {
			dataNonZeroLen++
		}
	}
	// access list is nil in txpool
	txpoolIntrinsicGas, _ := txpoolcfg.CalcIntrinsicGas(dataLen, dataNonZeroLen, 0, nil, false, true, true, true)
	// access list is not nil in execution
	executionIntrinsicGas, _ := IntrinsicGas(parsedTx.GetData(), parsedTx.GetAccessList(), false, true, true, true, 0)

	fmt.Printf("txpoolIntrinsicGas %d is less than executionIntrinsicGas %d: %t\n", txpoolIntrinsicGas, executionIntrinsicGas, txpoolIntrinsicGas < executionIntrinsicGas)
	fmt.Printf("txpoolIntrinsicGas %d is less than gas limit  %d: %t\n", txpoolIntrinsicGas, parsedTx.GetGas(), txpoolIntrinsicGas < parsedTx.GetGas())
	fmt.Printf("gas limit %d is less than executionIntrinsicGas %d: %t\n", parsedTx.GetGas(), executionIntrinsicGas, parsedTx.GetGas() < executionIntrinsicGas)
}

func encodeDecodeBinary(tx types.Transaction) (types.Transaction, error) {
	var buf bytes.Buffer
	var err error
	if err = tx.MarshalBinary(&buf); err != nil {
		return nil, fmt.Errorf("rlp encoding failed: %w", err)
	}
	var parsedTx types.Transaction
	if parsedTx, err = types.UnmarshalTransactionFromBinary(buf.Bytes(), false /* blobTxnsAreWrappedWithBlobs */); err != nil {
		return nil, fmt.Errorf("rlp decoding failed: %w", err)
	}
	return parsedTx, nil
}
```

2. The test result shows that the intrinsic gas in txpool is less than that in execution.

```
=== RUN   TestIntrinsicGasCalculation
txpoolIntrinsicGas 21000 is less than executionIntrinsicGas 25300: true
txpoolIntrinsicGas 21000 is less than gas limit  25000: true
gas limit 25000 is less than executionIntrinsicGas 25300: true
--- PASS: TestIntrinsicGasCalculation (0.00s)
PASS


Process finished with the exit code 0
```
