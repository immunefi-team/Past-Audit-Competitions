# 56454 bc insight gas undercharging threatens hayabusa network upgrade

**Submitted on Oct 16th 2025 at 08:25:34 UTC by @Angry\_Mustache\_Man for** [**Attackathon | VeChain Hayabusa Upgrade**](https://immunefi.com/audit-competition/vechain-hayabusa-upgrade-attackathon)

* **Report ID:** #56454
* **Report Type:** Blockchain/DLT
* **Report severity:** Insight
* **Target:** <https://github.com/vechain/thor/compare/master...release/hayabusa>
* **Impacts:**
  * Network not being able to confirm new transactions (total network shutdown)

## Description

## Vulnerability Details

During the HAYABUSA upgrade, the critical transition of `PoA→PoS` invokes the `native_isEndorsed` function. The `native_isEndorsed` function (in-scope for the attackathon) undercharges gas. See implementation:

<https://github.com/vechain/thor/blob/b4c914fe573ed6141daa159fa293e9193a96d74f/builtin/authority\\_native.go#L105-L137>

```go
{"native_isEndorsed", func(env *xenv.Environment) []any {
			var nodeMaster common.Address
			env.ParseArgs(&nodeMaster)

			env.UseGas(thor.SloadGas * 2)
			listed, endorsor, _, _, err := Authority.Native(env.State()).Get(thor.Address(nodeMaster))
			if err != nil {
				panic(err)
			}
			if !listed {
				return []any{false}
			}

			env.UseGas(thor.SloadGas)
			endorsement, err := Params.Native(env.State()).Get(thor.KeyProposerEndorsement)
			if err != nil {
				panic(err)
			}

			// Use staker Transition Period logic
			// to ensure that transitioning validators are marked as endorsed
			env.UseGas(thor.GetBalanceGas)
			isEndorsed, err := Staker.Native(env.State()).TransitionPeriodBalanceCheck(
				env.ForkConfig(),
				env.BlockContext().Number,
				endorsement,
			)(thor.Address(nodeMaster), endorsor)
			if err != nil {
				panic(err)
			}
			return []any{isEndorsed}
		}},
	}
```

Current gas charges in `native_isEndorsed`:

```go
env.UseGas(thor.SloadGas * 2)  // Authority.Get() - 2 SLOAD operations = 400 gas
env.UseGas(thor.SloadGas)      // Params.Get() - 1 SLOAD operation = 200 gas  
env.UseGas(thor.GetBalanceGas) // TransitionPeriodBalanceCheck balance check = 400 gas
```

Total charged GAS: 1000 gas.

However, `TransitionPeriodBalanceCheck` (called next) performs additional operations that are not charged here:

<https://github.com/vechain/thor/blob/b4c914fe573ed6141daa159fa293e9193a96d74f/builtin/staker/transition.go#L72-L98>

```go
func (s *Staker) TransitionPeriodBalanceCheck(fc *thor.ForkConfig, currentBlock uint32, endorsement *big.Int) authority.BalanceChecker {
	return func(validator, endorser thor.Address) (bool, error) {
		balance, err := s.state.GetBalance(endorser)
		if err != nil {
			return false, err
		}
		if balance.Cmp(endorsement) >= 0 {
			return true, nil
		}
		if currentBlock < fc.HAYABUSA { // before HAYABUSA fork, we only check the account balance
			return false, nil
		}
		validation, err := s.validationService.GetValidation(validator)
		if err != nil {
			return false, err
		}
		if validation == nil {
			return false, nil
		}
		if validation.Endorser != endorser {
			return false, nil // endorser mismatch
		}
		queuedVET := big.NewInt(0).SetUint64(validation.QueuedVET)
		queuedVET.Mul(queuedVET, bigE18) // convert to wei

		return queuedVET.Cmp(endorsement) >= 0, nil
	}
}
```

The code comment notes: "// before HAYABUSA fork, we only check the account balance". After HAYABUSA, `GetValidation` is invoked and performs extra storage reads (SLOADs) that are not accounted for in `native_isEndorsed`.

So the actual gas to be charged after HAYABUSA should be >= 1200 GAS: thor.SloadGas \* 2 + thor.SloadGas + thor.GetBalanceGas + (Gas for the GetValidation call)

The Missing Gas Charge is that `s.validationService.GetValidation(validator)` performs:

{% stepper %}
{% step %}

### Storage lookup / SLOADs

GetValidation does a storage lookup that involves one or more SLOAD operations to retrieve validation data.
{% endstep %}

{% step %}

### Additional gas (>= 1 SLOAD)

Those SLOADs cost >= 200 gas (>= thor.SloadGas) which is not charged in `native_isEndorsed`.
{% endstep %}

{% step %}

### Result: Undercharge (\~>=200 gas)

This >=200 gas is never charged in `native_isEndorsed`, resulting in >=16.7% gas undercharging (200/1200+).
{% endstep %}
{% endstepper %}

Every block validation during the HAYABUSA transition calls `native_isEndorsed`. The missing charge per call creates economic incentives for exploitation: each call saves >=200 gas, causing economic depletion of the protocol and potentially leading to network shutdown.

## Impact Details

{% hint style="warning" %}
Every validator check triggers the missing gas charge. Undercharging (\~>=16.7%) per call can be exploited repeatedly, causing economic depletion and risking a full network shutdown (new transactions cannot be confirmed).
{% endhint %}

## Proof of Concept

Create a new test file named `thor/builtin/authority_native_test.go` with the following tests to (a) statically verify that `native_isEndorsed` does not charge an SLOAD before calling `TransitionPeriodBalanceCheck`, and (b) measure gas consumed by `GetValidation`:

```go
package builtin_test

import (
	"math/big"
	"os"
	"strings"
	"testing"

	"github.com/stretchr/testify/require"
	"github.com/vechain/thor/v2/builtin"
	"github.com/vechain/thor/v2/builtin/gascharger"
	"github.com/vechain/thor/v2/builtin/staker"
	"github.com/vechain/thor/v2/thor"
	"github.com/vechain/thor/v2/vm"
	"github.com/vechain/thor/v2/xenv"
)

// This test performs a simple static check on authority_native.go to prove that
// native_isEndorsed calls TransitionPeriodBalanceCheck but does not charge an
// SLOAD (env.UseGas(thor.SloadGas)) immediately before that call.
func Test_POC(t *testing.T) {
	const path = "authority_native.go"

	b, err := os.ReadFile(path)
	require.NoError(t, err, "failed to read authority_native.go")

	src := string(b)

	start := strings.Index(src, "native_isEndorsed")
	require.NotEqual(t, -1, start, "native_isEndorsed not found in file")

	getBalanceIdx := strings.Index(src[start:], "env.UseGas(thor.GetBalanceGas)")
	require.NotEqual(t, -1, getBalanceIdx, "env.UseGas(thor.GetBalanceGas) not found in native_isEndorsed")

	transIdx := strings.Index(src[start:], "TransitionPeriodBalanceCheck")
	require.NotEqual(t, -1, transIdx, "TransitionPeriodBalanceCheck not found in native_isEndorsed")

	segment := src[start+getBalanceIdx : start+transIdx]

	// Check whether an SLOAD gas charge exists between GetBalanceGas and TransitionPeriodBalanceCheck
	if strings.Contains(segment, "UseGas(thor.SloadGas") {
		t.Fatalf("SLOAD gas charge found before TransitionPeriodBalanceCheck; expected no SLOAD charge (issue NOT present)")
	}

	t.Log("SLOAD gas charge is missing before TransitionPeriodBalanceCheck")
}



// TestGetValidationGasConsumptionSimple demonstrates that GetValidation consumes gas
// by creating a staker and measuring gas via the gascharger.
func Test_GasConsumedAtGetValidation(t *testing.T) {
	// Create a simple test setup
	setup := createTestSetup(t)

	// Create test addresses
	master1 := thor.BytesToAddress([]byte("master1"))
	endorsor1 := thor.BytesToAddress([]byte("endorsor1"))

	// Build a minimal xenv + contract so the gascharger can attach to env.UseGas()
	contract := &vm.Contract{Gas: 1_000_000}
	env := xenv.New(nil, nil, setup.state, nil, nil, nil, nil, contract, 0)
	charger := gascharger.New(env)

	// Create staker with a proper validation service wired to the charger
	stakerAddr := builtin.Staker.Address
	params := builtin.Params.Native(setup.state)
	stakerImpl := staker.New(stakerAddr, setup.state, params, charger)

	// Add validation to staker
	stakeAmountVET := uint64(25_000_000) // 25M VET
	stakeAmountWei := new(big.Int).SetUint64(stakeAmountVET)
	stakeAmountWei.Mul(stakeAmountWei, big.NewInt(1e18))

	// Fund endorsor
	currentBalance, err := setup.state.GetBalance(endorsor1)
	require.NoError(t, err)
	newBalance := new(big.Int).Add(currentBalance, stakeAmountWei)
	err = setup.state.SetBalance(endorsor1, newBalance)
	require.NoError(t, err)

	// Update staker contract state (effectiveVET + contract balance) to allow AddValidation
	effectiveVETBytes, err := setup.state.GetStorage(stakerAddr, thor.Bytes32{})
	require.NoError(t, err)
	effectiveVET := new(big.Int).SetBytes(effectiveVETBytes.Bytes())
	effectiveVET.Add(effectiveVET, stakeAmountWei)
	setup.state.SetStorage(stakerAddr, thor.Bytes32{}, thor.BytesToBytes32(effectiveVET.Bytes()))

	stakerBalance, err := setup.state.GetBalance(stakerAddr)
	require.NoError(t, err)
	newStakerBalance := new(big.Int).Add(stakerBalance, stakeAmountWei)
	err = setup.state.SetBalance(stakerAddr, newStakerBalance)
	require.NoError(t, err)

	// Add validation
	addValidationErr := stakerImpl.AddValidation(master1, endorsor1, thor.LowStakingPeriod(), stakeAmountVET)
	require.NoError(t, addValidationErr, "Failed to add validation to staker")

	// Measure gas around GetValidation
	before := charger.TotalGas()
	validation, getErr := stakerImpl.GetValidation(master1)
	after := charger.TotalGas()

	require.NoError(t, getErr, "GetValidation failed")
	require.NotNil(t, validation, "GetValidation returned nil")

	consumed := after - before

	t.Logf("GetValidation gas measurement:")
	t.Logf(" - total gas before: %d", before)
	t.Logf(" - total gas after : %d", after)
	t.Logf(" - gas consumed    : %d", consumed)

	// Expect 2 SLOAD operations (400 gas) - Validation struct spans 2 storage words
	require.Equal(t, uint64(2*thor.SloadGas), consumed, "GetValidation should consume 2 SLOAD operations (400 gas) for Validation struct")
}
```

Run the tests:

* go test -v ./builtin -run Test\_POC
* go test -v ./builtin -run Test\_GasConsumedAtGetValidation

<details>

<summary>Expected output for Test_POC</summary>

```
go test -v ./builtin -run Test_POC
=== RUN   Test_POC
    authority_native_test.go:45: SLOAD gas charge is missing before TransitionPeriodBalanceCheck
--- PASS: Test_POC (0.00s)
PASS
ok      github.com/vechain/thor/v2/builtin      0.017s
```

</details>

<details>

<summary>Expected output for Test_GasConsumedAtGetValidation</summary>

```
go test -v ./builtin -run Test_GasConsumedAtGetValidation
=== RUN   Test_GasConsumedAtGetValidation
    authority_native_test.go:109: GetValidation gas measurement:
    authority_native_test.go:110:  - total gas before: 121600
    authority_native_test.go:111:  - total gas after : 122000
    authority_native_test.go:112:  - gas consumed    : 400
--- PASS: Test_GasConsumedAtGetValidation (0.01s)
PASS
ok      github.com/vechain/thor/v2/builtin      0.027s
```

</details>
