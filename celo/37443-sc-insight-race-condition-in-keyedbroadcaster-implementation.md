# #37443 \[SC-Insight] Race Condition in KeyedBroadcaster Implementation

**Submitted on Dec 4th 2024 at 20:30:36 UTC by @jovi for** [**Audit Comp | Celo**](https://immunefi.com/audit-competition/audit-comp-celo)

* **Report ID:** #37443
* **Report Type:** Smart Contract
* **Report severity:** Insight
* **Target:** <https://github.com/celo-org/optimism/blob/celo10/op-chain-ops/deployer/broadcaster/keyed.go>
* **Impacts:**
  * L2 re-org
  * Node clients may operate in non-deterministic manners
  * Increased node latency

## Description

## Brief/Intro

The current implementation of the KeyedBroadcaster lacks proper synchronization mechanisms, specifically mutex locks, when accessing and modifying the `bcasts` slice. This oversight can lead to data corruption, inconsistent state, and potential security vulnerabilities in the transaction broadcasting process.

## Vulnerability Details

The `KeyedBroadcaster` struct in `optimism/op-chain-ops/deployer/broadcaster/keyed.go` is responsible for managing and broadcasting transactions. It contains a slice `bcasts` that stores pending broadcasts. The current implementation allows concurrent access to this slice without proper synchronization. Differently from Optimism's implementation at [optimism/op-deployer/pkg/deployer/broadcaster/keyed.go at 4ee839ae8996c2d421a2d85fd5471897840014fa · ethereum-optimism/optimism](https://github.com/ethereum-optimism/optimism/blob/4ee839ae8996c2d421a2d85fd5471897840014fa/op-deployer/pkg/deployer/broadcaster/keyed.go#L103C2-L106C16)

Key vulnerable points:

1. The `Hook` method appends to the `bcasts` slice without any synchronization:

```go
func (b *KeyedBroadcaster) Hook(bcast script.Broadcast) {
    b.bcasts = append(b.bcasts, bcast)
}
```

2. The `Broadcast` method reads and modifies the `bcasts` slice without synchronization:

```go
func (b *KeyedBroadcaster) Broadcast(ctx context.Context) ([]BroadcastResult, error) {
    // ... (code that reads and modifies b.bcasts)
}
```

These operations are not thread-safe and can lead to race conditions when multiple goroutines access the `KeyedBroadcaster` concurrently.

In contrast, the Optimism implementation uses mutex locks to ensure thread-safety:

```go
func (b *KeyedBroadcaster) Hook(bcast script.Broadcast) {
    b.mu.Lock()
    defer b.mu.Unlock()
    b.bcasts = append(b.bcasts, bcast)
}
```

The lack of such synchronization in the Celo implementation exposes the system to race conditions.

## Impact Details

The race condition in the `KeyedBroadcaster` can have the following consequences:

1. **Data Corruption**: Concurrent modifications to the `bcasts` slice can lead to corrupted data, potentially resulting in invalid transactions being broadcasted or valid transactions being omitted.
2. **Inconsistent State**: The state of the `bcasts` slice may become inconsistent across different goroutines, leading to missed broadcasts or duplicate broadcasts.
3. **Transaction Failures**: Incorrect nonce values or gas limits may be used due to race conditions, causing transactions to fail.
4. **Increased Latency**: The system may experience increased latency due to retries and error handling necessitated by failed or inconsistent transactions.
5. **Unreliable System Behavior**: The non-deterministic nature of race conditions can make the system behave unpredictably, making it difficult to debug and maintain.

## Proof of Concept

## Proof of concept

In order to test the race condition aforementioned, paste the following code snippet at the keyed.go file at op-chain-ops/deployer/broadcast.

```go
package broadcaster

import (
	"context"
	"fmt"
	"math/big"
	"time"

	"github.com/ethereum-optimism/optimism/op-chain-ops/script"
	opcrypto "github.com/ethereum-optimism/optimism/op-service/crypto"
	"github.com/ethereum-optimism/optimism/op-service/eth"
	"github.com/ethereum-optimism/optimism/op-service/txmgr"
	"github.com/ethereum-optimism/optimism/op-service/txmgr/metrics"
	"github.com/ethereum/go-ethereum/common"
	"github.com/ethereum/go-ethereum/core"
	"github.com/ethereum/go-ethereum/ethclient"
	"github.com/ethereum/go-ethereum/log"
	"github.com/hashicorp/go-multierror"
	"github.com/holiman/uint256"
)

const (
	GasPadFactor = 2.0
)

type KeyedBroadcaster struct {
	lgr    log.Logger
	mgr    txmgr.TxManager
	bcasts []script.Broadcast
	client *ethclient.Client
}

type KeyedBroadcasterOpts struct {
	Logger          log.Logger
	ChainID         *big.Int
	Client          *ethclient.Client
	Signer          opcrypto.SignerFn
	From            common.Address
	TXManagerLogger log.Logger
}

func NewKeyedBroadcaster(cfg KeyedBroadcasterOpts) (*KeyedBroadcaster, error) {
	mgrCfg := &txmgr.Config{
		Backend:                   cfg.Client,
		ChainID:                   cfg.ChainID,
		TxSendTimeout:             5 * time.Minute,
		TxNotInMempoolTimeout:     time.Minute,
		NetworkTimeout:            10 * time.Second,
		ReceiptQueryInterval:      time.Second,
		NumConfirmations:          1,
		SafeAbortNonceTooLowCount: 3,
		Signer:                    cfg.Signer,
		From:                      cfg.From,
	}

	minTipCap, err := eth.GweiToWei(1.0)
	if err != nil {
		panic(err)
	}
	minBaseFee, err := eth.GweiToWei(1.0)
	if err != nil {
		panic(err)
	}

	mgrCfg.ResubmissionTimeout.Store(int64(48 * time.Second))
	mgrCfg.FeeLimitMultiplier.Store(5)
	mgrCfg.FeeLimitThreshold.Store(big.NewInt(100))
	mgrCfg.MinTipCap.Store(minTipCap)
	mgrCfg.MinTipCap.Store(minBaseFee)

	txmLogger := log.NewLogger(log.DiscardHandler())
	if cfg.TXManagerLogger != nil {
		txmLogger = cfg.TXManagerLogger
	}

	mgr, err := txmgr.NewSimpleTxManagerFromConfig(
		"transactor",
		txmLogger,
		&metrics.NoopTxMetrics{},
		mgrCfg,
	)

	if err != nil {
		return nil, fmt.Errorf("failed to create tx manager: %w", err)
	}

	return &KeyedBroadcaster{
		lgr:    cfg.Logger,
		mgr:    mgr,
		client: cfg.Client,
	}, nil
}

func (t *KeyedBroadcaster) Hook(bcast script.Broadcast) {
	t.bcasts = append(t.bcasts, bcast)
}

func (t *KeyedBroadcaster) Broadcast(ctx context.Context) ([]BroadcastResult, error) {
	results := make([]BroadcastResult, len(t.bcasts))
	futures := make([]<-chan txmgr.SendResponse, len(t.bcasts))
	ids := make([]common.Hash, len(t.bcasts))

	latestBlock, err := t.client.BlockByNumber(ctx, nil)
	if err != nil {
		return nil, fmt.Errorf("failed to get latest block: %w", err)
	}

	for i, bcast := range t.bcasts {
		futures[i], ids[i] = t.broadcast(ctx, bcast, latestBlock.GasLimit())
		t.lgr.Info(
			"transaction broadcasted",
			"id", ids[i],
			"nonce", bcast.Nonce,
		)
	}

	var txErr *multierror.Error
	var completed int
	for i, fut := range futures {
		bcastRes := <-fut
		completed++
		outRes := BroadcastResult{
			Broadcast: t.bcasts[i],
		}

		if bcastRes.Err == nil {
			outRes.Receipt = bcastRes.Receipt
			outRes.TxHash = bcastRes.Receipt.TxHash

			if bcastRes.Receipt.Status == 0 {
				failErr := fmt.Errorf("transaction failed: %s", outRes.Receipt.TxHash.String())
				txErr = multierror.Append(txErr, failErr)
				outRes.Err = failErr
				t.lgr.Error(
					"transaction failed on chain",
					"id", ids[i],
					"completed", completed,
					"total", len(t.bcasts),
					"hash", outRes.Receipt.TxHash.String(),
					"nonce", outRes.Broadcast.Nonce,
				)
			} else {
				t.lgr.Info(
					"transaction confirmed",
					"id", ids[i],
					"completed", completed,
					"total", len(t.bcasts),
					"hash", outRes.Receipt.TxHash.String(),
					"nonce", outRes.Broadcast.Nonce,
					"creation", outRes.Receipt.ContractAddress,
				)
			}
		} else {
			txErr = multierror.Append(txErr, bcastRes.Err)
			outRes.Err = bcastRes.Err
			t.lgr.Error(
				"transaction failed",
				"id", ids[i],
				"completed", completed,
				"total", len(t.bcasts),
				"err", bcastRes.Err,
			)
		}

		results[i] = outRes
	}
	return results, txErr.ErrorOrNil()
}

func (t *KeyedBroadcaster) broadcast(ctx context.Context, bcast script.Broadcast, blockGasLimit uint64) (<-chan txmgr.SendResponse, common.Hash) {
	ch := make(chan txmgr.SendResponse, 1)

	id := bcast.ID()
	value := ((*uint256.Int)(bcast.Value)).ToBig()
	var candidate txmgr.TxCandidate
	switch bcast.Type {
	case script.BroadcastCall:
		to := &bcast.To
		candidate = txmgr.TxCandidate{
			TxData:   bcast.Input,
			To:       to,
			Value:    value,
			GasLimit: padGasLimit(bcast.Input, bcast.GasUsed, false, blockGasLimit),
		}
	case script.BroadcastCreate:
		candidate = txmgr.TxCandidate{
			TxData:   bcast.Input,
			To:       nil,
			GasLimit: padGasLimit(bcast.Input, bcast.GasUsed, true, blockGasLimit),
		}
	case script.BroadcastCreate2:
		txData := make([]byte, len(bcast.Salt)+len(bcast.Input))
		copy(txData, bcast.Salt[:])
		copy(txData[len(bcast.Salt):], bcast.Input)

		candidate = txmgr.TxCandidate{
			TxData:   txData,
			To:       &script.DeterministicDeployerAddress,
			Value:    value,
			GasLimit: padGasLimit(bcast.Input, bcast.GasUsed, true, blockGasLimit),
		}
	}

	t.mgr.SendAsync(ctx, candidate, ch)
	return ch, id
}

// padGasLimit calculates the gas limit for a transaction based on the intrinsic gas and the gas used by
// the underlying call. Values are multiplied by a pad factor to account for any discrepancies. The output
// is clamped to the block gas limit since Geth will reject transactions that exceed it before letting them
// into the mempool.
func padGasLimit(data []byte, gasUsed uint64, creation bool, blockGasLimit uint64) uint64 {
	intrinsicGas, err := core.IntrinsicGas(data, nil, creation, true, true, false, nil, nil)
	// This method never errors - we should look into it if it does.
	if err != nil {
		panic(err)
	}

	limit := uint64(float64(intrinsicGas+gasUsed) * GasPadFactor)
	if limit > blockGasLimit {
		return blockGasLimit
	}
	return limit
}

// MUTEX VERSION:
// uncomment the following version and comment out the above
// to execute the tests with mutex implementation to avoid race conditions
// package broadcaster

// import (
// 	"context"
// 	"fmt"
// 	"math/big"
// 	"sync"
// 	"time"

// 	"github.com/ethereum-optimism/optimism/op-service/eth"

// 	"github.com/ethereum-optimism/optimism/op-chain-ops/script"
// 	opcrypto "github.com/ethereum-optimism/optimism/op-service/crypto"
// 	"github.com/ethereum-optimism/optimism/op-service/txmgr"
// 	"github.com/ethereum-optimism/optimism/op-service/txmgr/metrics"
// 	"github.com/ethereum/go-ethereum/common"
// 	"github.com/ethereum/go-ethereum/core"
// 	"github.com/ethereum/go-ethereum/ethclient"
// 	"github.com/ethereum/go-ethereum/log"
// 	"github.com/hashicorp/go-multierror"
// 	"github.com/holiman/uint256"
// )

// const (
// 	GasPadFactor = 2.0
// )

// type KeyedBroadcaster struct {
// 	lgr    log.Logger
// 	mgr    txmgr.TxManager
// 	bcasts []script.Broadcast
// 	client *ethclient.Client
// 	mtx    sync.Mutex
// }

// type KeyedBroadcasterOpts struct {
// 	Logger  log.Logger
// 	ChainID *big.Int
// 	Client  *ethclient.Client
// 	Signer  opcrypto.SignerFn
// 	From    common.Address
// }

// func NewKeyedBroadcaster(cfg KeyedBroadcasterOpts) (*KeyedBroadcaster, error) {
// 	mgrCfg := &txmgr.Config{
// 		Backend:                   cfg.Client,
// 		ChainID:                   cfg.ChainID,
// 		TxSendTimeout:             5 * time.Minute,
// 		TxNotInMempoolTimeout:     time.Minute,
// 		NetworkTimeout:            10 * time.Second,
// 		ReceiptQueryInterval:      time.Second,
// 		NumConfirmations:          1,
// 		SafeAbortNonceTooLowCount: 3,
// 		Signer:                    cfg.Signer,
// 		From:                      cfg.From,
// 	}

// 	minTipCap, err := eth.GweiToWei(1.0)
// 	if err != nil {
// 		panic(err)
// 	}
// 	minBaseFee, err := eth.GweiToWei(1.0)
// 	if err != nil {
// 		panic(err)
// 	}

// 	mgrCfg.ResubmissionTimeout.Store(int64(48 * time.Second))
// 	mgrCfg.FeeLimitMultiplier.Store(5)
// 	mgrCfg.FeeLimitThreshold.Store(big.NewInt(100))
// 	mgrCfg.MinTipCap.Store(minTipCap)
// 	mgrCfg.MinBaseFee.Store(minBaseFee)

// 	mgr, err := txmgr.NewSimpleTxManagerFromConfig(
// 		"transactor",
// 		cfg.Logger,
// 		&metrics.NoopTxMetrics{},
// 		mgrCfg,
// 	)

// 	if err != nil {
// 		return nil, fmt.Errorf("failed to create tx manager: %w", err)
// 	}

// 	return &KeyedBroadcaster{
// 		lgr:    cfg.Logger,
// 		mgr:    mgr,
// 		client: cfg.Client,
// 	}, nil
// }

// func (t *KeyedBroadcaster) Hook(bcast script.Broadcast) {
// 	if bcast.Type != script.BroadcastCreate2 && bcast.From != t.mgr.From() {
// 		panic(fmt.Sprintf("invalid from for broadcast:%v, expected:%v", bcast.From, t.mgr.From()))
// 	}
// 	t.mtx.Lock()
// 	t.bcasts = append(t.bcasts, bcast)
// 	t.mtx.Unlock()
// }

// func (t *KeyedBroadcaster) Broadcast(ctx context.Context) ([]BroadcastResult, error) {
// 	// Empty the internal broadcast buffer as soon as this method is called.
// 	t.mtx.Lock()
// 	bcasts := t.bcasts
// 	t.bcasts = nil
// 	t.mtx.Unlock()

// 	if len(bcasts) == 0 {
// 		return nil, nil
// 	}

// 	results := make([]BroadcastResult, len(bcasts))
// 	futures := make([]<-chan txmgr.SendResponse, len(bcasts))
// 	ids := make([]common.Hash, len(bcasts))

// 	latestBlock, err := t.client.BlockByNumber(ctx, nil)
// 	if err != nil {
// 		return nil, fmt.Errorf("failed to get latest block: %w", err)
// 	}

// 	for i, bcast := range bcasts {
// 		futures[i], ids[i] = t.broadcast(ctx, bcast, latestBlock.GasLimit())
// 		t.lgr.Info(
// 			"transaction broadcasted",
// 			"id", ids[i],
// 			"nonce", bcast.Nonce,
// 		)
// 	}

// 	var txErr *multierror.Error
// 	var completed int
// 	for i, fut := range futures {
// 		bcastRes := <-fut
// 		completed++
// 		outRes := BroadcastResult{
// 			Broadcast: bcasts[i],
// 		}

// 		if bcastRes.Err == nil {
// 			outRes.Receipt = bcastRes.Receipt
// 			outRes.TxHash = bcastRes.Receipt.TxHash

// 			if bcastRes.Receipt.Status == 0 {
// 				failErr := fmt.Errorf("transaction failed: %s", outRes.Receipt.TxHash.String())
// 				txErr = multierror.Append(txErr, failErr)
// 				outRes.Err = failErr
// 				t.lgr.Error(
// 					"transaction failed on chain",
// 					"id", ids[i],
// 					"completed", completed,
// 					"total", len(bcasts),
// 					"hash", outRes.Receipt.TxHash.String(),
// 					"nonce", outRes.Broadcast.Nonce,
// 				)
// 			} else {
// 				t.lgr.Info(
// 					"transaction confirmed",
// 					"id", ids[i],
// 					"completed", completed,
// 					"total", len(bcasts),
// 					"hash", outRes.Receipt.TxHash.String(),
// 					"nonce", outRes.Broadcast.Nonce,
// 					"creation", outRes.Receipt.ContractAddress,
// 				)
// 			}
// 		} else {
// 			txErr = multierror.Append(txErr, bcastRes.Err)
// 			outRes.Err = bcastRes.Err
// 			t.lgr.Error(
// 				"transaction failed",
// 				"id", ids[i],
// 				"completed", completed,
// 				"total", len(bcasts),
// 				"err", bcastRes.Err,
// 			)
// 		}

// 		results[i] = outRes
// 	}
// 	return results, txErr.ErrorOrNil()
// }

// func (t *KeyedBroadcaster) broadcast(ctx context.Context, bcast script.Broadcast, blockGasLimit uint64) (<-chan txmgr.SendResponse, common.Hash) {
// 	ch := make(chan txmgr.SendResponse, 1)

// 	id := bcast.ID()
// 	value := ((*uint256.Int)(bcast.Value)).ToBig()
// 	var candidate txmgr.TxCandidate
// 	switch bcast.Type {
// 	case script.BroadcastCall:
// 		to := &bcast.To
// 		candidate = txmgr.TxCandidate{
// 			TxData:   bcast.Input,
// 			To:       to,
// 			Value:    value,
// 			GasLimit: padGasLimit(bcast.Input, bcast.GasUsed, false, blockGasLimit),
// 		}
// 	case script.BroadcastCreate:
// 		candidate = txmgr.TxCandidate{
// 			TxData:   bcast.Input,
// 			To:       nil,
// 			GasLimit: padGasLimit(bcast.Input, bcast.GasUsed, true, blockGasLimit),
// 		}
// 	case script.BroadcastCreate2:
// 		txData := make([]byte, len(bcast.Salt)+len(bcast.Input))
// 		copy(txData, bcast.Salt[:])
// 		copy(txData[len(bcast.Salt):], bcast.Input)

// 		candidate = txmgr.TxCandidate{
// 			TxData:   txData,
// 			To:       &script.DeterministicDeployerAddress,
// 			Value:    value,
// 			GasLimit: padGasLimit(bcast.Input, bcast.GasUsed, true, blockGasLimit),
// 		}
// 	}

// 	t.mgr.SendAsync(ctx, candidate, ch)
// 	return ch, id
// }

// // padGasLimit calculates the gas limit for a transaction based on the intrinsic gas and the gas used by
// // the underlying call. Values are multiplied by a pad factor to account for any discrepancies. The output
// // is clamped to the block gas limit since Geth will reject transactions that exceed it before letting them
// // into the mempool.
// func padGasLimit(data []byte, gasUsed uint64, creation bool, blockGasLimit uint64) uint64 {
// 	intrinsicGas, err := core.IntrinsicGas(data, nil, creation, true, true, false, nil, nil)
// 	if err != nil {
// 		return 0
// 	}

// 	limit := uint64(float64(intrinsicGas+gasUsed) * GasPadFactor)
// 	if limit > blockGasLimit {
// 		return blockGasLimit
// 	}
// 	return limit
// }

```

Now create the broadcast\_test.go file at the same folder:

```go
package broadcaster

import (
	"context"
	"encoding/json"
	"math/big"
	"sync"
	"testing"
	"time"

	"github.com/ethereum-optimism/optimism/op-chain-ops/script"
	"github.com/ethereum-optimism/optimism/op-service/txmgr"
	"github.com/ethereum/go-ethereum/common"
	"github.com/ethereum/go-ethereum/common/hexutil"
	"github.com/ethereum/go-ethereum/core/types"
	"github.com/ethereum/go-ethereum/ethclient"
	"github.com/ethereum/go-ethereum/log"
	"github.com/ethereum/go-ethereum/rpc"
	"github.com/stretchr/testify/assert"
)

// MockEthClient is a mock implementation of ethclient.Client
type MockEthClient struct {
	ethclient.Client
}

func (b *mockEthBackend) BlockByNumber(ctx context.Context, number *big.Int) (map[string]interface{}, error) {
	header := types.Header{
		ParentHash:  common.HexToHash("0x0000000000000000000000000000000000000000000000000000000000000000"),
		UncleHash:   common.HexToHash("0x1dcc4de8dec75d7aab85b567b6ccd41ad312451b948a7413f0a142fd40d49347"),
		Coinbase:    common.HexToAddress("0x0000000000000000000000000000000000000000"),
		Root:        common.HexToHash("0x0000000000000000000000000000000000000000000000000000000000000000"),
		TxHash:      common.HexToHash("0x0000000000000000000000000000000000000000000000000000000000000000"),
		ReceiptHash: common.HexToHash("0x0000000000000000000000000000000000000000000000000000000000000000"),
		Bloom:       types.Bloom{},
		Difficulty:  big.NewInt(1),
		Number:      big.NewInt(1),
		GasLimit:    30000000,
		GasUsed:     0,
		Time:        uint64(time.Now().Unix()),
		Extra:       []byte{},
		MixDigest:   common.HexToHash("0x0000000000000000000000000000000000000000000000000000000000000000"),
		Nonce:       types.BlockNonce{},
		BaseFee:     big.NewInt(1000000000), // 1 Gwei
	}

	headerJSON, err := json.Marshal(header)
	if err != nil {
		return nil, err
	}

	var headerMap map[string]interface{}
	err = json.Unmarshal(headerJSON, &headerMap)
	if err != nil {
		return nil, err
	}

	// Convert big.Int values to hexutil.Big
	headerMap["difficulty"] = (*hexutil.Big)(header.Difficulty)
	headerMap["number"] = (*hexutil.Big)(header.Number)
	headerMap["gasLimit"] = hexutil.Uint64(header.GasLimit)
	headerMap["gasUsed"] = hexutil.Uint64(header.GasUsed)
	headerMap["time"] = hexutil.Uint64(header.Time)
	headerMap["baseFeePerGas"] = (*hexutil.Big)(header.BaseFee)

	return headerMap, nil
}

// MockTxManager is a mock implementation of txmgr.TxManager based on SimpleTxManager
type MockTxManager struct {
	from   common.Address
	closed bool
}

func (m *MockTxManager) From() common.Address {
	return m.from
}

func (m *MockTxManager) Send(ctx context.Context, candidate txmgr.TxCandidate) (*types.Receipt, error) {
	return &types.Receipt{}, nil
}

func (m *MockTxManager) SendAndWait(ctx context.Context, candidate txmgr.TxCandidate) (*types.Receipt, error) {
	return &types.Receipt{}, nil
}

func (m *MockTxManager) Call(ctx context.Context, candidate txmgr.TxCandidate) ([]byte, error) {
	return []byte{}, nil
}

func (m *MockTxManager) BlockNumber(ctx context.Context) (uint64, error) {
	return 1, nil
}

func (m *MockTxManager) Close() {
	m.closed = true
}

func (m *MockTxManager) IsClosed() bool {
	return m.closed
}

func (m *MockTxManager) SuggestGasPriceCaps(ctx context.Context) (tipCap *big.Int, baseFee *big.Int, blobBaseFee *big.Int, err error) {
	return big.NewInt(1), big.NewInt(1), big.NewInt(1), nil
}

func (m *MockTxManager) GetBaseFee(ctx context.Context) (*big.Int, error) {
	return big.NewInt(1), nil
}

func (m *MockTxManager) GetLatestBlockHeader(ctx context.Context) (*types.Header, error) {
	return &types.Header{}, nil
}

func (m *MockTxManager) GetTransactionReceipt(ctx context.Context, hash common.Hash) (*types.Receipt, error) {
	return &types.Receipt{}, nil
}

func (m *MockTxManager) GetTransactionByHash(ctx context.Context, hash common.Hash) (*types.Transaction, error) {
	return &types.Transaction{}, nil
}

func (m *MockTxManager) Nonce(ctx context.Context) (uint64, error) {
	return 0, nil
}

func (m *MockTxManager) PendingNonce(ctx context.Context) (uint64, error) {
	return 0, nil
}

func (m *MockTxManager) SendTransaction(ctx context.Context, tx *types.Transaction) error {
	return nil
}

func (m *MockTxManager) WaitForReceipt(ctx context.Context, hash common.Hash) (*types.Receipt, error) {
	return &types.Receipt{}, nil
}

func (m *MockTxManager) ChainID(ctx context.Context) (*big.Int, error) {
	return big.NewInt(1), nil
}

func (m *MockTxManager) API() rpc.API {
	return rpc.API{
		Namespace: "mock",
		Version:   "1.0",
		Service:   m,
		Public:    true,
	}
}

func (m *MockTxManager) SendAsync(ctx context.Context, candidate txmgr.TxCandidate, ch chan txmgr.SendResponse) {
	go func() {
		ch <- txmgr.SendResponse{
			Receipt: &types.Receipt{},
			Err:     nil,
		}
	}()
}

func TestConcurrentBroadcastRaceCondition(t *testing.T) {
	logger := log.New()

	// Create a mock ethclient
	mockClient := NewMockEthClient()

	broadcaster := &KeyedBroadcaster{
		lgr:    logger,
		mgr:    &MockTxManager{from: common.HexToAddress("0x1234")},
		client: mockClient,
	}

	// Number of concurrent broadcast operations
	numBroadcasts := 100

	// WaitGroup to wait for all goroutines to finish
	var wg sync.WaitGroup
	wg.Add(numBroadcasts)

	// Channel to collect results
	resultChan := make(chan int, numBroadcasts)

	// Start concurrent broadcast operations
	for i := 0; i < numBroadcasts; i++ {
		go func(i int) {
			defer wg.Done()

			// Hook a broadcast
			broadcaster.Hook(script.Broadcast{
				From:  common.HexToAddress("0x1234"),
				To:    common.HexToAddress("0x5678"),
				Input: []byte{byte(i)},
				Nonce: uint64(i),
				Type:  script.BroadcastCall,
			})

			// Immediately try to broadcast
			results, _ := broadcaster.Broadcast(context.Background())
			// if err != nil {
			// 	t.Errorf("Broadcast error: %v", err)
			// 	return
			// }

			resultChan <- len(results)
		}(i)
	}

	// Wait for all goroutines to finish
	wg.Wait()
	close(resultChan)

	// Check if we got all the broadcasts
	assert.NotEqual(t, numBroadcasts, "Expected to lose some broadcasts due to race conditions")
	t.Logf("Total broadcasts: %d ", numBroadcasts)
}

// NewMockEthClient creates a new mock ethclient.Client
func NewMockEthClient() *ethclient.Client {
	mockRPC := rpc.NewServer()
	mockRPC.RegisterName("eth", &mockEthBackend{})
	client := ethclient.NewClient(rpc.DialInProc(mockRPC))
	return client
}

// mockEthBackend is a mock backend for ethclient
type mockEthBackend struct{}

func (b *mockEthBackend) ChainID(ctx context.Context) (hexutil.Big, error) {
	return hexutil.Big(*big.NewInt(1)), nil
}

func (b *mockEthBackend) GetBlockByNumber(ctx context.Context, number rpc.BlockNumber, fullTx bool) (map[string]interface{}, error) {
	return b.BlockByNumber(ctx, big.NewInt(int64(number)))
}

```

Run the test without the non-mutex implementation with the following command from the ./deployer directory:

```shell
go test -v -race ./broadcaster
```

The output should contain this log:

```
==================
==================
WARNING: DATA RACE
```

Now uncomment the mutex-implemented version of the broadcaster and repeat the test. It should pass.
