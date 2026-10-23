# 57179 bc insight during the call to native totalsupply there s missing gas charges

**Submitted on Oct 24th 2025 at 06:17:02 UTC by @emarai for** [**Attackathon | VeChain Hayabusa Upgrade**](https://immunefi.com/audit-competition/vechain-hayabusa-upgrade)

* **Report ID:** #57179
* **Report Type:** Blockchain/DLT
* **Report severity:** Insight
* **Target:** <https://github.com/vechain/thor/compare/v2.3.2...release/hayabusa>

## Description

### Brief/Intro

The `energy.go::TotalSupply` call was modified to account for the changes introduced by VIP-254 (energy growth changes). The native wrapper `energy_native::native_totalSupply` should be updated to account for the additional storage reads, but it was not. As a result, the `totalSupply` call is charged less gas than required.

### Vulnerability Details

Per native call conventions, every storage load must be preceded by a gas charge via `env.UseGas(size * thor.SloadGas)`, where `size` is how many storage words (slots) will be read. In `release/hayabusa`, `native_totalSupply` remained at charging for a single storage read:

{% code title="builtin/energy\_native.go (excerpt)" %}

```go
{"native_totalSupply", func(env *xenv.Environment) []any {
@1>	env.UseGas(thor.SloadGas)
			supply, err := Energy.Native(env.State(), env.BlockContext().Time).TotalSupply()
			if err != nil {
				panic(err)
			}
			return []any{supply}
		}}
```

{% endcode %}

However, `energy.go::TotalSupply` was changed to include additional storage reads (due to fetching stop time and issued), so the gas charge should be increased to account for 3 storage reads:

* The modified `TotalSupply` performs extra storage reads introduced by VIP-254:

{% code title="builtin/energy/energy.go (modified excerpt)" %}

```
```

{% endcode %}

```go
func (e *Energy) TotalSupply() (*big.Int, error) {
	initialSupply, err := e.getInitialSupply()
	if err != nil {
		return nil, err
	}

	// calc grown energy for total token supply
	acc := state.Account{
		Balance:   initialSupply.Token,
		Energy:    initialSupply.Energy,
		BlockTime: initialSupply.BlockTime,
	}

	// this is a virtual account, use account.CalcEnergy directly
@2>    stopTime, err := e.GetEnergyGrowthStopTime()
	if err != nil {
		return nil, err
	}
	grown := acc.CalcEnergy(e.blockTime, stopTime)

@3> 	issued, err := e.getIssued()
	if err != nil {
		return nil, err
	}

	return grown.Add(grown, issued), nil
}
```

* @1> indicates the existing gas charge that now lacks two additional storage read charges.
* @2> and @3> are the additional storage reads added by VIP-254.

The correct fix is to update the native wrapper gas charge to account for the additional storage reads, e.g. env.UseGas(3 \* thor.SloadGas) instead of env.UseGas(thor.SloadGas).

## Impact Details

Because the native wrapper undercharges gas, calls to `native_totalSupply` (and therefore the public `totalSupply()` function on `energy.sol`) are cheaper than they should be. This can cause inconsistencies in gas accounting and potentially be exploited in contexts where precise gas usage is relied upon.

The Solidity wrapper call is:

{% code title="contracts/Energy.sol (excerpt)" %}

```solidity
function totalSupply() public view returns(uint256) {
    return EnergyNative(this).native_totalSupply();
}
```

{% endcode %}

## References

* <https://github.com/vechain/thor/blob/b4c914fe573ed6141daa159fa293e9193a96d74f/builtin/energy/energy.go#L122>
* <https://github.com/vechain/thor/blob/b4c914fe573ed6141daa159fa293e9193a96d74f/builtin/energy\\_native.go#L23>

## Proof of Concept

<details>

<summary>PoC test (put into builtin/energy_poc_test.go and run)</summary>

Put the test below into `builtin/energy_poc_test.go` and run:

go test -timeout 30s -run ^TestPocEnergyTotalSupply$ github.com/vechain/thor/v2/builtin

The test checks gas usage; it will fail because the gas used is lower than expected and show the discrepancy.

{% code title="builtin/energy\_poc\_test.go" %}

```go
package builtin_test

import (
	"testing"

	"github.com/stretchr/testify/assert"

	"github.com/vechain/thor/v2/builtin"
	"github.com/vechain/thor/v2/chain"
	"github.com/vechain/thor/v2/muxdb"
	"github.com/vechain/thor/v2/runtime"
	"github.com/vechain/thor/v2/state"
	"github.com/vechain/thor/v2/thor"
	"github.com/vechain/thor/v2/xenv"
)

func TestPocEnergyTotalSupply(t *testing.T) {
	var (
		caller = thor.BytesToAddress([]byte("alice"))
	)

	fc := &thor.SoloFork
	fc.HAYABUSA = 0
	hayabusaTP := uint32(0)
	thor.SetConfig(thor.Config{HayabusaTP: &hayabusaTP})

	db := muxdb.NewMem()

	gene := buildGenesis(db, func(state *state.State) error {
		state.SetCode(builtin.Energy.Address, builtin.Energy.RuntimeBytecodes())

		return nil
	})

	repo, err := chain.NewRepository(db, gene)
	assert.NoError(t, err)

	bestSummary := repo.BestBlockSummary()
	state := state.NewStater(db).NewState(bestSummary.Root())

	rt := runtime.New(
		repo.NewBestChain(),
		state,
		&xenv.BlockContext{Time: bestSummary.Header.Timestamp()},
		fc,
	)

	test := &ctest{
		rt:     rt,
		abi:    builtin.Energy.ABI,
		to:     builtin.Energy.Address,
		caller: builtin.Energy.Address,
	}

	// @note this will fail, expecting `at least` 200 * 3, but will use less than that (494 on test)
	test.Case("totalSupply").
		Caller(caller).
		ShouldUseGas(200 * 3).Assert(t)
}
```

{% endcode %}

</details>

## Suggested Fix

Update the gas accounting in the native wrapper to match the actual number of storage reads performed by `Energy.TotalSupply()`. For example change:

```
```

to:

```
```

(or the appropriate number of storage reads based on the final implementation).
