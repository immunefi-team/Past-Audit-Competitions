# 57136 bc low txpool priority cache lets base fee swings reduce proposers tips

* **Submitted on:** Oct 23rd 2025 at 19:27:07 UTC by @v\_c0d35
* **Competition:** [Attackathon | VeChain Hayabusa Upgrade](https://immunefi.com/audit-competition/vechain-hayabusa-upgrade-attackathon)
* **Report ID:** #57136
* **Report Type:** Blockchain/DLT
* **Severity:** Low
* **Target:** <https://github.com/vechain/thor/compare/v2.3.2...release/hayabusa>
* **Impacts:** Modification of transaction fees outside of design parameters

## Summary

TxPool caches each transaction’s `priorityGasPrice` and prepaid cost the first time `Executable(...)` returns true (txpool/tx\_object.go:128-144) and never refreshes them. When base fee later increases, the cached values stay high even though `EffectivePriorityFeePerGas(...)` instantly drops. As a result, the txpool ordering keeps the stale entry at the front of the queue and the packer hands it to the packer first, delivering much smaller actual tips to proposers than current demand requires. This can be triggered by timing a base-fee jump after admission and degrades mempool fairness and proposer rewards.

## Vulnerability Details

* TxObject.Executable computes and caches `priorityGasPrice = EffectivePriorityFeePerGas(baseFee, …)` the first time the tx becomes executable and writes it under `if !o.executable` (txpool/tx\_object.go:128-144).
* Subsequent runs—triggered during housekeeping/wash and before every pack—skip that block, so `priorityGasPrice` remains fixed even though `EffectivePriorityFeePerGas(...)` is computed from the latest base fee.
* TxPool.wash sorts executable objects using the cached `priorityGasPrice` (txpool/tx\_pool.go:499-545), promoting stale values.
* The packer loop consumes that ordering directly (cmd/thor/node/packer\_loop.go:119-140). At runtime base-fee coverage is re-validated so the tx remains valid, but it now delivers an almost-zero tip.
* Because `o.cost` is cached the same way, `txObjectMap.UpdatePendingCost` can under-estimate the payer’s outstanding spend after a base-fee spike, allowing a payer to hold more txs than their real energy/fee balance supports.

## Impact Details

* Priority-fee mismatch: An attacker (or an unfortunate legitimate user) can be kept at the front of the queue with an effectively near-zero tip after a base-fee spike, starving proposers of expected rewards and delaying honest users who increase tips to match the new base fee.
* Energy accounting mismatch: Pending cost remains pegged to the old base fee, requiring consensus to re-evaluate and drop transactions repeatedly, increasing churn and reducing mempool fairness.

## References

* txpool/tx\_object.go — initial caching of priority/cost and missing refresh
* txpool/tx\_pool.go — wash ordering driven by cached `priorityGasPrice`
* cmd/thor/node/packer\_loop.go — packer consumes stale ordering without recomputing tips

## Proof of Concept (PoC)

The PoC is a Go test that reproduces the stale priority cache once Hayabusa/dPoS is active. It does the following:

{% stepper %}
{% step %}

### Phase 1 — Setup & attacker tx admission

* Boot a Solo chain fork with Hayabusa/dPoS active and two validators.
* Create and sign an attacker dynamic-fee transaction and resolve it to a `txpool.TxObject`.
* Call `Executable(...)` with a low base fee to populate the cached priority and cached cost.
* Verify the cached priority matches the expected priority at the low base fee.
  {% endstep %}

{% step %}

### Phase 2 — Base-fee spike & cached value staleness

* Simulate a base-fee spike.
* Call `Executable(...)` again (or check cached fields) for the attacker object.
* Observe that the cached `priorityGasPrice` still reflects the old (low-base-fee) effective priority while `EffectivePriorityFeePerGas(...)` computed with the new base fee is near-zero.
  {% endstep %}

{% step %}

### Phase 3 — Honest higher-tip tx arrives

* Create and resolve a second (honest) dynamic-fee transaction with a higher tip suitable for the new base fee.
* Call `Executable(...)` for the honest tx and verify its cached priority equals its actual effective priority under the new base fee.
* Observe the queue ordering: cached values keep the attacker ahead even though the honest tx pays a larger real tip.
  {% endstep %}

{% step %}

### Phase 4 — Outcome

* The stale cached ordering causes the attacker tx to be preferred in packing even though it provides far less real tip revenue after the base-fee jump.
* The test prints logs demonstrating the divergence between cached ordering and real tip amounts.
  {% endstep %}
  {% endstepper %}

### PoC test code

Create a folder `poc` and save the following file as `priority_fee_dpos_test.go`:

```go
package poc

import (
	"fmt"
	"math/big"
	"reflect"
	"testing"
	"unsafe"

	"github.com/stretchr/testify/require"

	"github.com/vechain/thor/v2/builtin"
	stakerpkg "github.com/vechain/thor/v2/builtin/staker"
	"github.com/vechain/thor/v2/genesis"
	"github.com/vechain/thor/v2/state"
	"github.com/vechain/thor/v2/test/testchain"
	"github.com/vechain/thor/v2/thor"
	"github.com/vechain/thor/v2/tx"
	"github.com/vechain/thor/v2/txpool"
)

// TestPriorityFeeStalenessDPoS reproduces the stale priority cache once Hayabusa/dPoS is active.
func TestPriorityFeeStalenessDPoS(t *testing.T) {
	chain, err := testchain.NewWithFork(&thor.SoloFork, 180)
	require.NoError(t, err)

	repo := chain.Repo()
	stater := chain.Stater()
	best := repo.BestBlockSummary()
	require.NotNil(t, best)

	fork := *chain.GetForkConfig()
	fork.VIP191 = 0
	fork.GALACTICA = 0
	fork.HAYABUSA = 10

	stakerState := stater.NewState(best.Root())
	params := builtin.Params.Native(stakerState)
	require.NoError(t, params.Set(thor.KeyMaxBlockProposers, big.NewInt(3)))

	ts := newTestStaker(stakerState)
	validator1 := thor.BytesToAddress([]byte("validator1"))
	endorser1 := thor.BytesToAddress([]byte("endorser1"))
	validator2 := thor.BytesToAddress([]byte("validator2"))
	endorser2 := thor.BytesToAddress([]byte("endorser2"))

	stakeAmount := stakerpkg.MinStakeVET
	require.NoError(t, ts.AddValidation(validator1, endorser1, thor.MediumStakingPeriod(), stakeAmount))
	require.NoError(t, ts.AddValidation(validator2, endorser2, thor.MediumStakingPeriod(), stakeAmount))

	originalTP := thor.HayabusaTP()
	defer thor.SetConfig(thor.Config{HayabusaTP: &originalTP})
	hayabusaTP := uint32(10)
	thor.SetConfig(thor.Config{HayabusaTP: &hayabusaTP})

	status, err := ts.impl.SyncPOS(&fork, 180)
	require.NoError(t, err)
	require.True(t, status.Active, "dPoS should be active after Hayabusa")
	t.Logf("[setup] Hayabusa/DPos active: %v", status.Active)

	signer := genesis.DevAccounts()[0]
	target := thor.BytesToAddress([]byte("sink"))

	attackerTx := tx.NewBuilder(tx.TypeDynamicFee).
		ChainTag(repo.ChainTag()).
		Clause(tx.NewClause(&target)).
		Gas(21000).
		BlockRef(tx.NewBlockRef(best.Header.Number())).
		Expiration(1000).
		MaxFeePerGas(big.NewInt(1200)).
		MaxPriorityFeePerGas(big.NewInt(100)).
		Nonce(0).
		Build()
	attackerTx = tx.MustSign(attackerTx, signer.PrivateKey)

	attackerObj, err := txpool.ResolveTx(attackerTx, true)
	require.NoError(t, err)

	baseFeeLow := big.NewInt(100)
	baseFeeHigh := big.NewInt(1199)

	chainView := repo.NewChain(best.Header.ID())
	stateLow := stater.NewState(best.Root())
	executable, err := attackerObj.Executable(chainView, stateLow, best.Header, &fork, baseFeeLow)
	require.NoError(t, err)
	require.True(t, executable)

	cachedLow := cachedPriority(attackerObj)
	require.NotNil(t, cachedLow)
	require.Equal(t, int64(100), cachedLow.Int64())
	t.Logf("[phase-1] attacker cached priority at baseFee=100: %s wei", cachedLow.String())

	legacyBaseGasPrice, err := builtin.Params.Native(stateLow).Get(thor.KeyLegacyTxBaseGasPrice)
	require.NoError(t, err)

	expectedLow := attackerTx.EffectivePriorityFeePerGas(baseFeeLow, legacyBaseGasPrice, big.NewInt(0))
	require.Equal(t, int64(100), expectedLow.Int64())
	t.Logf("[phase-1] attacker effective priority (same baseFee): %s wei", expectedLow.String())

	setExecutable(attackerObj, true)

	stateHigh := stater.NewState(best.Root())
	executable, err = attackerObj.Executable(chainView, stateHigh, best.Header, &fork, baseFeeHigh)
	require.NoError(t, err)
	require.True(t, executable)

	cachedHigh := cachedPriority(attackerObj)
	require.NotNil(t, cachedHigh)
	require.Equal(t, int64(100), cachedHigh.Int64())
	t.Logf("[phase-2] attacker cached priority after baseFee=1199: %s wei (stale)", cachedHigh.String())

	expectedHigh := attackerTx.EffectivePriorityFeePerGas(baseFeeHigh, legacyBaseGasPrice, big.NewInt(0))
	require.Equal(t, int64(1), expectedHigh.Int64())
	t.Logf("[phase-2] attacker effective priority after baseFee jump: %s wei (actual tip)", expectedHigh.String())

	honestTx := tx.NewBuilder(tx.TypeDynamicFee).
		ChainTag(repo.ChainTag()).
		Clause(tx.NewClause(&target)).
		Gas(21000).
		BlockRef(tx.NewBlockRef(best.Header.Number())).
		Expiration(1000).
		MaxFeePerGas(big.NewInt(2000)).
		MaxPriorityFeePerGas(big.NewInt(20)).
		Nonce(1).
		Build()
	honestTx = tx.MustSign(honestTx, signer.PrivateKey)

	honestObj, err := txpool.ResolveTx(honestTx, true)
	require.NoError(t, err)

	stateHonest := stater.NewState(best.Root())
	executable, err = honestObj.Executable(chainView, stateHonest, best.Header, &fork, baseFeeHigh)
	require.NoError(t, err)
	require.True(t, executable)

	setExecutable(honestObj, true)

	cachedHonest := cachedPriority(honestObj)
	require.NotNil(t, cachedHonest)
	require.Equal(t, int64(20), cachedHonest.Int64())
	t.Logf("[phase-3] honest cached priority at baseFee=1199: %s wei", cachedHonest.String())

	expectedHonest := honestTx.EffectivePriorityFeePerGas(baseFeeHigh, legacyBaseGasPrice, big.NewInt(0))
	require.Equal(t, int64(20), expectedHonest.Int64())
	t.Logf("[phase-3] honest effective priority at baseFee=1199: %s wei", expectedHonest.String())

	require.True(t, cachedHigh.Cmp(cachedHonest) > 0, "stale cache keeps attacker ahead")
	require.True(t, expectedHigh.Cmp(expectedHonest) < 0, "real tips show attacker pays less")
	t.Logf("[result] cached ordering => attacker(%s) > honest(%s)", cachedHigh.String(), cachedHonest.String())
	t.Logf("[result] real tips     => attacker(%s) < honest(%s)", expectedHigh.String(), expectedHonest.String())
}

type testStaker struct {
	addr  thor.Address
	state *state.State
	impl  *stakerpkg.Staker
}

func newTestStaker(state *state.State) *testStaker {
	ts := &testStaker{
		addr:  builtin.Staker.Address,
		state: state,
		impl:  builtin.Staker.Native(state),
	}
	state.SetStorage(ts.addr, thor.Bytes32{}, thor.Bytes32{})
	return ts
}

func (ts *testStaker) addBalance(amount uint64) error {
	wei := stakerpkg.ToWei(amount)
	current, err := ts.state.GetStorage(ts.addr, thor.Bytes32{})
	if err != nil {
		return err
	}
	effective := new(big.Int).SetBytes(current.Bytes())
	effective.Add(effective, wei)
	ts.state.SetStorage(ts.addr, thor.Bytes32{}, thor.BytesToBytes32(effective.Bytes()))

	balance, err := ts.state.GetBalance(ts.addr)
	if err != nil {
		return err
	}
	return ts.state.SetBalance(ts.addr, new(big.Int).Add(balance, wei))
}

func (ts *testStaker) subBalance(amount uint64) error {
	wei := stakerpkg.ToWei(amount)
	current, err := ts.state.GetStorage(ts.addr, thor.Bytes32{})
	if err != nil {
		return err
	}
	effective := new(big.Int).SetBytes(current.Bytes())
	if effective.Cmp(wei) < 0 {
		return fmt.Errorf("insufficient effective balance")
	}
	effective.Sub(effective, wei)
	ts.state.SetStorage(ts.addr, thor.Bytes32{}, thor.BytesToBytes32(effective.Bytes()))

	balance, err := ts.state.GetBalance(ts.addr)
	if err != nil {
		return err
	}
	if balance.Cmp(wei) < 0 {
		return fmt.Errorf("insufficient balance")
	}
	return ts.state.SetBalance(ts.addr, new(big.Int).Sub(balance, wei))
}

func (ts *testStaker) AddValidation(validator, endorser thor.Address, period uint32, stake uint64) error {
	if period != thor.LowStakingPeriod() && period != thor.MediumStakingPeriod() && period != thor.HighStakingPeriod() {
		period = thor.MediumStakingPeriod()
	}
	if err := ts.addBalance(stake); err != nil {
		return err
	}
	if err := ts.impl.AddValidation(validator, endorser, period, stake); err != nil {
		_ = ts.subBalance(stake)
		return err
	}
	return nil
}

func cachedPriority(obj *txpool.TxObject) *big.Int {
	val := reflect.ValueOf(obj).Elem().FieldByName("priorityGasPrice")
	if val.IsNil() {
		return nil
	}
	ptr := (*big.Int)(unsafe.Pointer(val.UnsafePointer()))
	return new(big.Int).Set(ptr)
}

func setExecutable(obj *txpool.TxObject, value bool) {
	val := reflect.ValueOf(obj).Elem().FieldByName("executable")
	reflect.NewAt(val.Type(), unsafe.Pointer(val.UnsafeAddr())).Elem().SetBool(value)
}
```

Run the test:

```bash
GOCACHE=$(pwd)/.cache-go go test -v ./poc -run TestPriorityFeeStalenessDPoS -count=1
```

## Expected Output

<details>

<summary>Verbose test output (click to expand)</summary>

```
=== RUN   TestPriorityFeeStalenessDPoS
    priority_fee_dpos_test.go:49: [setup] Hayabusa/DPos active: true
    priority_fee_dpos_test.go:68: [phase-1] attacker cached priority at baseFee=100: 100 wei
    priority_fee_dpos_test.go:74: [phase-1] attacker effective priority (same baseFee): 100 wei
    priority_fee_dpos_test.go:91: [phase-2] attacker cached priority after baseFee=1199: 100 wei (stale)
    priority_fee_dpos_test.go:95: [phase-2] attacker effective priority after baseFee jump: 1 wei (actual tip)
    priority_fee_dpos_test.go:114: [phase-3] honest cached priority at baseFee=1199: 20 wei
    priority_fee_dpos_test.go:118: [phase-3] honest effective priority at baseFee=1199: 20 wei
    priority_fee_dpos_test.go:125: [result] cached ordering => attacker(100) > honest(20)
    priority_fee_dpos_test.go:126: [result] real tips     => attacker(1) < honest(20)
--- PASS: TestPriorityFeeStalenessDPoS (0.31s)
PASS
```

</details>

## Notes

* The PoC inspects and toggles internal fields via reflection to demonstrate the caching behavior; it is intended for testing in a development environment.
* The core issue is deterministic: caching `priorityGasPrice` and `cost` once and never refreshing them allows stale values to persist when the base fee changes, leading to incorrect ordering and accounting.
