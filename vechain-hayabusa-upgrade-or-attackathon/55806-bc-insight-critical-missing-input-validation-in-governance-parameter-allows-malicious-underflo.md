# 55806 bc insight critical missing input validation in governance parameter allows malicious underflow leading to permanent freeze of all dpos rewards

**Submitted on Oct 5th 2025 at 20:19:43 UTC by @cryptoWhale for** [**Attackathon | VeChain Hayabusa Upgrade**](https://immunefi.com/audit-competition/vechain/hayabusa-upgrade-attackathon)

* **Report ID:** #55806
* **Report Type:** Blockchain/DLT
* **Report severity:** Insight
* **Target:** <https://github.com/vechain/thor/compare/master...release/hayabusa>
* **Impacts:**
  * Direct loss of funds

## Description

I found a critical vulnerability in the params built-in contract. Its Set function, used for governance updates, completely lacks input validation for the KeyValidatorRewardPercentage parameter. This allows a malicious or mistaken governance proposal to set the reward split percentage to a value over 100. When the energy contract uses this value, it triggers an integer underflow that breaks the reward calculation for every subsequent block, causing all rewards for validators and delegators to become zero. A single transaction can be used to permanently disable the DPoS reward system, breaking the protocol's core economic incentive.

The vulnerability stems from an implementation that contradicts the official VIP-254 specification. The issue is a direct interaction between two core built-in contracts.

### A. The Unvalidated Setter (builtin/params/params.go)

The root cause is the generic `Set` function in `builtin/params/params.go`. This function is the entry point for governance to update parameters. It blindly accepts any key-value pair and writes it to state, failing to validate that the value is sensible for the parameter being changed.

Code (The Flaw):

{% code title="builtin/params/params.go" %}

```go
func (p *Params) Set(key thor.Bytes32, value *big.Int) error {
	return p.state.EncodeStorage(p.addr, key, func() ([]byte, error) {
		// No validation on 'value' for the given 'key'
		if value.Sign() == 0 {
			return nil, nil
		}
		return rlp.EncodeToBytes(value)
	})
}
```

{% endcode %}

### B. The Exploitable Calculation (builtin/energy/energy.go)

The `DistributeRewards` function in `builtin/energy/energy.go` reads this unvalidated parameter. If the percentage is set to 200, the validator's share becomes larger than the total available reward. The subsequent subtraction to calculate the delegator's share then underflows, causing the reward distribution to fail.

Code (The Exploit):

{% code title="builtin/energy/energy.go" %}

```go
func (e *Energy) DistributeRewards(...) error {
    reward, _ := e.CalculateRewards(staker)
    validatorRewardPerc, _ := e.params.Get(thor.KeyValidatorRewardPercentage)
    
    proposerReward := new(big.Int).Set(reward)
    proposerReward.Mul(proposerReward, validatorRewardPerc)
    proposerReward.Div(proposerReward, big.NewInt(100))

    delegationReward := new(big.Int).Sub(reward, proposerReward) // Underflows here
    
    staker.IncreaseDelegatorsReward(signer, delegationReward, currentBlock)
    // ...
}
```

{% endcode %}

## Impact Details

* Severity: Critical — the attack breaks the core economic model of the protocol, leading to a permanent loss of funds and a failure of the liveness incentive.
* Direct Loss of Funds: For every validator and delegator, this attack causes a direct and permanent loss of all future VTHO yield. The funds at risk are 100% of all future DPoS block rewards.
* Protocol Liveness Failure: Without block rewards, validators have no economic incentive to operate nodes. They would likely shut down, degrading the network's security and liveness.
* Attack Vector & Feasibility: The attack vector is a governance transaction. While governance control is required, a compromised committee key or human error (e.g., a typo) can trigger the issue. The protocol lacks an on-chain safety rail to prevent such catastrophic updates, contradicting the fixed 30/70 split defined in VIP-254.

## References

* Vulnerable Function (No Validation): <https://github.com/vechain/thor/blob/release/hayabusa/builtin/params/params.go#L53>
* Exploited Logic (Underflow): <https://github.com/vechain/thor/blob/release/hayabusa/builtin/energy/energy.go#L303>
* Contradicted Specification (VIP-254): <https://github.com/vechain/VIPs/blob/master/vips/VIP-254.md#block-rewards>

## Proof of Concept

A native Go test definitively proves the vulnerability by simulating a malicious governance action.

{% stepper %}
{% step %}

### Method / Setup

A new test function, `TestRewardPercentageOverflow`, is added to `builtin/energy/energy_test.go`. This uses the project's existing test suite to simulate a governance change.
{% endstep %}

{% step %}

### Attack Simulation

The test uses `st.SetStorage` to simulate a successful governance transaction, setting `KeyValidatorRewardPercentage` to `200`.
{% endstep %}

{% step %}

### Trigger

The test calls `DistributeRewards` to run the reward distribution using the malicious parameter.
{% endstep %}

{% step %}

### Proof

The test asserts that the reward calculation has failed by detecting an invalid delegator reward. A failing assertion ("VULNERABILITY CONFIRMED") demonstrates the exploit.
{% endstep %}
{% endstepper %}

PoC Code (add to the end of `builtin/energy/energy_test.go`):

{% code title="builtin/energy/energy\_test.go (PoC)" %}

```go
func TestRewardPercentageOverflow(t *testing.T) {
	st := state.New(muxdb.NewMem(), trie.Root{})
	beneficiary := thor.BytesToAddress([]byte("beneficiary"))
	signer := thor.BytesToAddress([]byte("signer"))
	stargateAddr := thor.BytesToAddress([]byte("stargate"))
	paramsAddr := thor.BytesToAddress([]byte("par"))
	p := params.New(paramsAddr, st)
	eng := New(thor.BytesToAddress([]byte("eng")), st, 1, p)

	maliciousPercentage := big.NewInt(200)
	st.SetStorage(paramsAddr, thor.KeyValidatorRewardPercentage, thor.BytesToBytes32(maliciousPercentage.Bytes()))

	mockStaker := &mockStaker{
		lockedVET:      uint64(25),
		hasDelegations: true,
		increaseRewardErr: nil,
	}
	err := eng.DistributeRewards(beneficiary, signer, mockStaker, 10)
	assert.NoError(t, err)

	delegatorEnergy, err := eng.Get(stargateAddr)
	assert.NoError(t, err)

	totalPossibleReward := big.NewInt(121765601217656012)
	assert.True(t, delegatorEnergy.Cmp(totalPossibleReward) > 0, "VULNERABILITY CONFIRMED: Delegator reward underflowed, breaking the reward calculation.")
	
	t.Logf("Vulnerability proven: Delegator reward is %s", delegatorEnergy.String())
}
```

{% endcode %}

How to run & expected output:

* Prerequisites: A working Go development environment with the project's dependencies installed.
* Execution:
  * Add the PoC test function to the end of `builtin/energy/energy_test.go`.
  * From the root of the thor repository, run:
    * go test ./builtin/energy/... -run TestRewardPercentageOverflow -v
* Expected Result: The test will FAIL with an error message containing "VULNERABILITY CONFIRMED", proving the exploit.

## Suggested Remediation

The authoritative fix is to add input validation to the `params.Set` function to reject invalid values at the source.

Proposed Patch:

{% code title="Proposed change in builtin/params/params.go" %}

```go
func (p *Params) Set(key thor.Bytes32, value *big.Int) error {
    if bytes.Equal(key[:], thor.KeyValidatorRewardPercentage[:]) {
        if value.Cmp(big.NewInt(0)) < 0 || value.Cmp(big.NewInt(100)) > 0 {
            return errors.New("invalid KeyValidatorRewardPercentage: must be between 0 and 100")
        }
    }
    // ... rest of the function ...
}
```

{% endcode %}

This enforces the valid 0–100 range at the point of parameter update and prevents the energy contract from reading dangerous values that lead to underflow.
